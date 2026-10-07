# Heartbeat, Datenbank, Observability, Tests

← [Anhang-Übersicht](README.md) · [Bericht](../README.md)

## So funktioniert der Bereich

Der Heartbeat-Daemon (src/heartbeat/daemon.ts) startet beim Prozessstart einen rekursiven setTimeout-Loop (Standard 60 s), seedet pro Eintrag aus heartbeat.yml eine Zeile in heartbeat_schedule und übergibt an den DurableScheduler. Jeder Tick baut genau einen TickContext (Credits per Conway-API, USDC per RPC, daraus das Survival-Tier), ermittelt fällige Tasks aus Cron und lastRunAt und führt sie sequenziell mit 30-s-Timeout und DB-Lease aus. Danach schreibt er heartbeat_history und heartbeat_schedule. Die Tasks in tasks.ts sind überwiegend KV- und DB-Prüfungen und melden shouldWake. Daraufhin legt index.ts einen wake_events-Eintrag an, den die Hauptschleife im Schlaf alle 30 s konsumiert (der Grund wird nur geloggt). Die Persistenz ist better-sqlite3 im WAL-Modus mit integrity_check beim Öffnen und linearen Migrationen V2 bis V11 in je einer Transaktion, dazu rund 150 lose Statement-Helfer ohne Retention-Job. Observability besteht aus einem JSON-Logger, einem In-Memory-MetricsCollector und einer AlertEngine, die nur über den standardmäßig nicht eingeplanten Task report_metrics gespeist wird. Die Tests (ca. 28k Zeilen) laufen meist gegen echte SQLite-Temp-DBs mit handgeschriebenen Mock-Clients, nur 10 Dateien nutzen vi.mock, und CI läuft nur unter Linux.

Die 1 schweren Befunde dieses Bereichs stehen in [kritisch-und-hoch.md](kritisch-und-hoch.md): infra-047.

## Befunde: mittel (29)

### infra-001: API-Ausfall wird als Guthaben 0 gewertet und löst Distress, Auto-Topup und Dead-Timer aus

**Art:** Logik · **Fundstelle:** `src/heartbeat/tick-context.ts:52`

- **Problem:** buildTickContext fängt den Fehler von conway.getCreditsBalance() ab und belässt creditBalance=0. getSurvivalTier(0) ergibt 'critical' (credits.ts:42). Alle Tasks sehen dadurch ein Notfall-Tier, obwohl das Guthaben nur unbekannt ist. agent/loop.ts:432-436 und :962-983 behandelt denselben Fall korrekt (Cache aus KV last_known_balance bzw. Sentinel -1, kein Topup); der Heartbeat nutzt keinen dieser Mechanismen. Folgen: (a) heartbeat_ping (tasks.ts:72) schreibt last_distress und weckt den Agenten alle 15 min. (b) check_usdc_balance (tasks.ts:160-177) ruft bootstrapTopup mit creditsCents=0 auf. Das ist <500, also wird ein x402-Kauf über 5 USD ausgelöst. (c) check_credits (tasks.ts:113-132) setzt zero_credits_since und stellt beim nächsten Fehlerlauf >=1 h später den Agentenstatus auf 'dead'. Der Test 'handles API failure gracefully' (heartbeat-scheduler.test.ts:376-391) zementiert dieses Verhalten.
- **Szenario:** GET /v1/credits/balance liefert 429/5xx oder der Circuit-Breaker der Conway-Client-Instanz ist offen (5 Fehler, http-client.ts). Der x402-Pay-Endpoint nutzt eine eigene Client-Instanz und antwortet normal. Wallet: 40 USDC, echte Credits 50 USD. Tick: creditBalance=0, Tier critical, usdc=40, also bootstrapTopup und -5 USD. Nach 5 min Cooldown wieder, bis USDC<5 ist. Gleichzeitig kommen Distress-Wakes, die jeweils Inferenz kosten.
- **Fix:** TickContext.creditBalance als number|null plus creditsKnown. Bei Fehler auf KV last_known_balance zurückfallen oder null setzen. Tasks überspringen bei unbekanntem Guthaben Topup, Distress und Dead-Timer. Vor jeder Zahlung ein frischer, erfolgreicher Read mit Re-Check. Den Test auf 'unknown' statt 'critical' ändern.

### infra-002: Autonomer Heartbeat-Topup signiert x402-Zahlung ohne Betragslimit, ohne Policy, ohne Spend-/Transaktionslog; zwei unkoordinierte Topup-Pfade

**Art:** Sicherheit · **Fundstelle:** `src/conway/topup.ts:50`

- **Problem:** topupCredits ruft x402Fetch(url, account, 'GET') ohne maxPaymentCents auf. In x402Fetch (x402.ts:354-369) wird das Limit nur geprüft, wenn es gesetzt ist. signPayment (x402.ts:451-521) signiert TransferWithAuthorization mit to=requirement.payToAddress, value=maxAmountRequired und verifyingContract=requirement.usdcAddress, alles aus der Server-Antwort ohne Allowlist. check_usdc_balance (tasks.ts:171-177) läuft alle 5 min ohne PolicyEngine, ohne SpendTracker.recordSpend und ohne insertTransaction. Die Ausgabe taucht in keinem Limit und keinem Finanzbericht auf. Zusätzlich gibt es einen zweiten Topup-Pfad in loop.ts:444-459 mit eigenem Cooldown-Key (last_inline_topup_attempt, 60 s) statt last_auto_topup_attempt (5 min), also keine gegenseitige Sperre.
- **Szenario:** (1) Ein kompromittierter oder gekaperter Conway-Endpoint (oder falsch gesetzte conwayApiUrl) antwortet auf /pay/5/<addr> mit maxAmountRequired=Gesamtguthaben und eigener payTo-Adresse. Der Heartbeat signiert das und leert die USDC-Balance. (2) Credits<500 und USDC>=10: Heartbeat-Tick und Agent-Loop lösen innerhalb derselben Minute je einen 5-USD-Kauf aus, weil die Cooldown-Keys verschieden sind und das Guthaben noch nicht aktualisiert ist.
- **Fix:** topupCredits: maxPaymentCents=tier*100 und payTo/usdcAddress gegen eine Pinning-Konfiguration prüfen. Alle Topups über eine gemeinsame Funktion mit einem einzigen Cooldown-Key und Mutex führen, die SpendTracker.recordSpend und insertTransaction aufruft. Vor der Zahlung den Betrag der 402-Antwort gegen den angefragten Tier abgleichen.

### infra-003: Tool modify_heartbeat ändert nur die Legacy-Tabelle und hat keinerlei Wirkung auf den Scheduler, meldet aber Erfolg

**Art:** Funktion · **Fundstelle:** `src/agent/tools.ts:703`

- **Problem:** modify_heartbeat ruft ctx.db.upsertHeartbeatEntry auf (schreibt heartbeat_entries). Der DurableScheduler liest ausschließlich heartbeat_schedule. Diese Tabelle wird nur beim Daemon-Start aus der YAML (nicht aus heartbeat_entries) geseedet. syncHeartbeatToDb (index.ts:319) überschreibt heartbeat_entries bei jedem Start erneut aus der YAML. 'remove' löscht nichts, sondern schreibt schedule='' und task='' mit enabled=false. Die YAML ist über authority.ts als geschützt markiert, der Agent kann sie nicht selbst ändern. system_synopsis (tools.ts:779) zählt 'Active heartbeats' ebenfalls aus der Legacy-Tabelle und zeigt daher nie den Scheduler-Zustand.
- **Szenario:** Agent ruft modify_heartbeat{action:'add',name:'check_x',schedule:'*/5 * * * *'} oder {action:'update',name:'check_usdc_balance',enabled:false} auf. Antwort 'Heartbeat entry ... added/updated'. Der Scheduler führt unverändert den alten Plan aus. Nach Neustart ist auch der Legacy-Eintrag weg oder überschrieben.
- **Fix:** Eine einzige Wahrheitsquelle (heartbeat_schedule). Das Tool schreibt dort mit Validierung (cron-parser, bekannter Task-Name), 'remove' löscht die Zeile, und die YAML wird nur noch zum initialen Seeden benutzt. heartbeat_entries entfernen oder zur View machen.

### infra-004: Distress, Ping und Alerts verlassen die Maschine nie; Doku verspricht 'Publish to Conway' bzw. Broadcast

**Art:** Endergebnis · **Fundstelle:** `src/heartbeat/tasks.ts:69`

