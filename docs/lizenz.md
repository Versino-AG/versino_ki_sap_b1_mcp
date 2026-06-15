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

## Erneuerung (Abo)
Bei monatlicher/jährlicher Abrechnung verlängert sich die Lizenz automatisch. Wenn ihr
das optionale **Phone-Home** aktiviert habt, übernimmt die Installation den verlängerten
Schlüssel selbst — **kein manueller Key-Tausch** pro Abrechnungszeitraum:
```ini
SAP_LICENSE_VALIDATION_URL=https://aishop.versino.de/api/v1/validate
SAP_ENROLLMENT_TOKEN=<einmal-token aus der Auslieferung>
SAP_LICENSE_CACHE_FILE=license.renewed   # beschreibbarer Pfad → übernimmt den erneuerten Token
```
**Air-gapped / read-only Betrieb:** diese Variablen einfach weglassen — dann gilt der
ausgelieferte Token bis zum Ablaufdatum, und ihr erhaltet rechtzeitig eine neue
`versino.key`.

## Seats
`max_seats` begrenzt die Anzahl **distinkter** SAP-Nutzer. Werden mehr benötigt, über
das Lizenzportal aufstocken — die neue `versino.key` ersetzt die alte.

Fragen zur Lizenz: **support@versino.de**
