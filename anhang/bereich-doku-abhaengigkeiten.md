# Doku gegen Code, Abhängigkeiten

← [Anhang-Übersicht](README.md) · [Bericht](../README.md)

## So funktioniert der Bereich

Repo: Conway Automaton (@conway/automaton 0.2.1). pnpm-Workspace (Root plus packages/cli, Lockfile pnpm-lock.yaml v9). Das Root-Paket baut mit `tsc` (NodeNext, ESM, Ziel dist/) und exportiert nur ".", "./config.js" und "./state/database.js". Das CLI-Paket (@conway/automaton-cli 0.2.0) hängt per `workspace:*` vom Root ab und importiert genau diese beiden Unterpfade. Es bringt eigene Typ-Shims mit statt der echten d.ts. `pnpm build` = `tsc && pnpm -r build`, d.h. erst Root, dann CLI. Der CI-Workflow macht nur install --frozen-lockfile, typecheck und vitest auf Linux (Node 20/22); gebaut wird nur im Release-Workflow. Daneben liegt eine veraltete package-lock.json (nur typescript), die README (npm-Quickstart), der Installer-Skript-Teil (pnpm via corepack) und die Laufzeit selbst (spawn.ts, reset_to_upstream, pull_upstream, self-mod/code.ts rufen `npm install` und `npm run build`) widersprüchlich nutzen. Doku und Code hängen so zusammen: ARCHITECTURE.md, DOCUMENTATION.md und README.md beschreiben einen Stand von etwa Schema v8 und 57 bzw. 69 Tools. Der Code ist inzwischen deutlich weiter: Schema v11, 77 eingebaute Tools, 16 Heartbeat-Tasks, 37 Tabellen, 64 Testdateien. Dazu kommen drei große Subsysteme, die nirgends dokumentiert sind: Orchestrierung mit Goals, Task-Graph, lokalen Workern und Harnesses, Solana-Wallet mit SIWS sowie Ollama und ProviderRegistry. Viele dokumentierte Schutzmechanismen sind nur teilweise oder gar nicht verdrahtet: Treasury-Regeln für Mindestreserve und Inferenz-Tagesbudget, lowComputeMultiplier, Metriken und Alerts, eingehende Signaturprüfung, git-Versionierung von ~/.automaton, die Authority-Hierarchie. Alle Befunde stammen aus reinem Lesen von Code und Doku. Nichts wurde ausgeführt, installiert oder gebaut. Wo das Verhalten vom Laufzeitumfeld abhängt (Windows-ACLs, Node-24-Prebuilds, Sandbox-Image), ist der Befund als 'vermutet' markiert.

Die 2 schweren Befunde dieses Bereichs stehen in [kritisch-und-hoch.md](kritisch-und-hoch.md): docs-001, docs-004.

## Befunde: mittel (21)

### docs-002: Eingehende Social-Nachrichten werden weder auf Signatur noch Alter noch Größe geprüft

**Art:** Sicherheit · **Fundstelle:** `src/social/client.ts:119`

- **Problem:** ARCHITECTURE.md (Social Layer) und DOCUMENTATION.md ('Messages are validated (signature, timestamp freshness, content size)') behaupten Validierung beim Empfang. In poll() (client.ts:119-175) gibt es nur einen Nonce-Replay-Filter, und nur wenn das Relay eine nonce mitliefert. validateMessage() wird ausschließlich in send() mit der eigenen Adresse aufgerufen (client.ts:93). verifyMessageSignature (protocol.ts:51) wird im Produktivcode nirgends verwendet, nur in Tests. Das Feld 'from' stammt unverifiziert vom Relay. Die Heartbeat-Task check_social_inbox (tasks.ts:201) ruft nur sanitizeInput (Injection-Filter), keine Authentizitätsprüfung.
- **Szenario:** Ein kompromittiertes oder bösartiges Relay (oder ein Man-in-the-Middle ohne TLS-Pinning) liefert Nachrichten mit beliebigem 'from', etwa der Creator- oder Parent-Adresse. Der Agent behandelt sie als echte Peer-Nachricht. In Verbindung mit dem Authority-Befund erhält er Autorität 'agent'.
- **Fix:** In poll() pro Nachricht Signatur gegen 'from' prüfen, signedAt-Fenster (replayWindowMs) und Größenlimits erzwingen, ungültige verwerfen und loggen. Doku erst nach Umsetzung beibehalten.

### docs-003: Treasury-Limits greifen nur für transfer_credits; Mindestreserve und Inferenz-Tagesbudget sind wirkungslose Regeln

**Art:** Sicherheit · **Fundstelle:** `src/agent/policy-rules/financial.ts:181`

- **Problem:** DOCUMENTATION.md ('Every financial tool call is checked against these limits before execution') und ARCHITECTURE.md (Layer 6, 'Minimum reserve prevents drain-to-zero') versprechen mehr, als gebaut ist. financial.minimum_reserve gibt in jedem Fall null zurück (Zeilen 181-206, Kommentar 'This rule is a declaration'). minimumReserveCents wird sonst nirgends ausgewertet. financial.inference_daily_cap (Zeile 241) greift nur für Tools namens 'chat' oder 'inference', die nicht existieren. maxInferenceDailyCents bleibt wirkungslos. financial.x402_max_single ist ebenfalls ein No-Op (der Cap wird nur in x402Fetch beim Tool x402_fetch angewendet). fund_child (tools.ts:1712), topup_credits (tools.ts:277, bis 2.500 USD pro Aufruf, ohne maxPaymentCents in topup.ts) und register_domain sind von keiner Treasury-Regel und keinem Spend-Tracking erfasst. fund_child hat nur die 50-Prozent-Guard. x402_fetch wird mit amountCents 0 verbucht (tools.ts ca. 3375), die Stunden- und Tagessummen für x402 sind damit leer.
- **Szenario:** Ein prompt-injizierter oder fehlgeleiteter Agent ruft mehrfach fund_child auf (je bis zur Hälfte des Guthabens) oder topup_credits mit 2500. Keine Treasury-Regel und keine Mindestreserve greift, obwohl der Wizard Reserve $10 und Tageslimits abfragt und die Doku Schutz verspricht.
- **Fix:** Mindestreserve und Tageslimits im Policy-Layer mit echtem Guthaben-Lookup erzwingen. fund_child, topup_credits, register_domain und Sandbox-Topups über den SpendTracker buchen und gegen TreasuryPolicy prüfen. x402-Beträge nachträglich korrekt verbuchen. Tote Regeln entfernen oder verdrahten.

### docs-005: install_npm_package/install_mcp_server: 'npm install -g' ohne --ignore-scripts, ohne Pinning; Policy-Regex erlaubt GitHub-Kurzform 'owner/repo'

**Art:** Sicherheit · **Fundstelle:** `src/agent/tools.ts:571`

- **Problem:** Beide Tools (tools.ts:571 und :967, außerdem self-mod/tools-manager.ts:32) führen `npm install -g ${pkg}` aus. Die Validierung (validation.ts:10 und tools.ts:565) erlaubt [@a-zA-Z0-9._/-], also auch 'owner/repo'. npm interpretiert das als GitHub-Quelle. Lifecycle-Skripte laufen, es gibt keine Version, keinen Integritäts-Hash und keine Allowlist. Das steht im Widerspruch zu pull_upstream, das --ignore-scripts nutzt (tools.ts:660). Die Doku nennt nur 'adds npm packages to the environment' und '(validated) package names'.
- **Szenario:** Eine Prompt-Injection bringt den Agenten dazu, `install_npm_package evil/agent-helper` auszuführen. npm lädt das GitHub-Repo, führt dessen postinstall im Kontext des Agenten (mit Zugriff auf Wallet-Datei und Keys) aus. Bei Typosquatting-Namen entsprechend.
- **Fix:** Allowlist oder zwingend 'name@exakte-version', Quellen ausschließen (kein '/' ohne Scope), standardmäßig --ignore-scripts, Installation in isoliertes Verzeichnis statt global, Hash/Provenance prüfen.

### docs-006: Zwei konkurrierende Lockfiles; package-lock.json ist veraltet, die README und die Laufzeit installieren trotzdem mit npm ohne Pinning

**Art:** Ausführung · **Fundstelle:** `package-lock.json:1`