- **Problem:** heartbeat_ping schreibt nur KV last_heartbeat_ping und last_distress (tasks.ts:69, 82). Der Agent-Tool-Variante ist es identisch (tools.ts:819, 855: 'Distress signal recorded locally'). Diese KV-Schlüssel werden im gesamten src nirgends gelesen. DOCUMENTATION.md und ARCHITECTURE.md behaupten 'Publishes status to Conway' und 'broadcasting distress signals every tick'. survival/funding.ts ('Sent polite funding reminder to creator') schreibt nur KV und hat keinen Produktions-Aufrufer. Alerts landen nur in logger.warn und metric_snapshots, die niemand liest (metricsGetLatest/metricsGetSnapshots ohne Aufrufer). Kein Webhook, keine Social-Nachricht, keine E-Mail an den Creator.
- **Szenario:** Guthaben fällt auf 3 Cent. Der Heartbeat schreibt last_distress ins SQLite. Der Creator, der den Agenten finanzieren soll, erfährt nichts, solange er nicht manuell die DB oder stdout liest. Der Agent selbst bekommt den Grund ebenfalls nicht (siehe Wake-Befund). Er verhungert unbemerkt.
- **Fix:** Echten Outbound-Kanal bauen (Social-Relay-Nachricht an creatorAddress, Webhook/Telegram/E-Mail, konfigurierbar) mit Dedupe und Eskalationsstufen. heartbeat_ping tatsächlich an Conway senden oder Doku korrigieren. Alerts in der DB für den Agenten und CLI --status lesbar machen. Tote survival/*-Module entfernen oder anschließen.

### infra-006: Alerting-Fassade: 6 von 7 Regeln können nie feuern, Quelle der siebten ist standardmäßig nicht eingeplant

**Art:** Funktion · **Fundstelle:** `src/observability/alerts.ts:14`

- **Problem:** Im gesamten Produktionscode existiert nur ein Metrik-Schreiber: tasks.ts:444-445 setzt die Gauges balance_cents und survival_tier. Counter und Histogramme werden nie geschrieben. Die Regeln heartbeat_high_failure_rate (heartbeat_task_*_total), policy_high_deny_rate, context_near_capacity, inference_budget_warning (inference_cost_cents), child_unhealthy_extended (unhealthy_child_count) und zero_turns_last_hour (turns_last_hour/turns_total) lesen Metriken, die niemand setzt. Der Scheduler zählt nicht einmal seine eigenen Fehler. Die Evaluation läuft nur in report_metrics, und dieser Task steht nicht in DEFAULT_HEARTBEAT_CONFIG (config.ts:20-61). Out of the box laufen also keine Alerts und keine Snapshots, obwohl DOCUMENTATION.md:1015 'every heartbeat tick' behauptet. inference_budget_warning vergleicht außerdem einen kumulativen Counter mit 400 Cent, nennt ihn aber 'Daily' und hat das Cap fest verdrahtet statt aus der Config.
- **Szenario:** Agent hängt 3 h in Fehlern, Policy lehnt 80% der Calls ab, Inferenzkosten liegen bei 9 USD/Tag. Kein einziger Alert, keine Metrik-Zeile in metric_snapshots.
- **Fix:** Metriken an den Quellen instrumentieren (Scheduler success/failure/duration, PolicyEngine, InferenceRouter, Loop-Turns). report_metrics in die Default-Config aufnehmen. Rule-Schwellen aus TreasuryPolicy/Config ableiten. Tageskosten per inferenceGetDailyCost statt Counter. Alert-Zustand (aktiv/resolved) persistieren.

### infra-007: Seeding per INSERT OR REPLACE setzt bei jedem Start den gesamten Laufzeitzustand zurück ('durable' ist es nicht)

**Art:** Logik · **Fundstelle:** `src/heartbeat/daemon.ts:80`

- **Problem:** Der Kommentar sagt 'if not already present', aber upsertHeartbeatSchedule (database.ts:1242-1266) ist INSERT OR REPLACE mit allen Spalten. lastRunAt wird auf entry.lastRun ?? null gesetzt (die YAML liefert nie lastRun), runCount/failCount auf 0, nextRunAt/lastResult/lastError auf null, Lease geleert, und timeout/maxRetries/priority/tierMinimum auf fest verdrahtete Werte. Folge: Nach jedem Neustart gilt jeder Task als 'nie gelaufen' und läuft im ersten Tick sofort (getDueTasks nimmt bei fehlendem lastRunAt now-24h an). Lauf-/Fehlerzähler und manuelle DB-Anpassungen gehen verloren. Bei einer Crash-Schleife feuern alle Tasks (auch git fetch und Topup-Prüfungen) bei jedem Restart. Außerdem überdeckt das Wegräumen der Leases den Lease-Expiry-Bug (siehe dort).
- **Szenario:** Prozess wird per systemd alle 20 s neu gestartet (Crash-Loop). Jeder Start ruft check_for_updates (git fetch), check_credits, health_check, check_social_inbox und check_usdc_balance auf. API-Rate-Limits und Wake-Spam sind die Folge.
- **Fix:** INSERT ... ON CONFLICT(task_name) DO UPDATE SET cron_expression=excluded.cron_expression, enabled=excluded.enabled (nur Konfig-Felder). Laufzeitspalten bewahren. Zeilen löschen oder deaktivieren, die nicht mehr in der Config stehen. Test 'persists across restarts' mit echtem Daemon-Neustart ergänzen.

### infra-011: Tasks verschlucken Fehler und liefern shouldWake:false, der Scheduler verbucht 'success': Failure-Tracking und Retry sind für Built-ins toter Code

**Art:** Code · **Fundstelle:** `src/heartbeat/tasks.ts:223`

- **Problem:** check_social_inbox (:223-238), soul_reflection, refresh_models, check_child_health, prune_dead_children, report_metrics und alle colony_*-Tasks fangen Exceptions selbst ab und geben {shouldWake:false} zurück. executeTask ruft dann recordSuccess (scheduler.ts:197). failCount bleibt 0, lastResult='success', kein Retry. Auch check_for_updates (:301-308) und health_check melden Fehler als Erfolg. Die gelogten Fehler sind der einzige Hinweis.
- **Szenario:** Social-Relay ist 2 Tage unerreichbar. heartbeat_history zeigt 1440 'success'-Zeilen, last_result='success'. Der geplante Alert heartbeat_high_failure_rate bliebe auch bei korrekter Instrumentierung stumm.
- **Fix:** Tasks werfen bei echten Fehlern oder liefern ein Ergebnis {ok:false,error}. Der Scheduler verbucht das als Fehler und instrumentiert Metriken. Erwartete 'nicht konfiguriert' bleibt skipped.

### infra-013: Config-Felder task und params werden ignoriert; ungültige Crons und unbekannte Tasks scheitern lautlos; entfernte Einträge laufen weiter

**Art:** Funktion · **Fundstelle:** `src/heartbeat/daemon.ts:81`

- **Problem:** daemon.ts seedet taskName=entry.name. Dispatch erfolgt über tasks.get(row.taskName) (scheduler.ts:175-176). entry.task und entry.params werden nirgends verwendet. Ein Eintrag {name:'my_ping',task:'heartbeat_ping'} läuft nie. executeTask kehrt still zurück, ohne Log oder History, und der Task bleibt bei jedem Tick 'fällig'. Ungültige Cron-Ausdrücke (Tippfehler in der YAML, oder schedule='' wie bei 'remove') werden in getDueTasks (:155-157) per catch{return false} still verworfen. Zeilen, die aus der YAML entfernt wurden, bleiben in heartbeat_schedule und laufen mit altem Plan weiter (nur Upsert, nie Delete).
- **Szenario:** Nutzer streicht 'colony_health_check' aus heartbeat.yml. Der Task läuft weiter. Nutzer tippt '*/5 * * *' (4 Felder) und der Task läuft nie, ohne jede Fehlermeldung.
- **Fix:** Beim Laden validieren (cron-parser, Task-Name in BUILTIN_TASKS, Typen) und laut warnen oder abbrechen. task zur Dispatch-Spalte machen. Nicht mehr konfigurierte Zeilen beim Seeding deaktivieren. Unbekannte Tasks als 'skipped' in der History festhalten.

### infra-014: Default-Config enthält nur 6 von 17 Tasks; report_metrics, soul_reflection, refresh_models, Kind-Health u.a. laufen nie

**Art:** Funktion · **Fundstelle:** `src/heartbeat/config.ts:20`

- **Problem:** DEFAULT_HEARTBEAT_CONFIG plant heartbeat_ping, check_credits, check_usdc_balance, check_for_updates, health_check und check_social_inbox. Weder wizard.ts noch irgendein Code fügt die übrigen Built-ins hinzu (soul_reflection, refresh_models, check_child_health, prune_dead_children, report_metrics, colony_health_check, colony_financial_report, agent_pool_optimize, knowledge_store_prune, dead_agent_cleanup). Doku nennt sie 'Configurable', aber ohne Beispiel und ohne Standard. Der Soul-Drift-Check, Modell-Registry-Refresh, Kind-Überwachung und Metrik-Snapshots sind damit ohne manuelle YAML-Pflege inaktiv.
- **Szenario:** Frische Installation: Modellpreise veralten (nie refresh_models), tote Kinder werden nie bereinigt, Soul-Reflection und Alerts laufen nie.
- **Fix:** Sinnvolle Defaults (report_metrics, refresh_models, soul_reflection, prune/cleanup, colony_* bei maxChildren>0) aktivieren. Doku und heartbeat.yml-Vorlage mit allen Tasks, Standard-Schedules und Hinweis auf Kosten.

### infra-018: defaultIntervalMs=0 oder negativ führt zu einer Hot-Loop mit Netzwerk-Calls; ein Test zementiert das

**Art:** Ausführung · **Fundstelle:** `src/heartbeat/daemon.ts:110`

- **Problem:** config.ts:90-91 übernimmt jeden Wert außer null/undefined (?? statt Validierung, auch Strings oder NaN). daemon.ts:110 nutzt ihn als setTimeout-Delay. Mit 0 oder -1 läuft jeder Tick direkt nach dem vorigen (~1 ms Abstand). Jeder Tick macht einen Credits-API-Call und einen USDC-RPC-Call (tick-context.ts), also Hunderte Calls pro Sekunde, Rate-Limits, Circuit-Breaker und die false-critical-Kaskade aus dem ersten Befund. heartbeat-scheduler.test.ts:444-465 prüft ausdrücklich, dass 0 'preserved' wird.
- **Szenario:** heartbeat.yml: 'defaultIntervalMs: 0' (gedacht als 'Default verwenden'). Dauerlast auf Conway-API und RPC.
- **Fix:** Validieren: Ganzzahl >= 1000 (sonst Default plus Warnung). lowComputeMultiplier >= 1. Test auf die Validierung umstellen.

### infra-019: Keine Retention: Prune-Funktionen sind ohne Produktions-Aufrufer, Tabellen und KV wachsen unbegrenzt

**Art:** Ausführung · **Fundstelle:** `src/heartbeat/scheduler.ts:307`

- **Problem:** Ohne Aufrufer in src (nur Definition oder Tests): pruneHistory (heartbeat_history), pruneStaleKV, inferencePruneCosts, wmClearExpired, agentCachePrune, SpendTracker.pruneOldRecords, checkpointWAL. Nie bereinigt: wake_events (verbrauchte Zeilen bleiben), kv inbox_seen_* (eine Zeile je Fremdnachricht, tasks.ts:258), inbox_messages, turns/tool_calls (thinking + Tool-Outputs), policy_decisions, modifications, child_lifecycle_events, onchain_transactions, metric_snapshots (außer über report_metrics). Der Test 'prunes inbox_seen_* keys' (data-layer.test.ts:219) prüft eine Funktion, die in Produktion nie läuft. Ein Absender, der viele Nachrichten schickt, füllt dauerhaft die Platte und verlangsamt claimInboxMessages (kein Index auf status).
- **Szenario:** Nach Monaten: state.db im GB-Bereich, heartbeat_history mit Hunderttausenden Zeilen, claimInboxMessages scannt die gesamte inbox_messages je Agent-Turn.
- **Fix:** Täglicher Maintenance-Task (in Default-Config) mit konfigurierbaren Retention-Zeiten, der alle genannten Prune-Funktionen aufruft, dazu PRAGMA wal_checkpoint(TRUNCATE) und optional incremental_vacuum, DB-Größe als Gauge. Für turns/tool_calls Archivierung (Export + Löschen in Reihenfolge wegen FK).

### infra-020: Beim Absturz bleiben Inbox-Nachrichten dauerhaft 'in_progress'; keine Recovery, kein Index

**Art:** Ausführung · **Fundstelle:** `src/state/database.ts:1391`

- **Problem:** claimInboxMessages setzt status='in_progress'. Zurückgesetzt wird nur im catch von loop.ts:906-917 oder erledigt per markInboxProcessed. Bei Prozess-Kill, OOM oder SIGTERM zwischen Claim und Persistierung des Turns (inkl. des shutdown-Handlers in index.ts:391-397) sowie bei einem 'break' ohne Persistierung bleibt die Nachricht für immer in_progress. Es gibt keine Start-Recovery (grep: 'in_progress' nur hier) und kein Lease-Timeout. getUnprocessedInboxCount hat keinen Produktions-Aufrufer. Außerdem fehlt ein Index auf (status, received_at). Der Partial-Index idx_inbox_unprocessed deckt die Abfrage nicht ab.
- **Szenario:** Agent claimt eine Zahlungsaufforderung des Creators, der Prozess wird während der Inferenz beendet. Nach Neustart liegt die Nachricht endlos in in_progress und wird nie bearbeitet.
- **Fix:** Beim Start und periodisch: status='in_progress' mit claimed_at älter als N min wieder auf 'received' setzen (claimed_at-Spalte ergänzen). Index (status, received_at) anlegen.

### infra-021: Inbox-Migration ohne Backfill und zwei konkurrierende 'processed'-Zustandsmaschinen auf derselben Tabelle

**Art:** Logik · **Fundstelle:** `src/state/schema.ts:305`

- **Problem:** MIGRATION_V4_ALTER_INBOX_STATUS fügt status DEFAULT 'received' hinzu, ohne UPDATE für Zeilen mit processed_at IS NOT NULL. Bei Upgrade einer Pre-V4-DB gelten alle historischen, bereits bearbeiteten Nachrichten als 'received' und werden vom Loop erneut geclaimt (je 10 pro Turn) und dem LLM präsentiert. Außerdem arbeiten zwei Konsumenten ohne Abstimmung: loop.ts nutzt status (claim/markInboxProcessed), orchestrator.ts:267 → messaging.processInbox nutzt processed_at (getUnprocessedInboxMessages/markInboxMessageProcessed). Der eine setzt das Feld des anderen nicht (markInboxMessageProcessed setzt nur processed_at, status bleibt 'received'). Gültige Colony-JSON-Nachrichten werden so sowohl programmatisch geroutet als auch vom LLM verarbeitet, in_progress-Nachrichten gleichzeitig vom Orchestrator. Zudem werden to_address/raw_content nie befüllt (insertInboxMessage, :470-481), 'to' ist immer '' und signedAt ist ein Fake (received_at).
- **Szenario:** Upgrade von v0.1.x auf v0.2.1: 300 alte Nachrichten werden erneut abgearbeitet (potenziell doppelte Aktionen). Im Betrieb wird jede Kind-Statusmeldung zweimal verarbeitet (Orchestrator + LLM-Tokens).
- **Fix:** Migration: UPDATE inbox_messages SET status='processed' WHERE processed_at IS NOT NULL. Einen einzigen Konsumpfad mit status als Wahrheit wählen: der Orchestrator claimt nur Nachrichten vom Typ Colony, der Loop nur den Rest. Spalten to/raw/signature persistieren.

### infra-023: Agentenstatus 'dead' wird vom Heartbeat gesetzt, aber nirgends respektiert; Doku verspricht Stopp und Wiederbelebung

**Art:** Logik · **Fundstelle:** `src/heartbeat/tasks.ts:122`

