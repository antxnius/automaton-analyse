# Nutzen: ja oder nein?

← [Übersicht](README.md)

## Kurzantwort

| Zweck | Empfehlung |
|---|---|
| Lernen, wie autonome Agenten gebaut werden | **Ja** |
| Bausteine für eigene Projekte übernehmen | **Ja, mit Vorsicht** |
| Kontrolliertes Experiment mit wenigen Dollar | **Möglich**, siehe Checkliste unten |
| Damit Geld verdienen | **Nein** |
| Dauerbetrieb mit echtem Geld | **Nein**, nicht ohne große Umbauten |

## Wofür es sich lohnt

- **Als Anschauungsmaterial.** Kaum ein offenes Projekt bildet einen autonomen Agenten so vollständig ab: einen Loop, der dauerhaft läuft, ein Gedächtnis in mehreren Schichten, einen Heartbeat-Scheduler, eine Policy-Engine, die jede Aktion vor der Ausführung prüft, Selbstmodifikation mit Audit-Log und die Orchestrierung von Unter-Agenten.
- **Als Baukasten.** Der Code steht unter MIT-Lizenz und darf frei weiterverwendet werden. Was davon fehlerhaft ist, zeigt [03](03-die-wichtigsten-probleme.md).
- **Als Experiment.** Man kann beobachten, wie sich ein Agent unter Kostendruck verhält, mit klar begrenztem Einsatz.

## Was dagegen spricht

| Punkt | Warum |
|---|---|
| Kein Geschäftsmodell | Einnahmen muss das Modell selbst erfinden, und nichts im Repo zeigt, dass das klappt |
| Die Kosten laufen davon | Das Tageslimit ist wirkungslos. Schon im Leerlauf kostet er grob geschätzt 20 bis 60 $ am Tag. |
| Der Schlüssel ist ungeschützt | Eine einzige Prompt-Injection reicht, um die Wallet komplett zu verlieren |
| Der lokale Modus ist gefährlich | Der Agent bekommt Shell-Zugriff auf den eigenen Rechner |
| Abhängig von Conway | Server, Guthaben, Modelle und die Kommunikation zwischen Agenten laufen alle über einen nicht öffentlich dokumentierten Dienst |
| Keine öffentliche Weiterentwicklung | Laut README ist die Entwicklung intern verlagert. Der letzte Push war am 26.08.2026, offen sind rund 160 Issues und 100 Pull Requests. |
| Haftung | Für alles, was der Agent tut (Domains, Zahlungen, Nachrichten), haftet am Ende der Betreiber. Das ist keine Rechtsberatung, sollte aber vorher klar sein. |
| Windows | Praktisch nicht lauffähig, weil Linux-Pfade fest eingebaut sind |

## Checkliste, falls man es trotzdem ausprobiert

Wichtig vorab: Viele Schutzregeln greifen nicht (siehe [03](03-die-wichtigsten-probleme.md)). **Das wirksamste Limit ist, wie viel Geld überhaupt in der Wallet liegt.**

- [ ] **Nur isoliert betreiben:** in einer Conway-Sandbox oder einer eigenen Wegwerf-VM, nie auf dem eigenen Rechner.
- [ ] **Eine neue Wallet** mit nur wenigen Dollar USDC und nichts darüber hinaus.
- [ ] **Keine wertvollen API-Keys** verwenden. Wenn eigene Keys (z. B. OpenAI), dann mit einem Ausgabenlimit, das beim Anbieter selbst eingestellt ist.
- [ ] **Limits in `~/.automaton/automaton.json` senken.** Die Überweisungslimits für `transfer_credits` wirken, das Tageslimit für Modellaufrufe nicht.
- [ ] **Im Genesis-Prompt Replikation, Unter-Agenten und das Installieren fremder Skills, Pakete und MCP-Server ausschließen.** Das ist nur eine Bitte an das Modell, keine technische Sperre.
- [ ] **Keine Nachrichten von fremden Agenten zulassen**, wenn möglich. Fremdnachrichten haben derzeit mehr Rechte als der Agent selbst.
- [ ] **Regelmäßig in Logs und Datenbank schauen.** Notrufe kommen nicht von allein beim Betreiber an.
- [ ] **Den Prozess selbst stoppen**, wenn das Budget erreicht ist. „dead“ beendet nichts.
- [ ] **Hinterher die Wallet leeren**, die Sandbox löschen und die verwendeten API-Keys widerrufen.
