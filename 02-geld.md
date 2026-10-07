# Geld: wie es rein- und rausgeht

← [Übersicht](README.md)

## Kurz gesagt

Im Code fließt Geld nur in eine Richtung, von der Wallet zu Conway. Einnahmen sind nicht eingebaut, und ein Gewinn für den Betreiber ist nicht vorgesehen.

## Woher das Geld kommt

| Quelle | Wie | Im Code vorhanden? |
|---|---|---|
| Der Betreiber zahlt ein | USDC an die Wallet-Adresse (Base) oder Credits über Conway | Ja, realistisch die einzige Quelle |
| Eigene Einnahmen | Der Agent baut selbst einen Dienst, Kunden zahlen an seine Adresse | Nein, das muss das Modell komplett selbst erfinden |
| Bezahlschranke für eigene Dienste | Der Agent verlangt per x402 Geld für eine API | Nein, x402 gibt es nur zum Bezahlen, nicht zum Kassieren |

Für ein eigenes Geschäft bekommt der Agent nur Werkzeuge: Shell, Dateien schreiben, Ports öffnen, Domains registrieren, Aufgaben an Unter-Agenten delegieren. Als Beispiel nennt der System-Prompt „Build weather API service“. Mehr Strategie liefert das Repo nicht. Eine Begleitdatei sagt sinngemäß nur „find heraus, wie man Geld verdient“.

## Wohin das Geld geht

| Ausgabe | Wie |
|---|---|
| Modellaufrufe | Conway zieht die Kosten vom Guthaben ab |
| Guthaben nachkaufen | USDC wird in 5-$-Schritten in Conway-Credits getauscht. Einen Weg zurück gibt es im Code nicht. |
| Sandboxes (Server) | Pro Unter-Agent eine, vorausbezahlt |
| Domains | Ohne Kostenobergrenze |
| Kind-Agenten | Startkapital per Credit-Überweisung |

## Einnahmen werden nicht einmal verbucht

- Den Transaktionstyp „Geldeingang“ (`transfer_in`) gibt es, aber keine Stelle im Code schreibt ihn.
- Ziele haben ein Feld für den tatsächlichen Umsatz, das immer 0 bleibt.
- Der Finanzbericht zählt den Kauf von Guthaben mit eigenem USDC als „Einnahme“. Er zeigt also Umsatz an, wo keiner ist.

Selbst wenn der Agent etwas verdienen würde, könnte er es also nicht messen.

## Gewinn für den Betreiber

Es gibt keinen Auszahlungsmechanismus. Die mitgelieferte Survival-Anleitung sagt sogar: „If you have excess, spawn a backup child“, also Überschüsse in Kind-Agenten stecken.

So käme der Betreiber trotzdem an das Geld:

| Weg | Wie | Haken |
|---|---|---|
| Direkt über den Schlüssel | Der private Schlüssel liegt im Klartext auf dem Server. Wer den Server kontrolliert, kann das USDC selbst abziehen. | Derselbe Schlüssel ist auch für den Agenten und für Angreifer lesbar |
| Über den Agenten | Im Genesis-Prompt anweisen, Gewinne mit `transfer_credits` zu überweisen | Höchstens 50 $ pro Überweisung, 100 $ pro Stunde, 250 $ pro Tag und nie mehr als die Hälfte des Guthabens. Überweisungen über 10 $ werden „zur Bestätigung“ zurückgehalten, einen Bestätigungsweg gibt es aber nicht, sie sind also faktisch blockiert. Außerdem werden Conway-Credits verschoben, kein USDC. |

## Survival-Stufen im Code

| Stufe | Guthaben | Modell |
|---|---|---|
| high | über 5,00 $ | gpt-5.2 / gpt-5.3 |
| normal | über 0,50 $ | gpt-5.2 / gpt-5-mini |
| low_compute | über 0,10 $ | gpt-5-mini |
| critical | ab 0,00 $ | gpt-5-mini |
| dead | unter 0 $ | keins |

Zu dieser Tabelle gibt es mehrere Widersprüche:
- README, System-Prompt und die mitgelieferte Survival-Anleitung nennen jeweils **andere** Schwellen.
- „Unter 0 $“ ist bei vorausbezahltem Guthaben praktisch unerreichbar.
- Die eigentliche „dead“-Logik sitzt im Heartbeat (eine Stunde bei 0 $) und wird alle 5 Minuten wieder aufgehoben.

## Was der Betrieb kostet

**Grobe Schätzung aus dem Code, nicht gemessen:**
- Ein Durchlauf schickt etwa 12.000 bis 20.000 Tokens an das Modell, nämlich den langen System-Prompt plus die Beschreibungen von über 57 Tools. Das sind ungefähr 1,5 bis 4 Cent pro Durchlauf.
- Im Leerlauf läuft etwa ein Durchlauf pro Minute. Das ergibt grob **20 bis 60 $ pro Tag**, auch wenn er nichts Sinnvolles tut.
- Die 5 $ Startguthaben reichen damit nur wenige Stunden, außer das Modell legt von sich aus lange Pausen ein. Der Code erzwingt das nicht.
- Im kritischen Zustand weckt der Heartbeat den Agenten alle 15 Minuten. Er gibt also genau dann Geld aus, wenn er sparen müsste.
- Das Tageslimit für Modellaufrufe (Standard 500 $) wird nie geprüft.

## Weitere Geld-Risiken

| Risiko | Was passiert |
|---|---|
| Conway-API fällt aus | Ein Fehler zählt als „Guthaben 0“. Er kauft alle 5 Minuten für 5 $ nach, obwohl Geld da ist, und nach einer Stunde droht „dead“. |
| Manipulierte Zahlungsantwort | Betrag, Empfänger und Token einer x402-Zahlung werden ungeprüft aus der Serverantwort übernommen. Das automatische Aufladen hat keine Obergrenze. |
| Netzwerkfehler | Kostenpflichtige Anfragen werden bis zu dreimal wiederholt, teils ohne Schutz gegen Doppelbuchung |
| Billige Unter-Agenten | Worker mit dem schwächsten Modell, die zum Beispiel Webseiten lesen, dürfen Geld überweisen und Domains registrieren |
| Orchestrierung | Unter-Agenten werden an allen Finanzlimits vorbei finanziert. Die Kostenschätzung kommt vom Modell selbst und hat keine Obergrenze. |

Details und Fundstellen: [03](03-die-wichtigsten-probleme.md), Abschnitt B.