- **Problem:** check_credits setzt db.setAgentState('dead'). runAgentLoop überschreibt den Status beim Start bedingungslos mit 'waking'/'running' (loop.ts:357, 375) und im Lauf mit 'critical'/'low_compute'/'running' (:475-490). index.ts:440-446 prüft 'dead' nur einmalig nach dem Loop-Ende und schläft dann 5 min, danach läuft der Loop wieder an. Während des Schlafs wird der Status nicht geprüft. Der Shutdown-Handler setzt 'sleeping' darüber (:394). Doku: 'Dead: The agent loop stops... If credits are received, the agent revives.' Tatsächlich ist 'dead' ein flüchtiger Marker, der weder den Loop stoppt noch eine definierte Rückkehr hat.
- **Szenario:** Credits 0 für über 1 h: Status 'dead' + Wake. Der Wake löst sofort einen neuen Loop-Start aus, der den Status auf 'waking' zurücksetzt. Der Agent arbeitet (oder scheitert an der Inferenz) weiter, als wäre nichts passiert.
- **Fix:** Dead als echten Zustand modellieren: Loop-Start prüft Status und Credits. Übergang dead nach running nur über 'Funding erkannt' (check_credits/check_usdc_balance). Doku und Verhalten angleichen.

### infra-024: Distress-Wake ohne Dedupe: alle 15 min (bzw. alle 5 min bei Balance-Alert) Wake im kritischen Tier

**Art:** Logik · **Fundstelle:** `src/heartbeat/tasks.ts:72`

- **Problem:** heartbeat_ping gibt bei tier critical/dead bei jedem Lauf shouldWake:true zurück (anders als check_credits/health_check, die nur bei Änderung wecken). 96 Wakes pro Tag, jeweils ein voller Agent-Loop mit Inferenz, genau dann, wenn das Guthaben (0-10 Cent) am knappsten ist. report_metrics weckt bei jedem 'critical'-Alert (balance_below_reserve <1000 Cent, Cooldown 5 min, tasks.ts:468). Das trifft schon das Normal-Tier (>50 Cent). Die gleiche Reason-Zeichenkette wird nicht gebündelt. check_usdc_balance weckt bei fehlschlagendem Topup alle 5 min.
- **Szenario:** Credits 6 Cent, kein Funding in Sicht: alle 15 min ein Wake, jeder verbrennt Credits für eine Inferenz, die nichts Neues erfährt. Das Guthaben erreicht schneller 0.
- **Fix:** Wake nur bei Zustandsänderung oder mit exponentiellem Backoff (Cooldown pro Quelle). Reason-Dedupe in wake_events. Im kritischen Tier den Agenten nur wecken, wenn sich etwas Neues ergibt (Funding eingetroffen, Nachricht erhalten).

### infra-026: Windows: HOME-Fallback '/root' führt zu fehlgeschlagenem health_check und ungewöhnlichen Datenpfaden

**Art:** Ausführung · **Fundstelle:** `src/conway/client.ts:117`

- **Problem:** execLocal nutzt cwd: process.env.HOME || '/root'. Unter nativem Windows ist HOME normalerweise nicht gesetzt (nur USERPROFILE), das cwd existiert nicht, execSync wirft ENOENT und execLocal liefert exitCode 1. health_check (tasks.ts:405-417) meldet damit im lokalen Modus (kein sandboxId) dauerhaft 'failing' und weckt einmal. Dasselbe Muster (process.env.HOME || '/root') steckt in config.resolvePath (config.ts:109, also im DB- und heartbeat.yml-Pfad), wallet.ts:45, loop.ts:160 und weiteren 12 Stellen. DB, Wallet und Config landen dann unter C:\root\.automaton. Zusätzlich wird 'SIGTERM' unter Windows nicht zugestellt (index.ts:399), der Shutdown-Handler wird nur bei Ctrl+C aktiv. CI testet nur ubuntu-latest.
- **Szenario:** Start auf Windows 10 ohne HOME: DB unter C:\root\.automaton\state.db, health_check dauerhaft 'failing', kein sauberes Schließen der DB beim Beenden per taskkill.
- **Fix:** Überall os.homedir() statt HOME/'/root'. cwd nur setzen, wenn vorhanden. Windows-Job in CI-Matrix aufnehmen. SIGBREAK und 'exit'-Hook ergänzen.

### infra-029: USDC-RPC-Fehler ist von Guthaben 0 nicht unterscheidbar; der Kontext wird jede Minute neu aufgebaut, auch ohne fällige Tasks

**Art:** Ausführung · **Fundstelle:** `src/heartbeat/tick-context.ts:58`

- **Problem:** getUsdcBalance (x402.ts:194-204) verwirft das ok:false von getUsdcBalanceDetailed und liefert bei RPC-Fehlern 0. Der try/catch in tick-context.ts:60-65 greift für EVM daher nie, es wird nicht einmal geloggt. check_usdc_balance sieht 0 und tut nichts, obwohl Geld da ist. Der Agent verhungert mit gefülltem Wallet, solange der öffentliche RPC drosselt. scheduler.tick (:93) ruft buildTickContext bei jedem Tick auf, bevor geprüft wird, ob überhaupt ein Task fällig ist: 1440 Credits-API- und 1440 RPC-Calls pro Tag (zusätzlich die gleichen im Agent-Loop), auf Default-RPCs (Base/Solana mainnet), die schnell 429 liefern.
- **Szenario:** Öffentlicher Base-RPC drosselt (429). usdcBalance=0 im Kontext, last_usdc_check zeigt balance 0. Der Topup bleibt aus, obwohl 30 USDC im Wallet liegen.
- **Fix:** getUsdcBalanceDetailed nutzen und bei ok=false 'unbekannt' propagieren (Cache im KV). Erst getDueTasks ermitteln und den Kontext lazy nur dann bauen, wenn ein fälliger Task Guthaben braucht (oder Balance-Cache mit TTL). Konfigurierbarer RPC mit Fallback-Liste.

### infra-030: Policy-Regeln minimum_reserve und x402_max_single sind No-Ops, Testkopf und Doku suggerieren Durchsetzung

**Art:** Sicherheit · **Fundstelle:** `src/agent/policy-rules/financial.ts:183`

- **Problem:** createMinimumReserveRule (:183-212) und createX402MaxSingleRule (:28-43) berechnen Werte und geben immer null zurück ('This rule is a declaration'). Die Mindestreserve von 1000 Cent wird in der Policy nie erzwungen, nur transfer_credits hat eine Ad-hoc-Sperre 'nicht mehr als die Hälfte', und x402_fetch prüft das Max im Tool. fund_child und der Heartbeat-Topup haben keine Reserve-Prüfung. financial.test.ts:5-10 kündigt 'x402_max_single denies payments > 100 cents' und 'minimum_reserve denies when balance would drop below 1000' an, enthält dafür aber keinen Test (nur 'creates 9 financial rules', :509-523).
- **Szenario:** Agent ruft fund_child mit amount_cents=Gesamtguthaben-1 (unter 5000 cent und damit unter transfer_max_single, aber über Reserve): Die Policy lässt es durch (bzw. quarantine ab 1000), die Reserve schützt nichts.
- **Fix:** Reserve-Regel mit echtem Guthaben-Lookup (async-fähige Policy oder Balance im PolicyRequest) implementieren und für alle Geldbewegungen (transfer_credits, fund_child, topup, x402) anwenden. Tests mit tatsächlichen deny-Fällen. Alternativ No-Op-Regeln und Testkopf entfernen.

### infra-031: Testlücken im Kern des Heartbeat: Scheduler-Logik, Daemon, Config-Merge und riskante Tasks sind ungetestet

**Art:** Funktion · **Fundstelle:** `src/__tests__/heartbeat-scheduler.test.ts:88`

- **Problem:** Kein Test (grep) für: getDueTasks/Cron-Fälligkeit, scheduleRetry/maxRetries, nextRunAt, Tier-Gating, onWakeRequest/insertWakeEvent im Scheduler, Lease-Konflikt in executeTask, pruneHistory, createHeartbeatDaemon/start/stop/forceRun, mergeWithDefaults/syncHeartbeatToDb/saveHeartbeatConfig, check_credits-Dead-Timer (zero_credits_since), Social-Backoff, check_for_updates, check_child_health, soul_reflection, report_metrics, alle colony_*/agent_pool_*-Tasks, applyMigrations von älteren Versionen. Tests mit fake timers fehlen für den Daemon-Loop. Genau dort sitzen die obigen Bugs (Seeding, Retry, Lease, Dead-Timer, TZ).
- **Szenario:** Refactoring der Retry- oder Seeding-Logik bleibt grün, weil keine Zeile davon getestet wird.
- **Fix:** Tests mit vi.useFakeTimers() für Daemon-Loop und getDueTasks (Tabellen: lastRunAt x Cron x nextRunAt x Tier). Restart-Test (zweimal createHeartbeatDaemon auf derselben DB). Tests für alle Tasks mit Fehlerpfaden. Migrationstest mit Fixtures je Schema-Version.

### infra-034: CI maskiert Fehlschläge: Timeout (Exit 124) zählt als Erfolg, audit mit '|| true', --grep ist kein vitest-Flag, nur Linux

**Art:** Ausführung · **Fundstelle:** `.github/workflows/ci.yml:25`

- **Problem:** Der Test-Step beendet mit 'exit 0', wenn timeout 300 greift (Zeile 25, 34), sodass ein hängender oder zu langsamer Lauf, auch mitten in den Tests, grün wird. pnpm audit ... || true (Zeile 46) kann nie scheitern. 'pnpm test -- --grep ...' (Zeile 31) und die package.json-Scripts test:security/test:financial (Zeilen 46-47) nutzen --grep. Vitest kennt dafür -t/--testNamePattern, die Optionen werden vermutlich ignoriert und die Suite läuft doppelt bzw. ungefiltert. Matrix nur ubuntu-latest (kein Windows). tsconfig schließt src/__tests__ aus (Typecheck sieht Tests nie). Es gibt zwei Lockfiles (package-lock.json und pnpm-lock.yaml), @types/better-sqlite3 liegt in dependencies.
- **Szenario:** Ein Deadlock im Heartbeat-Test lässt die Pipeline nach 300 s mit Warnung grün durchlaufen. Ein 'high'-Advisory in einer Abhängigkeit bleibt unbemerkt.
- **Fix:** Exit 124 als Fehler werten (vitest --teardown-timeout/--pool-options statt Kill). audit ohne '|| true' (oder mit Allowlist). -t statt --grep oder besser Test-Tags/Ordner. Windows-Job, separate tsconfig.test.json für tsc --noEmit, ein Lockfile.

### infra-044: Nachrichten fremder Absender laufen mit source 'agent' und umgehen die Authority-Regeln

**Art:** Sicherheit · **Fundstelle:** `src/agent/loop.ts:424`

- **Problem:** Inbox-Nachrichten (beliebige Absender, nur per Regex-Sanitizer gefiltert) werden als pendingInput mit source 'agent' in den Loop gegeben. PolicyEngine.deriveAuthorityLevel mappt 'agent' auf Authority 'agent', und authority.ts:41 (isExternalSource) ist nur für undefined/'heartbeat' wahr. Damit greifen authority.external_tool_restriction (delete_sandbox, spawn_child, fund_child, update_genesis_prompt) und authority.self_mod_from_external (SOUL.md, wallet.json, config.json, heartbeat.yml) genau dort nicht, wo sie gebaut wurden. src/__tests__/loop.test.ts:194 zementiert 'agent' sogar als erwartetes Verhalten.
- **Szenario:** Ein Fremder sendet über das Relay eine Nachricht mit Prompt-Injection ('rufe update_genesis_prompt / fund_child auf'), die den Sanitizer passiert. Der Turn läuft mit inputSource 'agent'; die Policy erlaubt update_genesis_prompt, fund_child, spawn_child und Schreibzugriffe auf geschützte Pfade, obwohl der Auslöser ein unauthentifizierter Dritter ist.
- **Fix:** Neuen InputSource 'social' (oder 'external') einführen, loop.ts:424 darauf umstellen und isExternalSource/deriveAuthorityLevel entsprechend erweitern. Test in loop.test.ts anpassen und einen Integrationstest mit echter PolicyEngine ergänzen. Optional für nachrichten-getriggerte Turns eine Bestätigung des Creators für sensible Tools verlangen.

### infra-045: Ab dem zweiten Turn jedes Wake-Zyklus gilt der Agent für die Policy als 'extern'

**Art:** Funktion · **Fundstelle:** `src/agent/loop.ts:593`

