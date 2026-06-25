# Troubleshooting

## `license.refused` beim Start
Keine gültige Lizenz gefunden. Prüfen:
- liegt `versino.key` **neben** dem Binary? (oder zeigt `SAP_LICENSE_FILE` darauf?)
- ist der Schlüssel nicht abgelaufen? (neue über das [Lizenzportal](https://aishop.versino.de))

Details: [lizenz.md](lizenz.md).

## Windows-Defender / SmartScreen-Warnung
Ist das Binary noch nicht signiert, kann Windows einen Fehlalarm zeigen.
- „Weitere Informationen" → „Trotzdem ausführen", bzw. die Datei in Defender zulassen.
- Für die produktive Auslieferung ist Code-Signing vorgesehen — dann entfällt die Warnung.

## Client erreicht den Server nicht
- Host/Port und den **`/mcp`**-Pfad in der URL prüfen.
- Bei Zugriff von anderen Rechnern: Server mit `--host 0.0.0.0` starten, **Firewall**
  für den Port freigeben und die **interne IP/DNS** des Servers in der Client-URL verwenden.
- Claude Desktop bevorzugt **HTTPS**; für blankes `http://` die `mcp-remote`-Bridge nutzen
  (siehe [installation-windows.md](installation-windows.md)).

## Verbindung zu SAP schlägt fehl
- `SAP_BASE_URL` korrekt? (`https://<host>:50000/b1s/v2/`)
- selbstsigniertes SL-Zertifikat → `SAP_ALLOW_SELF_SIGNED_CERT=true`.
- richtiger `SAP_AUTH_MODE` (`basic` vs `ropc`)?
- gewählte CompanyDB in `SAP_DATABASES` enthalten?

## Schreib-Tools fehlen / werden abgelehnt
- `SAP_OPERATION_MODE=READ_WRITE` setzen (Default ist `READ_ONLY`).
- Schreib-/Lösch-Tools sind zusätzlich an die **Edition** gebunden (PRO/ENTERPRISE),
  siehe [lizenz.md](lizenz.md).

## Windows: Fenster schließt sich sofort wieder
Meist ist der **Port schon belegt** (ein anderer Dienst lauscht auf `8000`; im Log
`WinError 10048` / „… nur jeweils einmal verwendet werden"). Lösung:
- Server auf einem **freien Port** starten: `sapb1-mcp.exe --port 8765` — und im Client
  dieselbe Portnummer verwenden (`…:8765/mcp`).
- Oder den belegenden Dienst beenden.
- Damit ihr die Fehlermeldung **seht** statt eines sofort schließenden Fensters: die `.exe`
  aus einem geöffneten Terminal (PowerShell) starten, nicht per Doppelklick.

## `connect` liefert keinen Login-Link (Web-Login)
Der Browser-Login-Fallback braucht die **öffentliche Adresse** der Instanz. In der `.env`
`SAP_PUBLIC_URL` setzen (im Netz `https://…`, lokal `http://127.0.0.1:8000`) und den Server
neu starten. Siehe [konfiguration.md](konfiguration.md) und
[installation-zentral.md](installation-zentral.md).

## Login-Seite reagiert beim Klick nicht / keine Bestätigung
Beim Klick auf „Anmelden" passiert nichts und es kommt keine Erfolgsseite — fast immer ist
der **SAP Service Layer nicht erreichbar** (die Anmeldung läuft ins Leere/Timeout):
- Ist der SAP-Host vom **Server** aus erreichbar? (oft **VPN** nötig, Firewall, Port `50000`).
  Test: `Test-NetConnection <sap-host> -Port 50000` (Windows) bzw. `nc -vz <sap-host> 50000`.
- `SAP_BASE_URL` korrekt (`https://<host>:50000/b1s/v2/`)?
- Tickets sind kurzlebig: zwischen Linköffnen und Login nicht zu lange warten.

## `npx` / Node.js nicht gefunden (Bridge)
Die `mcp-remote`-Bridge benötigt **Node.js**. Node LTS von
[nodejs.org](https://nodejs.org) installieren, Client neu starten. Tipp: in der Client-Config
`npx` durch `cmd /c npx` ersetzen (Windows), falls der Befehl nicht gefunden wird.

## Linux: Binary startet nicht
- ausführbar gemacht? `chmod +x sapb1-mcp`
- 64-bit Linux mit glibc (x86-64) vorausgesetzt.

Weiterhin Probleme? **support@versino.de** (bitte mit Logausschnitt und Versionsnummer).
