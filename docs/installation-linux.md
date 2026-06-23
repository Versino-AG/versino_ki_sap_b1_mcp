# SAP-B1-MCP unter Linux betreiben

Anleitung für die Linux-Auslieferung (`sapb1-mcp-<version>-linux-x64`). Der Server läuft
als **per-user HTTP-Server** (Streamable HTTP) — eine Instanz bedient mehrere Nutzer;
jeder meldet sich beim Verbinden mit seinen **eigenen** SAP-B1-Zugangsdaten an.

## 1. Herunterladen & ablegen
Aus den [Releases](../../releases/latest) `sapb1-mcp-<version>-linux-x64` laden und in
**einen** Ordner legen (z. B. `/opt/sapb1-mcp/`):
- das Binary (zur Vereinfachung umbenennen: `mv sapb1-mcp-*-linux-x64 sapb1-mcp`)
- `versino.key` — eure Lizenz (wird **automatisch neben dem Binary** gefunden)
- `.env` — Konfiguration (→ [konfiguration.md](konfiguration.md))

Ausführbar machen:
```bash
chmod +x sapb1-mcp
```

## 2. `.env` (minimal)
Kommentare immer in eine **eigene Zeile** (nicht hinter den Wert — Inline-Kommentare
können je nach Parser den Wert verfälschen):
```ini
# Service-Layer-URL eurer SAP-B1-Instanz
SAP_BASE_URL=https://ihr-sap-host:50000/b1s/v2/

# Wählbare CompanyDBs (eine oder mehrere, kommagetrennt)
SAP_DATABASES=SBO_IhreFirma

# Anmeldemodus: "basic" oder "ropc" (Keycloak/SSO)
SAP_AUTH_MODE=basic

# Zugriffsmodus: READ_ONLY oder READ_WRITE
SAP_OPERATION_MODE=READ_ONLY

# Nur bei selbstsigniertem SL-Zertifikat
SAP_ALLOW_SELF_SIGNED_CERT=true

# Öffentliche Adresse dieser Instanz — nötig für den Browser-Login (sonst liefert
# "connect" keinen Login-Link). Lokal: http://127.0.0.1:8000 (Port muss zum
# Serverstart passen). Im Netzbetrieb die HTTPS-URL (siehe installation-zentral.md).
SAP_PUBLIC_URL=http://127.0.0.1:8000
```
Vollständige Optionsliste: [konfiguration.md](konfiguration.md).

## 3. Server starten
```bash
cd /opt/sapb1-mcp
./sapb1-mcp --env-file .env --host 127.0.0.1 --port 8000
```
Erfolg: Log zeigt `server.per_user_start` und `Uvicorn running on http://127.0.0.1:8000`.
Der MCP-Endpunkt ist dann **`http://127.0.0.1:8000/mcp`**.

- **Lokaler Zugriff:** `--host 127.0.0.1` genügt.
- **Netzwerkzugriff:** `--host 0.0.0.0`, interne IP/DNS in der URL verwenden, Port in der
  Firewall freigeben. Für Netzwerkbetrieb wird **TLS** (Reverse-Proxy) empfohlen.

> Für eine **zentrale** Instanz, die alle Arbeitsplätze bedient (ohne Installation je
> Arbeitsplatz), siehe [installation-zentral.md](installation-zentral.md).

## 4. Als systemd-Dienst (optional)
`/etc/systemd/system/sapb1-mcp.service`:
```ini
[Unit]
Description=SAP B1 MCP Server
After=network-online.target

[Service]
WorkingDirectory=/opt/sapb1-mcp
ExecStart=/opt/sapb1-mcp/sapb1-mcp --env-file /opt/sapb1-mcp/.env --host 0.0.0.0 --port 8000
Restart=on-failure
User=sapb1mcp

[Install]
WantedBy=multi-user.target
```
```bash
sudo systemctl enable --now sapb1-mcp
```

## 5. An LLM-Clients anbinden
- **Streamable-HTTP-fähige Clients** (Claude Desktop Custom Connector, Cline, Continue,
  Cursor …): auf `http://<host>:8000/mcp` zeigen.
- **Nur-stdio-Clients:** Bridge `npx mcp-remote http://<host>:8000/mcp` (benötigt Node.js).

Die Schritte zur Claude-Desktop-Anbindung sind identisch zu Windows —
siehe [installation-windows.md](installation-windows.md), Abschnitt 4.

## 6. Erste Nutzung
Im Client das Tool **`connect`** aufrufen → SAP-Anmeldung (Eingabedialog/Browser-Login);
die Zugangsdaten gelangen nie in den Chat-/LLM-Kontext. Danach stehen die SAP-Tools
bereit (`sap_query_odata`, `sap_help`, `sap_create`, …).

## Hinweise
- Voraussetzung: 64-bit Linux mit glibc (x86-64).
- **Container-Betrieb (Docker):** auf Anfrage — **support@versino.de**.
- Häufige Fälle (`license.refused`, Verbindungsprobleme) in [troubleshooting.md](troubleshooting.md).