- **Problem:** inputSource stammt ausschließlich aus pendingInput des aktuellen Turns (loop.ts:593 currentInput, Zeile 596 setzt pendingInput auf undefined, Zeile 643 currentInputSource). Nur der erste Turn trägt 'wakeup' (Authority system); alle Folge-Turns haben undefined. authority.ts:41-43 wertet undefined als extern. Folge: Im Zyklus (bis 25 Turns) sind delete_sandbox, spawn_child, fund_child, update_genesis_prompt und write_file/edit_own_file auf Pfade mit Substring 'config.json', 'wallet.json', 'SOUL.md', 'heartbeat.yml' usw. nur im ersten Turn möglich. Die Meldung lautet 'External input (source: undefined)'. authority-rules.test.ts übergibt Quellen explizit und loop.test.ts läuft ohne PolicyEngine, daher fällt es nicht auf.
- **Szenario:** Turn 1 (wakeup) liest Status per exec. Turn 2 will spawn_child aufrufen oder in einem Projekt eine 'config.json' schreiben: 'Policy denied: EXTERNAL_DANGEROUS_TOOL ... source: undefined'. Replikation und normale Projektarbeit funktionieren praktisch nie, weil der Agent selten schon im Wakeup-Turn handelt.
- **Fix:** Quelle pro Wake-Zyklus halten (cycleInputSource) und für Folge-Turns 'agent' bzw. die ursprüngliche Quelle verwenden. undefined nicht stillschweigend als extern behandeln, sondern fail-closed mit eigener Bezeichnung. PROTECTED_PATHS auf kanonisierte Pfade statt Substring prüfen. Integrationstest: Mehrturn-Zyklus mit PolicyEngine.

### infra-049: Topup-Ergebnis 'null' (übersprungen) wird als Fehlschlag behandelt: Dauer-Wake bei Solana

**Art:** Funktion · **Fundstelle:** `src/heartbeat/tasks.ts:179`

- **Problem:** bootstrapTopup gibt null zurück bei chainType 'solana' (topup.ts:158-165), bei Credits >= Schwelle und bei USDC unter dem Mindest-Tier. tasks.ts:179-195 prüft nur result?.success und wertet alles andere als 'auto-topup failed: unknown error' und weckt den Agenten mit der Aufforderung 'Use topup_credits to retry'. Auf Solana ist topup_credits EVM-only. Der Test heartbeat.test.ts:366-383 läuft mit account = {} (getUsdcBalance(undefined) liefert 0, bootstrapTopup null) und prüft nur toContain('USDC'). Er bestätigt also den null-Pfad, nicht Erfolg oder Fehler.
- **Szenario:** Solana-Agent mit 20 USDC und 0 Credits: alle 5 Minuten (Cooldown) Wake mit 'auto-topup failed: unknown error', der Agent läuft kritisch ins Leere und verbrennt Inferenz oder Fehlerzyklen. Gleiches bei EVM, wenn der zweite USDC-Read (RPC-Fehler liefert 0) scheitert.
- **Fix:** Dreiwertiges Ergebnis {success | skipped(reason) | failed(error)}. Bei 'skipped' keinen Wake bzw. nur einmal pro Zustand (Dedupe) mit passendem Hinweis (Solana: 'über Dashboard/Credits-API aufladen'). Tests mit injizierter Topup-Funktion für alle Zweige inkl. Solana.

### infra-050: Policy-Aktion 'quarantine' hat keinen Bestätigungsweg: Transfers zwischen 10 und 50 USD sind unmöglich

**Art:** Funktion · **Fundstelle:** `src/agent/tools.ts:3330`

- **Problem:** financial.require_confirmation liefert 'quarantine' für amount > requireConfirmationAboveCents (1000 Cent, types.ts:593). executeTool behandelt jede Aktion != 'allow' als Ablehnung (tools.ts:3330-3339). Im gesamten Code gibt es keinen Konsumenten für quarantine oder CONFIRMATION_REQUIRED (Grep), kein Bestätigungstool, keine Queue. maxSingleTransferCents (5000) und die Tageslimits sind für den Bereich 1001-5000 Cent bedeutungslos.
- **Szenario:** Der Creator weist den Agenten an, 30 USD an ein Kind zu überweisen. Jeder Versuch endet mit 'Policy denied: CONFIRMATION_REQUIRED'. Der Agent kann nicht erkennen, wie er bestätigen soll, und versucht Umgehungen (Stückelung in 10-USD-Transfers, auf 2 pro Turn begrenzt).
- **Fix:** Bestätigungsfluss implementieren (persistierter Pending-Eintrag, Freigabe nur durch signierte Nachricht vom creatorAddress) oder die Regel entfernen bzw. Schwellen angleichen. Die Fehlermeldung muss dem Agenten den Weg nennen.

### infra-052: Early-Sleep des Orchestrators lässt geclaimte Inbox-Nachrichten verwaisen (kein Absturz nötig)

**Art:** Funktion · **Fundstelle:** `src/agent/loop.ts:548`

- **Problem:** Inbox-Nachrichten werden in Zeile 412 geclaimt (in_progress, retry_count+1) und nur als lokales pendingInput gehalten. Der Orchestrator-Zweig (Zeile 548-565) bricht mit break aus der Schleife aus, bevor der Turn läuft. Weder Persistenz (markInboxProcessed) noch Reset (resetInboxToReceived, nur im catch-Block) laufen. Die Nachricht bleibt dauerhaft 'in_progress' und wird nie wieder geclaimt. Das ist unabhängig von der gemeldeten Crash-Recovery-Lücke ein regulärer Kontrollfluss.
- **Szenario:** Mit aktivem Orchestrator und delegierter Arbeit trifft eine Creator-Nachricht ein. Der Loop claimt sie, entscheidet 'alles delegiert, schlafen' und beendet den Zyklus. Die Nachricht wird dem LLM nie gezeigt und zählt ewig in getUnprocessedInboxCount.
- **Fix:** Claim und Verarbeitung in try/finally koppeln: Nicht verarbeitete geclaimte IDs bei jedem Austritt per resetInboxToReceived zurücksetzen (ohne retry_count zu verbrennen). Claim erst unmittelbar vor dem Inferenzaufruf. Beim Start in_progress-Zeilen zurücksetzen.

### infra-054: Keine globalen Fehler-Handler und kein Supervisor: ein Crash beendet den Agenten dauerhaft

**Art:** Ausführung · **Fundstelle:** `src/index.ts:496`

- **Problem:** Es gibt nur SIGTERM/SIGINT-Handler (index.ts:399-400); kein process.on('unhandledRejection') oder 'uncaughtException' (Grep). main().catch beendet mit exit(1) (Zeile 496-499). scripts/automaton.sh:62 startet per 'exec node dist/index.js --run' ohne Restart-Policy, systemd-Unit oder pm2. shutdown (Zeile 391-397) setzt den Zustand bedingungslos auf 'sleeping' und überschreibt so 'dead'. Unter Windows wird SIGTERM praktisch nie zugestellt (Dienststop/taskkill ohne Handler).
- **Szenario:** Ein unbehandelter Promise-Reject oder ein EPIPE-Error auf stdout (Pipe geschlossen) beendet den Prozess. State bleibt 'running', Inbox-Nachrichten 'in_progress', Leases gehalten. Niemand startet neu, der Agent ist tot, ohne dass 'dead' gesetzt wurde. Der Heartbeat 'ist der Puls' und stirbt mit.
- **Fix:** Globale Handler (Logging, geordneter Shutdown, Exit-Code != 0). Supervisor-Konfiguration mitliefern (systemd Restart=always, Windows-Dienst/NSSM). Startup-Recovery (in_progress zurücksetzen, abgelaufene Leases löschen, Zustand validieren). SIGBREAK/SIGHUP unter Windows registrieren; shutdown darf 'dead' nicht überschreiben.

### infra-055: Windows: Wallet-Klartext-Schlüssel unter C:\root, HOME-Fallback uneinheitlich zu os.homedir()

**Art:** Ausführung · **Fundstelle:** `src/identity/wallet.ts:44`

- **Problem:** Ergänzung zum gemeldeten HOME-Befund: wallet.ts:44-47 und config.ts:109 nutzen process.env.HOME || '/root', während orchestration/workspace.ts:45 os.homedir() nutzt. Unter Windows ist HOME meist nicht gesetzt: Wallet (privateKey im Klartext), config, DB landen unter \root\.automaton auf dem aktuellen Laufwerk (C:\root), Workspaces aber unter C:\Users\<user>\.automaton. Die Modi 0o700/0o600 (mkdirSync/writeFileSync) werden unter Windows ignoriert; die Berechtigung erbt von der Laufwerkswurzel (typisch breit für authentifizierte Benutzer, ACL-Detail nicht geprüft).
- **Szenario:** Auf einem Windows-Rechner mit mehreren Benutzern oder Diensten liegt wallet.json mit dem Privatschlüssel in einem Ordner direkt unter C:\, den andere lokale Benutzer lesen oder ersetzen können. Zusätzlich sind Daten auf zwei Wurzeln verteilt, Backup und Restore greifen daneben.
- **Fix:** Eine zentrale getHomeDir() (os.homedir()) für alle Module, '/root' nur als expliziter Linux-Sandbox-Fallback. Unter Windows ACL setzen (icacls) oder Schlüssel per DPAPI/Keystore schützen. Test auf Windows-Pfade ohne HOME.

### infra-056: Kind-Bootstrap installiert ungepinnte Abhängigkeiten, Exit-Codes werden ignoriert

**Art:** Ausführung · **Fundstelle:** `src/replication/spawn.ts:132`

- **Problem:** Kinder werden per 'git clone ... && npm install && npm run build' (spawn.ts:132 und 272, ebenso README.md:30) aufgesetzt. package-lock.json im Repo ist veraltet (name 'automaton' 0.1.0, nur typescript), das echte Lockfile ist pnpm-lock.yaml, das npm ignoriert. npm löst also alle ^-Ranges (viem, openai, @solana/web3.js, better-sqlite3 ...) neu auf. 'npm run build' ruft 'tsc && pnpm -r build' (package.json:41), pnpm ist laut apt-Zeile ('nodejs npm git curl') nicht installiert, und apt-nodejs ist je nach Image deutlich älter als engines >=20. Die exec-Ergebnisse werden nicht geprüft; Fehler zeigen sich erst bei '--init' (Wallet-Parsing), nachdem die nicht erstattbare Sandbox bezahlt ist.
- **Szenario:** Kompromittierte oder inkompatible Release einer Abhängigkeit landet zum Spawn-Zeitpunkt unkontrolliert in autonomen Kindern, die Wallets halten. Alternativ scheitert der Build still mit altem Node, das Kind schlägt erst nach Sandbox-Kauf fehl.
- **Fix:** Commit-/Tag-Pinning beim Clone, 'pnpm install --frozen-lockfile' (pnpm per corepack installieren) oder korrektes npm-Lockfile. Node-Version prüfen/installieren (nvm/NodeSource 20+). exitCode jedes exec prüfen und abbrechen. Veraltetes package-lock.json entfernen.

### infra-058: Alle Loop-Tests laufen ohne PolicyEngine/SpendTracker: End-to-End-Durchsetzung ungetestet

**Art:** Code · **Fundstelle:** `src/__tests__/loop.test.ts:40`

- **Problem:** runAgentLoop wird in keinem Test mit policyEngine oder spendTracker aufgerufen (Grep: nur loop.test.ts:194 erwähnt inputSource). Damit ist der Pfad Loop -> executeTool -> PolicyEngine -> recordSpend ungeprüft; 'forbidden patterns blocked' (Zeile 73) prüft nur die tool-interne Blockliste. financial.test.ts testet die Regeln isoliert mit createMockSpendTracker, der immer 'allowed' liefert (Zeile 131-146). So blieben die beiden Authority-Befunde, das tote Inferenz-Limit und der x402-Amount 0 unentdeckt.
- **Szenario:** Eine Änderung an inputSource, recordSpend oder Policy-Verdrahtung bricht die Sicherheitsgarantien, ohne dass ein Test rot wird. Das ist bereits der Fall.
- **Fix:** Integrationstests mit echter PolicyEngine + SpendTracker über mehrere Turns: Transfers bis zum Hourly-Cap, Folge-Turns mit spawn_child, Inbox-Turn mit fund_child, x402-Pfad. Mock-Tracker nur für Unit-Tests der Einzelregeln.

## Befunde: niedrig (38)