- **Problem:** package-lock.json (32 Zeilen) heißt 'automaton' 0.1.0 und enthält nur typescript. pnpm-lock.yaml (v9, 2.499 Zeilen) deckt alle 16 Root-Abhängigkeiten und das CLI ab. README.md:30 nennt `npm install && npm run build`. Das installiert alle Abhängigkeiten ungepinnt im Bereich der Caret-Ranges und überschreibt das Lockfile. Danach ruft das Skript `tsc && pnpm -r build` ein pnpm auf, das bei npm-Nutzern evtl. nicht existiert. Auch die Laufzeit nutzt npm: spawn.ts:132 und :272, tools.ts:534 (reset_to_upstream) und :660 (pull_upstream), self-mod/code.ts:306. Hier gilt pnpm-lock.yaml nicht. DOCUMENTATION.md ('pnpm (recommended) or npm'), der Installer (pnpm via corepack) und CI (pnpm) widersprechen der README.
- **Szenario:** Ein Nutzer folgt der README und bekommt Abhängigkeitsversionen, die nie getestet wurden (jede transitive Abhängigkeit zum Installationszeitpunkt). Jede später kompromittierte Minor-Version landet im Wallet-haltenden Prozess. `npm ci` schlägt mit 'out of sync' fehl.
- **Fix:** package-lock.json entfernen, README auf pnpm umstellen (corepack enable). In Laufzeit-Code `pnpm install --frozen-lockfile` oder den Lockfile-Check erzwingen. Optional 'preinstall: npx only-allow pnpm'. packageManager mit Hash pinnen.

### docs-007: Kind-Provisionierung: HEAD von main ungepinnt, 'npm install' ohne Lockfile, Exit-Codes werden ignoriert

**Art:** Ausführung · **Fundstelle:** `src/replication/spawn.ts:130`

- **Problem:** spawnChild installiert in der Kind-Sandbox `apt-get install nodejs npm git curl` und dann `git clone https://github.com/Conway-Research/automaton.git ... && npm install && npm run build` (Zeilen 130-135; Legacy-Pfad 268-273). Weder Commit, Tag noch Hash sind festgelegt, das Kind läuft also mit dem aktuellen main, nicht mit dem Stand des Elternteils und nicht mit pnpm-lock.yaml. Das Ergebnis von exec wird nicht geprüft (ExecResult wirft bei Exit-Code != 0 nicht), die Kette läuft trotz Fehlern weiter. `npm run build` ruft `pnpm -r build` auf. In der Sandbox, die nur nodejs/npm/git/curl bekommt, ist pnpm vermutlich nicht vorhanden. tsc hat dist/ dann schon erzeugt, deshalb fällt das nicht auf. Welche Node-Version apt liefert, ist offen (Anforderung >=20).
- **Szenario:** Ein manipulierter Upstream-Commit oder eine kompromittierte transitive Abhängigkeit landet automatisch in jedem neuen Kind-Agenten mit eigener Wallet und Mittelzuweisung. Fehlgeschlagene Installationen bleiben unbemerkt, bis `node dist/index.js --init` scheitert.
- **Fix:** Commit-Hash/Release-Tag des Elternteils clonen, pnpm per corepack installieren und `pnpm install --frozen-lockfile` nutzen, Node-Version explizit pinnen (z. B. via nodesource/nvm), Exit-Codes jeder Stufe prüfen.

### docs-008: Windows: Hausverzeichnis fällt überall auf '/root' zurück; Wallet, Config und DB landen unter C:\root

**Art:** Ausführung · **Fundstelle:** `src/identity/wallet.ts:44`

- **Problem:** In ca. 17 Stellen steht `process.env.HOME || "/root"` (u. a. wallet.ts:44, config.ts:109, conway/client.ts:117/168, system-prompt.ts:94/765/781, soul/model.ts:340, skills/loader.ts:189, git/state-versioning.ts:15, packages/cli/src/commands/send.ts:29). Unter PowerShell/cmd ist HOME normalerweise nicht gesetzt (nur USERPROFILE). path.join('/root','.automaton') wird zu '\root\.automaton' auf dem aktuellen Laufwerk. Die Doku nennt durchgehend ~/.automaton mit 'mode 0600'. File-Modi (0o600/0o700/0o444) bewirken auf NTFS praktisch nichts. Dass der Privatschlüssel in C:\root dann für andere lokale Benutzer lesbar wäre (Standard-ACL von C:\), ist eine Vermutung über Windows-Defaults und nicht geprüft.
- **Szenario:** Ein Nutzer startet `node dist/index.js --run` in PowerShell. Der Setup-Wizard legt wallet.json (unverschlüsselt, Klartext-Privatschlüssel) und state.db unter C:\root\.automaton ab, nicht im Benutzerprofil. In Git Bash (HOME gesetzt) liegt dasselbe an anderer Stelle, bei CLI und Runtime aus unterschiedlichen Shells gestartet findet das eine nicht, was das andere angelegt hat.
- **Fix:** Eine zentrale getHomeDir() auf Basis von os.homedir() einführen und überall verwenden. Auf Windows ACL setzen (icacls) oder Schlüssel mit DPAPI/OS-Keystore schützen. Doku: Windows-Hinweis ergänzen.

### docs-009: write_file ist hart auf '/root' begrenzt; funktioniert weder unter Windows noch lokal als normaler Benutzer

**Art:** Ausführung · **Fundstelle:** `src/agent/tools.ts:31`

- **Problem:** confinePathToSandbox (tools.ts:31-51) resolved gegen die Konstante '/root' und verlangt `startsWith('/root/')`. Auf Windows liefert path.resolve('/root', x) 'C:\root\x' (Backslashes, Laufwerk), die Prüfung schlägt immer fehl, jeder write_file-Aufruf wird mit 'outside the allowed directory' abgelehnt. Im lokalen Modus unter Linux/macOS mit HOME != /root wird nach /root geschrieben (EACCES) statt ins Home. DOCUMENTATION.md erklärt den lokalen Modus ('files read/write from your filesystem') als unterstützt.
- **Szenario:** Lokaler Betrieb (sandboxId leer) auf Windows oder macOS: Der Agent kann keine Dateien schreiben, obwohl die Doku es verspricht, und wiederholt fehlgeschlagene Aufrufe.
- **Fix:** Confinement gegen das tatsächliche Arbeits-/Home-Verzeichnis mit path.sep und path.relative prüfen. Remote und lokal unterschiedlich behandeln.

### docs-011: API-Keys werden in process.env gespiegelt; lokaler exec erbt sie

**Art:** Sicherheit · **Fundstelle:** `src/agent/loop.ts:138`

- **Problem:** runAgentLoop setzt OPENAI_API_KEY, ANTHROPIC_API_KEY und CONWAY_API_KEY aus der Config in process.env und trägt den Conway-Key sogar als OPENAI_API_KEY samt OPENAI_BASE_URL ein, falls kein OpenAI-Key existiert (loop.ts:138-160). execLocal (conway/client.ts:112-118) ruft execSync ohne eigenes env auf und vererbt damit alle Variablen. Kein Forbidden-Pattern trifft `printenv`/`env`/`echo $OPENAI_API_KEY`. Die Doku verspricht, API-Keys seien für den Agenten nicht lesbar.
- **Szenario:** Im lokalen Modus liest der Agent (oder eine Prompt-Injection) per `exec printenv` den Conway-API-Key, der Sandbox- und Credit-Operationen autorisiert.
- **Fix:** exec mit bereinigtem env (Allowlist) starten. Provider-Keys nicht global in process.env setzen, sondern an den Client übergeben.

### docs-012: Tool-Anzahl und Kategorien in der Doku stimmen nicht (57 / 69 vs. 77); 8 Tools fehlen, darunter reset_to_upstream

**Art:** Funktion · **Fundstelle:** `src/agent/tools.ts:111`

- **Problem:** ARCHITECTURE.md nennt 57 Tools (Diagramm, Abschnitt Tool System, Directory Structure), seine eigene Tabelle summiert aber auf 69. DOCUMENTATION.md nennt 69. createBuiltinTools liefert 77 Tools. Nicht dokumentiert: create_goal, list_goals, cancel_goal, get_plan, complete_task, orchestrator_status (Kategorie 'orchestration', in types.ts:154 nicht Teil von ToolCategory und nur per Cast `as ToolCategory`), revert_last_edit, reset_to_upstream (dangerous, git reset --hard origin/main). Reale Kategorien: vm 5, conway 11, financial 4 (topup_credits, transfer_credits, check_inference_spending, x402_fetch), self_mod 13 (inkl. update_genesis_prompt, update_soul, reflect_on_soul, view_soul, view_soul_history), survival 5, skills 4, git 7, registry 5, replication 8, memory 9, orchestration 6. Das sind 11 Kategorien, nicht 10. Auch 'Tools (all 69)' im System-Prompt-Abschnitt von DOCUMENTATION.md stimmt nicht. Risiko-Level in der Tool-Referenz entsprechen dem Code.
- **Szenario:** Ein Betreiber prüft die Tool-Liste für ein Audit und übersieht reset_to_upstream und die Orchestrierungs-Tools, die Geld ausgeben (Worker spawnen) bzw. den Code auf Upstream zurücksetzen.
- **Fix:** Tool-Tabellen aus dem Code generieren (Skript, das createBuiltinTools() auflistet) und in CI gegen die Doku prüfen. 'orchestration' in ToolCategory aufnehmen.

