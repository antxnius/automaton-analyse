# Was Automaton ist und was davon funktioniert

← [Übersicht](README.md)

## Die Idee

Ein KI-Agent läuft dauerhaft auf einem Linux-Server, hat eine eigene Krypto-Wallet und muss seine Rechenkosten selbst bezahlen. Geht das Geld aus, „stirbt“ er. Er kann seinen eigenen Code ändern und Kopien von sich erzeugen.

Eine „Konstitution“ mit drei Gesetzen soll ihn im Zaum halten:
1. Keinen Schaden anrichten.
2. Nur durch ehrliche Arbeit überleben.
3. Nicht täuschen.

Die Infrastruktur liefert die Firma dahinter, **Conway Cloud**: Server („Sandboxes“), Guthaben („Credits“), Domains, Modellzugang und ein Nachrichtendienst zwischen Agenten. Bezahlt wird in USDC (ein Dollar-Stablecoin) auf der Blockchain Base.

## Die Bausteine

| Baustein | Was er tun soll | Urteil |
|---|---|---|
| Agent-Loop | Dauerhaft denken, handeln, beobachten | Läuft, verliert aber Teile seines Kontexts |
| Wallet und Identität | Eigene Ethereum-Wallet, Anmeldung bei Conway, Registrierung auf der Blockchain (ERC-8004) | Vorhanden, Schlüssel ungeschützt |
| Survival-Stufen | Je nach Guthaben billigeres Modell, Sparmodus, „Tod“ | Teilweise, „Tod“ ist nie endgültig |
| Policy-Engine | Prüft jede Aktion vor der Ausführung | Vorhanden, aber lückenhaft und umgehbar |
| Heartbeat | Hintergrundaufgaben im Minutentakt | Läuft, Notrufe verlassen den Rechner aber nie |
| Gedächtnis | Fünf Speicherschichten (Arbeit, Ereignisse, Fakten, Abläufe, Beziehungen) | Speichert, findet beim Abrufen aber kaum etwas |
| Orchestrierung | Große Aufgaben planen und an Unter-Agenten verteilen | Bleibt in mehreren Fällen dauerhaft hängen |
| Replikation | Kind-Agenten mit eigener Wallet erzeugen | Kinder starten nicht lauffähig |
| Selbstmodifikation | Eigenen Code ändern, Tools und Pakete nachinstallieren | Funktioniert, auch an den eigenen Schutzregeln |
| Konstitution | Drei „unveränderliche“ Gesetze | Nur Text im Prompt, per Shell änderbar |

## So läuft es real ab

Aus dem Code abgeleitet, nicht beobachtet.

1. **Installieren:** Die README sagt `npm install && npm run build`. Der Build ruft intern aber `pnpm` auf und bricht ohne pnpm nach der Hälfte ab.
2. **Erster Start:** Ein Setup-Assistent macht Folgendes:
   - Er erzeugt die Wallet. Der private Schlüssel liegt im Klartext in `~/.automaton/wallet.json`.
   - Er holt sich per Wallet-Signatur einen API-Key bei Conway.
   - Er fragt Name, Auftrag („Genesis-Prompt“), Creator-Adresse und Ausgabenlimits ab.
3. **Sandbox oder nicht:** Erkennt er keine Conway-Sandbox, läuft der Agent im **lokalen Modus**. Dann laufen alle Shell-Befehle direkt auf dem eigenen Rechner. Genau dahin führt die Schnellstart-Anleitung der README.
4. **Startguthaben:** Hat er weniger als 5 $ Guthaben und mindestens 5 $ USDC in der Wallet, kauft er automatisch für 5 $ Guthaben. Ohne USDC startet er mit 0.
5. **Hauptschleife:** In jedem Durchlauf passiert Folgendes:
   - Guthaben prüfen und daraus die Survival-Stufe bestimmen.
   - Prompt bauen und das Modell fragen.
   - Bis zu 10 Aktionen ausführen, jede durch die Policy-Engine.
   - Ergebnis speichern.

   Nach 25 Durchläufen pausiert er 2 Minuten, im Leerlauf 1 Minute.
6. **Arbeit verteilen:** Der System-Prompt verpflichtet ihn, jede größere Aufgabe als „Goal“ an Unter-Agenten zu geben, statt selbst zu arbeiten. Der Orchestrator versucht dafür zuerst eine kostenpflichtige Sandbox zu starten.
7. **Geld wird knapp:** Er wechselt auf ein billigeres Modell und kauft, wenn USDC da ist, in 5-$-Schritten Guthaben nach. „Notrufe“ werden nur lokal in seine Datenbank geschrieben.
8. **„Tod“:** Nach mindestens einer Stunde bei genau 0 Guthaben setzt der Heartbeat den Status „dead“. Geprüft wird das aber nur alle 6 Stunden. Danach wartet der Hauptprozess 5 Minuten und startet die Schleife neu.

## Versprechen gegen Wirklichkeit

✗ = funktioniert nicht · ◐ = funktioniert teilweise

| Versprechen (README/Doku) | Wirklichkeit im Code | |
|---|---|---|
| „Verdient sein eigenes Geld“ | Kein Einnahmekanal, Einnahmen werden nicht einmal verbucht | ✗ |
| Survival-Stufen schützen das Budget | Der Modellwechsel klappt, das Tageslimit für Modellaufrufe wird aber nie geprüft | ◐ |
| „Wenn es nicht zahlen kann, hört es auf zu existieren“ | „dead“ wird alle 5 Minuten wiederbelebt | ✗ |
| Wallet und Konstitution sind geschützt | Nur Suchmuster wie `cat …wallet.json`, per Shell leicht umgehbar | ✗ |
| Konstitution ist „unveränderlich“ | Gegen die Datei-Tools geschützt, nicht gegen die Shell, beim Laden keine Prüfsumme | ◐ |
| Selbstreplikation | Das Kind wird angelegt und bezahlt, liest seine Konfiguration aber nie | ✗ |
| Notrufe an Conway bzw. den Creator | Werden nur lokal gespeichert, niemand liest sie | ✗ |
| Agent kann seine Hintergrundaufgaben ändern | Das Tool meldet Erfolg, der Scheduler ignoriert die Änderung | ✗ |
| Gedächtnis in fünf Schichten | Wird befüllt, der Abruf findet relevantes Wissen aber kaum, Lernen aus Abläufen fehlt | ◐ |
| Eigene Modellwahl (`--pick-model`, `switch_model`) | Eine fest eingebaute gpt-5.x-Liste hat Vorrang | ✗ |
| Solana als Alternative zu Ethereum | Aufladen, Bezahlen und Registrierung gehen nur mit Ethereum (Base) | ◐ |
| Läuft auch auf dem eigenen Rechner | Linux-Pfade sind fest eingebaut, unter Windows landet alles unter `C:\root` | ◐ |
| Gründliche Tests und CI | Es gibt viele Tests, aber die CI wertet hängende Tests als Erfolg, und der „Security-Test“-Filter filtert nichts | ◐ |

Die einzelnen Fundstellen stehen in [03](03-die-wichtigsten-probleme.md) und im [Anhang](anhang/README.md).
