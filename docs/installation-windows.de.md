<!-- translation-of: installation-windows.md@b1c629aea651 -->
# SAP-B1-MCP an Claude Desktop & andere LLMs anbinden (Windows, manuell)

> 🌐 [English](installation-windows.md) · **Deutsch** · [Česky](installation-windows.cs.md)

> **Schneller geht es mit dem geführten Installer** — er nimmt euch die Schritte unten
> (Download, `.env` schreiben, Lizenz ablegen, Verbindung prüfen, Dienst, Claude-Config)
> ab: [installer.de.md](installer.de.md). Diese Anleitung beschreibt die **manuelle**
> Einrichtung für eigene Verzeichnisse, eigenen Dienst-Wrapper oder feinere Anpassung.

Anleitung für die Windows-Auslieferung (`sapb1-mcp.exe`). Der Server läuft als
**per-user HTTP-Server** (Streamable HTTP) — eine Instanz bedient mehrere Nutzer;
jeder meldet sich beim Verbinden mit seinen **eigenen** SAP-B1-Zugangsdaten an.

## 1. Herunterladen & ablegen
Aus den [Releases](../../releases/latest) `sapb1-mcp-<version>-windows-x64.exe` laden.
Zur Vereinfachung in `sapb1-mcp.exe` umbenennen und mit den übrigen Dateien in **einen**
Ordner legen (z. B. `C:\sapb1-mcp\`):
- `sapb1-mcp.exe` — der Server
- `versino.key` — eure Lizenz (wird **automatisch neben der .exe** gefunden, kein Pfad nötig)
- `.env` — Konfiguration (→ [konfiguration.de.md](konfiguration.de.md))

## 2. `.env` konfigurieren (minimal)
**Wichtig:** Kommentare immer in eine **eigene Zeile** schreiben — **nicht** hinter den
Wert in dieselbe Zeile (Inline-Kommentare können je nach Parser den Wert verfälschen).

```ini
# Service-Layer-URL eurer SAP-B1-Instanz
SAP_BASE_URL=https://ihr-sap-host:50000/b1s/v2/

# Wählbare CompanyDBs (eine oder mehrere, kommagetrennt)
SAP_DATABASES=SBO_IhreFirma

# Anmeldemodus: "basic" (User/Passwort) oder "bearer" (Browser-SSO über Keycloak)
SAP_AUTH_MODE=basic

# Zugriffsmodus: READ_ONLY oder READ_WRITE
SAP_OPERATION_MODE=READ_ONLY

# Nur bei selbstsigniertem SL-Zertifikat
SAP_ALLOW_SELF_SIGNED_CERT=true

# Öffentliche Adresse dieser Instanz — nötig für den Browser-Login in Claude
# Desktop (sonst liefert "connect" keinen Login-Link). Lokal/gleiche Maschine:
# http://127.0.0.1:8000 — der Port MUSS zum Serverstart (Abschnitt 3) passen.
# Im Netzbetrieb stattdessen die HTTPS-URL (siehe installation-zentral.de.md).
SAP_PUBLIC_URL=http://127.0.0.1:8000

# Phone-Home / automatische Abo-Erneuerung läuft automatisch — der Enrollment-Token
# ist in der versino.key eingebacken, hier ist NICHTS einzutragen. Nur bei bewusst
# air-gapped Betrieb (kein Internet) findet kein Phone-Home statt.
```
Die Lizenz wird über `versino.key` neben der .exe gezogen — `SAP_LICENSE_FILE` muss
**nicht** gesetzt werden. Vollständige Optionsliste: [konfiguration.de.md](konfiguration.de.md).

## 3. Server starten
```powershell
cd C:\sapb1-mcp
.\sapb1-mcp.exe --env-file .env --port 8000
```
Erfolg: Log zeigt `server.per_user_start` und `Uvicorn running on http://127.0.0.1:8000`.
Der MCP-Endpunkt ist dann **`http://127.0.0.1:8000/mcp`**.

- **Standard:** Der Server bindet **alle Interfaces** (`0.0.0.0`) — andere Rechner im Netz
  erreichen ihn direkt über `http://<interne-IP/DNS>:8000/mcp`; die **Windows-Firewall**
  für den Port freigeben. Für Netzwerkbetrieb wird **TLS** (Reverse-Proxy) empfohlen.
- **Nur lokal:** in der `.env` `SAP_BIND_HOSTS=127.0.0.1` setzen (kommagetrennte Liste
  möglich, z. B. `127.0.0.1,192.168.1.10`); ein explizites `--host` gewinnt.

> Für eine **zentrale** Instanz, die alle Arbeitsplätze bedient (ohne Installation je
> Arbeitsplatz), siehe [installation-zentral.de.md](installation-zentral.de.md).

## 4. An Claude Desktop anbinden
**Variante A — Custom Connector (URL):** Einstellungen → *Connectors* →
*Custom connector hinzufügen* → URL `http://127.0.0.1:8000/mcp` (bzw. die interne
HTTPS-URL). *Hinweis:* Claude Desktop bevorzugt **HTTPS** — für blankes `http://`
nutzt Variante B.

**Variante B — Bridge über die Konfigdatei** (funktioniert auch mit `http://`;
benötigt Node.js): `%APPDATA%\Claude\claude_desktop_config.json`:
```json
{
  "mcpServers": {
    "sapb1-mcp": {
      "command": "npx",
      "args": ["mcp-remote", "http://127.0.0.1:8000/mcp"]
    }
  }
}
```
Danach Claude Desktop neu starten.

## 5. Andere MCP-Clients (Cline, Continue, Cursor, eigene Agents …)
- **Streamable-HTTP-fähige Clients:** direkt auf `http://<host>:8000/mcp` zeigen.
- **Nur-stdio-Clients:** dieselbe `mcp-remote`-Bridge wie oben (`npx mcp-remote <url>`).

## 6. Erste Nutzung
Im Client das Tool **`connect`** aufrufen. Der Server fordert die SAP-Anmeldung an
(Eingabedialog des Clients bzw. Browser-Login) — die **Zugangsdaten gelangen nie in
den Chat-/LLM-Kontext**. Danach stehen die SAP-Tools bereit
(`sap_query_odata`, `sap_help`, `sap_create`, …); `list_databases` zeigt die CompanyDBs.

## Troubleshooting
Häufige Fälle (Defender-Warnung, `license.refused`, Verbindungsprobleme) findet ihr in
[troubleshooting.de.md](troubleshooting.de.md).