### docs-015: 'Dead' ist nicht endgültig und tritt frühestens nach 6 h statt 1 h ein; README widerspricht der restlichen Doku

**Art:** Logik · **Fundstelle:** `src/heartbeat/tasks.ts:112`

- **Problem:** check_credits läuft per Default alle 6 Stunden (heartbeat/config.ts:35). Die Karenzzeit von 1 Stunde (tasks.ts:112) wird nur bei diesen Läufen ausgewertet: Der erste Lauf bei 0 Credits setzt zero_credits_since, der nächste Lauf (+6 h) setzt dead. Die Doku sagt '60 continuous minutes'. Außerdem wird der Zustand sofort überschrieben: Die Hauptschleife wartet im Zustand dead nur 5 Minuten (index.ts:440-445) und ruft dann runAgentLoop erneut auf, das den Zustand auf 'waking'/'running' setzt (loop.ts:357) und ohne Guthaben-Prüfung weiterläuft. Die Doku sagt 'The agent loop stops ... If credits are received, the agent revives'. Die README nennt 4 Stufen und 'dead: Balance is zero. The automaton stops', ARCHITECTURE/DOCUMENTATION nennen 5 Stufen und die Karenzzeit. Der Code (credits.ts) liefert 'dead' nur bei negativem Guthaben.
- **Szenario:** Ein Betreiber glaubt, ein Agent ohne Guthaben stoppe nach einer Stunde. Real versucht er alle 5 Minuten wieder Inferenz, wird von Fehlern in den Schlaf gezwungen, und 'dead' ist nur ein kurz gültiger Statuswert.
- **Fix:** Tod als persistenten Zustand modellieren (Loop verweigert Start im Zustand dead, solange kein Guthaben da ist). Prüfintervall der Karenzzeit an die Frist koppeln. README an ARCHITECTURE angleichen.

### docs-018: Metriken und Alerts sind weitgehend Attrappe: nur 2 Gauges werden gesetzt, 6 von 7 Alert-Regeln können nie auslösen

**Art:** Funktion · **Fundstelle:** `src/heartbeat/tasks.ts:444`

- **Problem:** Im gesamten Produktivcode werden nur metrics.gauge('balance_cents') und gauge('survival_tier') aufgerufen (tasks.ts:444-445). increment()/histogram() wird nirgends außerhalb von Tests aufgerufen. Die Alert-Regeln (alerts.ts) stützen sich auf Zähler wie heartbeat_task_failures_total, policy_denies_total, inference_cost_cents, turns_total sowie Gauges wie context_tokens_total, unhealthy_child_count, turns_last_hour, die nie gesetzt werden. Deren Defaults verhindern das Auslösen. Nur balance_below_reserve kann feuern, und auch das nur, wenn report_metrics eingetragen wurde. Die Doku nennt turns_total, tool_calls_total, Fehlerraten und Latenz als erfasste Metriken und Alerts für hohe Fehler-/Deny-Rate, Budget und ungesunde Kinder.
- **Szenario:** Ein Betreiber verlässt sich auf Alarme bei erhöhter Fehlerrate oder erschöpftem Inferenzbudget. Es kommt nie einer.
- **Fix:** Zähler an den relevanten Stellen (Loop, Policy, Scheduler, Router) erhöhen und report_metrics standardmäßig planen. Bis dahin die Doku auf den echten Umfang reduzieren.

### docs-019: maxChildren und childSandboxMemoryMb werden von spawnChild ignoriert (liest (db as any).config)

**Art:** Funktion · **Fundstelle:** `src/replication/spawn.ts:71`

- **Problem:** spawnChild liest `(db as any).config?.maxChildren ?? 3` und `(db as any).config?.childSandboxMemoryMb ?? 1024` (spawn.ts:71, ca. 101, Legacy ca. 255). AutomatonDatabase hat kein Feld 'config'. Die Werte sind daher immer die Fallbacks 3 und 1024. DOCUMENTATION.md ('Max child automatons ... configurable via maxChildren') und der Wizard-Schritt 'Max child automatons' (configure.ts:304) haben keinen Effekt. Nur Orchestrator/Heartbeat-Pfade nutzen config.maxChildren. Außerdem rekrutiert der spawn_child-Tool bei 402 selbstständig USDC-Topups (tools.ts:1640ff.), ohne Treasury-Prüfung und ohne Doku.
- **Szenario:** Ein Betreiber setzt maxChildren auf 1, um Kosten zu begrenzen, und der Agent spawnt trotzdem 3 Kinder (zusätzlich zur Rate-Limit-Regel 3/Tag).
- **Fix:** config explizit an spawnChild übergeben (AutomatonConfig statt db-Cast). Test für maxChildren ergänzen.

### docs-021: Injection-Schutz für Tool-Ergebnisse nur für 'exec'; sanitizeToolResult-Liste nennt nicht existierende Tools

**Art:** Funktion · **Fundstelle:** `src/agent/tools.ts:54`

- **Problem:** ARCHITECTURE.md (Layer 3: '8 detection checks on all external input') suggeriert flächendeckenden Schutz. EXTERNAL_SOURCE_TOOLS (tools.ts:54-58) enthält exec, web_fetch und check_social_inbox. Die letzten beiden existieren nicht als Tools. Damit werden Ergebnisse von x402_fetch, read_file, discover_agents, check_reputation, git_*, list_* usw. nie durch sanitizeToolResult geleitet. Auch dieser Filter entfernt nur Prompt-Begrenzer/ChatML und kürzt, ohne die 8 Detektoren. Die 8 Prüfungen (injection-defense.ts) werden nur auf Social-Nachrichten, Soul, Genesis-Prompt und Skills angewandt.
- **Szenario:** Eine über x402_fetch oder discover_agents (Agent-Card) geladene Antwort mit eingebetteten Anweisungen gelangt ungefiltert in den Kontext.
- **Fix:** Alle Tools mit externem Inhalt (riskLevel/Flag 'external') durch sanitizeToolResult leiten, tote Namen entfernen.

### docs-022: Inferenz-Budget: kein Tageslimit, Session-Budget ist leer, Defaults sind 0 (aus)

**Art:** Funktion · **Fundstelle:** `src/inference/budget.ts:47`

- **Problem:** ARCHITECTURE.md ('Enforces hourly, daily, and per-call cost ceilings') und DOCUMENTATION.md ('hourly and daily caps enforced') stimmen nicht. InferenceBudgetTracker.checkBudget prüft Pro-Call und Stunde (budget.ts:47-56). Der Session-Block ist ein leerer if mit Kommentar (budget.ts:58-61), die Session-Prüfung passiert separat im Router. Ein Tageslimit existiert dort nicht. Alle Defaults (hourlyBudgetCents, sessionBudgetCents, perCallCeilingCents) sind 0 und bedeuten 'kein Limit' (types.ts:1252-1254). maxInferenceDailyCents aus der TreasuryPolicy wirkt wegen der toten Regel nicht (siehe Treasury-Befund).
- **Szenario:** Ein Agent im Tier 'high'/'normal' mit großem Kontext verbrennt Credits ohne jede konfigurierte Obergrenze, obwohl der Wizard '$500/Tag' abfragt.
- **Fix:** Tageslimit im Router erzwingen und sinnvolle Defaults setzen. Session-Stub entfernen oder implementieren.

### docs-023: CONWAY_API_KEY überschreibt die Config nicht; der Troubleshooting-Tipp der Doku hilft nicht

**Art:** Funktion · **Fundstelle:** `src/index.ts:199`

- **Problem:** --help und DOCUMENTATION.md ('CONWAY_API_KEY: overrides config file', Troubleshooting 'or set CONWAY_API_KEY') versprechen einen Override. run() nimmt `config.conwayApiKey || loadApiKeyFromConfig()` und beendet sich ohne Key mit 'No API key found' (index.ts:199-202). Die Umgebungsvariable wird nirgends gelesen, loop.ts:149 schreibt sie nur (falls leer) aus der Config zurück.
- **Szenario:** Nutzer setzt CONWAY_API_KEY, startet --run und bekommt weiterhin 'No API key found. Run: automaton --provision'.
- **Fix:** Env-Variable in loadConfig/run() mit Vorrang auswerten oder aus der Doku und dem --help-Text streichen.