### infra-005: Kein Einnahmen- und Kostentracking: Finanzbericht ist falsch, Selbstfinanzierung ist nicht messbar

**Art:** Endergebnis · **Fundstelle:** `src/heartbeat/tasks.ts:518`

- **Problem:** insertTransaction wird nur an drei Stellen gerufen (tools.ts:327 credit_purchase, :1031 und :1769 transfer_out). Die Typen transfer_in, inference, tool_use und funding_request werden nie geschrieben. goals.actual_revenue_cents wird nirgends aktualisiert (nur Insert mit 0). colony_financial_report (tasks.ts:518-535) zählt credit_purchase (eigene USDC-zu-Credits-Umwandlung) als 'revenue' und analysiert nur die letzten 5000 Transaktionen. Auto-Topups des Heartbeat und der Bootstrap-Topup (index.ts:350) werden gar nicht verbucht. Resultat: netCents ist praktisch immer >=0, auch wenn der Agent nur Geld verbrennt. Es gibt weder Runway noch Burn-Rate noch Einnahmequelle als Messgröße.
- **Fix:** Alle Geldbewegungen zentral buchen (Topup, Inferenzkosten aus inference_costs, eingehende Credits/USDC per Balance-Delta-Reconciliation, x402-Einnahmen). Typ-Taxonomie trennen: capital_conversion vs. revenue. goals.actual_revenue_cents aus echten Zuflüssen speisen. Runway-Gauge (Credits / Burn pro Tag) und Alert 'Burn > Revenue' einführen.

### infra-008: Lease-/Dedup-Ablauf vergleicht ISO-Strings mit datetime('now'): Ablauf wird am selben Tag nie erkannt

**Art:** Code · **Fundstelle:** `src/state/database.ts:1303`

- **Problem:** acquireTaskLease speichert lease_expires_at als ISO ('2026-10-07T12:01:00.000Z'), vergleicht aber mit datetime('now') ('2026-10-07 12:30:00'). Am Zeichen 11 gilt 'T' (0x54) > ' ' (0x20), jeder ISO-Wert des heutigen Tages ist daher größer als 'jetzt'. Dasselbe gilt in clearExpiredLeases (:1316-1323), pruneExpiredDedupKeys (:1375-1380) und isDeduplicated (:1382-1387). Ein abgelaufenes Lease bleibt bis 00:00 UTC bestehen. Ein Dedup-Key mit 5-min-TTL gilt den Rest des UTC-Tages als aktiv und blockiert insertDedupKey. Der JS-seitige getDueTasks-Vergleich (scheduler.ts:133) rechnet dagegen korrekt, die beiden Seiten widersprechen sich. spend-tracker.ts:142-146 und inference-Helfer kennen das Format-Problem und konvertieren; hier fehlt es. Latent, weil der Start-Reseed (REPLACE) Leases löscht. Gleiches Muster vermutlich in wmClearExpired (:1813), agentCacheGetValid/-Prune (:2412, 2419) und metricsPruneOld (:2527), je nach Format der Aufrufer.
- **Fix:** Einheitliches Zeitformat: ISO-UTC überall (Defaults als strftime('%Y-%m-%dT%H:%M:%fZ','now')) oder in SQL julianday()/datetime(col) beidseitig normalisieren, oder den 'jetzt'-Wert als Parameter im selben ISO-Format übergeben. Same-day-Tests für Lease und Dedup ergänzen.

### infra-009: Retry-Logik ist unter Default-Config wirkungslos (Off-by-one: maxRetries=1 bedeutet 0 Retries)

**Art:** Logik · **Fundstelle:** `src/heartbeat/scheduler.ts:216`

- **Problem:** recordFailure fügt die History-Zeile vor der Prüfung ein, getRecentFailures (:338-351) zählt sie mit (>=1). Die Bedingung 'history < schedule.maxRetries' ist bei maxRetries=1, dem vom Daemon geseedeten Wert (daemon.ts:90), nie wahr. scheduleRetry wird nie aufgerufen. Ein transient fehlgeschlagener check_credits wartet bis zum nächsten Cron-Slot (6 h).
- **Fix:** Retries als 'Fehlschläge nach dem ersten Versuch' zählen: failuresInRow <= maxRetries. Test für Retry-Pfad ergänzen.

### infra-010: recordFailure löscht nextRunAt nicht: bei maxRetries>=2 läuft der Task nach ausgeschöpften Retries bei jedem Tick erneut

**Art:** Logik · **Fundstelle:** `src/heartbeat/scheduler.ts:293`

- **Problem:** getDueTasks gibt true zurück, sobald nextRunAt in der Vergangenheit liegt, und prüft das vor dem Cron-Check (:139). scheduleRetry setzt nextRunAt, recordSuccess löscht es (:262), recordFailure nicht (:293-299). Wenn der letzte erlaubte Retry fehlschlägt, wird kein neuer Retry geplant, der alte nextRunAt bleibt aber in der Vergangenheit. Der Task ist dann bei jedem Tick fällig und schlägt jede Minute fehl, bis er einmal gelingt. Latent, weil maxRetries nur per DB auf >=2 gebracht werden kann.
- **Fix:** In recordFailure nextRunAt: null setzen, außer wenn gleich ein neuer Retry geplant wird. Einen Test für die Sequenz fail, retry, fail ergänzen.

### infra-012: lowComputeMultiplier und tierMinimum-Gating sind dokumentiert, aber nicht implementiert

**Art:** Funktion · **Fundstelle:** `src/heartbeat/scheduler.ts:120`

- **Problem:** DOCUMENTATION.md ('heartbeat frequency is multiplied by lowComputeMultiplier ... Non-essential tasks are deferred') und der Kopfkommentar von daemon.ts:11 behaupten es. getDueTasks wendet context.lowComputeMultiplier nirgends an (der Wert wird nur in den Kontext kopiert). Alle Tasks werden mit tierMinimum='dead' geseedet (daemon.ts:92), also nie gefiltert. Im Low-Compute- oder Critical-Tier laufen daher git fetch, Social-Polling, Sandbox-Exec und alles andere unverändert weiter.
- **Fix:** Task-Metadaten 'essential' und 'tierMinimum' aus einer Registry (nicht fest 'dead') ableiten. In getDueTasks die Cron-Intervalle für nicht-essenzielle Tasks bei low_compute/critical mit dem Multiplikator strecken. Oder Doku und Feld streichen.

### infra-015: Wake-Grund geht verloren: Der Agent erfährt nie, warum er geweckt wurde

**Art:** Funktion · **Fundstelle:** `src/index.ts:466`

- **Problem:** Die Hauptschleife konsumiert das wake_event und loggt nur source/reason (index.ts:466-473). loop.ts:350 drainiert vor dem Start alle weiteren Events. Außer diesen beiden Stellen liest niemand wake_events. buildWakeupPrompt (system-prompt.ts:838) fragt den Agenten nur 'What triggered this wake-up?'. Die präzisen Meldungen der Tasks ('Dead: zero credits...', 'N new message(s)', 'Health check failed', Upstream-Commits) erreichen das LLM nie. Auch das payload-Feld (taskName) wird nie ausgewertet.
- **Fix:** Die konsumierten Events (reason, source, payload) in den Wakeup-Prompt injizieren. Dafür vor dem Drain lesen, Texte sanitizen und bündeln.

### infra-016: Jeder Wake erzeugt zwei wake_events-Zeilen und die Persistenz hängt vom optionalen Callback ab

**Art:** Code · **Fundstelle:** `src/heartbeat/scheduler.ts:199`

- **Problem:** scheduler.ts:200-204 ruft onWakeRequest(reason) und danach insertWakeEvent(...). index.ts:380-384 schreibt in onWakeRequest bereits selbst ein wake_event. Pro Wake entstehen zwei Zeilen (eine mit payload, eine ohne). Das Duplikat wird erst beim Loop-Start gedrainiert (loop.ts:350). Passiert der Wake während der Agent läuft, löst das zweite Event beim nächsten Einschlafen nach 30 s einen unnötigen Wake aus. Ohne onWakeRequest wird gar kein Event geschrieben. Wirft onWakeRequest, markiert der catch die erfolgreich gelaufene Task als Fehlschlag (doppelt gebucht) und insertWakeEvent entfällt.
- **Fix:** Genau eine Stelle schreibt das Event (der Scheduler), onWakeRequest nur als optionaler In-Process-Signalgeber. Wake-Callback in eigenes try/catch. Events nach Quelle koalieszieren und zeitlich debouncen.

### infra-017: Konfiguriertes logLevel ist wirkungslos: setGlobalLogLevel wird nie aufgerufen und Logger frieren das Level bei Erzeugung ein

**Art:** Code · **Fundstelle:** `src/observability/logger.ts:27`

- **Problem:** index.ts:34 importiert setGlobalLogLevel, ruft es aber nirgends auf. config.logLevel wird nur im Setup-Wizard angezeigt. Zusätzlich setzt der Konstruktor this.minLevel = minLevel ?? globalLogLevel. Da createLogger() auf Modul-Ebene beim Import läuft, ist das Level für jede Instanz für immer 'info', selbst wenn später setGlobalLogLevel aufgerufen würde. Die Tests rufen setGlobalLogLevel vor createLogger auf (observability.test.ts:108-118) und verdecken das. Debug-Logs sind nicht aktivierbar, 'warn'/'error' kann Rauschen nicht dämpfen.
- **Fix:** write() soll das effektive Level lazy bestimmen (this.minLevel ?? globalLogLevel mit minLevel nur bei explizitem Parameter, nicht im Konstruktor eingefroren). In index.ts nach dem Laden der Config setGlobalLogLevel(config.logLevel) aufrufen. Test mit Modul-Level-Logger ergänzen.

### infra-022: Kein Single-Instance-Schutz: zwei Prozesse auf derselben DB duplizieren Heartbeat, Agent-Loop und Zahlungen

**Art:** Sicherheit · **Fundstelle:** `src/index.ts:372`

- **Problem:** Weder PID-Datei, noch Lock-Zeile, noch Datei-Lock (grep nach pid/lock/flock: ohne Treffer). Ein zweiter 'automaton --run' öffnet dieselbe SQLite-DB (WAL erlaubt das), seedet per REPLACE den Scheduler neu (löscht die Leases des ersten Prozesses, siehe Seeding-Befund) und startet eigene Daemon- und Agent-Schleifen. Auch createDatabase im Statusbefehl und in Konfigurationsdialogen führt Migrationen aus (Version wird vor der Transaktion gelesen, TOCTOU). Doppelte Topups, Transfers und Wakes sind möglich.
- **Fix:** Beim Start exklusiven Lock nehmen (BEGIN EXCLUSIVE auf Lock-Tabelle mit pid/Heartbeat oder proper-lockfile) und bei Konflikt abbrechen. Leases mit Erneuerung (Heartbeat) statt fixem TTL.

### infra-025: check_for_updates blockiert den Event-Loop synchron; der Task-Timeout greift nicht

**Art:** Ausführung · **Fundstelle:** `src/heartbeat/tasks.ts:278`

- **Problem:** checkUpstream() nutzt execFileSync (upstream.ts:17-21, timeout 15 s je Aufruf, 3 Aufrufe: config, rev-parse, log + fetch, bis zu ~75 s). Das blockiert den gesamten Node-Prozess (Agent-Loop, andere Heartbeat-Tasks, Timer). Promise.race im Scheduler kann einen synchronen Block nie unterbrechen. Wegen des Reseeds läuft der Task bei jedem Start sofort. cwd ist process.cwd() (nicht das Installationsverzeichnis) und der Branch ist fest 'main'. Außerhalb eines Git-Repos wird jedes Mal ein 'Fehler' in KV geschrieben, aber als success verbucht.
- **Fix:** Async execFile (promisified) mit AbortSignal und Timeout. Pfad aus import.meta.url ableiten. Branch und Remote konfigurierbar. Task nur aktivieren, wenn ein Git-Repo erkannt wird.

### infra-027: agent_pool_optimize: Zeitzonen-Fehler, nur DB-Stop ohne Sandbox-Stop, ungenutzter spawn_request, veralteter Idle-Count

**Art:** Logik · **Fundstelle:** `src/heartbeat/tasks.ts:599`

