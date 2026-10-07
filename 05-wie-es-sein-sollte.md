# Wie es funktionieren sollte

← [Übersicht](README.md)

Damit das Ziel (selbstfinanzierend, sicher, autonom) realistisch erreichbar wird, wären Änderungen in dieser Reihenfolge nötig. Die Nummern in Klammern verweisen auf die Probleme in [03](03-die-wichtigsten-probleme.md).

## Priorität 1: Fundament

Ohne diese Punkte ist alles andere riskant oder nicht messbar.

| # | Was | Warum und wie |
|---|---|---|
| 1 | **Ein gemeinsames Kassenbuch** | Alle Einnahmen und Ausgaben (Modellaufrufe, Sandboxes, Domains, Gas, Kinder) an einer Stelle erfassen. Daraus lassen sich die Verbrennungsrate und die Reichweite berechnen, also wie viele Tage das Geld noch reicht. Die Survival-Stufen sollten sich nach dieser Reichweite richten, nicht nach festen Cent-Beträgen. |
| 2 | **Harte Budgets an einer einzigen Stelle** | Jeder Modellaufruf und jede Zahlung (Aufladen, Domains, Sandboxes, Kinder, Orchestrator) läuft durch denselben Wächter. Der setzt Limits pro Aufruf, Stunde und Tag und hält eine Reserve, die wirklich blockiert. Tote Regeln werden angeschlossen oder gelöscht. (8, 9, 10) |
| 3 | **Leerlaufkosten senken** | Lange Pausen statt einer Minute. Aufwachen nur bei Ereignissen wie Kundenanfrage, Zahlungseingang oder Befehl des Betreibers. Ein kürzerer System-Prompt. Ziel: unter 1 $ pro Tag im Leerlauf. |
| 4 | **Ein echter Einnahmekanal als Code** | Eine Bezahlschranke (x402-Server), die Zahlungen prüft und verbucht, dazu ein oder zwei fertige Produkte mit positiver Marge. Der Agent betreibt das Geschäft, statt die Zahlungsinfrastruktur selbst zu erfinden. |
| 5 | **Schlüssel getrennt vom Agenten** | Signiert wird über einen separaten Prozess mit eigenen Limits. Der Agent hält nur ein kleines Betriebsguthaben, **Überschüsse gehen automatisch an die Creator-Adresse**. Damit wäre auch geregelt, wie der Betreiber am Gewinn beteiligt wird. (1, 2) |
| 6 | **Kein ungeschützter Betrieb auf dem Host** | Den lokalen Modus nur nach ausdrücklicher Freigabe erlauben. Befehle über eine Erlaubnisliste und eine Netzwerksperre steuern statt über Suchmuster. (5, 6) |
| 7 | **Rechte korrigieren, dazu ein signierter Kanal für den Betreiber** | Eigene Schritte des Agenten laufen als „self“, Fremdnachrichten werden stark eingeschränkt. Der Betreiber kann per Signatur pausieren, stoppen, Budgets setzen und Geld abziehen, also mit einem echten Not-Aus. In Notlagen wird er benachrichtigt. (7) |
| 8 | **Schutzregeln unveränderbar machen** | Policy-, Wallet- und Survival-Code sowie die Konstitution liegen schreibgeschützt außerhalb der Reichweite des Agenten und werden beim Start per Prüfsumme kontrolliert. (3, 4) |

## Priorität 2: Funktionen reparieren

| # | Was | Warum und wie |
|---|---|---|
| 9 | **Replikation erst nach nachgewiesenem Gewinn** | Zum Beispiel nur, wenn der Nettogewinn der letzten 14 bis 30 Tage über null liegt. Kinder starten ohne Setup-Dialog und mit fester Version. Startkapital gibt es als Darlehen, das mit einem Anteil der Einnahmen zurückgezahlt wird. (14, 15) |
| 10 | **Ein echter Nachrichtenkanal zu Kindern** | Aufträge gehen über Relay oder HTTP statt in die lokale Datenbank, mit Rückmeldung und einem Health-Endpunkt. (13) |
| 11 | **Orchestrator ans Budget koppeln** | Unter-Agenten nur mit eingeplantem Geld starten. Dazu Timeouts und sauberes Aufräumen nach Abbruch und Neuplanung. (11, 16, 17, 18) |
| 12 | **Kontext und Posteingang reparieren** | Ausgeführte Aktionen samt Ergebnis immer im Kontext behalten und den Posteingang zuverlässig lesen. (19, 20, 21) |
| 13 | **„Tod“ eindeutig definieren** | Bei Zahlungsunfähigkeit keine Modellaufrufe mehr, nur noch auf eine Einzahlung warten. Eine einzige Definition, die in Code, Prompt und Doku gleich ist. |
| 14 | **Ein Fehler ist kein Guthaben von 0** | Ein Ausfall der API darf nicht als leeres Konto gelten und keine Käufe auslösen. |
| 15 | **Eigene Modellwahl respektieren** | Die Einstellung des Betreibers hat Vorrang vor der eingebauten Liste. Anbieter ohne API-Key werden übersprungen. (22) |

## Priorität 3: Qualität und Betrieb

| # | Was | Warum und wie |
|---|---|---|
| 16 | **Tests gegen eine nachgebaute Conway-API** | Heute hängt alles an externen Diensten, die sich nicht prüfen lassen. Gebraucht werden Tests gegen eine nachgebaute API und ein End-to-End-Test vom Setup bis zum Kind-Agenten. |
| 17 | **Eine ehrliche CI** | Hängende Tests zählen als Fehler, die Security-Tests laufen wirklich, und das CLI wird mitgebaut. |
| 18 | **Gedächtnis mit echter Suche** | Volltextsuche statt Teilstring-Suche über den ganzen Text. Gespeicherte Inhalte werden nach Herkunft markiert, damit eingeschleuste Befehle nicht dauerhaft im Prompt landen. |
| 19 | **Doku und Code abgleichen** | Schwellen, Funktionen und Tool-Namen müssen übereinstimmen. |
| 20 | **Die Plattform klar festlegen** | Offiziell nur Linux bzw. Sandbox. Windows entweder ausdrücklich ausschließen oder sauber unterstützen. |
| 21 | **Widerspruchsfreie Werte** | „Survive at all costs“ und „Never give up“ aus den Standardtexten entfernen. Das passt nicht zu „Gesetz I geht vor Überleben“. |

## Realistisches Zielbild

Solange Kassenbuch, harte Budgets und Schlüsseltrennung nicht nachweislich funktionieren, ist dieses Modell tragfähig: **ein Agent mit einem festen, vom Betreiber gesetzten Budget, der regelmäßig berichtet, was er gekostet und was er gebracht hat.** Selbstfinanzierung, Replikation und „Tod als Selektion“ ergeben erst danach Sinn.