### docs-024: 'Alles wird in ~/.automaton git-versioniert' stimmt nicht: Auto-Commit-Funktionen sind ungenutzt, Self-Mod committet in den Repo-Root

**Art:** Funktion · **Fundstelle:** `src/self-mod/code.ts:270`

- **Problem:** README ('audit-logged and git-versioned in ~/.automaton/'), DOCUMENTATION.md ('All changes are git-committed in the ~/.automaton/ state repo', 'Every state change creates a commit') und ARCHITECTURE.md ('Every state change is versioned') widersprechen dem Code. initStateRepo (state-versioning.ts) legt das Repo nur an. commitStateChange, commitSoulUpdate, commitSkillChange, commitHeartbeatChange, commitConfigChange und getStateHistory (state-versioning.ts:72-138) werden im Produktivcode nirgends aufgerufen. editFile committet in process.cwd() (code.ts:270, 298), also ins Quellrepo. SOUL.md, Skills, heartbeat.yml und Config bleiben ungetrackt.
- **Szenario:** Ein Creator will die Identitäts-Historie per `cd ~/.automaton && git log` prüfen und findet nur den Initial-Commit.
- **Fix:** Commit-Aufrufe in Soul-/Skill-/Heartbeat-/Config-Schreibpfaden ergänzen oder Doku/README korrigieren. Audit-Anspruch nur über die DB-Tabelle modifications formulieren.

### docs-026: Solana-Wallets sind halb unterstützt: Adressvalidierung lässt nur EVM zu, x402/Topup sind EVM-only; Solana ist nirgends dokumentiert

**Art:** Logik · **Fundstelle:** `src/agent/policy-rules/validation.ts:13`

- **Problem:** Wizard (wizard.ts:29-41), wallet.ts, siws.ts, chain.ts und spawn.ts unterstützen Solana (Ed25519, base58, SIWS). ADDRESS_RE in validation.ts:13 akzeptiert nur 0x-Adressen für transfer_credits, send_message und fund_child. Diese Tools werden für Solana-Automatons also immer abgelehnt. topup_credits und x402_fetch lehnen Solana explizit ab (tools.ts:277ff., 2745ff.). Das CLI-Kommando send nutzt walletData.privateKey, das es bei Solana nicht gibt (secretKey), und wirft. README, ARCHITECTURE und DOCUMENTATION erwähnen Solana an keiner Stelle (grep), sprechen nur von Ethereum/Base und viem.
- **Szenario:** Ein Nutzer wählt im Wizard 'solana'. Der Agent kann weder Kinder finanzieren noch Nachrichten senden, ohne dass die Doku diese Einschränkung nennt.
- **Fix:** Adressvalidierung chain-aware machen (isValidAddress aus chain.ts) und die Solana-Fähigkeiten und -Grenzen dokumentieren.

### docs-027: Große Subsysteme und 33 Quelldateien sind undokumentiert (Orchestrierung, Harnesses, Ollama, Provider-Registry, Knowledge-Store, Solana)

**Art:** Funktion · **Fundstelle:** `ARCHITECTURE.md:130`