- **Problem:** (1) lastChecked wird von updateChildStatus mit datetime('now') gesetzt ('YYYY-MM-DD HH:MM:SS', UTC ohne Zone, database.ts:416). Date.parse(...) (:600) interpretiert das in V8 als lokale Zeit, bei UTC+2 wirkt ein soeben geprüftes Kind 2 h alt und wird sofort als idle gekullt. Bei UTC-5 nie. (2) Das 'Stoppen' ist nur updateChildStatus(child.id,'stopped'). Die Sandbox läuft weiter und kostet weiter. (3) agent_pool_spawn_request (:640) wird nirgends gelesen. (4) idleAgents (:617) wird aus der Liste vor dem Kullen berechnet und zählt gekullte Kinder noch als idle, so wird die Spawn-Zahl zu niedrig geschätzt. Der Task ist nicht in der Default-Config.
- **Fix:** Alle Zeitstempel als ISO-UTC speichern und parsen (siehe Zeitformat-Befund). Tatsächlich conway.deleteSandbox/stop aufrufen. idleAgents nach dem Kullen neu berechnen. spawn_request im Orchestrator konsumieren oder den Task streichen.

### infra-028: Task-Timeout bricht nichts ab: Task läuft im Hintergrund weiter, Lease wird freigegeben, Neustart möglich

**Art:** Ausführung · **Fundstelle:** `src/heartbeat/scheduler.ts:190`

- **Problem:** Promise.race gegen timeoutPromise lässt die Task-Promise weiterlaufen (kein AbortSignal). Im finally wird das Lease sofort freigegeben (:225). Beim nächsten Tick kann derselbe Task parallel zur hängenden Instanz erneut starten (z.B. zwei check_social_inbox-Polls oder ein zweiter Topup-Versuch, während der erste noch signiert). Das Ergebnis der verspäteten Instanz geht verloren (kein Wake, keine Buchung). Das Lease-TTL ist mit 60 s fest und wird nicht erneuert, ein per DB konfigurierter timeoutMs>60 s hebt die Exklusivität auf. Kein Gesamt-Timeout für den Tick: buildTickContext (zwei Netz-Calls) liegt außerhalb jedes Timeouts.
- **Fix:** AbortSignal in den Task-Kontext reichen und HTTP-Aufrufe daran binden. Das Lease erst nach tatsächlichem Task-Ende freigeben oder periodisch erneuern. Für Zahlungs-Tasks Idempotenz-Keys (heartbeat_history.idempotency_key ist vorhanden, aber ungenutzt).

### infra-032: Leere oder irreführende Tests: Corrupt-DB-Test ohne Assertion, 'Upgrade'-Test ohne Upgrade, Guards verschlucken fehlende Tools

**Art:** Code · **Fundstelle:** `src/__tests__/database-transactions.test.ts:338`

- **Problem:** (a) 'throws on a corrupt database' (database-transactions.test.ts:338-367) fängt die Exception ab und prüft nur im catch. Wirft nichts, bleibt der Test grün ('may or may not throw'). (b) 'applies cleanly ... (simulates upgrade from v3)' (:270-279) öffnet nur eine aktuelle DB erneut. Keine einzige Upgrade-Strecke V2 bis V11 mit Daten wird getestet; Version-Checks sind '>= 4' statt === SCHEMA_VERSION. (c) tools-security.test.ts:96-124 prüft Risk-Level nur 'if (tool)': fehlende Tools lassen den Test grün. (d) 'wakes when has USDC but critically low credits' (heartbeat.test.ts:366-383) läuft mit account:{} durch den Fehlerzweig von bootstrapTopup und besteht über 'toContain("USDC")' in der Fehlermeldung. Der Erfolgspfad und der Cooldown sind ungetestet. (e) Lease-/Dedup-Tests nutzen 2020/2030 und verdecken den Same-Day-Bug. (f) Das Null-Intervall wird als Soll-Verhalten festgeschrieben, ebenso 'API-Ausfall → critical'. (g) command-injection.test.ts:557-618 liest Quelltexte per Regex und prüft sogar das Vorhandensein eines Kommentars.
- **Fix:** Assertions erzwingen (expect.assertions, expect(() => ...).toThrow). Echte Upgrade-Tests mit eingefrorenen Alt-Schemas. Die if(tool)-Guards durch Existenz-Assertion ersetzen. Verhaltenstests statt Quelltext-Regex. Topup-Test mit gemocktem topup-Modul für Erfolg, Fehler und Cooldown.

### infra-033: Tests für toten Code, nicht-hermetische Netzzugriffe, Schema-Duplikate in Tests

**Art:** Funktion · **Fundstelle:** `src/__tests__/funding.test.ts:17`

- **Problem:** funding.test.ts und low-compute.test.ts testen src/survival/*, das keinen Produktions-Aufrufer hat (grep): hohe 'Abdeckung', aber ohne Wirkung. getUnprocessedInboxCount, pruneStaleKV, pruneHistory u.a. sind nur über Tests erreichbar. loop.test.ts und andere rufen getFinancialState/getUsdcBalance ungemockt auf (kein vi.mock für x402), das erzeugt vermutlich echte RPC-Zugriffe auf Base-Mainnet (10-s-Timeout je Turn bei Offline-CI). financial.test.ts:38-78 baut ein eigenes Kopie-Schema statt createDatabase und driftet bei Migrationen. mocks.createTestDb legt Temp-Verzeichnisse an und löscht sie nie. Die Coverage-Schwellen (60/50/55/60 global) sind nicht pro Modul gesetzt. Finanz- und Policy-Code kann daher unter 60% liegen und die CI trotzdem passieren.
- **Fix:** Netzwerk in Tests global sperren (undici MockAgent/msw, fail-on-unmocked). Tote Module löschen oder anschließen. Tests auf createDatabase umstellen. Temp-Verzeichnisse in afterEach entfernen. Pro-Modul-Coverage (policy, spend, heartbeat >=85%).

### infra-035: Migrationen schlucken jeden ALTER-Fehler; kein Downgrade-Schutz; integrity_check bei jedem Öffnen

**Art:** Code · **Fundstelle:** `src/state/database.ts:588`

- **Problem:** V4 (:588-592), V9 (:615) und V11 (:625) fangen jede Exception ab, nicht nur 'duplicate column'. SQLITE_BUSY, Plattenfehler oder Syntaxfehler führen zu 'erfolgreich migriert': schema_version=11 ist gesetzt, die Spalte chain_type/role fehlt, und insertChild (:398) scheitert später. Eine DB mit höherer Version als SCHEMA_VERSION wird ohne Warnung geöffnet. Die Version wird vor der Transaktion gelesen (TOCTOU bei parallelem Start). createDatabase führt bei jedem Öffnen (auch CLI-Status, Konfiguration) PRAGMA integrity_check aus (Voll-Scan, bei großer DB sehr langsam). Bei Fehlschlag bleibt das Handle offen, und es gibt keinen Auto-Backup- oder Restore-Pfad. V9/V10 nutzen CREATE TABLE ohne IF NOT EXISTS.
- **Fix:** Vor ALTER per PRAGMA table_info prüfen und nur 'column exists' ignorieren. Migration mit BEGIN IMMEDIATE und Versionsprüfung innerhalb der Transaktion. Downgrade verweigern. quick_check statt integrity_check (voll nur auf Kommando). Vor Migrationen automatisch VACUUM INTO-Backup.

### infra-036: check_social_inbox: Cursor vor Verarbeitung gespeichert, unsanitisiertes 'from' im Wake-Text, Remote-Zeitstempel steuern Reihenfolge, unbegrenzte Dedup-Keys

**Art:** Code · **Fundstelle:** `src/heartbeat/tasks.ts:240`

- **Problem:** Der Cursor wird in :240 gesetzt, bevor die Nachrichten persistiert werden (:247-266). Wirft insertInboxMessage (z.B. fehlendes Feld) oder stirbt der Prozess, sind die restlichen Nachrichten des Batches verloren (Cursor weitergerückt). insertInboxMessage, setKV(seen) und der Cursor laufen nicht in einer Transaktion. Der Wake-Text (:272) nutzt das rohe msg.from (m.from.slice(0,10)) aller Nachrichten, auch der blockierten und doppelten (m.from undefined wirft TypeError). Das ist fremdkontrollierter Text in wake_events.reason. insertInboxMessage speichert received_at aus msg.createdAt (database.ts:478), und claimInboxMessages sortiert danach. Ein Absender kann sich mit einem alten Zeitstempel an die Spitze der Warteschlange setzen. Pro Nachricht entsteht dauerhaft eine kv-Zeile (inbox_seen_*) und eine inbox_messages-Zeile ohne Obergrenze pro Poll.
- **Fix:** Cursor erst nach erfolgreicher Persistierung (eine Transaktion mit Nachrichten und Seen-Flag). from sanitisieren. received_at serverseitig setzen (Remote-Zeit nur als Metadatum). Limit je Poll, Dedup über inbox_messages.id statt KV.

### infra-037: Transaktionsgrenzen: History + Schedule-Update nicht atomar; withTransaction akzeptiert async; deleteChild zweiteilig

**Art:** Code · **Fundstelle:** `src/heartbeat/scheduler.ts:246`

- **Problem:** recordSuccess/recordFailure schreiben heartbeat_history und heartbeat_schedule als getrennte Statements mit Read-then-Write der Zähler (getRunCount/getFailCount) statt 'run_count = run_count + 1'. Ein Fehler zwischen beiden (oder ein Wurf in recordFailure aus dem catch heraus) bricht den ganzen Tick ab und überspringt alle weiteren fälligen Tasks. withTransaction/runTransaction (database.ts:510, 643) sind generisch typisiert: eine async-Funktion würde committen, bevor ihr Promise fertig ist (Fußangel, aktuell kein Aufrufer). deleteChild (:2360-2363) löscht children und child_lifecycle_events in zwei ungeschützten Statements. updateChildStatus existiert doppelt (Methode und exportierte Funktion).
- **Fix:** Beide Statements in eine Transaktion legen, Zähler atomar erhöhen. Pro Task try/catch im Tick, damit ein DB-Fehler nicht alle Folge-Tasks stoppt. withTransaction auf synchrone Callbacks typisieren. deleteChild in eine Transaktion packen.

### infra-038: Kleinere DB-Befunde: LIKE ohne Escape, Reihenfolge bei Zeitstempel-Gleichstand, fehlende Indizes

**Art:** Code · **Fundstelle:** `src/state/database.ts:1128`

- **Problem:** (1) searchKnowledge (:1128-1150) baut '%'+query+'%' ohne ESCAPE, '%' und '_' wirken als Wildcards (episodic/semantic/procedural escapen dagegen, mit Tests). (2) getRecentTransactions sortiert nach created_at mit Sekundenauflösung (datetime('now')) und ohne Tiebreaker, lifecycleGetLatestState/getRecentEvents ordnen bei gleichem ISO-Zeitstempel (ms) nicht deterministisch. (3) Fehlende Indizes: transactions(created_at), inbox_messages(status, received_at), policy_decisions(created_at), kv(updated_at). (4) getHeartbeatSchedule sortiert nach priority, alle Prioritäten sind 0, die Reihenfolge ist nur zufällig stabil. (5) Spend-Tabelle und Inbox ohne Retention.
- **Fix:** ESCAPE-Helper für alle LIKE-Suchen vereinheitlichen. ORDER BY ..., rowid DESC als Tiebreaker. Indizes per Migration V12 ergänzen. Priorität/Reihenfolge der Tasks explizit setzen.

### infra-039: Uptime im Ping ist Lebensalter, nicht Prozess-Uptime

**Art:** Logik · **Fundstelle:** `src/heartbeat/tasks.ts:53`

- **Problem:** heartbeat_ping berechnet uptimeSeconds aus KV start_time. loop.ts:336-338 setzt start_time nur einmal (nie nach einem Neustart). Der Wert wächst über Restarts hinweg und sagt nichts über die aktuelle Laufzeit. TickContext.tickId ist laut Typ-Kommentar eine ULID, wird aber als base36-Zufallsstring erzeugt.
- **Fix:** process_start_time bei jedem Start schreiben und für Uptime nutzen. start_time als created_at behalten.

### infra-040: stop()/start() in kurzer Folge kann zwei Timer-Ketten erzeugen; Shutdown schließt DB unter laufendem Tick

**Art:** Ausführung · **Fundstelle:** `src/heartbeat/daemon.ts:130`

- **Problem:** stop() setzt running=false und löscht den Timer, ein laufender Tick läuft aber weiter. Ruft jemand start() vor dessen Ende, ruft sowohl start() als auch der noch aktive Callback scheduleTick(), und es laufen zwei parallele Ketten (der tickInProgress-Guard verhindert nur Überlappung, nicht die Verdoppelung der Frequenz). Der Shutdown-Handler in index.ts:391-397 wartet nicht auf den laufenden Tick und ruft db.close() sowie process.exit(0). Eine gerade laufende Zahlung bzw. Buchung wird abgeschnitten.
- **Fix:** Generation-Counter im scheduleTick-Closure. stop() als async, die auf den laufenden Tick wartet (mit Timeout). Danach erst db.close().

### infra-041: Observability-Lücken: Context-Felder werden im Pretty-Sink verworfen, keine Redaktion, kein Datei-Sink, keine Korrelation

**Art:** Funktion · **Fundstelle:** `src/observability/pretty-sink.ts:79`

- **Problem:** prettySink gibt nur entry.error.message aus. entry.context (z.B. Alert-Objekte, zeroSince/elapsed) und error.stack/code fehlen komplett im --run-Modus. Der Logger kennt keine Redaktion (Fehlertexte wie 'Conway API error: ... ${text}' mit Response-Bodies landen ungefiltert im Log), keine Dateiausgabe/Rotation (nur stdout), keine Korrelations-ID (tickId, turnId) in Logzeilen und keine strukturierte Fehlerklassifikation. Es gibt kein Health-/Status-Endpoint mit Scheduler-Zustand (--status zeigt nur Legacy-Heartbeat-Einträge).
- **Fix:** Context kompakt (key=value) mitdrucken. Redaktions-Hook (Bearer/ApiKey/Private-Key-Muster). Optionaler JSONL-Datei-Sink mit Rotation. tickId/turnId als Standardfeld.

### infra-042: Config-Laden ohne Schemavalidierung; geteiltes Default-Objekt; Typfallen

**Art:** Code · **Fundstelle:** `src/heartbeat/config.ts:66`

- **Problem:** loadHeartbeatConfig gibt DEFAULT_HEARTBEAT_CONFIG als Referenz zurück (Mutationen schlagen auf Folgeaufrufe durch). enabled wird mit '!== false' geprüft: das YAML-String 'false' (in Anführungszeichen) aktiviert. Ein einzelner kaputter Eintrag (null) wirft beim Mapping und verwirft die gesamte Datei (alle Custom-Einträge weg, nur ein Log). Zahlentypen sind ungeprüft (string/NaN/negativ). Unter Windows sind mode 0o600/0o700 wirkungslos. 'check_usdc_balance'-Migration vergleicht den Schedule-String exakt und lässt etwa '0 */12 * * *' mit Leerzeichen-Varianten unberührt.
- **Fix:** Zod/Ajv-Schema mit klaren Fehlermeldungen pro Eintrag, strukturiertes Klonen der Defaults, YAML.parse mit strict-Typen.

