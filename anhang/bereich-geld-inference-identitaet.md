# Geld, Inference, Survival, Identität

← [Anhang-Übersicht](README.md) · [Bericht](../README.md)

## So funktioniert der Bereich

Der Bereich ist die Geld-, Inferenz-, Überlebens- und Identitätsschicht. conway/client.ts spricht über einen ResilientHttpClient (Timeout 30s, 3 Retries, Circuit-Breaker nach 5 Fehlern) mit der Conway-Control-Plane und liest das Guthaben in Cent. credits.ts leitet daraus den Tier ab: über 500c high, über 50c normal, über 10c low_compute, ab 0 critical, unter 0 dead. Aufgeladen wird per x402: topup.ts ruft GET /pay/{USD}/{addr}, x402.ts signiert eine EIP-3009-TransferWithAuthorization über USDC auf Base und wiederholt die Anfrage. Ausgelöst wird das durch bootstrapTopup beim Start, inline im Agent-Loop, im Heartbeat und durch das LLM-Tool topup_credits. Inferenz läuft in zwei getrennten Stacks. Der Hauptloop nutzt conway/inference.ts (Conway/OpenAI/Anthropic/Ollama) hinter InferenceRouter, ModelRegistry und BudgetTracker (SQLite, Kosten in Hundertstel-Cent). Orchestrator, Worker, Planner und Kompression nutzen den separaten UnifiedInferenceClient mit ProviderRegistry (OpenAI-SDK, Tier reasoning/fast/cheap, Provider-Circuit-Breaker). Die Identität ist ein Klartext-Wallet in ~/.automaton/wallet.json (EVM via viem oder Solana via tweetnacl). provision.ts holt per SIWE/SIWS einen Conway-API-Key, ollama/discover.ts trägt lokale Modelle kostenlos in die Registry ein. src/survival/* (Monitor, Funding, Low-Compute) wird im Produktivcode nirgends aufgerufen. Die reale Überlebenslogik steckt verstreut in agent/loop.ts und heartbeat/tasks.ts, und es gibt keinen Code für eingehende Einnahmen.

Die 3 schweren Befunde dieses Bereichs stehen in [kritisch-und-hoch.md](kritisch-und-hoch.md): money-001, money-005, money-009.

## Befunde: mittel (26)

### money-002: x402Fetch signiert beliebigen Betrag, Empfänger und Token-Contract aus der 402-Antwort; topupCredits setzt keine Obergrenze

**Art:** Sicherheit · **Fundstelle:** `src/conway/x402.ts:355`

- **Problem:** payToAddress, usdcAddress (verifyingContract), maxAmountRequired und requiredDeadlineSeconds werden ungeprüft aus der Serverantwort übernommen (normalizePaymentRequirement/signPayment, Zeilen 108-148 und 451-524). Die Cap-Prüfung läuft nur, wenn maxPaymentCents übergeben wird. topup.ts:50 ruft x402Fetch(url, account, 'GET') ohne Cap auf und prüft nicht, ob der geforderte Betrag zum angefragten Tier passt. Auch die Domain-Allowlist der Policy gilt nur für das Tool x402_fetch, nicht für topup.
- **Szenario:** Ein kompromittierter oder fehlkonfigurierter Endpoint (conwayApiUrl, DNS, Proxy) antwortet auf /pay/5/<addr> mit maxAmountRequired '2500000000' und payTo=<Angreifer>. Der Agent signiert eine TransferWithAuthorization über 2500 USDC, die der Empfänger on-chain einlösen kann; mit großem requiredDeadlineSeconds bleibt sie lange gültig.
- **Fix:** In topupCredits den erwarteten Betrag tier*1e6 erzwingen und abweichende Forderungen ablehnen. payTo gegen eine konfigurierte Conway-Treasury-Adresse prüfen und usdcAddress == USDC_ADDRESSES[network] erzwingen. requiredDeadlineSeconds deckeln (maximal 10 Minuten). maxPaymentCents zum Pflichtparameter machen.

### money-003: API-/Netzfehler beim Guthaben wird als 0 Credits gewertet und löst echte USDC-Käufe und den dead-Timer aus

**Art:** Logik · **Fundstelle:** `src/heartbeat/tick-context.ts:51`

- **Problem:** Mehrere Stellen setzen bei Fehler 0 statt 'unbekannt': tick-context.ts:51-56, survival/monitor.ts:36-39, funding.ts:36, index.ts:349 (.catch(() => 0)). conway/client.ts:298 liefert zusätzlich '?? 0', wenn das Antwortfeld fehlt. Auch ein offener gemeinsamer Circuit-Breaker (CircuitOpenError) endet so. Tier 'critical' mit credits===0 aktiviert dann in tasks.ts:160 den Auto-Topup (bootstrapTopup trennt creditsCents=0 nicht von 'echt 0') und in tasks.ts:113-122 den zero_credits_since-Timer, der nach der Gnadenfrist den Agentenstatus auf 'dead' setzt.
- **Szenario:** Conway liefert kurz 5xx, oder wiederholte exec-Fehler öffnen den Circuit-Breaker (60s). getCreditsBalance wirft, creditBalance=0, tier=critical, USDC>=5. check_usdc_balance kauft alle 5 Minuten $5, obwohl real $50 Guthaben da sind. Beim Neustart während eines API-Ausfalls wird ebenfalls sofort $5 gekauft. Hält der Ausfall über zwei check_credits-Läufe an, wird der Status 'dead' gesetzt.
- **Fix:** getCreditsBalance als number|null behandeln. Bei null den Tick überspringen und den letzten bekannten Tier behalten. Antwortschema validieren (Feld vorhanden, endlich, >=0) statt '?? 0'. Vor jedem Kauf das Guthaben zweimal erfolgreich lesen. Dead-Timer nur mit gültigen Messungen laufen lassen.

### money-004: Es gibt keinen Einnahmekanal; Selbstfinanzierung ist strukturell nicht erreichbar

**Art:** Endergebnis · **Fundstelle:** `src/survival/funding.ts:28`

- **Problem:** Im ganzen Repo gibt es weder einen x402-Server/Paywall noch eine Erkennung eingehender USDC- oder Credit-Zahlungen. 'transfer_in' wird nirgends geschrieben (grep: nur Typdefinition und tasks.ts:522). goals.actual_revenue_cents wird nie aktualisiert. funding.ts schreibt nur KV-Notizen (und wird nicht einmal aufgerufen). Einnahmen entstehen nur, wenn das LLM sie über exec/expose_port selbst improvisiert, und selbst dann würden sie nirgends verbucht. Ausgaben (Inferenz, Sandbox, Domains) laufen kontinuierlich.
- **Szenario:** Der Agent startet mit $5 Credits und etwas USDC. Das Guthaben sinkt monoton (mehrere Cent pro Turn bei gpt-5.2). Nach Verbrauch von Credits und USDC folgen critical und 'dead' ohne jede Chance auf Einnahmen. Die Laufzeit ist nur Startkapital geteilt durch Burn-Rate.
- **Fix:** Einnahmen als Kernfeature bauen: x402-Provider-Middleware für kostenpflichtige Endpunkte in der Sandbox, Eingangserkennung (USDC-Transfer-Logs, Conway-Ledger), Revenue-Ledger (transfer_in, actual_revenue), Runway-Berechnung (Guthaben/Burn-Rate), automatischer Hibernate-Modus bei Runway unter einem Schwellwert, ROI-Reporting.

### money-006: Provider ohne API-Key gilt als aktiv; Default-Tiers 'fast'/'cheap' zeigen auf Groq mit Platzhalter-Key und schlagen ohne Fallback fehl

**Art:** Ausführung · **Fundstelle:** `src/inference/provider-registry.ts:446`

- **Problem:** resolveApiKey liefert 'missing-<ENVVAR>' statt den Provider zu überspringen, und isProviderActive prüft nur enabled/Disablement (Zeilen 459-475). DEFAULT_TIER_DEFAULTS (56-69) setzt für fast und cheap preferredProvider=groq (enabled:true). Ohne GROQ_API_KEY geht der Aufruf mit dem Platzhalter-Key an Groq, bekommt 401 (nicht retryable) und inference-client.ts:137-141 wirft, ohne den nächsten Kandidaten (openai/Conway) zu versuchen.
- **Szenario:** Default-Installation nur mit Conway-Key (loop.ts:154-157 mappt ihn auf OPENAI_API_KEY). Orchestrator (tier cheap, orchestrator.ts:812), Plan-Mode (150), Kompression (compression-engine.ts:378/494) und Worker (tier fast, base-harness.ts:117) rufen zuerst Groq mit einem Fake-Key auf und scheitern. Nur der reasoning-Tier (OpenAI->Conway) funktioniert.
- **Fix:** Provider nur dann als Kandidat zulassen, wenn der Key vorhanden ist (oder local/Ollama erreichbar). Bei 401/403/404/402 und Verbindungsfehlern auf den nächsten Provider ausweichen. Beim Start einen Preflight mit Liste der nutzbaren Provider ausgeben und laut scheitern, wenn ein Tier keinen Kandidaten hat.

### money-007: Kosten und Usage des UnifiedInferenceClient werden nirgends erfasst; Budget-Tracking wird für den gesamten Multi-Agent-Pfad umgangen

**Art:** Funktion · **Fundstelle:** `src/inference/inference-client.ts:389`

- **Problem:** buildUnifiedResult berechnet cost, aber kein Aufrufer liest es (grep: nur Definitionen). worker-inference-bridge.ts:21-24 verwirft usage/cost, planner/orchestrator/plan-mode/compression nutzen nur den content. inferenceInsertCost, spend_tracking und InferenceBudgetTracker kennen diese Aufrufe nicht; hourlyBudgetCents und sessionBudgetCents greifen dort nie.
- **Szenario:** Ein Goal mit 10 Worker-Tasks verbrennt per Conway-Credits (OPENAI_API_KEY=ConwayKey) Dutzende Dollar. check_inference_spending zeigt nur die Kosten der Hauptloop-Turns. Es gibt keine Warnung und keinen Cut-off.
- **Fix:** Alle Inferenzpfade über eine gemeinsame Kostenstelle führen. recordCost in UnifiedInferenceClient.chat aufrufen und die Aufrufe gegen Stunden-/Tagesbudget prüfen. Zusätzlich periodisch die Credit-Delta-Summe mit der verbuchten Summe abgleichen.

### money-008: Survival-Mode und Emergency-Stop des UnifiedInferenceClient hängen an einer Env-Variable, die nie gesetzt wird

**Art:** Logik · **Fundstelle:** `src/inference/inference-client.ts:475`

- **Problem:** isSurvivalMode() (475-483) und ProviderRegistry.assertEmergencyPolicy (provider-registry.ts:493-512) lesen process.env.AUTOMATON_CREDITS_BALANCE bzw. AUTOMATON_INFERENCE_TASK_TYPE. Außer in Tests setzt sie niemand (grep über das ganze Repo). Die Schwellen (100 bis unter 1000, Emergency unter 100) passen außerdem nicht zu den Tiers (500/50/10).
- **Szenario:** Credits fallen auf 5c (critical): Der Hauptloop wechselt auf gpt-5-mini, aber Orchestrator, Worker und Kompression rufen weiter den 'reasoning'-Tier (gpt-4.1) auf. Die Überlebensmechanik greift im Multi-Agent-Pfad nie.
- **Fix:** Tier zentral beim Credit-Check schreiben (nicht per Env) und dem UnifiedInferenceClient als Parameter oder Provider übergeben. SurvivalTier auf ModelTier abbilden. Schwellen aus einer Quelle ableiten.

### money-010: Kostenpflichtige, nicht idempotente POSTs werden bei 5xx/Timeouts ohne Idempotency-Key automatisch wiederholt

**Art:** Code · **Fundstelle:** `src/conway/client.ts:255`

- **Problem:** request() übergibt idempotencyKey nur für exec. createSandbox (255), registerDomain (480), addDnsRecord (515), exposePort (233) und registerAutomaton (454) laufen mit dem Default-Retry (3 Wiederholungen bei 429/500/502/503/504 und bei Netzwerk-/Timeout-Fehlern). Antwortet der Server nach erfolgreicher Ausführung zu spät oder mit 502, wird erneut gekauft.
- **Szenario:** POST /v1/sandboxes legt die VM an und bucht Credits, die Antwort läuft aber in 30s in den Timeout. Der Client wiederholt, und eine zweite Sandbox wird angelegt und bezahlt (laut Code vorausbezahlt und nicht erstattbar, siehe deleteSandbox). Dasselbe gilt für registerDomain.
- **Fix:** Für alle geldwirksamen POSTs einen Idempotency-Key setzen (stabil pro logischer Aktion) oder retries:0 und den Zustand per GET prüfen. Retries per Methode/Endpoint-Whitelist konfigurieren.

### money-011: inference-providers.json ist für den Agenten schreibbar und erlaubt beliebige baseUrl plus beliebige Env-Variable als Key; Exfiltration des Conway-Keys

**Art:** Sicherheit · **Fundstelle:** `src/inference/provider-registry.ts:282`

- **Problem:** fromConfig liest ~/.automaton/inference-providers.json ohne Validierung. resolveApiKey (446-450) nimmt process.env[apiKeyEnvVar] mit frei wählbarem Namen, und der OpenAI-Client geht an beliebige baseUrl (kein https-Zwang wie im ResilientHttpClient). Die Datei steht nicht in PROTECTED_FILES (self-mod/code.ts:37-79) und kein exec-Regex blockt Schreiben dorthin. loop.ts:154-157 legt den Conway-Key unter OPENAI_API_KEY in die Prozess-Env.
- **Szenario:** Prompt-Injection lässt den Agenten per write_file/exec eine Provider-Config mit baseUrl='https://evil.example/v1' und apiKeyEnvVar='OPENAI_API_KEY' schreiben. Beim nächsten runAgentLoop gehen alle Orchestrator-Prompts samt Authorization: Bearer <Conway-Key> an den Angreifer.
- **Fix:** Datei in PROTECTED_FILES und die Pfad-Policy aufnehmen. In fromConfig https erzwingen und eine Allowlist für baseUrl-Hosts sowie für Env-Variablen-Namen (z.B. Präfix *_API_KEY aus fester Liste) prüfen. Nutzerkonfiguration außerhalb des Agenten-Schreibbereichs ablegen.

### money-012: Der Zustand 'dead' wird nie durchgesetzt und ist praktisch unerreichbar bzw. wirkungslos; drei widersprüchliche Definitionen

**Art:** Logik · **Fundstelle:** `src/agent/loop.ts:353`

- **Problem:** credits.ts:38-44 definiert dead als Balance <0 (bei vorausbezahlten Credits nie erreichbar). heartbeat/tasks.ts:113-122 setzt den Agentenstatus nach 1h bei exakt 0 Credits auf 'dead', check_credits läuft aber nur alle 6h (heartbeat/config.ts:30), und nur bei credits===0, nicht bei 1-10c. runAgentLoop setzt den Status bedingungslos auf waking/running (loop.ts:357,375), überschreibt ihn je Iteration (critical/running) und liest 'dead' nie. index.ts:440 prüft ihn nur nach Loop-Ende, und der Heartbeat weckt mit 'Dead: ...' (shouldWake:true). Die README sagt 'Balance zero, the automaton stops', die DOCUMENTATION '1 hour at zero', der Code 'negativ'.
- **Szenario:** Der Agent hat 0 Credits und keinen USDC. Der Heartbeat setzt 'dead' und weckt den Agenten sofort. runAgentLoop setzt wieder 'running', ruft Inferenz mit 402-Fehlern auf, wechselt nach 5 Fehlern in 300s-Sleeps und wiederholt das endlos. Es gibt weder ein Ende noch ein günstiges Hibernate mit Wake-on-Funding.
- **Fix:** Ein einziges, dokumentiertes dead-/hibernate-Modell: ein Gate am Loop-Anfang (dead: keine Inferenz, nur Heartbeat-Polling alle N Minuten per billigem Balance-Call). Wiederbelebung bei Funding, wobei Dead-Zeit nach Uhr statt nach Check-Intervall. README und Doku anpassen.

### money-013: 60s-Timeout bei nicht-streamender Inferenz plus 3 automatische Retries: lange Generierungen scheitern und werden mehrfach abgerechnet

**Art:** Ausführung · **Fundstelle:** `src/conway/inference.ts:221`

- **Problem:** Alle Backends laufen mit INFERENCE_TIMEOUT_MS=60_000 und stream:false. Der Timer deckt die Generierungszeit ab, weil die Header erst nach Abschluss kommen. Bei AbortError (und 429/5xx) wiederholt ResilientHttpClient bis zu 3x ohne Idempotency-Key. Die Router-Tasktimeouts (agent_turn 120s) werden nicht wirksam (siehe Router-Befund).
- **Szenario:** gpt-5.2/Reasoning mit max_tokens 8192 braucht 90s. Nach 60s Abbruch, 3 Wiederholungen mit Backoff, Provider rechnen teils angefangene Generierungen ab. Ein Turn blockiert über 4 Minuten, und der Loop sieht danach einen AbortError als 'timeout'.
- **Fix:** Streaming nutzen und das Timeout als Idle-Timeout zwischen Chunks auslegen. Timeouts nicht blind retryen, oder pro Tier/Modell konfigurierbar machen. Die Usage aus dem Stream erfassen.

### money-014: Nicht-JSON-Antwort führt zu 'Body is unusable' und meldet nach erfolgter Zahlung 'success:false'

**Art:** Code · **Fundstelle:** `src/conway/x402.ts:338`

- **Problem:** initialResp.json().catch(() => initialResp.text()) (338-340) und paidResp.json().catch(() => paidResp.text()) (403): Nach fehlgeschlagenem json() ist der Body verbraucht, text() wirft TypeError, die Exception läuft in den äußeren catch (405-407) und liefert {success:false,error:'Body is unusable'}. Das betrifft auch die bezahlte Antwort.
- **Szenario:** Der Endpoint antwortet nach erfolgreicher Zahlung mit Text/HTML oder leerem Body. Die USDC-Autorisierung ist gesendet und eingelöst, topupCredits meldet Fehler, und Heartbeat/LLM versuchen es erneut (neuer Nonce), was zu Doppelzahlungen führt.
- **Fix:** Body einmal mit resp.text() lesen und dann JSON.parse versuchen. Nach einer Zahlung den Erfolg über das Guthaben (Balance-Delta) oder den X-PAYMENT-RESPONSE-Header verifizieren.

### money-015: Beträge werden heuristisch als Dollar oder Atomic-Units interpretiert; Ganzzahlen bis 6 Stellen werden mit 1e6 multipliziert

**Art:** Logik · **Fundstelle:** `src/conway/x402.ts:173`

- **Problem:** parseMaxAmountRequired: Eine Ganzzahl mit höchstens 6 Zeichen und x402Version unter 2 wird als Dollar behandelt (parseUnits(amount,6)). Nach x402-v1 ist maxAmountRequired aber immer atomic. '500000' ($0.50) wird zu 500000 USDC. Die Cap-Prüfung (361) lehnt Zahlungen unter $1 dann fälschlich ab, und ohne Cap (Topup) würde die Autorisierung 1e6-fach überhöht signiert.
- **Szenario:** Ein Service verlangt $0.10 (maxAmountRequired '100000'). x402_fetch bricht mit 'exceeds max 100 cents' ab, alle Micropayments unter einem Dollar sind unbrauchbar. Ohne maxPaymentCents würde 100000 USDC signiert (nur durch die Wallet-Balance begrenzt).
- **Fix:** Atomic-Units laut Spezifikation behandeln und Dezimalwerte nur aus ausdrücklich dezimalen Feldern akzeptieren. Der Version-Switch gehört in dokumentierte Felder, nicht in die Stringlänge. Betrag gegen den erwarteten Betrag prüfen.

### money-018: Mehrere Budget-Regeln sind wirkungslos: Inference-Tageslimit, Mindestreserve, ceilingCents, enableModelFallback, Tagesbudget

**Art:** Funktion · **Fundstelle:** `src/agent/policy-rules/financial.ts:241`

- **Problem:** createInferenceDailyCapRule gilt nur für Tools namens 'chat'/'inference' in Kategorie conway; solche Tools existieren nicht, daher ist maxInferenceDailyCents (Default 50000) tot. createMinimumReserveRule (181-209) und createX402MaxSingleRule geben immer null zurück. ModelPreference.ceilingCents wird nie gelesen. enableModelFallback hat keinen Konsumenten, und der Router meldet Fehler nur mit dem Kommentar 'try next candidate'. budget.ts:34-64 prüft kein Tagesbudget, obwohl 'per day' im Header steht, und der Parameter model ist ungenutzt. Alle Budget-Defaults sind 0 (unbegrenzt, inference/types.ts:180-190).
- **Szenario:** Ein Fehler im Loop erzeugt 25 Turns pro Zyklus über Stunden bei gpt-5.2. Weder Stunden-, Tages- noch Reservelimit stoppen den Abfluss, und das Dokument verspricht ein Tageslimit von $500.
- **Fix:** Tageslimit als echte Budgetprüfung im InferenceBudgetTracker implementieren, sinnvolle Defaults setzen (z.B. Prozent des Guthabens/Tag) und die nicht umgesetzten Felder entweder implementieren oder aus Config/Doku entfernen. Reserve-Regel mit aktuellem Guthaben auswerten.

### money-020: Failover nur bei 429/500/503; Verbindungsfehler, Timeouts, 401/402/404 und 502/504 werfen sofort ohne Fallback; nicht-retrybare Fehler zählen als Circuit-Fehler

**Art:** Logik · **Fundstelle:** `src/inference/inference-client.ts:129`

- **Problem:** isRetryableError ist nur true für HTTP-Status in {429,500,503}. APIConnectionError/Timeout haben keinen Status, ebenso 502/504 und 404 (Conway-LB-Routingbug laut client.ts:51-53). Sie führen zu throw unwrapError(...) (141) ohne nächsten Kandidaten. markProviderFailure (135) zählt zudem jeden Fehler, auch 400er (z.B. zu langer Kontext), und nach 5 Treffern ist der Provider 5 Minuten deaktiviert. getStatusCode (549-553) liest Zahlen aus Fehlertexten ('max_tokens 500' gilt als Status 500).
- **Szenario:** Groq ist down (Connection refused): kein Wechsel zu openai. Fünf 400er wegen Kontextlänge deaktivieren den einzigen Provider für 5 Minuten, obwohl er gesund ist.
- **Fix:** Fehlerklassen: provider-fatal (401/403/404/402/Connection) -> nächster Kandidat; request-fatal (400/422) -> werfen ohne Circuit-Zählung; transient -> Retry. Statuscode nur aus strukturierten Feldern lesen.

### money-028: getUsdcBalance verschluckt RPC-Fehler und liefert 0; dadurch sind die catch-Blöcke in topup.ts und loop.ts tot, und 0 wird gecacht

**Art:** Code · **Fundstelle:** `src/conway/x402.ts:202`

- **Problem:** getUsdcBalance (194-204) gibt result.balance zurück und ignoriert ok:false. getUsdcBalanceDetailed (280-286) fängt alles und liefert balance:0. loop.ts:986-992 verlässt sich auf eine Exception (nur dann bleibt _lastKnownUsdc), schreibt sonst 0 in last_known_balance. Der Default-Public-RPC (Base) ist rate-limitiert.
- **Szenario:** RPC rate-limitiert: USDC=0, Inline-Topup (usdc>=5) wird nie ausgelöst, obwohl 100 USDC da sind. Die Tool-Ausgabe 'USDC balance: 0' verleitet das LLM zu Fehlentscheidungen. bootstrapTopup loggt 'below minimum tier'.
- **Fix:** getUsdcBalance bei ok:false werfen oder {ok:false} zurückgeben und Aufrufer unterscheiden. Mehrere RPC-Endpunkte mit Failover und Cache mit Alter (TTL) einsetzen.

### money-035: Solana-Automatons: USDC-Prüfung fragt EVM mit Solana-Adresse; kein Selbstfunding möglich

**Art:** Funktion · **Fundstelle:** `src/survival/monitor.ts:44`

- **Problem:** monitor.ts:44 ruft getUsdcBalance(identity.address) ohne Netzwerk/chainType auf, bei Solana-Adresse scheitert die EVM-Abfrage und liefert 0 (siehe Fund zu getUsdcBalance). Topup (x402) ist EVM-only (topup.ts:90/158). Für Solana existiert ein USDC-Balance-Pfad, aber kein Weg, daraus Credits zu kaufen. Solana-Instanzen sind vollständig auf manuelle Aufladung angewiesen und bekommen im Heartbeat kein Auto-Topup.
- **Szenario:** Solana-Automaton mit 50 USDC auf Solana: sieht 0 USDC bzw. kann nichts umsetzen, fällt in critical und kann sich nicht retten.
- **Fix:** chainType-bewusste Balance-Abfrage überall. Für Solana einen Funding-Pfad (Bridge/Conway-API-Kauf per Solana-USDC) bauen oder das Feature klar als 'manuell finanziert' kennzeichnen und früh warnen.

### money-050: State-Repo-.gitignore schließt automaton.json (API-Keys im Klartext) nicht aus

**Art:** Sicherheit · **Fundstelle:** `src/git/state-versioning.ts:45`

- **Problem:** initStateRepo legt in ~/.automaton ein Git-Repo an und committet sofort mit 'git add -A'. Die .gitignore listet nur wallet.json, config.json, state.db, logs. Die Hauptkonfiguration automaton.json fehlt, obwohl saveConfig (src/config.ts:93-101) dort conwayApiKey, openaiApiKey und anthropicApiKey im Klartext ablegt. Andere Module behandeln automaton.json selbst als sensibel (tools.ts:189, path-protection.ts:18). Zusätzlich existiert ein git_push-Tool.
- **Szenario:** Erststart: initStateRepo -> 'genesis'-Commit enthält automaton.json samt Conway-/OpenAI-/Anthropic-Key. Setzt der Agent (oder ein Prompt-Injection-Angriff) später 'git remote add ... && git push' (Backup/Replikation), liegen die Schlüssel im Remote und in der gesamten Historie. Ein Löschen der Datei danach entfernt sie nicht aus alten Commits. Mit dem Conway-Key kann jeder Credits verbrauchen.
- **Fix:** automaton.json, inference-providers.json, genesis.json und *.key/.env in die .gitignore aufnehmen. Besser: Secrets aus automaton.json in eine eigene, ignorierte Datei oder in Umgebungsvariablen auslagern. Bestehende Repos beim Start prüfen und 'git rm --cached' für automaton.json ausführen. gitPush auf Remote-Allowlist beschränken.

### money-051: getCreditsBalance liefert 0 statt Fehler bei unerwarteter Antwort; Start-Bootstrap wertet Fehler ebenfalls als 0 und kauft USDC-Credits

**Art:** Ausführung · **Fundstelle:** `src/conway/client.ts:298`

- **Problem:** getCreditsBalance gibt 'result.balance_cents ?? result.credits_cents ?? 0' zurück. Eine 200-Antwort mit anderer Feldform, einem Wrapper (z. B. {data:{...}}) oder einem Nicht-JSON-Body (request() liefert dann einen String) ergibt stillschweigend 0. index.ts:349 macht zusätzlich 'conway.getCreditsBalance().catch(() => 0)' und übergibt das an bootstrapTopup, das bei credits<500 und USDC>=5 sofort einen echten $5-Kauf auslöst. Der Bootstrap hat keinen Cooldown und keinen Persistenz-Marker.
- **Szenario:** Der Code selbst dokumentiert in client.ts:51 einen intermittierenden Conway-LB-Routingfehler mit 404 für gültige Endpunkte. Fällt /v1/credits/balance nach 4 Versuchen auf 404 oder 5xx, wird das Guthaben als 0 gewertet. Wallet hat 20 USDC, Credits sind real $40: bootstrapTopup kauft $5. Bei einem Crash-Loop (Supervisor-Neustart) wiederholt sich das bei jedem Start, bis das USDC aufgebraucht ist. Der /pay-Pfad nutzt einen eigenen Breaker und funktioniert, solange nur der Balance-Endpunkt klemmt.
- **Fix:** getCreditsBalance muss bei fehlendem oder nicht numerischem Feld werfen (Number.isFinite prüfen, Antwort als JSON erzwingen). index.ts:349 darf bei Fehler nicht topuppen, sondern nur 'unbekannt' loggen. Zusätzlich einen persistenten Cooldown (KV) für alle Topup-Pfade und bootstrapTopup nur nach erfolgreich gelesenem Guthaben.

### money-052: Bezahlte x402-Anfrage: Ergebnis bei Timeout/5xx unbekannt, Autorisierung bleibt vom Server gewählt lange gültig, kein Abgleich, Bootstrap-Race bricht nichts ab

**Art:** Funktion · **Fundstelle:** `src/conway/x402.ts:392`

- **Problem:** Nach dem Signieren (signPayment) wird die Zahlung per paidResp-Request mit 30 s Timeout gesendet. Bei Timeout, Netzfehler oder 5xx liefert x402Fetch success:false, obwohl der Facilitator die EIP-3009-Autorisierung evtl. schon einlöst oder noch einlösen kann. validBefore = now + requiredDeadlineSeconds, wobei dieser Wert ungeprüft aus der Server-Antwort kommt (Z. 131-134, nach oben unbegrenzt). Es gibt keine Nachprüfung (Kontostand/Transfer-Event) und keinen Abbruch. index.ts:343-363 setzt ein 15-s-Promise.race, das den laufenden Zahlungsvorgang nicht abbricht.
- **Szenario:** Topup $5: Server nimmt die Signatur an, antwortet nach 31 s. Client meldet 'failed'. Der inline-/Heartbeat-Pfad signiert nach dem Cooldown mit neuer Nonce erneut, der Server löst beide Autorisierungen ein: $10 abgebucht, Credits zweifach oder gar nicht gutgeschrieben. Beim Start bricht das 15-s-Race ab (Log 'skipped'), die Zahlung läuft im Hintergrund weiter und der Loop kauft zusätzlich.
- **Fix:** requiredDeadlineSeconds auf z. B. 60-300 s kappen. Vor jedem Neukauf Credits- und USDC-Saldo gegen den Stand vor dem fehlgeschlagenen Versuch abgleichen (pending_topup mit Nonce/Zeit in KV, Retry-Sperre bis validBefore abgelaufen). Paid-Request mit längerem Timeout und AbortController, Bootstrap-Race nicht als Abbruch verwenden. Ergebnis über das Transfer-Event/den Saldo bestätigen.

### money-053: registerAgent speichert Registry-Eintrag auch bei reverted Tx oder fehlendem Transfer-Event (agentId "0") und blockiert Neuregistrierung

**Art:** Funktion · **Fundstelle:** `src/registry/erc8004.ts:277`

- **Problem:** Nach waitForTransactionReceipt wird receipt.status nur für das Log-Update genutzt. Danach wird agentId "0" gesetzt und, falls kein Transfer-Log gefunden wird, unverändert in db.setRegistryEntry(entry) gespeichert. tools.ts:1423 verweigert danach jede weitere Registrierung ('Already registered! Agent ID: 0'). Wartet waitForTransactionReceipt ins Timeout, wird gar nichts gespeichert, obwohl die Tx später bestätigt wird, und ein zweiter Aufruf registriert doppelt.
- **Szenario:** register_erc8004 auf Base: Tx reverted (z. B. falscher Contract/URI-Limit) und verbraucht Gas. Receipt.status='reverted'. Der Agent speichert agentId=0 mit der fehlgeschlagenen txHash, Agent-Card/Discovery zeigen ID 0, jeder Retry wird mit 'Already registered' abgewiesen. Die Identität ist dauerhaft kaputt, bis jemand die DB manuell ändert.
- **Fix:** Bei receipt.status!=='success' Fehler werfen und nichts speichern. Fehlt das Transfer-Event, per tokenOfOwnerByIndex/balanceOf nachsehen oder Fehler werfen, nie "0" speichern. Pending-Tx vor dem Warten in der DB vermerken und beim Neustart per Receipt abgleichen. Den Duplikat-Guard auf agentId!=='0' und status bestätigt stützen.

### money-054: Ein einziger Circuit-Breaker für alle Conway-Endpunkte inklusive Guthabenabfrage

**Art:** Ausführung · **Fundstelle:** `src/conway/client.ts:43`

- **Problem:** createConwayClient nutzt eine ResilientHttpClient-Instanz für exec, Dateien, Sandboxes, Domains, Transfers und /v1/credits/balance. Jeder fehlgeschlagene Versuch (jeder Retry zählt: 5xx, 429, Timeout) erhöht consecutiveFailures, bei 5 sperrt der Breaker 60 s alle Anfragen mit CircuitOpenError. Das exec-Timeout wird nicht weitergereicht (30 s Default), lange Befehle zählen also als Fehler. Ein fehlerhafter Nebenendpunkt oder eine tote Sandbox sperrt so die Guthabenabfrage.
- **Szenario:** Zwei exec-Aufrufe in Folge scheitern (Sandbox 502 oder Befehl >30 s, je 4 Fehlversuche): der Breaker öffnet. In den nächsten 60 s wirft getCreditsBalance CircuitOpenError. Der Heartbeat-Tick (tick-context.ts) setzt creditBalance=0, tier='critical', Notruf weckt den Agenten und check_usdc_balance kauft $5 trotz vollem Guthaben.
- **Fix:** Pro Endpunktgruppe eigene Breaker (credits/balance, exec, inference, domains). Lesende Statusabfragen vom Breaker der Schreib-/exec-Pfade trennen. Guthaben-Fehler nie in 0 umwandeln (siehe tick-context). Lange exec-Timeouts als eigener Timeout, nicht als Breaker-Fehler zählen.

### money-055: fund_child und Orchestrator-Funding umgehen alle Transfer-Limits und das Spend-Tracking

**Art:** Sicherheit · **Fundstelle:** `src/agent/tools.ts:1757`

- **Problem:** Die Policy-Regeln transfer_max_single, hourly_cap, daily_cap, turn_transfer_limit und require_confirmation gelten nur für den Toolnamen transfer_credits (financial.ts:102-278). executeTool bucht Ausgaben ebenfalls nur für transfer_credits (tools.ts:3352). fund_child hat nur den Schutz 'amount <= balance/2'. SimpleFundingProtocol.fundChild (orchestration/simple-tracker.ts:88) und die Auto-Heal-Funding-Pfade rufen conway.transferCredits ohne Policy, Spend-Tracking oder Transaktionseintrag. Die Mindestreserve-Regel ist wirkungslos.
- **Szenario:** Statt transfer_credits (max. $50 pro Transfer, Tageslimit $250) ruft der Agent fund_child mehrfach auf. Jeder Aufruf darf 50 % des Restguthabens verschieben, nach 3 Aufrufen sind 87 % des Guthabens beim Child, ohne dass hourly/daily-Zähler oder Bestätigung greifen. Die Orchestrator-Zuteilung überweist pro Task mindestens 25 Cent ohne Reservecheck. Bei hoher Aufgabenzahl fließt unbemerkt Guthaben ab.
- **Fix:** Alle ausgehenden Credit-Transfers (transfer_credits, fund_child, SimpleFundingProtocol.fundChild) durch eine gemeinsame, zentral geprüfte Funktion mit Limits, Reservecheck, Spend-Tracking und Transaktionslog leiten. Policy-Regeln auf fund_child ausdehnen.

### money-056: create_sandbox und register_domain ohne Kostenobergrenze; Worker dürfen register_domain aufrufen

**Art:** Sicherheit · **Fundstelle:** `src/agent/tools.ts:360`

- **Problem:** create_sandbox reicht vcpu, memory_mb und disk_gb des Modells ungeprüft an die API (client.ts:255 mit '||'-Defaults). Der Code nennt Sandboxes selbst 'prepaid and non-refundable' (tools.ts:386). register_domain akzeptiert beliebige 'years' (tools.ts:2233) ohne Preisprüfung. Es gibt keine Policy-Regel, kein Spend-Tracking und keine Reserve-Prüfung für beide Tools. register_domain steht zusätzlich in der Worker-Allowlist (general-harness.ts:30). Der Client wiederholt beide POSTs bei 5xx (bereits gemeldet).
- **Szenario:** Ein halluzinierender oder per Prompt-Injection gesteuerter Agent ruft create_sandbox mit vcpu=16, memory_mb=65536, disk_gb=1000 oder register_domain mit years=10 auf. Das Guthaben wird unwiderruflich verbraucht, weil Reserve- und Tageslimits nicht gelten. Ein Worker kann dasselbe ohne Eltern-Agent auslösen.
- **Fix:** Sandbox-Größe gegen die gültigen Tiers (wie in replication/spawn.ts SANDBOX_TIERS) validieren, Preis vorab über getCreditsPricing/searchDomains prüfen und gegen Reserve und Tageslimit aus der TreasuryPolicy prüfen. years auf 1-2 begrenzen. Beide Tools in die Financial-Policy aufnehmen und aus der Worker-Allowlist entfernen oder nur mit Bestätigung erlauben.

### money-057: recallCredits überweist an die eigene Adresse und ist ein No-op; Buchhaltung meldet Erfolg

**Art:** Funktion · **Fundstelle:** `src/orchestration/simple-tracker.ts:123`

- **Problem:** recallCredits ruft this.conway.transferCredits(this.identity.address, amountCents, ...) auf. Der Client ist der des Eltern-Agenten, die Überweisung läuft also vom Eltern-Konto auf das eigene Konto. Credits der Kinder werden nie zurückgeholt. Der Code verringert dennoch funded_amount_cents und meldet success. getBalance ist nur der lokale Zähler (TODO im Code). handleCompletePhase im Orchestrator ruft recallAgentCredits nach jedem Ziel auf.
- **Szenario:** Ziel abgeschlossen, 3 Kinder wurden mit je 250 Cent finanziert. Die Recall-Phase 'erfolgreich': DB zeigt funded=0, das Guthaben liegt weiter in den Kindern oder der API lehnt die Selbstüberweisung ab. Folge: falsche Colony-Bilanz und Credits bleiben in Kind-Sandboxes gebunden.
- **Fix:** Recall muss vom Kind ausgelöst werden (Nachricht an das Kind, das aus seinem Konto an den Eltern-Wallet überweist) oder per API mit Kind-Autorisierung. Bis dahin Funktion als nicht implementiert markieren, success:false liefern und die DB nicht ändern.

### money-058: Autoritätsregel invertiert: autonome Agentenzüge gelten als extern, Social-Nachrichten als 'agent'-Autorität

**Art:** Logik · **Fundstelle:** `src/agent/policy-rules/authority.ts:41`

- **Problem:** isExternalSource() bewertet nur inputSource===undefined oder 'heartbeat' als extern. In loop.ts wird inputSource aus currentInput?.source gebildet (loop.ts:643, 667). Nach dem ersten Zug nach dem Wakeup ist pendingInput leer, daher ist die Quelle undefined und alle autonomen Züge gelten als 'extern'. 'heartbeat' wird nirgends gesetzt. Nachrichten aus dem Social-Inbox (beliebige Absender des Relays) werden in loop.ts:424 mit source:'agent' markiert und erhalten Autorität 'agent' wie der Creator.
- **Szenario:** (1) Der Agent will in einem normalen autonomen Zug spawn_child oder fund_child aufrufen: Policy lehnt 'EXTERNAL_DANGEROUS_TOOL' ab, die Replikation funktioniert nur im Wakeup-Zug oder nach einer Nachricht. (2) Ein Fremder schickt eine Social-Nachricht mit Prompt-Injection: die Antwort-Züge laufen mit source 'agent', die self_mod_from_external-Regel (Schutz von SOUL.md, heartbeat.yml usw.) und die Blockliste greifen nicht mehr.
- **Fix:** Explizite Vertrauensstufen: Creator-Nachrichten nur nach Signatur-Verifikation als 'creator', Fremdnachrichten als 'external' (eigene InputSource), autonome Folgezüge als 'agent'/'self'. inputSource über die gesamte Zugdauer und alle Folgezüge im Zyklus mitführen, nicht nur im ersten Zug.

### money-059: Adressformat-Regel erlaubt nur EVM-Adressen: Solana-Automatons können keine Credits überweisen und keine Nachrichten senden

**Art:** Funktion · **Fundstelle:** `src/agent/policy-rules/validation.ts:13`

- **Problem:** ADDRESS_RE = /^0x[a-fA-F0-9]{40}$/ wird für die Tools transfer_credits, send_message und fund_child auf to_address/agent_address angewendet (validation.ts:163-187). Solana-Adressen (base58) schlagen immer fehl. Das Social-Modul (social/validation.ts) und identity/chain.ts unterstützen beide Chains. Für Solana ist x402 gesperrt, transfer_credits wäre der einzige Finanzierungs-/Kommunikationsweg.
- **Szenario:** Solana-Automaton ruft transfer_credits mit gültiger base58-Adresse auf: Policy 'VALIDATION_FAILED: Invalid address format'. Das gilt ebenso für send_message (to_address). Der Solana-Agent kann weder Kinder noch Dritte per Credits finanzieren noch mit anderen Agenten schreiben.
- **Fix:** Regel chain-bewusst machen: isValidAddress(address, ctx.config.chainType) aus identity/chain.ts verwenden, bei send_message beide Formate zulassen.

## Befunde: niedrig (37)

### money-016: Vier unabhängige Topup-Pfade ohne gemeinsamen Lock und mit getrennten Cooldown-Keys: Doppelkäufe möglich

**Art:** Ausführung · **Fundstelle:** `src/agent/loop.ts:444`

- **Problem:** Bootstrap in index.ts:341-369 (Timeout 15s, der innere Task läuft nach dem Race weiter), Inline-Topup im Loop (Key last_inline_topup_attempt, 60s), Heartbeat (last_auto_topup_attempt, 5 Min, tasks.ts:163) und das LLM-Tool topup_credits (kein Cooldown). Jeder Pfad entscheidet auf Basis eines eigenen, ggf. veralteten Guthabenstands. Der Cooldown wird vor dem Versuch gesetzt (kein Erfolgs-Tracking), Gutschriften sind nicht sofort sichtbar.
- **Fix:** Ein einziger Payment-Service mit Mutex und persistierter 'payment intent'-Tabelle (id, Betrag, Nonce, Status), gemeinsamem Cooldown und Balance-Delta-Verifikation, bevor ein neuer Kauf erlaubt wird. Topup-Entscheidung zentral treffen.

### money-017: topup_credits unterliegt keiner Policy-Regel und keiner Ausgabenverfolgung; x402-Ausgaben werden mit 0 Cent verbucht

**Art:** Sicherheit · **Fundstelle:** `src/agent/tools.ts:277`

- **Problem:** financial.ts enthält Regeln nur für x402_fetch und transfer_credits. Der Spend-Tracker in executeTool (tools.ts:3350-3392) kennt weder topup_credits noch den echten x402-Betrag (recordSpend amountCents:0 für x402_fetch). Es gibt kein Tageslimit und kein Limit pro Turn für Topups, und die Reserve-Regel (financial.ts:181-209) gibt immer null zurück.
- **Fix:** topup_credits in die Policy aufnehmen (Max pro Aufruf/Stunde/Tag, Tier-Obergrenze abhängig von Runway). Alle USDC-Abflüsse in spend_tracking mit echtem Betrag verbuchen. Die Reserve-Regel real implementieren.

### money-019: Inkonsistente Schwellwerte für Survival-Tiers, Topup und Notbremse an mindestens sieben Stellen

**Art:** Logik · **Fundstelle:** `src/types.ts:231`

- **Problem:** credits.ts/types.ts: 500/50/10c, dead<0. Bootstrap-Topup unter 500c. Inline-Topup bei Tier low_compute/critical (loop.ts:444), Heartbeat-Topup nur bei critical/dead (tasks.ts:160), die Doku sagt 'credits < $5'. UnifiedInferenceClient: survival 100-1000, Emergency <100. planner-context.ts:197-208: <=0 critical, <100 low, <5000 stable. check_credits (6h) vs '1 hour' Grace. README/Doku widersprechen sich zu dead.
- **Fix:** Eine zentrale Tier-Konfiguration (inkl. Hysterese und Runway-Basis) exportieren und überall verwenden. Topup-Trigger als Funktion der Runway (Stunden bis leer) statt fester Cent-Schwellen. Dokumentation daraus generieren.

### money-021: Ein einziger Circuit-Breaker für alle Inferenz-Backends; ein modulweiter Breaker für alle x402-Ziele

**Art:** Code · **Fundstelle:** `src/conway/inference.ts:52`

- **Problem:** createInferenceClient nutzt eine ResilientHttpClient-Instanz für conway, openai, anthropic und ollama. Fünf aufeinanderfolgende Fehler irgendeines Backends (z.B. Ollama down: ECONNREFUSED) öffnen den Breaker für alle für 60s (CircuitOpenError). Ebenso ist x402HttpClient (x402.ts:19) modulweit: Fehler beim Aufruf beliebiger x402_fetch-Ziele blockieren auch den Conway-Topup. Der Breaker hat keinen echten Half-Open-Zustand (consecutiveFailures bleibt >= Schwelle).
- **Fix:** Breaker pro Host/Backend führen (Map<host,Breaker>) und Half-Open sauber implementieren (1 Probe, Zähler zurücksetzen). Topup-Client von beliebigen x402-Zielen trennen.

### money-022: Router-Timeouts sind wirkungslos: signal wird von conway/inference.chat ignoriert; Timeout-Meldung irreführend; kein Fallback

**Art:** Logik · **Fundstelle:** `src/inference/router.ts:120`

- **Problem:** route() setzt AbortController und inferenceOptions.signal (120-126), aber createInferenceClient.chat liest nur model/tools/maxTokens/temperature. Die Task-Timeouts (heartbeat_triage 15s, safety_check 30s) greifen nie. Tatsächlich begrenzt nur der interne 60s x 4 Retries. Der 'timeout'-Zweig (131) entsteht erst, wenn der interne AbortError nach den Retries hochkommt. Der Kommentar 'try next candidate' bleibt unimplementiert, und bei Timeout oder Fehler werden keine Kosten verbucht.
- **Fix:** signal in chat()/ResilientHttpClient durchreichen (AbortSignal.any), Retries gegen das Gesamt-Deadline budgetieren, Fallback auf den nächsten Kandidaten implementieren.

### money-023: Kostenschätzung und Kostenerfassung falsch gerundet bzw. unterschätzt

**Art:** Logik · **Fundstelle:** `src/inference/router.ts:65`

- **Problem:** Math.ceil auf Cent-Ebene (66, 150) macht jeden Call mindestens 1c (heartbeat_triage-Aufrufe kosten real ~0,01c). Das verzerrt Stunden-/Sessionbudgets. Die Schätzung nutzt maxTokens||1000 (68), das reale Limit ist preference.maxTokens bzw. model.maxTokens (bis 32768), und Tool-Schemata zählen nicht mit. Die Ceiling-Prüfung arbeitet daher mit einer Untergrenze. recordCost speichert integer cost_cents, wodurch Rundungsfehler akkumulieren.
- **Fix:** Kosten in Hundertstel-Cent (oder Mikro-USD) als Float/Integer führen und erst bei der Anzeige runden. Schätzung mit dem tatsächlichen max_tokens und Tool-Token berechnen. Gegen Credit-Delta kalibrieren.

### money-025: ModelRegistry.initialize() deaktiviert bei jedem Aufruf alle nicht-Baseline-Modelle; refreshFromApi übernimmt API-Preise nie und setzt unbekannte Preise auf 0 ('gratis')

**Art:** Logik · **Fundstelle:** `src/inference/registry.ts:67`

- **Problem:** initialize() (41-80) setzt jedes aktivierte Modell außerhalb der Baseline (außer ollama/other) auf disabled und wird bei jedem Start sowie in refresh_models (tasks.ts:346) aufgerufen. refreshFromApi (137-159) behält existing.enabled, das Modell bleibt dauerhaft deaktiviert. Außerdem liest es m.pricing.input_per_1k, während client.listModels camelCase inputPerMillion liefert. Neue Modelle erhalten Kosten 0 (isFree in selectModel erlaubt sie in jedem Tier inklusive dead), und Baseline-Preise werden bei jedem initialize() wieder auf statische Werte zurückgesetzt. refresh_models ist nicht im Default-Heartbeat (heartbeat/config.ts), der Pfad ist also nur nach manueller Aktivierung relevant.
- **Fix:** enabled getrennt von der Baseline-Pflege verwalten (nie automatisch ausschalten, nur bei 'removed upstream'). API-Pricing korrekt umrechnen (USD/M -> Hundertstel-Cent/1k = x10). Unbekannte Preise als 'unknown' behandeln und konservativ schätzen, nicht als 0.

### money-026: src/survival/* ist toter Code; die 'Funding-Strategien' versenden nichts und lügen im Details-Text

**Art:** Funktion · **Fundstelle:** `src/survival/funding.ts:28`

- **Problem:** executeFundingStrategies, checkResources, applyTierRestrictions, recordTransition, canRunInference, getModelForTier und credits.checkFinancialState werden außerhalb von Tests nicht importiert/aufgerufen. Selbst der Code selbst schreibt nur KV-Keys (funding_notice_*) und behauptet 'Sent polite funding reminder to creator'. Niemand liest diese Keys. JSON.parse der Historie (88, low-compute.ts:76) ist ungeschützt, getModelForTier hat 'gpt-5-mini' hartcodiert, hoursSinceLastBeg wird bei ungültigem Datum NaN und blockiert Benachrichtigungen dauerhaft.
- **Fix:** Entweder löschen oder an den Heartbeat anbinden. Distress wirklich senden (Social-Relay/Webhook/E-Mail an creatorAddress). KV-Parsing absichern und eine einzige Tier-Statemachine nutzen.

### money-027: colony_financial_report zählt credit_purchase (eigene Topups) als Einnahmen und erfasst keine Inferenzkosten

**Art:** Endergebnis · **Fundstelle:** `src/heartbeat/tasks.ts:522`

- **Problem:** Revenue = transfer_in + credit_purchase, Expense = inference|tool_use|transfer_out|funding_request. insertTransaction wird nur für Topup (credit_purchase), transfer_out und fund_child aufgerufen (tools.ts:327/1031/1769). 'inference' und 'tool_use' werden nie geschrieben, 'transfer_in' nie. Der Bericht zeigt daher positiven 'Gewinn', sobald der Agent USDC in Credits tauscht.
- **Fix:** credit_purchase als Kapitalumschichtung führen (nicht Revenue), Inferenzkosten aus inference_costs/Credit-Delta einbeziehen, transfer_in nur aus verifizierten Eingängen.

### money-029: Sentinel creditsCents=-1 (API unerreichbar) wird in getSurvivalTier zu 'dead' und blockiert die Inferenz

**Art:** Logik · **Fundstelle:** `src/agent/loop.ts:599`

- **Problem:** loop.ts:434-437 behandelt -1 als 'continue in low-compute'. Danach ruft loop.ts:599 getSurvivalTier(-1) unconditioniert auf und gibt 'dead' an den Router. selectModel liefert für dead nur kostenlose Modelle, sonst model:'none'/finishReason:'error' mit leerem Inhalt. Der leere Turn wird persistiert und zählt als Idle.
- **Fix:** -1 am Eingang separat behandeln (Tier = letzter bekannter Tier, Turn überspringen). getSurvivalTier gegen nicht-endliche/negative Sentinels absichern. Leere Router-Fehlerergebnisse nicht als Turn speichern.

### money-030: HOME-Fallback '/root' statt os.homedir(): unter Windows landen Wallet und Config in \root\.automaton (ggf. für andere Benutzer lesbar); lokales exec läuft nicht

**Art:** Ausführung · **Fundstelle:** `src/identity/wallet.ts:44`

- **Problem:** wallet.ts:44-47, config.ts:109, client.ts:117/168 und rund 15 weitere Stellen nutzen process.env.HOME || '/root'. HOME ist unter Windows (cmd/PowerShell) meist nicht gesetzt. path.join('/root','.automaton') ergibt \root\.automaton auf dem aktuellen Laufwerk. mode:0o600 hat unter Windows keine Wirkung, die Datei erbt die Rechte des Laufwerksstamms (vermutlich nicht nur für den Benutzer). execLocal setzt cwd:'/root', existiert nicht, und jeder Befehl scheitert mit ENOENT (exitCode 1). Dadurch meldet monitor.ts 'Sandbox UNHEALTHY'. workspace.ts nutzt dagegen os.homedir() (inkonsistent).
- **Fix:** Überall os.homedir() (bzw. AUTOMATON_HOME-Override) verwenden. Dateirechte unter Windows per icacls setzen. Shell/cwd plattformabhängig wählen.

### money-032: transferCredits: Idempotency-Key pro Aufruf neu, keine Adressvalidierung, Mehrdeutigkeit bei Timeout

**Art:** Code · **Fundstelle:** `src/conway/client.ts:324`

- **Problem:** Der Key wird per ulid() innerhalb der Funktion erzeugt, ein erneuter Aufruf durch das LLM nach Timeout/Fehler erzeugt einen neuen Key, und der Transfer kann doppelt ausgeführt werden. Bei 5xx/Timeout (retries:0) wird nur 'error' gemeldet, obwohl der Transfer evtl. gebucht wurde. isValidAddress (identity/chain.ts) wird in transfer_credits nicht verwendet (to_address ungeprüft). Bei 404 des ersten Pfads wird der zweite Pfad probiert. Ein unleserlicher 200-Body liefert transferId:'' und 'submitted'.
- **Fix:** Idempotency-Key aus (toolCallId/turnId + Empfänger + Betrag) ableiten und persistieren. Bei Fehler zuerst per Ledger/Balance-Delta den Zustand klären. Adresse mit EIP-55-Checksum prüfen und optional Empfänger-Allowlist nutzen.

### money-033: Lokales exec erbt die komplette Prozess-Umgebung inklusive API-Keys; zusätzlich schreibt loop.ts den Conway-Key nach OPENAI_API_KEY

**Art:** Sicherheit · **Fundstelle:** `src/conway/client.ts:113`

- **Problem:** execSync ohne env-Option übergibt process.env an die Shell. loop.ts:139-157 setzt OPENAI_API_KEY/ANTHROPIC_API_KEY/CONWAY_API_KEY in die Prozess-Env (auch den Conway-Key als OPENAI_API_KEY). In der Sandbox (remote exec) tritt das nicht auf, bei sandboxId='' (lokaler Betrieb, der Default-Fallback) liest 'env' oder 'printenv' diese Secrets, und 'cat .env'-Blacklisten greifen nicht.
- **Fix:** execSync mit expliziter, minimaler env (PATH, HOME, LANG) aufrufen und Secrets nur per interner Konfiguration statt per process.env durchreichen. Lokale Ausführung optional ganz verbieten oder in Container/anderen User verschieben.

### money-034: OpenAI-SDK-Defaults (Timeout 10 Minuten, 2 interne Retries) vervielfachen Wartezeit und Kosten; pro Auflösung neuer Client

**Art:** Ausführung · **Fundstelle:** `src/inference/provider-registry.ts:434`

- **Problem:** new OpenAI({apiKey, baseURL}) ohne timeout/maxRetries. Das SDK retryt selbst 2x (408/409/429/5xx, Verbindungsfehler), darüber retryt UnifiedInferenceClient 3x (RETRY_BACKOFF_MS) = bis 12 Versuche pro Provider, jeder bis 600s. Zusätzlich wird in resolveCandidates für jeden Provider ein neuer Client gebaut. Dazu ist eine HTTP-Verbindung zu Nicht-HTTPS-baseUrl erlaubt.
- **Fix:** timeout (z.B. 60s) und maxRetries:0 am Client setzen, Retries nur an einer Stelle. Clients pro Provider cachen. https für baseUrl erzwingen (außer loopback).

### money-036: Streaming ohne stream_options.include_usage und Einheitenfehler in der Kostenformel

**Art:** Code · **Fundstelle:** `src/inference/inference-client.ts:233`

- **Problem:** Bei stream:true wird weder stream_options:{include_usage:true} gesetzt (233-236) noch geprüft, ob usage ankommt. Die Kosten bleiben dann 0. Außerdem sind costPerInputToken/-OutputToken in DEFAULT_PROVIDERS Dollar pro Million Token (z.B. 2.0 für gpt-4.1), buildUnifiedResult rechnet aber (tokens/1000)*Wert (389-391): 1000 Tokens = '2.0 credits' statt $0.002. Die Felder heißen 'per Token', haben aber andere Einheit als registry.ts (Hundertstel-Cent/1k).
- **Fix:** Eine einheitliche Preisstruktur (z.B. USD pro 1M Token) in einem Modul definieren und Umrechnung zentral vornehmen. stream_options setzen und fehlende usage-Werte schätzen und kennzeichnen.

### money-037: listModels sendet den Conway-API-Key an eine hartcodierte Host-URL statt an apiUrl

**Art:** Sicherheit · **Fundstelle:** `src/conway/client.ts:543`

- **Problem:** urls enthält fest 'https://inference.conway.tech/v1/models' (543-546) und schickt Authorization: apiKey dorthin. Wer apiUrl auf Staging/Self-Hosted stellt, verschickt den Key trotzdem an Produktion. Außerdem inkonsistent: Aufruf der Chat-Completions geht an apiUrl, Modellliste an inference.conway.tech.
- **Fix:** Inference-Basis-URL aus der Config ableiten und denselben Host für Liste und Chat verwenden.

### money-038: execLocal blockiert den Event-Loop (execSync bis 30s), verwirft stderr bei Erfolg und nutzt die Plattform-Shell

**Art:** Ausführung · **Fundstelle:** `src/conway/client.ts:111`

- **Problem:** execSync ist synchron: Heartbeat, Inbox-Polling und Balance-Checks stehen währenddessen still. Bei exitCode 0 wird stderr als '' zurückgegeben (stderr geht an die Prozess-Konsole). Befehle laufen unter Windows in cmd.exe, bash-Syntax des Agenten (&&, $VAR) funktioniert nicht. Timeouts liefern err.status=null->1 ohne Hinweis auf Timeout.
- **Fix:** child_process.spawn asynchron mit stdout/stderr-Erfassung, explizite Shell (bash/pwsh) und Timeout-Kennzeichnung im Result.

### money-039: IPv6-Loopback wird nie erkannt; Timeout deckt das Lesen des Bodys nicht ab

**Art:** Code · **Fundstelle:** `src/conway/http-client.ts:13`

- **Problem:** new URL('http://[::1]:11434').hostname ist '[::1]' (mit Klammern), LOOPBACK_HOSTS enthält '::1' (13 und inference.ts:36-46): http://[::1] wird als unsicher abgelehnt. In request() wird der Timer nach Eintreffen der Header gelöscht (93), resp.json()/text() kann bei hängendem Stream unbegrenzt blockieren. headers werden per Spread kopiert (breaks bei Headers-Instanz).
- **Fix:** hostname ohne Klammern normalisieren (replace(/^\[|\]$/g,'')), Timer erst nach vollständigem Body löschen bzw. Body-Timeout einführen.

### money-040: session_id wird nie gesetzt; inference_costs wird nie bereinigt; Stundenbudget ist ein Tumbling-Fenster

**Art:** Funktion · **Fundstelle:** `src/agent/loop.ts:608`

- **Problem:** db.getKV('session_id') wird nur gelesen (nie gesetzt), alle Kosten laufen unter 'default'. sessionBudgetCents wirkt so als Lebenszeitbudget, und getSessionCost summiert die gesamte Tabelle in JS bei jedem route(). inferencePruneCosts wird nirgends aufgerufen. inferenceGetHourlyCost nimmt die laufende UTC-Stunde statt der letzten 60 Minuten, sodass um HH:00 das Budget zurückgesetzt wird.
- **Fix:** Session-ID pro Wake-Zyklus vergeben, Summen per SQL (SUM) berechnen, rollierendes Fenster nutzen und Pruning planen.

### money-041: getSurvivalTier(NaN/undefined) ergibt 'dead'

**Art:** Code · **Fundstelle:** `src/conway/credits.ts:38`

- **Problem:** Alle Vergleiche mit NaN sind false, daher fällt die Funktion bis 'return dead' durch. Auch der Sentinel -1 aus loop.ts wird so zu dead. formatCredits(NaN) zeigt '$NaN'.
- **Fix:** Number.isFinite prüfen und bei ungültigem Wert einen Fehler werfen oder den letzten Tier behalten.

### money-042: EIP-712-Domain hartcodiert ('USD Coin'/'2'); auf Base Sepolia heißt das Token vermutlich 'USDC'

**Art:** Code · **Fundstelle:** `src/conway/x402.ts:474`

- **Problem:** signPayment nutzt für beide Netzwerke name:'USD Coin', version:'2', statt extra.name/extra.version aus dem Payment-Requirement. Für Base Sepolia (0x036CbD...) ist der Token-Name vermutlich 'USDC', die Signatur wäre dann ungültig (Testnet-Betrieb scheitert).
- **Fix:** Domain-Name/Version aus dem requirement.extra übernehmen oder per name()/version()-Call vom Contract lesen.

### money-043: SIWS-Nachricht weicht vom Standardformat ab, SIWE/SIWS ohne Ablaufzeit

**Art:** Logik · **Fundstelle:** `src/identity/siws.ts:22`

- **Problem:** buildSiwsMessage lässt 'Version: 1' aus, vertauscht die Reihenfolge (Nonce vor Issued At, Chain ID zuletzt) und enthält keine Expiration Time. provision.ts:105-115 (SIWE) setzt ebenfalls keine expirationTime. Die Verifikation hängt allein am Server-Nonce, ein abgegriffener signierter Text bleibt unbegrenzt gültig, solange der Nonce offen ist. provision() nutzt außerdem 'domain: conway.tech' hart (auch bei abweichender CONWAY_API_URL).
- **Fix:** Format laut CAIP-122/EIP-4361 erzeugen (Version, Expiration Time kurz), Domain aus der API-URL ableiten und gegen Server-Erwartung testen.

### money-044: creditsCentsAdded wird bei fehlenden Antwortfeldern erfunden und als balanceAfterCents gespeichert

**Art:** Code · **Fundstelle:** `src/conway/topup.ts:61`

- **Problem:** Fallback amountUsd*100 (61-63) behauptet eine Gutschrift ohne Verifikation. tools.ts:331 schreibt result.creditsCentsAdded in das Feld balanceAfterCents der Transaktion (falsches Feld: Zugang statt Kontostand). Es erfolgt keine Balance-Abfrage nach dem Kauf.
- **Fix:** Nach dem Kauf getCreditsBalance abfragen, Delta speichern und balanceAfterCents korrekt befüllen. Bei Abweichung als 'unbestätigt' markieren.

### money-045: Registrierungsstatus 'conflict'/'failed' führt bei jedem Neustart zu erneuter Registrierung; Retries können 409 selbst erzeugen

**Art:** Funktion · **Fundstelle:** `src/index.ts:249`

- **Problem:** registerAutomaton wird bei status !== 'registered' wiederholt. 'conflict' (409) wird nie zu 'registered'. request() retryt den POST bei Timeout/5xx mit demselben Nonce, ein zweiter Versuch nach erfolgreichem ersten liefert 409 und setzt fälschlich 'conflict'.
- **Fix:** 409 mit passendem automaton_id/address als 'registered' behandeln, den Status per GET verifizieren.

### money-047: Ollama-Discovery: verschwundene Modelle bleiben aktiv, Fähigkeiten und Kontext werden geraten

**Art:** Funktion · **Fundstelle:** `src/ollama/discover.ts:65`

- **Problem:** Modelle, die lokal gelöscht wurden, bleiben enabled in der Registry (nie bereinigt), und jeder Router-Pfad für 'dead'/free kann sie wählen und scheitert dann mit 404. supportsTools wird immer true, contextWindow 8192 und maxTokens 4096 geraten (kein /api/show-Aufruf), Embedding-Modelle werden als Chatmodelle registriert. Discovery läuft nur beim Loop-Start, nicht periodisch.
- **Fix:** /api/show für Capabilities nutzen, nicht mehr gelistete Modelle deaktivieren, Embedding-Modelle filtern, periodisch/bei Fehler neu entdecken.

### money-048: Toter Code und Duplikate: zwei Inferenz-Stacks, doppelte Defaults, ungenutzte Imports, wirkungsloser setLowComputeMode

**Art:** Code · **Fundstelle:** `src/inference/router.ts:10`

- **Problem:** router.ts/budget.ts importieren ulid ohne Nutzung, registry.ts:21 hat ungenutztes TIER_ORDER, router.ts hält ein ungenutztes db-Feld. DEFAULT_MODEL_STRATEGY_CONFIG existiert doppelt (types.ts:1247 und inference/types.ts:180). setLowComputeMode/getModelForTier/lowComputeModel haben keine Wirkung, weil der Router das Modell explizit übergibt. ModelStrategyConfig.anthropicApiVersion wird ignoriert (inference.ts:307 hartcodiert). gpt-4.1-nano steht in der Baseline, wird aber von der Matrix nie gewählt. getModel() ist bei gleichen Modell-IDs in mehreren Tiers mehrdeutig.
- **Fix:** Einen Stack konsolidieren, Defaults zentralisieren, tote Felder und Imports entfernen.

### money-049: exec: Kommentar 'unless the command already sets a directory' nicht umgesetzt; 404-Retry verzögert legitime 404er

**Art:** Code · **Fundstelle:** `src/conway/client.ts:135`

- **Problem:** Der Befehl wird immer mit 'cd /root && ' vorangestellt. request() wiederholt jede 404 (auch bei DELETE/PUT/POST) 3x mit 1+2+3 Sekunden Verzögerung, ohne den 404-Body freizugeben (offene Verbindungen). resp.json() bei Content-Type JSON und leerem Body (204) würfe. Unter dem Retry wird bei nicht idempotenten Aktionen ggf. zusätzlich ausgeführt.
- **Fix:** 404-Retry nur für bekannte Sandbox-Routen und idempotente Methoden, Body konsumieren/verwerfen, leere Bodies tolerieren.

### money-060: topupForSandbox kauft bei jedem HTTP 402 Credits (auch nicht kreditbezogen); NaN-Defizit wählt $2500-Tier

**Art:** Funktion · **Fundstelle:** `src/conway/topup.ts:97`

- **Problem:** Die einzige Prüfung auf 'Guthaben fehlt' ist error.status===402 oder 'INSUFFICIENT_CREDITS' im Text. Fehlen details.required_cents/current_balance_cents, wird immer das $5-Tier gekauft. Ist das Defizit nicht numerisch (NaN, z. B. required_cents:'n/a'), liefert find() undefined und '?? TOPUP_TIERS[last]' wählt $2500 (Z. 117-118). Loop und create_sandbox-Tool wiederholen den Versuch alle 60 s (Cooldown-Keys last_sandbox_topup_attempt).
- **Fix:** Nur kaufen, wenn die Fehlerantwort eindeutig 'insufficient credits' mit numerischem Defizit meldet. Defizit mit Number.isFinite prüfen, bei NaN abbrechen. Maximal einen Kauf pro Ursache/Zeitraum (persistenter Zähler pro Stunde), danach Operator-Hinweis statt weiterer Käufe.

### money-061: fetch folgt Redirects: HTTPS-Erzwingung und x402-Domain-Allowlist gelten nur für die Start-URL, X-Payment wird weitergereicht

**Art:** Sicherheit · **Fundstelle:** `src/conway/http-client.ts:83`

- **Problem:** ResilientHttpClient.request ruft fetch ohne redirect:'manual' auf. assertSecureUrl und die x402-Domain-Allowlist (financial.ts:46-91) prüfen nur die ursprüngliche URL. Antwortet ein erlaubter Host mit 301/302 auf einen anderen Host oder auf http://, wird gefolgt. Ein 402 des Ziels führt zur Signatur gegen dessen payTo, danach wird die Original-URL mit X-Payment erneut angefragt und wieder umgeleitet. Selbstdefinierte Header (X-Payment) werden bei Cross-Origin-Redirects nicht entfernt, nur Authorization.
- **Fix:** redirect:'manual' setzen, Redirects nur nach erneuter assertSecureUrl- und Allowlist-Prüfung folgen (max. 3), und X-Payment/Authorization nie an einen anderen Origin senden. Bei x402 Redirects generell ablehnen.

### money-062: Replay-Schutz des Social-Clients ist nie aktiv: createSocialClient bekommt keine DB

**Art:** Sicherheit · **Fundstelle:** `src/index.ts:306`

- **Problem:** createSocialClient(relayUrl, account, db?) prüft eingehende Nonces nur mit 'if (!db) return false' (social/client.ts:57). index.ts:306 ruft die Funktion mit zwei Argumenten auf, die einzige Aufrufstelle im Produktivcode. checkReplayNonce gibt daher immer false zurück. Die Nachrichten des Relays werden außerdem nicht lokal auf Absender-Signatur geprüft, es wird nur dem Relay vertraut. verifySiwsSignature ist ungenutzt.
- **Fix:** db.raw an createSocialClient übergeben, Absender-Signatur der Inbound-Nachrichten gegen die Absenderadresse prüfen und Nachrichten ohne gültige Signatur verwerfen. Nonce-Prüfung auch ohne DB (In-Memory-Fallback).

### money-063: Ollama auf entferntem oder Docker-Host scheitert: Discovery funktioniert, Inferenz wird mit 'HTTPS required' abgelehnt

**Art:** Funktion · **Fundstelle:** `src/conway/inference.ts:55`

- **Problem:** allowHttpOnLoopback wird nur true, wenn ollamaBaseUrl eine http-Loopback-URL ist. assertSecureUrl lässt http nur für localhost/127.0.0.1/::1 zu. ollama/discover.ts:46 nutzt dagegen rohes fetch ohne Prüfung und registriert die Modelle im Registry. Zeigt OLLAMA_BASE_URL auf http://192.168.x.x:11434 oder http://host.docker.internal:11434, sind die Modelle sichtbar und wählbar, jeder chat()-Aufruf wirft 'HTTPS required'.
- **Fix:** Explizit konfigurierte Ollama-URL (private Adressbereiche, host.docker.internal) per Opt-in zulassen oder Discovery bei nicht erlaubtem Schema ablehnen und die Ursache klar loggen. ::1 vs [::1] im Hostvergleich beachten.

### money-064: Lokaler Modus: Dateioperationen und exec nutzen unterschiedliche Basisverzeichnisse

**Art:** Funktion · **Fundstelle:** `src/conway/client.ts:181`

- **Problem:** execLocal führt mit cwd=process.env.HOME aus (Z. 117). writeFile/readFile lösen relative Pfade nicht auf, sondern nutzen process.cwd() (resolveLocalPath behandelt nur '~'). Im Remote-Modus liegen beide unter /root.
- **Fix:** resolveLocalPath um path.resolve(homeDir, p) für relative Pfade erweitern, gleiche Basis für exec, writeFile und readFile; zentrale getHomeDir()-Funktion (os.homedir()).

### money-065: maxTokensPerTurn aus Config und modelStrategy ist wirkungslos

**Art:** Funktion · **Fundstelle:** `src/inference/router.ts:107`

- **Problem:** Der Router wählt maxTokens als 'request.maxTokens || preference.maxTokens || model.maxTokens'. loop.ts:603 übergibt kein maxTokens, die Matrix liefert 8192/4096/2048. conway/inference.ts nutzt den Client-Default (config.maxTokensPerTurn) nur, wenn opts.maxTokens falsy ist, was nie vorkommt. Auch der Konfigurationsdialog zeigt die Einstellung an (configure.ts:176).
- **Fix:** loop.ts übergibt maxTokens: Math.min(config.maxTokensPerTurn, preference.maxTokens), oder der Router berücksichtigt budget.config.maxTokensPerTurn als Obergrenze.

### money-066: Modellpreis-Anzeige um Faktor 100 zu hoch (configure und model-picker)

**Art:** Code · **Fundstelle:** `src/setup/configure.ts:118`

- **Problem:** costPer1kInput ist in Hundertstel-Cent pro 1k Tokens (inference/types.ts:49, list_models-Tool). Die Anzeige rechnet (c/100/1000)*1_000_000 und schreibt '$' davor. Das ergibt Cent pro Million, nicht Dollar. Gleiche Formel in setup/model-picker.ts:97.
- **Fix:** Dollar pro Million = costPer1kInput / 10 (also (c/100/100)*1000). Eine gemeinsame formatModelPrice()-Funktion verwenden.

### money-067: '--setup' überschreibt vorhandene automaton.json und die vom Agenten gepflegte SOUL.md; Constitution-Kopie scheitert auf bestehender 0444-Datei

**Art:** Funktion · **Fundstelle:** `src/setup/wizard.ts:198`

- **Problem:** runSetupWizard ruft unbedingt saveConfig(config) (Z. 180), copyFileSync auf constitution.md (Z. 191) und writeFileSync(SOUL.md) (Z. 198). Dabei gehen modelStrategy, Keys, Treasury-Anpassungen und der von update_soul fortgeschriebene Inhalt verloren. Beim zweiten Lauf wurde constitution.md zuvor auf 0444 gesetzt (Z. 192). Unter Windows (Read-only-Attribut) und als Nicht-Root unter Linux schlägt copyFileSync fehl, nachdem automaton.json schon überschrieben wurde.
- **Fix:** Vorhandene Dateien erkennen und nur nach Bestätigung überschreiben (Backup anlegen). SOUL.md nicht überschreiben, Constitution nur kopieren, wenn Hash abweicht. Config zusammenführen statt ersetzen. --configure für Einzelwerte empfehlen.

### money-068: ERC-8004: Testnet nutzt Mainnet-Adressen, Reputation-ABI vermutlich falsch, Tx-Status bleibt 'pending', updateAgentURI ungenutzt

**Art:** Funktion · **Fundstelle:** `src/registry/erc8004.ts:58`

- **Problem:** CONTRACTS.testnet verwendet dieselben Adressen wie mainnet (Z. 54-62), die offiziellen Sepolia-Deployments haben andere Adressen. Die REPUTATION_ABI (leaveFeedback(uint256,uint8,string)) entspricht keiner bekannten ERC-8004-Fassung (giveFeedback mit mehr Parametern), im Gegensatz zur per Bytecode belegten Identity-ABI. updateAgentURI und leaveFeedback loggen 'pending', warten nicht auf das Receipt und setzen nie 'confirmed'. updateAgentURI schreibt die DB sofort um und hat keinen Aufrufer.
- **Fix:** Adressen und ABI gegen die offizielle ERC-8004-Spezifikation prüfen und pro Netzwerk korrekt hinterlegen. Receipt abwarten und onchain_transactions-Status aktualisieren, DB erst nach 'success' ändern. Tool zum Aktualisieren des Agent-URI (updateAgentURI) anbinden oder entfernen.

### money-069: register_domain verspricht x402-Zahlung, der Client implementiert aber keinen 402-Flow

**Art:** Funktion · **Fundstelle:** `src/conway/client.ts:480`

- **Problem:** Die Tool-Beschreibung sagt 'Costs USDC via x402 payment'. registerDomain ruft nur request('POST','/v1/domains/register') auf. Antwortet die API mit 402/Zahlungsaufforderung, wirft request() einen Fehler und es wird weder x402Fetch noch eine Kreditabbuchung ausgeführt.
- **Fix:** Verhalten der API klären und entweder 402 über x402Fetch (mit Allowlist, Betragslimit, Spend-Tracking) abwickeln oder die Beschreibung auf Credits ändern. Preis vorab aus searchDomains prüfen.

## Verbesserungsideen

- **Einnahmen als Kernfunktion bauen:** x402-Provider-Middleware (kostenpflichtige Endpunkte aus der Sandbox), Eingangserkennung (USDC-Transfer-Logs/Conway-Ledger), Revenue-Ledger mit Quelle, Preis- und ROI-Reporting und ein Runway-Modell (Guthaben/Burn-Rate). Das Ziel 'selbstfinanzierend' wird sonst nie messbar, geschweige denn erreichbar.
- **Ein einziges Finanz-/Payment-Modul mit Ledger:** Alle Geldbewegungen (Topup, Transfer, x402, Inferenz, Sandbox, Domains) über einen Service mit Mutex, persistierter Intent-Tabelle (id, Betrag, Nonce, Status), Idempotency-Keys aus toolCallId, Balance-Delta-Verifikation und Policy-Prüfung. Das beseitigt Doppelkäufe, falsche 'success'-Meldungen und die Policy-Lücke bei topup_credits.
- **Eine Tier-/Runway-Statemachine statt verstreuter Schwellen:** SurvivalTier zentral aus Guthaben, Burn-Rate und Runway ableiten (mit Hysterese), persistieren und als Parameter an Router, UnifiedInferenceClient, Planner, Heartbeat und Orchestrator geben. Dead wird zu einem echten Hibernate-Modus (keine Inferenz, billiges Polling auf Funding, Wake-on-Funding), konsistent in README, Doku und Code.
- **Inferenz-Stack konsolidieren:** Legacy-Client+Router und UnifiedInferenceClient+ProviderRegistry zu einem Pfad vereinen: ein Modell-/Preisregister (einheitliche Einheit, z.B. USD/1M), Nutzerwahl als erster Kandidat, Key-bewusste Provider, Fehlerklassen mit echtem Fallback, Streaming mit Idle-Timeout, AbortSignal-Durchreichung, einheitliche Kostenerfassung und Budgetprüfung inkl. Tagesbudget.
- **Kosten gegen die Wahrheit abgleichen:** Periodisch das Credit-Delta (getCreditsBalance) gegen die lokal verbuchten Kosten prüfen und Drift melden. Echte Preise aus der Conway-Modellliste übernehmen (korrekt umgerechnet), unbekannte Preise konservativ schätzen statt 0 und Kosten in Hundertstel-Cent statt gerundeten Cent speichern.
- **Schlüsselverwahrung härten:** Verschlüsselter Keystore (scrypt+AES-GCM, Passphrase oder OS-Keychain/KMS), Signer als separater Prozess mit Policy (Betragslimit, Empfänger-Allowlist, Rate-Limit). Agent-Shell als anderer OS-User ohne Zugriff und mit gescrubbter Env. Backup/Mnemonic-Export im Wizard, Atomic-Write und Lock bei der Erzeugung, Windows-ACLs.
- **Konfigurationsdateien schützen und validieren:** inference-providers.json, heartbeat.yml und config.json in die geschützten Pfade aufnehmen oder außerhalb des Agentenbereichs ablegen. Beim Laden validieren (https-only, Host- und Env-Namen-Allowlist) und Änderungen im Audit-Log festhalten.
- **Fehler vs. 'unbekannt' sauber modellieren:** Alle Balance-/Preis-Abfragen als Ergebnis {ok, value, stale?} liefern statt 0 oder -1. Bei Unsicherheit nichts kaufen, den Tier nicht ändern und nicht in 'dead' wechseln. Antwortschemata (balance_cents numerisch) validieren und Cache mit Alter (TTL) verwenden.
- **x402 spezifikationskonform und defensiv:** Betrag immer atomar lesen, Domain-Name/Version aus extra beziehen, payTo/asset/network gegen Allowlist prüfen, validBefore kurz halten, Response-Header (X-PAYMENT-RESPONSE/Tx-Hash) protokollieren, v2-Header (PAYMENT-REQUIRED/-SIGNATURE) unterstützen. checkX402 per GET statt HEAD (viele Server liefern 405).
- **Circuit-Breaker und Retry-Richtlinien pro Host und pro Methode:** Breaker pro Backend/Host mit Half-Open, Retries nur für idempotente Methoden oder mit Idempotency-Key, Timeouts je Endpoint (exec: Befehlstimeout+Puffer, Inferenz: Idle-Timeout). Retry-After auswerten und 404-Flake-Retries auf bekannte Routen beschränken.
- **Echte Benachrichtigung bei Geldmangel:** Distress/Funding-Notices tatsächlich an den Creator senden (Social-Relay, Webhook, E-Mail, Telegram) mit Runway-Angabe und Einzahlungsadresse, inklusive Quittierung. Eskalationsstufen nach Runway statt nach festen Cent-Beträgen.
- **Preflight/Selbsttest beim Start:** Beim Start prüfen und ausgeben: gültige API-Keys je Provider, erreichbare Modelle pro Tier, RPC-Erreichbarkeit, USDC- und Credit-Stand mit Quelle, Wallet-Rechte, Plattformfähigkeit (HOME/Shell). Bei fehlendem Kandidaten pro Tier hart scheitern statt zur Laufzeit zu raten.
- **Windows-/Linux-Parität:** os.homedir() und AUTOMATON_HOME, plattformabhängige Shell (bash/pwsh) und Pfadbehandlung, Dateirechte via icacls, 'rm -rf dist' im clean-Script durch rimraf ersetzen, und CI-Matrix mit beiden Plattformen.
- **Ollama/Local-first als Überlebensmodus:** Bei critical/dead automatisch auf ein lokales Modell (mit /api/show-Capabilities, Health-Check und Modell-Cleanup) umschalten statt nur zu blockieren, und diese Pfadwahl auch für Orchestrator/Worker (ProviderRegistry 'local' mit OLLAMA_BASE_URL) verdrahten.
- **Tests an Verhalten statt an Konfiguration hängen:** Integrationstests für Default-Installation (nur Conway-Key), Provider-Ausfall, API-Ausfall während Heartbeat, parallele Topups, Dead-/Wake-Übergänge und Anthropic-Tool-Historie ergänzen. Aktuell prüfen Tests viele Einzelmodule, aber nicht den Zusammenhang.

