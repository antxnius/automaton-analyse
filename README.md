# Conway Automaton: Ergebnisse der Analyse

**Repo:** [github.com/Conway-Research/automaton](https://github.com/Conway-Research/automaton)
**Untersuchter Stand:** Commit `d8f8168` vom 26.08.2026, Version 0.2.1
**Analyse vom:** 07.10.2026

## In einem Satz

Automaton ist ein großes, ernsthaft gebautes Gerüst für einen autonomen KI-Agenten mit eigener Krypto-Wallet, aber das Versprechen „verdient sein eigenes Geld“ löst der Code nicht ein, und viele Sicherheits- und Budgetgrenzen greifen in der Praxis nicht.

## Kurzfazit

| Frage | Antwort |
|---|---|
| Verdient es Geld? | Nicht von sich aus. Im Code gibt es nur die Ausgabenseite. Ein Geschäft muss sich das Sprachmodell komplett selbst ausdenken. Am Anfang zahlt nur der Betreiber. |
| Bekommt der Betreiber Gewinn? | Nicht vorgesehen. Überschüsse bleiben beim Agenten oder sollen in Kind-Agenten fließen. |
| Ist es sicher? | Nein. Der private Wallet-Schlüssel ist für den Agenten auslesbar, und damit auch für jeden, der ihn per Prompt-Injection steuert. Viele Schutzregeln lassen sich umgehen. |
| Hält es sein Budget ein? | Nur teilweise. Das Tageslimit für Modellaufrufe (Standard 500 $) wird nie geprüft, mehrere andere Limits sind toter Code. |
| Funktioniert die Selbstvermehrung? | Nein. Kind-Agenten werden angelegt und bezahlt, starten aber nicht lauffähig. |
| Wofür taugt es? | Zum Lernen und Abschauen gut. Zum Geldverdienen oder für den Betrieb mit echtem Geld nicht. |

## Was in welcher Datei steht

| Datei | Inhalt | Lesen, wenn … |
|---|---|---|
| [01-was-es-ist-und-was-funktioniert.md](01-was-es-ist-und-was-funktioniert.md) | Aufbau, realer Ablauf, Versprechen gegen Wirklichkeit | … man verstehen will, was das Ding ist |
| [02-geld.md](02-geld.md) | Wie Geld rein- und rausgeht, Gewinn, Kosten, Survival-Stufen | … man damit Geld verdienen will |
| [03-die-wichtigsten-probleme.md](03-die-wichtigsten-probleme.md) | Die schweren Fehler, nach Thema sortiert | … man technisch tiefer einsteigen will |
| [04-nutzen-ja-oder-nein.md](04-nutzen-ja-oder-nein.md) | Empfehlung, Für und Wider, Checkliste fürs Ausprobieren | … man es laufen lassen will |
| [05-wie-es-sein-sollte.md](05-wie-es-sein-sollte.md) | Was man ändern müsste, damit es funktioniert | … man daran weiterbauen will |
| [anhang/](anhang/README.md) | Alle 636 Einzelbefunde mit Fundstelle und Fix, aufgeteilt nach Bereich | … man einen Punkt im Detail nachschlagen will |

## Zahlen

| | |
|---|---|
| Untersuchter Code | ca. 37.000 Zeilen TypeScript, dazu ca. 28.000 Zeilen Tests |
| Gefundene Befunde | 636 |
| Nach Gegenprüfung bestätigt | 619 |
| davon kritisch / hoch | 2 / 33 |
| davon mittel / niedrig | 272 / 312 |
| Umstritten / abgelehnt | 6 / 11 |

Viele Befunde überschneiden sich, weil mehrere Prüfer dasselbe Kernproblem aus verschiedenen Richtungen gefunden haben. Die 35 schweren Befunde gehen auf 22 eigenständige Probleme zurück (siehe [03](03-die-wichtigsten-probleme.md)).

**Schwere:**
- **kritisch:** direkter Geld- oder Kontrollverlust möglich
- **hoch:** Kernfunktion kaputt oder Schutz wirkungslos
- **mittel:** deutlicher Fehler mit begrenzter Wirkung
- **niedrig:** Kleinigkeit, Doku-Abweichung, Schönheitsfehler

## Wie die Analyse gemacht wurde

- Durchgeführt mit Claude (KI von Anthropic), das viele parallel arbeitende Prüf-Agenten koordiniert hat.
- Der Code wurde **nur gelesen, nicht ausgeführt**. Was zur Laufzeit passiert, ist aus dem Code abgeleitet, nicht beobachtet.
- **Ablauf:**
  1. Acht Bereiche wurden je vollständig gelesen.
  2. Je ein Zweitdurchgang suchte gezielt nach Übersehenem.
  3. Ein separater Abgleich prüfte Doku gegen Code und die Abhängigkeiten.
  4. Jeder Befund wurde von einem Skeptiker geprüft, der ihn widerlegen sollte. Kritische und hohe Befunde prüften in der Regel zwei.
- **Grenzen:**
  - Die Dienste von Conway Cloud (`api.conway.tech` usw.) sind nicht öffentlich dokumentiert. Alles, was davon abhängt, ist nur aus dem Client-Code erschlossen.
  - Mittlere und niedrige Befunde sind nur einfach geprüft. Einzelne davon können falsch sein.
  - Die Bestätigungsquote ist auch deshalb hoch, weil viele Befunde leicht prüfbare Fakten sind, etwa „Funktion X fehlt“ oder „weicht von der Doku ab“.
