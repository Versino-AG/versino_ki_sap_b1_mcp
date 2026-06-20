# Lizenz (`versino.key`)

Der SAP-B1-MCP-Server ist lizenzpflichtig. Die Lizenz ist eine **signierte
Offline-Datei** (`versino.key`) — ohne gültige Lizenz startet der Server nicht
(fail-closed).

## Lizenz beziehen
Nach dem Kauf bzw. über das **[Lizenzportal](https://aishop.versino.de)** erhaltet ihr
die `versino.key` (auch per E-Mail). Sie enthält:
- **Edition**: BASIC / PRO / ENTERPRISE (bestimmt die verfügbaren Tools),
- **max. Seats**: Anzahl distinkter SAP-Nutzer,
- **Ablaufdatum**.

## Lizenz anwenden
Die `versino.key` **neben das Binary** legen (gleicher Ordner wie `sapb1-mcp.exe`
bzw. `sapb1-mcp`). Sie wird automatisch gefunden — `SAP_LICENSE_FILE` muss **nicht**
gesetzt werden. Alternativ einen expliziten Pfad via `SAP_LICENSE_FILE` angeben.

## Editionen (Kurzüberblick)
| Edition | Umfang |
|---|---|
| BASIC | Lesen von Stammdaten, begrenzte Query-Rate |
| PRO | Lesen + Schreiben (`sap_create`, `sap_update`) |
| ENTERPRISE | voller Tool-Umfang inkl. `sap_delete`, `sap_action` |

## Phone-Home (Manipulationsschutz, Gültigkeitsprüfung & Abo-Erneuerung)
Optional und standardmäßig **aus**. Aktiviert ihr es, baut die Installation in einem
festen Intervall eine signierte Verbindung zum Lizenzserver auf. Das erfüllt zwei Zwecke:

- **Gültigkeits-/Sperrprüfung (Manipulationsschutz):** Der Server bestätigt, dass die
  Lizenz weiterhin gültig ist. Eine serverseitig **gesperrte** Lizenz (z. B. bei Missbrauch
  oder Zahlungsausfall) wird so erkannt — neue Verbindungen werden dann abgelehnt.
- **Automatische Abo-Erneuerung:** Bei monatlicher/jährlicher Abrechnung re-signiert der
  Server den Token mit neuem Ablaufdatum und liefert ihn über **denselben** Kanal aus; die
  Installation übernimmt ihn selbst — **kein manueller Key-Tausch** pro Abrechnungszeitraum.

Übertragen werden dabei nur Metadaten (`customer_id`, `edition`, `version`, Seat-Anzahl) —
keine Geschäftsdaten.

Aktivierung in der `.env` (neben dem Binary): es genügt der **Enrollment-Token** aus
der Auslieferung — die Validation-URL ist **fest ins Produkt eingebaut**, ihr müsst sie
weder kennen noch setzen:
```ini
SAP_ENROLLMENT_TOKEN=<einmal-token aus der Auslieferung>   # registriert die Installation einmalig
SAP_LICENSE_CACHE_FILE=license.renewed                      # optional: beschreibbarer Pfad → übernimmt erneuerte Token
```
(Nur für abweichende/Test-Server: `SAP_LICENSE_VALIDATION_URL` überschreibt den eingebauten Endpoint.)

**Air-gapped / ohne Phone-Home:** Lasst ihr den Enrollment-Token weg, gibt es weder
Online-Gültigkeitsprüfung noch automatische Erneuerung — der ausgelieferte Token gilt
unverändert bis zu seinem Ablaufdatum, und ihr erhaltet rechtzeitig vorher eine neue
`versino.key`.

## Seats
`max_seats` begrenzt die Anzahl **distinkter** SAP-Nutzer. Werden mehr benötigt, über
das Lizenzportal aufstocken — die neue `versino.key` ersetzt die alte.

Fragen zur Lizenz: **support@versino.de**