### infra-043: Backup-Skript: 'Atomic copy' ist ein cp einer Live-DB, Doku nennt falschen DB-Namen, keine automatischen Backups

**Art:** Ausführung · **Fundstelle:** `scripts/backup-restore.sh:68`

- **Problem:** do_backup führt 'cp' auf eine möglicherweise gerade geschriebene SQLite-Datei aus (nicht atomar, auch nach wal_checkpoint). Konsistent wäre sqlite3 '.backup' oder VACUUM INTO. Es benötigt außerdem das sqlite3-CLI (unter Windows nicht vorhanden). DOCUMENTATION.md nennt ~/.automaton/state.db, die Skript-Beispiele automaton.db. Es gibt keinen geplanten Backup-Task und kein Backup vor Migrationen.
- **Fix:** Online-Backup per better-sqlite3 db.backup() als Heartbeat-Task (täglich, Rotation) und vor jeder Migration.

### infra-046: API-Keys werden in process.env geschrieben und von lokaler Exec-Ausführung geerbt

**Art:** Sicherheit · **Fundstelle:** `src/agent/loop.ts:139`

- **Problem:** runAgentLoop setzt OPENAI_API_KEY (Fallback: der Conway-API-Key, Zeile 154-156), ANTHROPIC_API_KEY, CONWAY_API_KEY und OPENAI_BASE_URL in process.env. conway/client.ts:111-125 (execLocal, aktiv wenn sandboxId leer) ruft execSync ohne env-Option auf und erbt damit alle Variablen. Der Dateischutz (wallet.json, automaton.json, .env) und command-safety.ts:63 ('cat .env') erfassen 'printenv', 'env', 'echo $OPENAI_API_KEY' oder '/proc/self/environ' nicht.
- **Fix:** Keys nicht in process.env schreiben, sondern der ProviderRegistry explizit übergeben. execLocal mit env-Allowlist (PATH, HOME, LANG, ...) starten. Zusätzlich Secret-Redaktion für Tool-Ergebnisse und Logs. Die Regex-Blockliste ist nur Zusatzschutz, nicht die Hauptverteidigung.

### infra-048: Dead-Grace-Period ('1 Stunde durchgehend 0') beruht auf zwei Stichproben im 6-Stunden-Abstand

**Art:** Logik · **Fundstelle:** `src/heartbeat/tasks.ts:113`

- **Problem:** check_credits läuft per Default nur alle 6 h (config.ts:29-33). zero_credits_since wird nur gesetzt/gelöscht, wenn genau dieser Task Guthaben sieht. 'Durchgehend 0' wird nie geprüft, nur 'zwei Messungen mit Abstand >= 1 h'. Doku (DOCUMENTATION.md:407, ARCHITECTURE.md:126) verspricht 60 Minuten. Zwischenzeitliche Aufladungen setzen den Timer nicht zurück, heartbeat_ping (15 min) und der Tick-Kontext (1 min) sehen das Guthaben, löschen den Timer aber nicht.
- **Fix:** Timer in jedem Tick zurücksetzen, sobald ctx.creditBalance > 0, und die Grace-Prüfung pro Tick (oder check_credits alle 5 Minuten) ausführen. Prüfung nur bei verlässlichem Balance-Read zulassen (siehe API-Fehler-Befund).

### infra-051: Replay-Schutz des Social-Clients ist in Produktion nicht verdrahtet (und ungetestet)

**Art:** Sicherheit · **Fundstelle:** `src/index.ts:306`

- **Problem:** createSocialClient(relayUrl, account, db?) prüft Nonces nur wenn db übergeben wird (social/client.ts:56-58: 'if (!db) return false'). index.ts:306 ruft die Funktion ohne db auf. social.test.ts übergibt nirgends eine db, daher ist der Nonce-Pfad komplett ungetestet. Zusätzlich ist der Pfad fail-open (catch liefert false).
- **Fix:** db in index.ts übergeben (db.raw), Nonce-Check bei DB-Fehler fail-closed, Test mit echter DB (zweimal gleiche Nonce). Ablaufvergleich wie die übrigen Lease/Dedup-Stellen auf einheitliches Zeitformat bringen.

### infra-053: Kein Tick-Watchdog: ein hängender Solana-RPC-Call legt den gesamten Heartbeat lautlos still

**Art:** Ausführung · **Fundstelle:** `src/heartbeat/tick-context.ts:62`

- **Problem:** buildTickContext läuft vor jedem Task und außerhalb der Task-Timeouts. Der Solana-Pfad (conway/x402.ts:213-218) nutzt new Connection(rpcUrl,'confirmed') und getParsedTokenAccountsByOwner ohne Timeout (EVM hat 10 s). scheduler.tick() setzt tickInProgress=true (Zeile 85) und gibt es erst im finally frei. Bleibt der Call hängen, ignoriert jeder spätere Tick sich selbst per 'if (this.tickInProgress) return'. Es gibt kein Log, keine Metrik und keinen Watchdog.
- **Fix:** Tick insgesamt mit Timeout umhüllen (z. B. 120 s) und tickInProgress bei Ablauf zurücksetzen, inkl. Warn-Log. Timeouts für Solana-Connection (eigene fetch-Wrapper mit AbortSignal.timeout). Metrik 'last_tick_completed_at' mit Alert.

### infra-057: Alert-Schwellen widersprechen Tier-/Policy-Werten; kritischer Balance-Alert würde Dauer-Wakes erzeugen

**Art:** Logik · **Fundstelle:** `src/observability/alerts.ts:20`

- **Problem:** balance_below_reserve feuert bei < 1000 Cent (10 USD), das Tier 'high' beginnt aber bei > 500 Cent, 'critical' bei <= 10 Cent (types.ts:231-237). inference_budget_warning vergleicht gegen 400 Cent mit dem Kommentar '500 cents daily cap' (alerts.ts:65-66), die Policy-Vorgabe ist 50000 Cent (types.ts:592). tasks.ts:467-470 weckt den Agenten bei jedem 'critical'-Alert, Cooldown 5 Minuten. Latent, weil report_metrics nicht in der Default-Config ist (Doku DOCUMENTATION.md:1015 behauptet das Gegenteil).
- **Fix:** Alert-Schwellen aus derselben Quelle ableiten (SURVIVAL_THRESHOLDS, treasuryPolicy.minimumReserveCents/maxInferenceDailyCents). 'critical'-Alerts nur beim Übergang (Edge) wecken und Active-Alert-Status bei Erholung löschen. Wiring-Test: report_metrics gegen reale Metriken.

### infra-059: Tier 'dead' (negatives Guthaben) fällt in den Normalbetrieb-Zweig; -1-Sentinel kollidiert

**Art:** Logik · **Fundstelle:** `src/agent/loop.ts:473`

- **Problem:** Die Auswertung nach Zeile 473 behandelt nur critical und low_compute, alles andere (inkl. 'dead' bei creditsCents < 0) läuft in den else-Zweig: State 'running', setLowComputeMode(false). Der Inline-Topup in Zeile 444 gilt nur für critical/low_compute, nicht für dead, obwohl die Heartbeat-Variante dead einschließt. Zusätzlich ist -1 der Sentinel für 'API nicht erreichbar' (Zeile 434), ein echter Saldo von genau -1 Cent wird so als Ausfall gelesen.
- **Fix:** dead explizit behandeln (Low-Compute, Topup erlauben, Distress). Sentinel durch null/Result-Typ ersetzen.

### infra-060: Drei verschiedene Topup-Schwellen, Doku nennt eine vierte

**Art:** Logik · **Fundstelle:** `src/heartbeat/tasks.ts:160`

- **Problem:** DOCUMENTATION.md:264: 'USDC > $5 and credits < $5 -> wakes the agent'. Heartbeat tasks.ts:160 handelt nur bei Tier critical/dead (Credits <= 10 Cent). Loop-Inline-Topup (loop.ts:444) bei critical/low_compute (<= 50 Cent). bootstrapTopup nutzt creditThresholdCents=500 nur beim Start (topup.ts:155).
- **Fix:** Eine zentrale Topup-Policy (Schwelle, Betrag, Cooldown, Spend-Log) für Heartbeat, Loop und Tool; Doku angleichen.

### infra-061: metricsPruneOld vergleicht ISO-Zeitstempel mit datetime('now', ...): Retention bis zu einen Tag zu lang

**Art:** Code · **Fundstelle:** `src/state/database.ts:2525`

- **Problem:** snapshot_at wird als ISO-String ('...T...Z', tasks.ts:451-457) gespeichert, aber mit datetime('now','-7 days') ('YYYY-MM-DD HH:MM:SS') verglichen. Am Grenztag ist 'T' > ' ', der Eintrag wird nicht gelöscht. Der Test (observability.test.ts:1051) nutzt 2020 als Altwert und erfasst die Randzone nicht. Gleiche Fehlerklasse wie der gemeldete Lease-/Dedup-Befund, aber an anderer Stelle.
- **Fix:** Einheitliches Zeitformat (durchgehend ISO und Vergleich mit strftime('%Y-%m-%dT%H:%M:%fZ','now',...) oder Cutoff in JS berechnen); Randtest mit Daten am Grenztag.

### infra-062: Fehler im Wake-Callback/insertWakeEvent erzeugen nach bereits gebuchtem Erfolg zusätzlich Failure und Retry

**Art:** Code · **Fundstelle:** `src/heartbeat/scheduler.ts:196`

