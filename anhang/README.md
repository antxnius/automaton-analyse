# Anhang: alle Befunde im Detail

← [Bericht](../README.md)

Hier stehen alle 636 Einzelbefunde der Analyse. Jeder hat eine ID wie `core-001`. Das Präfix nennt den Bereich, die Zahl ist fortlaufend. Kritische und hohe Befunde stehen gesammelt in einer eigenen Datei, alle anderen in der Datei ihres Bereichs.

Jeder Eintrag enthält:
- **Problem:** was falsch ist oder fehlt
- **Szenario:** wie es konkret schiefgeht
- **Fix:** was man ändern sollte
- **Gegenprüfung:** nur bei kritischen und hohen Befunden; die Begründung des Prüfers, der versucht hat, den Befund zu widerlegen

| Datei | Inhalt | Befunde |
|---|---|---|
| [kritisch-und-hoch.md](kritisch-und-hoch.md) | Alle schweren Befunde ausführlich | 35 |
| [bereich-kern-setup-cli.md](bereich-kern-setup-cli.md) | Kern, Setup, CLI, Skripte, CI: so funktioniert der Bereich, mittlere und niedrige Befunde, Verbesserungsideen | 77 |
| [bereich-agent-loop.md](bereich-agent-loop.md) | Agent-Loop und Harnesses: so funktioniert der Bereich, mittlere und niedrige Befunde, Verbesserungsideen | 83 |
| [bereich-policy-selbstmodifikation.md](bereich-policy-selbstmodifikation.md) | Policy-Regeln, Selbstmodifikation, Git, Skills: so funktioniert der Bereich, mittlere und niedrige Befunde, Verbesserungsideen | 65 |
| [bereich-geld-inference-identitaet.md](bereich-geld-inference-identitaet.md) | Geld, Inference, Survival, Identität: so funktioniert der Bereich, mittlere und niedrige Befunde, Verbesserungsideen | 63 |
| [bereich-gedaechtnis.md](bereich-gedaechtnis.md) | Gedächtnis: so funktioniert der Bereich, mittlere und niedrige Befunde, Verbesserungsideen | 57 |
| [bereich-orchestrierung.md](bereich-orchestrierung.md) | Orchestrierung: so funktioniert der Bereich, mittlere und niedrige Befunde, Verbesserungsideen | 55 |
| [bereich-replikation-soul-social.md](bereich-replikation-soul-social.md) | Replikation, Soul, Registry, Social: so funktioniert der Bereich, mittlere und niedrige Befunde, Verbesserungsideen | 71 |
| [bereich-heartbeat-datenbank-tests.md](bereich-heartbeat-datenbank-tests.md) | Heartbeat, Datenbank, Observability, Tests: so funktioniert der Bereich, mittlere und niedrige Befunde, Verbesserungsideen | 67 |
| [bereich-doku-abhaengigkeiten.md](bereich-doku-abhaengigkeiten.md) | Doku gegen Code, Abhängigkeiten: so funktioniert der Bereich, mittlere und niedrige Befunde, Verbesserungsideen | 46 |
| [umstritten-und-abgelehnt.md](umstritten-und-abgelehnt.md) | Befunde, die der Gegenprüfung nicht oder nur teilweise standhielten | 17 |
| [gesamtanalyse.md](gesamtanalyse.md) | Der ausführliche Ende-zu-Ende-Durchlauf: realer Ablauf, Hindernisse, Soll-Zustand | – |

**Präfixe:** `core` Kern/Setup/CLI · `agent` Agent-Loop · `policy` Policy/Selbstmodifikation · `money` Geld/Inference/Identität · `memory` Gedächtnis · `orchestration` Orchestrierung · `replication` Replikation/Social · `infra` Heartbeat/Datenbank/Tests · `docs` Doku/Abhängigkeiten

**Hinweis:** Die Einträge sind die Originaltexte der Prüf-Agenten, nur leicht formatiert. Mittlere und niedrige Befunde wurden von einem Prüfer bestätigt, kritische und hohe in der Regel von zwei.