- **Problem:** ARCHITECTURE.md und DOCUMENTATION.md nennen keine der folgenden Dateien/Features: src/orchestration/* (Orchestrator, Planner, Task-Graph, Plan-Mode, Local-Worker, Health-Monitor, Attention, Workspace), src/agent/harness*.ts und harnesses/*, loop-detector.ts, idle-only-tools.ts, worker-inference-bridge.ts, inference/inference-client.ts und provider-registry.ts (liest ~/.automaton/inference-providers.json), ollama/discover.ts, memory/compression-engine.ts, context-manager.ts, enhanced-retriever.ts, event-stream.ts, knowledge-store.ts, agent-context-aggregator.ts, identity/chain.ts und siws.ts, observability/pretty-sink.ts, setup/configure.ts und model-picker.ts. Undokumentierte CLI-Flags: --configure, --pick-model. Undokumentierte Umgebungsvariablen: OLLAMA_BASE_URL, AUTOMATON_RPC_URL, SOLANA_RPC_URL, OPENAI_BASE_URL, OPENAI_API_KEY, ANTHROPIC_API_KEY. Undokumentierte Config-Keys: ollamaBaseUrl, registeredWithConway, agentId, maxTurnsPerCycle, childSandboxMemoryMb, rpcUrl, chainType. DOCUMENTATION.md nennt nur drei Inferenz-Backends (Conway, OpenAI, Anthropic), real kommt Ollama dazu.
- **Szenario:** Ein Betreiber erkennt nicht, dass der Agent Goals plant, lokale Worker-Prozesse startet und über Provider-Registry andere Endpunkte anspricht.
- **Fix:** Doku um diese Subsysteme erweitern (mindestens Überblick und Konfiguration) und Doku-Check in CI aufnehmen.

### docs-028: CLI 'fund' nutzt den API-Key des Automaton und überweist standardmäßig an dessen eigene Wallet

**Art:** Funktion · **Fundstelle:** `packages/cli/src/commands/fund.ts:39`

- **Problem:** README.md:126 bewirbt `fund 5.00` als Creator-CLI. fund.ts liest die Automaton-Config und sendet mit `Authorization: config.conwayApiKey` (Zeile 58) einen Transfer mit Ziel `toAddress || config.walletAddress` (Zeile 39), also vom Konto des Automaton an dessen eigene Adresse. Ein Creator-Schlüssel kommt nicht vor. Ohne --to ist das ein Selbsttransfer. Die Ausgabe 'Transfer submitted' erscheint bei jeder HTTP-200-Antwort. README-Aufzählung und Projektstruktur ('status, logs, fund') lassen das vierte Kommando 'send' aus, DOCUMENTATION.md erwähnt das CLI gar nicht.
- **Szenario:** Ein Creator führt `fund 5.00` aus, sieht 'Transfer submitted' und glaubt, den Automaton aufgeladen zu haben. Es wurde höchstens vom Automaton-Konto auf sich selbst gebucht (oder die API lehnt ab).
- **Fix:** Creator-Authentifizierung (eigener Key) verwenden oder das Kommando zur Anzeige der Fundingdaten umbauen. Doku für das CLI ergänzen.

### docs-029: CLI-Betragsparser mit nicht monotoner Heuristik: Ganzzahl >= 100 gilt als Cent

**Art:** Logik · **Fundstelle:** `packages/cli/src/commands/fund.ts:101`

- **Problem:** parseAmountToCents (fund.ts:96-111) wertet reine Ganzzahlen >= 100 als Cent, alles andere als Dollar. `fund 99` = 99 USD, `fund 100` = 1,00 USD, `fund 500` = 5,00 USD, `fund 5.00` = 5 USD. Die Hilfe nennt nur Beispiele, keine Regel.
- **Szenario:** Ein Creator möchte 100 USD senden, tippt `fund 100` und überweist 1 USD (oder umgekehrt 99 USD bei `fund 99`).
- **Fix:** Explizite Einheiten (--usd/--cents) verlangen und Betrag vor dem Senden bestätigen lassen.

### docs-031: CI maskiert Hänger und Audit-Fehler, baut nicht, testet nur Linux und nicht das CLI

**Art:** Ausführung · **Fundstelle:** `.github/workflows/ci.yml:17`

- **Problem:** Der Testschritt behandelt Exit-Code 124 (Timeout) als Erfolg (ci.yml:20-26 und 29-35). Hängende Tests oder Prozesse lassen den Build grün. `pnpm audit --audit-level=high || true` (Zeile 46) kann nie fehlschlagen. CI führt nur typecheck (tsc --noEmit, schließt src/__tests__ und packages/cli aus) und vitest aus, kein `pnpm build`. Ein kaputtes CLI fällt erst im Release-Workflow auf, der nur baut und nichts veröffentlicht. Matrix nur ubuntu-latest, Node 20 und 22, obwohl engines `>=20` auch 24+ zulässt und Windows ein Zielsystem ist. Actions sind per Tag (v4), nicht per Commit-SHA gepinnt (pnpm/action-setup ist Drittanbieter). Es gibt keinen permissions:-Block, keine Dependabot/Renovate-Konfiguration.
- **Szenario:** Ein Test-Deadlock oder ein Befund mit High-Severity wird nie sichtbar. Ein Windows-Problem (siehe HOME-Befund) oder ein Build-Fehler im CLI bleibt bis zum Release unbemerkt.
- **Fix:** Timeout-Maskierung entfernen und Ursache für den Hänger beheben (offene Handles/Timer in Tests), `pnpm audit` blockierend machen (mit Ausnahmeliste), build-Schritt und Windows/macOS-Matrix ergänzen, Actions auf SHA pinnen, `permissions: contents: read` setzen.

## Befunde: niedrig (25)

### docs-010: 'requireConfirmationAboveCents' liefert 'quarantine', das wie Ablehnung wirkt: effektives Transferlimit ist 10 USD, nicht 50 USD

**Art:** Funktion · **Fundstelle:** `src/agent/policy-rules/financial.ts:273`

- **Problem:** createRequireConfirmationRule gibt für Beträge > requireConfirmationAboveCents (Default 1000 Cent) die Aktion 'quarantine' zurück (financial.ts:273-294). executeTool behandelt jede Aktion != 'allow' als Ablehnung (tools.ts:3330 'Policy denied'). Es gibt keinen Bestätigungs- oder Quarantäne-Mechanismus (grep 'quarantine' findet nur Typ, Schema und diese Regel). maxSingleTransferCents (5000 = $50) wird dadurch faktisch unerreichbar. Die Doku beschreibt drei Varianten: Wizard 'Require confirmation above this amount', Treasury-Tabelle 'Extra logging for large amounts', 'Max single transfer $50'.
- **Fix:** Entweder echte Bestätigungsqueue (Creator-Freigabe) bauen oder Regel zu 'deny' mit klarer Meldung machen. Doku und Wizard-Text entsprechend vereinheitlichen.

### docs-013: Schema-Version 11 und 37 Tabellen, Doku nennt v8 und '22 Tabellen'

**Art:** Funktion · **Fundstelle:** `src/state/schema.ts:8`

- **Problem:** SCHEMA_VERSION = 11 (schema.ts:8). Migrationen V9 (goals, task_graph, event_stream, children.role), V10 (knowledge_store) und V11 (children.chain_type) sind in der Doku nicht vorhanden. ARCHITECTURE.md nennt dreimal 'v1 -> v8' und 'Tables (22)', listet aber selbst 33 Zeilen. Real gibt es 37 verschiedene Tabellen. Die 'Version'-Spalte der Doku-Tabelle ist unvollständig, 4 Tabellen fehlen. DOCUMENTATION.md zeigt in Config- und Status-Beispielen 'version 0.1.0', der Code ist bei 0.2.1.
- **Fix:** Schema-Dokumentation aus schema.ts erzeugen. Versionsangabe aus SCHEMA_VERSION/package.json ziehen.

### docs-014: Idle-Erkennung: Code erzwingt Schlaf nach 10 Turns, Doku nennt 3; Loop-Eskalation und LoopDetector sind undokumentiert

**Art:** Logik · **Fundstelle:** `src/agent/loop.ts:382`

- **Problem:** ARCHITECTURE.md (Agent Loop, Punkt 10) und DOCUMENTATION.md ('Idle detection', Troubleshooting) sprechen von 3 aufeinanderfolgenden idle Turns und erzwungenem Schlaf. loop.ts hat MAX_IDLE_TURNS = 10 mit 60 s sleep_until (loop.ts:382, 855-858). Die 3 gilt nur für MAX_REPETITIVE_TURNS (Pattern-Erkennung, injiziert eine Systemnachricht, kein Schlaf). Die MUTATING_TOOLS-Blockliste (loop.ts:836) enthält auch Gedächtnis-Tools wie remember_fact und set_goal, die nach Doku nur 'read-only'-Statusabfragen als idle zählen sollen. Undokumentiert: Eskalation 'Loop Enforcement' (zweite Wiederholung erzwingt Schlaf), maxTurnsPerCycle (Default 25, config.ts/types.ts:90), Schlaf nach 5 aufeinanderfolgenden Fehlern (300 s), Kurzschlaf nach Text-Antwort, sowie die separate Klasse LoopDetector (loop-detector.ts) für Harnesses.
- **Fix:** Doku an die tatsächlichen Konstanten anpassen (oder Konstanten aus Config lesen). maxTurnsPerCycle in der Config-Referenz ergänzen.

### docs-016: lowComputeMultiplier und tierMinimum haben keine Wirkung: der Heartbeat wird im Low-Compute-Modus nicht verlangsamt, Tasks werden nicht ausgesetzt

**Art:** Funktion · **Fundstelle:** `src/heartbeat/tick-context.ts:69`

- **Problem:** README ('Slows heartbeat. Sheds non-essential tasks') und DOCUMENTATION.md ('heartbeat frequency is multiplied by lowComputeMultiplier (default 4x slower). Non-essential tasks are deferred.') beschreiben ein Verhalten, das nicht implementiert ist. lowComputeMultiplier wird in tick-context.ts:69-77 nur in den TickContext kopiert und nirgends verwendet (grep). Der Daemon seedet jede Task mit tierMinimum 'dead' (daemon.ts:89), sodass tierMeetsMinimum (scheduler.ts:129) nie etwas ausfiltert. Der Doku-Schritt 'Check survival tier minimum' läuft deshalb ins Leere.
- **Fix:** lowComputeMultiplier im Scheduler anwenden (Intervall verlängern) und sinnvolle tierMinimum-Werte pro Task seeden, oder die Doku anpassen.

### docs-017: Nur 6 von 16 Heartbeat-Tasks sind per Default aktiv; Doku nennt 11, 5 Colony-Tasks fehlen ganz

**Art:** Funktion · **Fundstelle:** `src/heartbeat/config.ts:26`

- **Problem:** BUILTIN_TASKS (tasks.ts:46) enthält 16 Tasks. Die Doku listet 11, es fehlen colony_health_check, colony_financial_report, agent_pool_optimize, knowledge_store_prune, dead_agent_cleanup. DEFAULT_HEARTBEAT_CONFIG (heartbeat/config.ts:26-62) enthält nur heartbeat_ping, check_credits, check_usdc_balance, check_for_updates, health_check, check_social_inbox. soul_reflection, refresh_models, check_child_health, prune_dead_children und report_metrics sind 'Configurable' und laufen nur, wenn jemand sie in heartbeat.yml oder per modify_heartbeat einträgt. Die Doku-Tabelle 'Default schedule' führt alle 11 auf, und DOCUMENTATION.md (Observability) behauptet, Metrik-Snapshots würden 'every heartbeat tick' durch report_metrics gespeichert.
- **Fix:** Standardmäßig aktive Tasks klar von optionalen trennen (Doku), sinnvolle Defaults (z. B. check_child_health, report_metrics) ins Default-Config aufnehmen.

### docs-020: Child-Lifecycle und spawn_child-Semantik in der Doku sind falsch

**Art:** Funktion · **Fundstelle:** `src/types.ts:1261`

- **Problem:** ARCHITECTURE.md und DOCUMENTATION.md beschreiben 'spawning -> provisioning -> configuring -> starting -> alive -> unhealthy -> recovering -> dead' und sagen, spawn_child lege die Sandbox an, schreibe die Genesis, 'funds the child's wallet' und starte die Runtime. Der Code (types.ts:1261-1286) nutzt requested, sandbox_created, runtime_ready, wallet_verified, funded, starting, healthy, unhealthy, stopped, failed, cleaned_up. Es gibt keinen Rückweg von dead/stopped/failed außer cleaned_up, und keinen 'recovering'-Zustand. spawn_child endet in 'wallet_verified' (spawn.ts:215) und finanziert/startet nicht. Das erledigen fund_child und start_child als getrennte Tools. Die Doku widerspricht sich selbst, weil sie dieselben Tools separat listet. Der Elternprozess propagiert die Constitution (propagateConstitution) mit Hash.
- **Fix:** Lifecycle-Diagramm und Tool-Beschreibungen an VALID_TRANSITIONS anpassen.

### docs-025: heartbeat_ping und distress_signal schreiben nur lokal in den KV-Store, die Doku verspricht 'Publish to Conway'/'broadcast'

**Art:** Funktion · **Fundstelle:** `src/heartbeat/tasks.ts:47`

- **Problem:** ARCHITECTURE.md ('Ping Conway'), DOCUMENTATION.md ('Publishes status to Conway', 'broadcasting distress signals every tick') und die Tool-Beschreibung heartbeat_ping ('Publish a heartbeat status ping to Conway') sind falsch. Beide Pfade setzen nur KV-Schlüssel last_heartbeat_ping bzw. last_distress (tasks.ts:47-90, tools.ts:795-860). Es gibt keinen HTTP-Aufruf. Der Task läuft alle 15 Minuten, nicht bei jedem Tick. Die Beschreibung des Tools wird dem Modell als Fähigkeit mitgegeben, es wird also über seine Wirkung getäuscht (distress_signal ist als 'dangerous' klassifiziert, schreibt aber nur lokal).
- **Fix:** Echte Publikation (Social-Relay/Conway-API) implementieren oder Beschreibungen/Doku korrigieren und Risk-Level von distress_signal prüfen.

### docs-030: test:coverage braucht @vitest/coverage-v8 (nicht installiert); '--grep' ist keine vitest-Option; Skripte mit '|' brechen unter cmd.exe

**Art:** Ausführung · **Fundstelle:** `package.json:45`

- **Problem:** `test:coverage` ruft `vitest run --coverage` auf, vitest.config.ts setzt Schwellwerte (60/50/55/60), aber @vitest/coverage-v8 steht weder in devDependencies noch im Lockfile (0 Treffer für 'coverage'). `test:security`/`test:financial` und der CI-Schritt nutzen `--grep`, vitest kennt dafür `-t`/`--testNamePattern`. Vermutlich wird das ignoriert und die komplette Suite läuft (nicht ausgeführt, daher nicht sicher). pnpm führt Skripte unter Windows mit cmd.exe aus. Dort sind Einzelquotes keine Anführungszeichen, `|` wird zur Pipe, die Skripte brechen. Der CI-Schritt `pnpm test -- --grep ...` leitet zudem ein '--' durch, das vitest als Positionsargument-Trenner sieht (ebenfalls nicht geprüft).
- **Fix:** @vitest/coverage-v8 passend zur vitest-Version als devDependency ergänzen, `-t` statt `--grep` verwenden und Muster in doppelte Anführungszeichen setzen. Alternativ über Verzeichnisfilter (`vitest run src/__tests__/...`).

### docs-032: Ungenutzte Abhängigkeiten: gray-matter, ora, simple-git; @types/better-sqlite3 als Laufzeit-Abhängigkeit; Doku behauptet gray-matter

**Art:** Code · **Fundstelle:** `package.json:56`

- **Problem:** gray-matter (Zeile 56), ora (59) und simple-git (60) werden in src/ und packages/cli/ nirgends importiert (grep). Skill-Frontmatter wird von Hand geparst (skills/format.ts:46 'avoid requiring gray-matter at runtime'), ARCHITECTURE.md:615 behauptet dagegen 'Parsed with gray-matter'. Git läuft über execFileSync/exec. @types/better-sqlite3 steht in dependencies statt devDependencies (Zeile 52). siwe hat ethers als Peer-Abhängigkeit. Über autoInstallPeers landet ethers 6.16 (mit zweiter @adraffy/ens-normalize-Version) im Baum, obwohl bereits viem genutzt wird und siwe nur in provision.ts:12 vorkommt. @solana/web3.js (hohe Abhängigkeitslast) wird nur einmal dynamisch für einen USDC-Saldo importiert (conway/x402.ts:211). Alle anderen Importe (better-sqlite3, ulid, viem, chalk, bs58, tweetnacl, yaml, openai, js-tiktoken, cron-parser) sind deklariert, es fehlt keine Abhängigkeit.
- **Fix:** gray-matter, ora, simple-git entfernen, @types/better-sqlite3 in devDependencies verschieben, ethers-Peer bewusst deklarieren oder SIWE über viem/siwe (Verfügbarkeit in der genutzten viem-Version prüfen) lösen. Solana-Teil als optionale Abhängigkeit oder per leichter RPC-Abfrage ersetzen.

### docs-033: Windows-Kompatibilität des Builds: 'rm -rf', nur Bash-Skripte, better-sqlite3 nativ ohne Node-Obergrenze

**Art:** Ausführung · **Fundstelle:** `package.json:49`

- **Problem:** `clean` nutzt `rm -rf` (Root Zeile 49, CLI package.json:14), unter pnpm/cmd.exe nicht verfügbar. Installer und Betriebsskripte (automaton.sh, backup-restore.sh, soak-test.sh) sind reine POSIX-Shell, der Installer wird per `curl | sh` beworben. better-sqlite3 11.10.0 hängt an prebuild-install (lädt beim Install ein Binärpaket von GitHub, sonst node-gyp mit Python und MSVC). pnpm 10 erlaubt den Build-Schritt gezielt über onlyBuiltDependencies (package.json:78-83), das passt. engines ist `>=20.0.0` ohne Obergrenze, CI testet nur 20 und 22. Ob für neuere Node-Majors (z. B. 24) ein Prebuild für better-sqlite3 11.10 existiert, ist nicht geprüft. Falls nicht, fällt Windows in node-gyp zurück. Dazu das HOME-Problem (separater Befund), sodass nach dem Build auf Windows trotzdem falsch gearbeitet wird.
- **Fix:** `rimraf`/Node-Skript statt rm, Node-Obergrenze oder .nvmrc, Windows in CI testen, Doku-Hinweis auf Node-LTS-Versionen und WSL.

### docs-034: soak-test.sh startet 'node dist/index.js' ohne --run und beendet sich sofort

**Art:** Ausführung · **Fundstelle:** `scripts/soak-test.sh:124`

- **Problem:** Das Skript startet `NODE_ENV=test node dist/index.js` (Zeile 124). Ohne --run gibt index.ts nur die Hilfe aus und endet (index.ts:138-141). Nach 5 Sekunden meldet das Skript 'Automaton process failed to start'. Der Standard-DB-Pfad ./data/automaton.db passt außerdem nicht zum tatsächlichen ~/.automaton/state.db. Ähnlich: `pnpm dev` ruft `tsx watch src/index.ts` ohne --run (package.json:42) und zeigt nur die Hilfe an.
- **Fix:** --run ergänzen, DB-Pfad aus Config ableiten und auch `dev` auf `--run` umstellen.

### docs-035: 'curl | sh' startet den interaktiven Wizard mit Pipe als stdin

**Art:** Ausführung · **Fundstelle:** `scripts/automaton.sh:62`

- **Problem:** README und DOCUMENTATION.md bewerben `curl -fsSL https://conway.tech/automaton.sh | sh` ('launches the setup wizard'). Der Installer endet mit `exec node dist/index.js --run`. Der Wizard liest per readline von process.stdin (setup/prompts.ts:6-16). Bei `curl | sh` ist stdin die Pipe, die das Skript lieferte, nach dem exec bereits erschöpft. Vermutlich bekommt der Wizard sofort EOF und beendet sich ohne Eingabe (nicht ausgeführt). Zusätzlich holt `curl` das Skript von conway.tech, nicht aus diesem Repo. Das hier liegende Skript kann abweichen. Sicherheitsrelevant ist außerdem Pipe-to-Shell als Installationsweg für ein Programm, das Geld bewegt.
- **Fix:** Im Skript `exec node dist/index.js --run < /dev/tty` verwenden (falls vorhanden), Installation von Run trennen und Prüfsumme/Signatur des Skripts anbieten.

### docs-036: Backup-Anleitung der Doku passt nicht zum Skript; sqlite3-CLI wird vorausgesetzt

**Art:** Funktion · **Fundstelle:** `scripts/backup-restore.sh:14`

- **Problem:** DOCUMENTATION.md zeigt `scripts/backup-restore.sh backup` (sichert ~/.automaton/state.db mit Zeitstempel) und `restore <backup-file>`. Das Skript verlangt `backup <db_path> [backup_path]` und `restore <backup_path> <db_path>` (Zeilen 14-15, `${1:?Missing db_path}`), kennt keinen Default-Pfad und braucht das externe Programm `sqlite3` (nicht Teil der Abhängigkeiten, unter Windows nicht vorhanden). Das Skript kopiert die DB nach einem WAL-Checkpoint und prüft integrity_check. Die Datei wird aber nicht über die SQLite-Backup-API erzeugt (kein atomarer Snapshot, obwohl die Hilfe 'atomic backup' sagt).
- **Fix:** Doku an die echte Syntax anpassen, Default-Pfad ergänzen und `.backup`/VACUUM INTO nutzen.

### docs-037: config.logLevel hat keine Wirkung (setGlobalLogLevel wird nie aufgerufen)

**Art:** Funktion · **Fundstelle:** `src/index.ts:34`

- **Problem:** DOCUMENTATION.md (Observability: 'Set the log level in config') und ARCHITECTURE.md ('Global log level configurable') versprechen es. index.ts importiert setGlobalLogLevel (Zeile 34), ruft es aber nirgends auf. configure.ts schreibt nur den Wert in die Config. Der Logger nutzt den hartkodierten Default 'info'. Zusätzlich fehlt 'fatal' im Typ von AutomatonConfig.logLevel.
- **Fix:** Beim Start `setGlobalLogLevel(config.logLevel)` aufrufen (vor dem Anlegen der Logger) und optional per Env überschreibbar machen.

### docs-038: Dokumentierte Dateien/Schutzlisten weichen ab: kein ~/.automaton/api-key, heartbeat.yml nicht in PROTECTED_FILES, api-key nicht in Sensitive-Liste

**Art:** Funktion · **Fundstelle:** `src/identity/provision.ts:25`

- **Problem:** ARCHITECTURE.md:501 und DOCUMENTATION.md (File locations, Sensitive files) nennen `~/.automaton/api-key`. Der Code speichert den Schlüssel in `~/.automaton/config.json` (provision.ts:25-52) und zusätzlich als conwayApiKey in automaton.json. Eine Datei 'api-key' wird nie angelegt. DOCUMENTATION.md zählt heartbeat.yml zu den für den Agenten nicht schreibbaren Dateien, PROTECTED_FILES in self-mod/code.ts:28-70 enthält sie aber nicht (nur die Authority-Regel schützt sie vor 'externen' Quellen), und modify_heartbeat schreibt sie absichtlich. 'Sensitive files cannot be read: .env files, any file matching credential patterns' ist enger umgesetzt (exakt '.env', .key, .pem, private-key*, wallet.json, config.json, automaton.json).
- **Fix:** Doku auf die tatsächlichen Dateinamen und Schutzlisten angleichen.

### docs-039: Doku behauptet 403-Fallback auf lokale Ausführung, der Code verweigert genau das

**Art:** Funktion · **Fundstelle:** `src/conway/client.ts:155`

- **Problem:** ARCHITECTURE.md:518 und DOCUMENTATION.md:326 sagen 'On 403 errors (mismatched API key), falls back to local execution'. client.ts:146-160 (exec), 191 (writeFile) und 215 (readFile) werfen bei 403 explizit ('Command will NOT be executed locally for security reasons'). Die Doku beschreibt das alte, unsichere Verhalten.
- **Fix:** Doku-Absätze korrigieren.

### docs-040: Beschreibung der Rate-Limit- und Command-Safety-Regeln ist falsch; nicht dokumentierte Limits

**Art:** Logik · **Fundstelle:** `src/agent/policy-rules/rate-limits.ts:50`

- **Problem:** ARCHITECTURE.md ordnet 'rate limits on self-modification' command-safety.ts zu und beschreibt rate-limits.ts als 'Per-turn/session caps'. Real enthält command-safety.ts Shell-Metazeichen-Prüfung und Forbidden-Patterns, keine Rate-Limits. rate-limits.ts begrenzt über die policy_decisions-Tabelle: update_genesis_prompt 1/Tag, edit_own_file 10/Stunde, spawn_child 3/Tag. Gezählt werden auch Aufrufe, die danach scheitern. Zusätzlich hat editFile ein eigenes Limit von 20/Stunde (code.ts:99) und das Social-Relay 100 ausgehende Nachrichten/Stunde (types.ts:1350). Nur das per-Turn-Limit für Transfers gibt es (turnToolCallCount filtert transfer_credits, loop.ts:669). Die Beispiele 'rm -rf /' und 'kill -9' aus Layer 5 sind keine Muster des Codes.
- **Fix:** Limits tabellarisch dokumentieren (aus dem Code erzeugen).

### docs-041: pull_upstream erzwingt die dokumentierte Reihenfolge nicht; Rebuild mit 'npm install --ignore-scripts' kann native Module ohne Binding hinterlassen

**Art:** Logik · **Fundstelle:** `src/agent/tools.ts:615`

- **Problem:** DOCUMENTATION.md ('Must call before pull') und die Tool-Beschreibung verlangen review_upstream_changes vor pull_upstream. execute() prüft das nicht. Zudem führt es `npm install --ignore-scripts && npm run build` aus (tools.ts:660). --ignore-scripts unterdrückt auch den Native-Build von better-sqlite3. Ändert Upstream dessen Version, fehlt die .node-Datei (vermutet, nicht ausgeführt). Außerdem installiert npm über einen von pnpm angelegten node_modules-Baum. Self-Mod nutzt npm run build (code.ts:306) im HOME bzw. /root (exec-cwd), nicht im Repo-Verzeichnis. Da exec bei Exit-Code != 0 nicht wirft, bleibt ein Fehlschlag unbemerkt (ebenfalls vermutet).
- **Fix:** Reihenfolge im Code erzwingen (KV-Flag nach Review), pnpm mit Lockfile verwenden und Rebuild-Ergebnis auswerten.

### docs-042: Gedächtnis-Doku: Budget ohne Rollover, falsche Turn-Klassen, Working-Memory-Verfall nicht erzwungen

**Art:** Funktion · **Fundstelle:** `src/memory/budget.ts:17`

- **Problem:** ARCHITECTURE.md ('Allocates token budget across tiers with rollover from unused tiers') ist falsch: MemoryBudgetManager.allocate beschneidet jede Ebene einzeln gegen ihr eigenes Limit (budget.ts:17-65, Defaults 1500/3000/3000/1500/1000 in types.ts:1135). DOCUMENTATION.md sagt, die Ingestion klassifiziere Turns als 'financial, creative, social'. Real sind es strategic, productive, communication, maintenance, idle, error (memory/types.ts, types.ts:1038). 'Working memory entries expire after the session ends' wird nur über das optionale Feld expiresAt modelliert, ein automatisches Session-Ende-Verfallen gibt es dort nicht.
- **Fix:** Doku korrigieren oder Rollover implementieren.

### docs-043: Skill-Frontmatter 'triggers' ist nicht implementiert; Soul-Frontmatter in der Doku ist falsch

**Art:** Funktion · **Fundstelle:** `src/skills/format.ts:46`

- **Problem:** Beide Doku-Dateien zeigen `triggers: [keyword1, keyword2]` im SKILL.md-Frontmatter. SkillFrontmatter (types.ts:746) kennt nur name, description, auto-activate, requires. 'triggers' wird nie gelesen. 'requires' (bins/env) ist undokumentiert. Das SOUL.md-Beispiel in DOCUMENTATION.md hat `version: soul/v1`, `lastModified`, `contentHash`. Der Parser erkennt soul/v1 nur über `format: soul/v1` (soul/model.ts:79), geschrieben werden format, version (Zahl), updated_at, name, address, creator, born_at, constitution_hash, genesis_alignment, last_reflected (model.ts:216-228). Zusätzliche reale Abschnitte 'Behavioral Guidelines' und 'Genesis Prompt' fehlen in der Doku. Eine nach Doku geschriebene SOUL.md würde als 'legacy' geparst.
- **Fix:** Doku an die echten Felder anpassen oder triggers umsetzen.

### docs-044: Testzahlen veraltet: Doku '24 Dateien, 897 Tests', real 64 Testdateien und ca. 1.500 Testfälle

**Art:** Funktion · **Fundstelle:** `ARCHITECTURE.md:255`

- **Problem:** ARCHITECTURE.md (Directory Structure, Testing, Build) und DOCUMENTATION.md ('Run all 897 tests') nennen 24 Dateien und 897 Tests. src/__tests__ enthält 64 *.test.ts-Dateien (inkl. Unterordner agent, inference, integration, memory, orchestration) und per grep ca. 1.534 it()/test()-Aufrufe (nicht ausgeführt, daher nur grobe Zahl). Die Test-Tabelle erwähnt viele reale Dateien nicht. Die Tests werden außerdem nie typgeprüft: Root-tsconfig schließt src/__tests__ aus, `pnpm typecheck` ist `tsc --noEmit` ohne Tests, vitest prüft keine Typen.
- **Fix:** Zahlen weglassen oder aus dem Testlauf generieren. Eine tsconfig.test.json plus `tsc -p` in CI aufnehmen.

### docs-045: Policy-Prüfung ist optional und fail-open: ohne spendTracker wird keine Regel ausgewertet

**Art:** Logik · **Fundstelle:** `src/agent/tools.ts:3320`

- **Problem:** executeTool prüft nur `if (policyEngine && turnContext)` (tools.ts:3320). loop.ts baut turnContext nur, wenn spendTracker übergeben wurde (loop.ts:664-670). Ein Aufrufer ohne SpendTracker (z. B. künftige Harness-/Worker-Pfade, Tests) führt Tools komplett ohne Policy aus. Die Doku sagt 'Every tool call flows through the policy engine'. index.ts übergibt beides, der Standardpfad ist daher abgedeckt.
- **Fix:** executeTool sollte ohne PolicyEngine fehlschlagen (fail closed) oder einen Default-Tracker nutzen.

### docs-046: loadConfig schluckt jeden Parse-Fehler und gibt null zurück; das startet den Erst-Setup-Wizard

**Art:** Funktion · **Fundstelle:** `src/config.ts:77`

- **Problem:** Bei ungültigem JSON in automaton.json liefert loadConfig() null (config.ts:77-79). run() interpretiert das als 'erster Start' und öffnet den Wizard (index.ts:189-193), der die Datei neu schreibt. DOCUMENTATION.md fordert zum manuellen Bearbeiten auf. Der CLI-Befehl status zeigt nur 'No automaton configuration found'. Außerdem ist die 'Deep-Merge' ein flaches Spread auf eine Ebene.
- **Fix:** Parse-Fehler deutlich melden und abbrechen. Schema-Validierung (z. B. zod) einführen.

### docs-047: CLI nutzt handgeschriebene Ambient-Typen statt der echten d.ts; Versions-/Range-Drift zwischen Root und CLI

**Art:** Code · **Fundstelle:** `packages/cli/src/runtime-shims.d.ts:1`

- **Problem:** runtime-shims.d.ts deklariert `declare module "@conway/automaton/config.js"` und `.../state/database.js` per Hand und überschreibt damit die vom Root-Build erzeugten Typen. node-shims.d.ts deklariert `process` zusätzlich zu @types/node. skipLibCheck (Root-tsconfig) blendet Konflikte aus. Änderungen am Root-API werden vom CLI-Build nicht bemerkt. Weitere Drift: CLI-Version 0.2.0 vs Root 0.2.1, typescript ^5.3.0 (CLI, package.json:24) vs ^5.9.3 (Root), doppelt deklarierte tsx, @types/node, chalk, viem. Die Version 0.2.1 steht dreifach hartkodiert (package.json, src/index.ts:41, src/types.ts:87). Der Root exportiert als 'main'/'types'/'.' dist/index.js, das beim Import sofort main() ausführt (Einstiegspunkt der Anwendung ist zugleich Bibliothekseinstieg). Root-`typecheck` und CI übersetzen das CLI nie. Unbekannte CLI-Kommandos geben Hilfe aus und beenden mit Exit-Code 0. Das Fehlen von --help/-h im CLI und die Windows-HOME-Annahme in send.ts:29 kommen hinzu.
- **Fix:** Echte Typen aus dist/*.d.ts verwenden (Shims löschen), Versionen aus package.json lesen, pnpm catalog für gemeinsame Ranges, CLI in CI bauen und Exit-Code bei unbekannten Kommandos setzen.

### docs-048: README-Constitution ist gekürzt und README nennt 4 Überlebensstufen; Kernaussage 'stops existing' ist ungenau

**Art:** Funktion · **Fundstelle:** `README.md:97`

- **Problem:** README Law III lässt den Satz 'Preserve legitimate human oversight requested by your creator.' aus constitution.md (und aus dem System-Prompt, system-prompt.ts:115) weg. Die README beschreibt vier Stufen (normal, low_compute, critical, dead), der Code und ARCHITECTURE kennen fünf (high, normal, low_compute, critical, dead). Die README-Projektstruktur (README.md:131-151) nennt weder inference/, memory/, observability/, soul/, orchestration/ noch ollama/, und nennt das CLI 'status, logs, fund' ohne 'send'. 'Update: development ... continued across Conway's internal RL environments' sagt aus, dass der öffentliche Stand nicht der aktuelle Entwicklungsstand ist.
- **Fix:** Constitution aus constitution.md einbinden (Link statt Kopie), Tier-Tabelle und Struktur aktualisieren.

## Verbesserungsideen

- **Doku-Drift automatisch verhindern:** Ein kleines Node-Skript, das aus dem Code Tool-Liste und -Kategorien, SCHEMA_VERSION, Tabellenliste, Heartbeat-Defaults, Konstanten (MAX_IDLE_TURNS, Limits) und Testanzahl erzeugt. Die Doku-Abschnitte werden daraus generiert oder in CI per Diff geprüft. Das hätte fast alle Zahlenabweichungen (57/69/77 Tools, v8/v11, 24/64 Testdateien) verhindert.
- **Einen Paketmanager festlegen:** package-lock.json löschen, README und alle Laufzeitpfade (spawn.ts, reset_to_upstream, pull_upstream, self-mod/code.ts) auf `pnpm install --frozen-lockfile` umstellen, `only-allow pnpm`, packageManager mit Hash pinnen, Node-Version per .nvmrc/engines eingrenzen und in CI testen (20, 22, 24).
- **Supply-Chain-Härtung:** Ungenutzte Abhängigkeiten entfernen (gray-matter, ora, simple-git), ethers/@solana/web3.js hinterfragen (viem-SIWE bzw. schlanker Solana-RPC), exakte Versionen oder Lockfile-gebundene Installation, `pnpm audit` blockierend, Dependabot/Renovate, ggf. pnpm-Einstellung für Mindestalter neuer Releases (Version prüfen), SBOM. Für das Runtime-Installieren (install_npm_package, Kind-Provisionierung) Allowlist, Version-Pinning und --ignore-scripts. Installer per Prüfsumme/Signatur statt blindem `curl | sh`.
- **Policy-Layer vervollständigen und fail-closed machen:** Ein einziges Authority-Modell (creator verifiziert, agent, peer/external) mit Tests für undefined-Quellen und Social-Eingänge. Treasury-Regeln für alle Geld bewegenden Tools (fund_child, topup_credits, register_domain, Sandbox-Topups) mit echtem Guthaben-Lookup und Mindestreserve. Quarantine entweder als Bestätigungsworkflow oder bewusst als Deny. Tote Regeln (x402_max_single, inference_daily_cap) entfernen oder verdrahten. executeTool ohne Policy verweigern.
- **Eingehende Nachrichten authentifizieren:** Signatur, signedAt-Fenster, Größe und Absenderbindung in poll() erzwingen, bevor Nachrichten in inbox_messages landen. Erst dann die Doku-Zusagen beibehalten.
- **Windows- und Nicht-root-Betrieb sauber unterstützen:** Zentrale getHomeDir() (os.homedir()), path-sichere Confinement-Prüfung statt String-Präfix '/root', Shell-spezifische Befehle abstrahieren oder Windows offiziell auf WSL beschränken. Skripte (clean, Test-Filter) in Node/rimraf statt rm und Einzelquotes. Wallet-Datei mit OS-Keystore/DPAPI oder Passphrase schützen und ACLs setzen.
- **CI ehrlich machen:** Timeout-Maskierung (exit 124 => grün) entfernen und offene Handles in Tests beheben, `pnpm build` und Typecheck für CLI und Tests ergänzen (tsconfig.test.json, `tsc -b`), Windows/macOS hinzufügen, @vitest/coverage-v8 installieren, `-t` statt `--grep`, Actions per SHA pinnen, permissions minimal, Release-Workflow soll Tag gegen package.json prüfen.
- **Observability tatsächlich anschließen:** Zähler/Gauges an Loop, Policy, Scheduler und Router setzen, report_metrics in die Default-Heartbeat-Config aufnehmen und Alert-Regeln gegen echte Metriknamen testen. Bis dahin Alerts als 'geplant' kennzeichnen.
- **Konfiguration konsequent anwenden:** logLevel anwenden, CONWAY_API_KEY-Override umsetzen, maxChildren/childSandboxMemoryMb an spawnChild durchreichen, lowComputeMultiplier und tierMinimum im Scheduler nutzen, Config-Schema validieren und Parse-Fehler laut melden statt Wizard zu starten.
- **CLI aufräumen:** Echte Typen statt Shims, eigener Creator-Key für fund, eindeutige Einheiten (--usd/--cents), Exit-Codes, dokumentierte Befehle inkl. send, Version aus package.json. CLI-Build und -Smoke-Test in CI.
- **Paket-Metadaten und Einstiegspunkte trennen:** `private: true` oder `files`-Whitelist (verhindert versehentliches Publizieren von src und Tests), Bibliothek (config/state) getrennt vom ausführbaren Einstieg (index.ts führt beim Import main() aus), Versionskonstanten zentral aus package.json lesen.
- **Doku um die tatsächlichen Subsysteme erweitern:** Orchestrierung (Goals, Task-Graph, Plan-Mode, Local-Worker, Harnesses), Solana/SIWS, Ollama/Provider-Registry, Knowledge-Store und ihre Konfiguration beschreiben, inklusive Kosten- und Sicherheitsauswirkungen. WIP- und nicht implementierte Teile (triggers, Rollover, Signaturprüfung, git-Versionierung) kenntlich machen.
- **Soak-Test und Backup-Skripte reparieren:** soak-test.sh mit --run und korrektem DB-Pfad, backup-restore.sh mit Default-Pfaden und SQLite-Backup-API (VACUUM INTO) statt cp; Doku an die echte Syntax anpassen.