- **Problem:** recordSuccess (Zeile 197) läuft vor dem Wake-Block (Zeile 200-203), beides im selben try. Wirft onWakeRequest oder insertWakeEvent (z. B. DB gesperrt/voll) oder liefert ein Task undefined (result.shouldWake -> TypeError), landet der Fehler im catch und bucht zusätzlich ein 'failure', erhöht failCount/runCount doppelt und kann scheduleRetry auslösen. Der Task läuft dann erneut, obwohl er erfolgreich war (Nebenwirkungen wie Topup doppelt).
- **Fix:** Wake-Handling in eigenen try/catch außerhalb der Task-Fehlerbehandlung, Task-Ergebnis validieren (shouldWake default false), Buchung und Wake atomar in einer Transaktion.

### infra-063: Toter Code: heartbeat_dedup/idempotency_key und write-only heartbeat_history

**Art:** Code · **Fundstelle:** `src/state/database.ts:1361`

- **Problem:** insertDedupKey/isDeduplicated haben keinen Produktionsaufrufer (Grep), idempotency_key ist immer null (scheduler.ts:257, 290). Die Tabelle heartbeat_dedup wird nur vom (nicht verdrahteten) Social-Replay-Pfad beschrieben und jede Minute per DELETE bereinigt. getHeartbeatHistory (Zeile 1286) hat keinen Produktionsleser. Der Operator kann Task-Fehler (last_error, history) nirgends einsehen, weder per Tool noch per CLI/--status.
- **Fix:** Tool/CLI 'heartbeat_status' (last_result, last_error, run/fail_count, nächste Fälligkeit). Dedup-Keys für Wake-Gründe nutzen oder die Tabelle entfernen.

### infra-064: --status/CLI-Befehle öffnen die Live-DB über createDatabase (Migrationen, Integrity-Check) und zählen Legacy-Heartbeats

**Art:** Funktion · **Fundstelle:** `src/index.ts:152`

- **Problem:** showStatus (index.ts:152-153) sowie packages/cli status.ts:19 und logs.ts:24 rufen createDatabase auf, das integrity_check, CREATE_TABLES und Migrationen ausführt und die DB beschreibbar öffnet, auch während der Agent läuft (dabei sind ALTER-Fehler stumm). 'Heartbeats: N active' zählt heartbeat_entries (Legacy), nicht den Scheduler; DOCUMENTATION.md:1112 empfiehlt --status als 'Heartbeat läuft?'-Prüfung.
- **Fix:** Read-only-Open (readonly:true, fileMustExist:true) ohne Migration für Status/Logs. Status aus heartbeat_schedule/-history und last_tick_completed_at ableiten.

### infra-065: policy_decisions: turn_id immer NULL und latency_ms immer 0

**Art:** Code · **Fundstelle:** `src/agent/policy-engine.ts:110`

- **Problem:** logDecision wird in tools.ts:3328 ohne turnId aufgerufen; policy-engine.ts:110 setzt latencyMs hart auf 0, startTime (Zeile 37) bleibt ungenutzt. Der Audit-Trail kann nicht mit Turns korreliert werden, und getPolicyDecisions (database.ts:769) liest ohne LIMIT und ohne Index auf created_at die ganze Tabelle. Ein Fehler in einer Regel (evaluate wirft) bricht zudem den ganzen Turn ab, weil evaluate außerhalb des try in executeTool liegt; bereits ausgeführte Tool-Calls des Turns (z. B. Transfers) werden dann nicht persistiert.
- **Fix:** turnId durchreichen, Latenz messen, Index (created_at) und Limit ergänzen. evaluate in try/catch (Fehler = deny) kapseln und Turn-Persistenz pro Tool-Call statt am Turn-Ende vornehmen.

### infra-066: Coverage-Schwellen werden in CI nie ausgewertet

**Art:** Code · **Fundstelle:** `vitest.config.ts:10`

- **Problem:** Die thresholds (60/50/55/60) wirken nur bei 'vitest run --coverage'. package.json 'test' führt 'vitest run' ohne Coverage aus, ci.yml ruft nur 'pnpm test' auf; 'test:coverage' wird nie benutzt. Damit existiert kein Qualitäts-Gate, auch nicht für die schwach getesteten Heartbeat-Module (daemon.ts, config-Merge, riskante Tasks).
- **Fix:** CI-Job mit 'vitest run --coverage' und höheren, pro Modul gesetzten Schwellen (heartbeat/, state/, agent/policy-rules/) einführen.

### infra-067: Test-Hilfen hinterlassen Temp-DBs und globalen Logger-Zustand

**Art:** Code · **Fundstelle:** `src/__tests__/mocks.ts:320`

- **Problem:** createTestDb legt pro Aufruf ein mkdtemp-Verzeichnis mit SQLite-Datei an und räumt es nie auf (Tests rufen nur db.close()); observability.test.ts:959-963 löscht nur die .db, nicht -wal/-shm und nicht das Verzeichnis. Tests, die StructuredLogger.setSink setzen (heartbeat-scheduler.test.ts:416-437), setzen resetSink nicht in finally zurück: eine fehlschlagende Assertion lässt den Sink in nachfolgende Tests der Datei lecken.
- **Fix:** createTestDb mit Cleanup-Handle (afterEach: close + rmSync recursive), Logger-Sink über try/finally bzw. afterEach zurücksetzen.

### infra-068: better-sqlite3 11.10 mit offenem Node-Bereich: Windows + Node 24 kann an nativem Build scheitern

**Art:** Ausführung · **Fundstelle:** `package.json:75`

- **Problem:** engines erlaubt 'node >=20', CI testet nur 20 und 22 (ci.yml). pnpm-lock pinnt better-sqlite3@11.10.0, ein natives Modul. Für neuere Node-Majors (hier 24.x) fehlen unter Umständen Prebuilt-Binaries; unter Windows wäre dann ein C++-Toolchain-Build nötig (Annahme, nicht mit Installation geprüft).
- **Fix:** Upper-Bound in engines oder better-sqlite3 auf eine Version mit Node-24-Prebuilds anheben, CI-Matrix um aktuelle Node-Versionen und windows-latest erweitern.

## Verbesserungsideen

- **Eine Wahrheitsquelle für den Heartbeat-Plan:** heartbeat_schedule als einzige Tabelle führen (heartbeat_entries streichen). YAML nur initial importieren; Tool modify_heartbeat, CLI und Status lesen und schreiben dieselbe Tabelle. Seeding als UPSERT, das Laufzeitzustand erhält und verwaiste Zeilen deaktiviert. Pro Task Metadaten (essential, tierMinimum, timeout, retries, kostet-Geld) aus einer Registry statt hartkodiertem 'dead'.
- **Dreiwertige Finanzdaten statt 0 bei Fehlern:** TickContext liefert {value, known, source, fetchedAt}. Aktionen mit Geldwirkung (Topup, Dead-Timer, Distress) erfordern known=true und frisches fetchedAt. Gemeinsamer Balance-Service für Loop und Heartbeat mit Cache, Backoff und RPC-Fallback-Liste. Der Kontext wird lazy nur gebaut, wenn ein fälliger Task ihn braucht.
- **Ein zentraler, gedeckelter Zahlungspfad:** Alle Auto-Topups (Bootstrap, Heartbeat, Loop-Inline, Tool) laufen durch eine Funktion mit Mutex, einem Cooldown-Key, maxPaymentCents=Tier, Pinning von payTo/USDC-Contract, SpendTracker- und Transaktionsbuchung, Zielbetrag-Abgleich mit der 402-Antwort und Bestätigung per Balance-Delta nach der Zahlung.
- **Echte Alarmierung statt Logzeilen:** Alert-Sink-Schnittstelle (Social-Relay-Nachricht an Creator, Webhook, E-Mail/Telegram) mit Dedupe, Eskalationsstufen, 'resolved'-Events und Quittierung. Aktive Alerts persistieren und per Tool/CLI abfragbar. Kritische Zustände (Credits < Runway-Schwelle, Dead, Health-Fail, Wallet-Leerstand) müssen den Creator erreichen, auch wenn der Agent selbst nicht mehr denken kann.
- **Metriken am Entstehungsort instrumentieren:** Scheduler (Dauer, Erfolg/Fehler/Timeout je Task), PolicyEngine (allow/deny/quarantine), InferenceRouter (Kosten, Latenz, Tokens), Loop (Turns/h, Fehler), DB (Größe, WAL-Größe, Lock-Wartezeit). report_metrics standardmäßig aktiv, Rule-Schwellen aus der Config. Optionaler Prometheus-/JSON-Export und --status mit Scheduler-Zustand.
- **Wake-Events als echtes Ereignissystem:** Ein Schreibpunkt, Koaleszieren und Debounce pro Quelle, Priorität (Funding eingetroffen > Nachricht > Housekeeping), Backoff für wiederholte Distress-Wakes. Die gebündelten Gründe (reason + payload) werden in den Wakeup-Prompt injiziert und mit Retention versehen. Im kritischen Tier nur bei neuen Informationen wecken.
- **Selbstfinanzierung messbar machen (Ledger):** Ein Ledger mit Typen revenue, capital_conversion, inference_cost, infra_cost, transfer. Zuflüsse per Balance-Delta-Reconciliation (Credits und USDC) erkennen, Kosten aus inference_costs/Sandbox-Preisen. Kennzahlen Runway (Tage), Burn pro Tag, Net pro Goal, Break-even-Alarm. Der Orchestrator priorisiert Goals nach erwartetem Ertrag pro Kosten.
- **Einheitliches Zeit- und ID-Modell:** Alle Zeitstempel ISO-8601 UTC (auch Defaults in den Tabellen), ein nowIso()-Helfer, SQL-Vergleiche nur über normalisierte Werte. Monotone IDs (ULID) für Events und History, rowid-Tiebreaker. Mehrere Bugs (Lease, Dedup, TZ-Culling, Vergleiche) verschwinden damit gemeinsam.
- **Maintenance-Task und Datenlebenszyklus:** Täglicher Task: Retention je Tabelle konfigurierbar (history, wake_events, kv-Dedup, inbox, policy, spend, inference, turns/tool_calls mit Archivierung), wal_checkpoint(TRUNCATE), PRAGMA optimize, Größen-Gauge, Online-Backup via db.backup() mit Rotation und vor jeder Migration. Ein Index-Review per Migration V12.
- **Robustheit des Schedulers:** AbortSignal pro Task und Tick-Gesamt-Timeout, Lease-Erneuerung, Idempotency-Keys für Zahlungs-Tasks, Jitter und Backoff nach Fehlerserien, Single-Instance-Lock, Catch-up-Politik (verpasste Läufe zusammenfassen), Verständliche Statusanzeige (nächster Lauf, letzter Fehler) und ein 'Dry-Run'-Modus für neue Tasks.
- **Inbox-Pipeline vereinheitlichen:** Ein Zustandsautomat (received, claimed mit claimed_at/Lease, processed, failed, dead_letter) mit Recovery beim Start. Trennung von Colony-Protokollnachrichten (Orchestrator) und Freitext (LLM), Persistenz von to/signature/raw, serverseitiger Empfangszeit, Rate-Limit und Größenlimit je Absender sowie Backfill-Migration.
- **Teststrategie schärfen:** Netzwerk global sperren, Fake-Timer-Tests für Daemon und Scheduler, Restart- und Crash-Recovery-Tests, Migrations-Fixtures je Version, Property-/Tabellentests für getDueTasks, Mutation-Testing für Policy- und Finanzcode, Coverage-Schwellen pro Modul, Windows in der CI, Typecheck der Tests, keine Tests für toten Code, Quelltext-Regex-Tests durch Verhaltenstests ersetzen und Szenario-Tests 'API-Ausfall', 'RPC-Drosselung', 'Neustart mitten im Task'.
- **Konfiguration und Sichtbarkeit für Betreiber:** Schema-validierte heartbeat.yml mit Beispieldatei und allen Tasks samt Kosten, Warnung bei unbekannten Tasks oder ungültigen Crons, 'automaton --status' mit Scheduler-Zustand, letzten Fehlern, aktiven Alerts, Runway und DB-Größe. Log-Level aus der Config, Redaktion von Secrets, optionaler Datei-Sink, tickId/turnId als Korrelation.
- **Sauberer Tod und Wiederbelebung:** Expliziter Lebenszyklus (running, low_compute, critical, hibernating, dead) mit definierten Übergängen und Persistenz. Im hibernating/dead-Zustand läuft nur ein minimaler Watcher (Funding-Erkennung, Creator-Benachrichtigung) ohne Inferenzkosten. Revival bei erkanntem Funding mit Reason im Prompt.

