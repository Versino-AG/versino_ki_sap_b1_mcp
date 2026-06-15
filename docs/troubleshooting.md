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

## Linux: Binary startet nicht
- ausführbar gemacht? `chmod +x sapb1-mcp`
- 64-bit Linux mit glibc (x86-64) vorausgesetzt.

Weiterhin Probleme? **support@versino.de** (bitte mit Logausschnitt und Versionsnummer).
