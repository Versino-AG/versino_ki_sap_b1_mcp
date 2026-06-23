# SAP-B1-MCP zentral betreiben (eine Instanz für alle Arbeitsplätze)

Statt das Binary auf jedem Arbeitsplatz einzeln einzurichten, läuft **eine** Instanz
zentral auf einem Server im Kundennetz. Alle Nutzer verbinden ihren LLM-Client über
das Netzwerk darauf — am Arbeitsplatz ist **keine** `.exe`, keine `versino.key` und
keine `.env` nötig, nur ein Eintrag der zentralen URL im Client.

Der Per-User-Charakter bleibt erhalten: Jeder meldet sich beim Verbinden mit den
**eigenen** SAP-B1-Zugangsdaten an, Seats zählen distinkte SAP-Nutzer über alle
Arbeitsplätze hinweg.

```
  Arbeitsplatz A ─┐
  Arbeitsplatz B ─┼─ HTTPS ─▶ [ Reverse-Proxy (TLS) ]
  Arbeitsplatz C ─┘                   │  http://127.0.0.1:8000
                                      ▼
                            [ sapb1-mcp (Dienst) ] ─ HTTPS:50000 ─▶ SAP B1 Service Layer
                              + versino.key + .env
```

## 1. Server wählen
- Ein dauerhaft laufender Windows- oder Linux-Server (VM oder physisch), erreichbar im
  LAN/VPN der Nutzer.
- **Nahe am SAP Service Layer** platzieren: der Server braucht Zugriff auf
  `https://<sap-host>:50000/b1s/v2/` (Latenz/Firewall beachten).
- Empfehlung: Reverse-Proxy und MCP-Server auf **derselben** Maschine — der MCP-Server
  lauscht dann nur lokal (`127.0.0.1:8000`), nach außen ist ausschließlich der
  TLS-Proxy sichtbar.

## 2. Grundinstallation
Binary, `versino.key` und `.env` in **einen** Ordner legen — exakt wie in
[installation-windows.md](installation-windows.md) bzw. [installation-linux.md](installation-linux.md)
beschrieben. Nur **einmal** auf dem Server, nicht pro Arbeitsplatz.

## 3. Zentrale `.env`
Gegenüber dem Einzelplatz-Setup sind drei Einstellungen für den zentralen Betrieb
wichtig:
Kommentare jeweils in eine **eigene Zeile** (nicht hinter den Wert):
```ini
# Service-Layer-URL eurer SAP-B1-Instanz
SAP_BASE_URL=https://ihr-sap-host:50000/b1s/v2/

# alle wählbaren CompanyDBs (kommagetrennt)
SAP_DATABASES=SBO_FirmaA,SBO_FirmaB

# "basic" oder "ropc" (Keycloak/SSO)
SAP_AUTH_MODE=basic

# READ_WRITE nur bei gewolltem Schreibzugriff
SAP_OPERATION_MODE=READ_ONLY

# --- zentral besonders relevant ---
# Login NUR über Browser/Dialog, nie als Chat-Argument
SAP_DISABLE_INLINE_LOGIN=true

# öffentliche HTTPS-URL der Instanz (die des TLS-Proxys) — für den Web-Login
SAP_PUBLIC_URL=https://mcp.kunde.intern
```
- **`SAP_PUBLIC_URL`** ist die URL, unter der die Nutzer den Server erreichen (die des
  TLS-Proxys). Sie **muss `https://`** sein — `http://` ist nur für `127.0.0.1` erlaubt.
  Über sie baut der Server die Browser-Login-URL (`<SAP_PUBLIC_URL>/login?t=…`).
- **`SAP_DISABLE_INLINE_LOGIN=true`** erzwingt, dass SAP-Zugangsdaten ausschließlich über
  den Web-Login erfasst werden und nie in den LLM-Kontext geraten — im Mehrbenutzerbetrieb
  dringend empfohlen.
- Vollständige Optionsliste: [konfiguration.md](konfiguration.md).

## 4. Als Dienst dauerhaft betreiben
Damit die Instanz Neustarts überlebt und ohne angemeldeten Benutzer läuft.

**Linux (systemd)** — `/etc/systemd/system/sapb1-mcp.service`:
```ini
[Unit]
Description=SAP B1 MCP Server
After=network-online.target

[Service]
WorkingDirectory=/opt/sapb1-mcp
ExecStart=/opt/sapb1-mcp/sapb1-mcp --env-file /opt/sapb1-mcp/.env --host 127.0.0.1 --port 8000
Restart=on-failure
User=sapb1mcp

[Install]
WantedBy=multi-user.target
```
```bash
sudo systemctl enable --now sapb1-mcp
```

**Windows (Dienst via NSSM)** — [nssm.cc](https://nssm.cc):
```powershell
nssm install sapb1-mcp "C:\sapb1-mcp\sapb1-mcp.exe" "--env-file" "C:\sapb1-mcp\.env" "--host" "127.0.0.1" "--port" "8000"
nssm set sapb1-mcp AppDirectory "C:\sapb1-mcp"
nssm start sapb1-mcp
```
(`--host 127.0.0.1`, weil der Proxy auf derselben Maschine vorgeschaltet wird. Liegt der
Proxy auf einem anderen Host, `--host 0.0.0.0` und den Port nur intern freigeben.)

## 5. TLS-Reverse-Proxy (für Netzbetrieb erforderlich)
Im Netz laufen Login-Tickets und Anmeldungen über die Leitung — daher **muss** TLS davor.
Der Proxy terminiert HTTPS und leitet **alle** Pfade (`/mcp`, `/login`, `/api/login`) an
den lokalen Server weiter. Wichtig: **langes Lese-Timeout** und **kein Response-Buffering**
(Streamable HTTP / Server-Sent Events).

**Caddy** (einfachste Variante inkl. automatischem Zertifikat) — `Caddyfile`:
```
mcp.kunde.intern {
    reverse_proxy 127.0.0.1:8000 {
        flush_interval -1          # SSE/Streaming nicht puffern
    }
}
```

**nginx** — Server-Block:
```nginx
server {
    listen 443 ssl;
    server_name mcp.kunde.intern;
    ssl_certificate     /etc/ssl/mcp.kunde.intern.crt;
    ssl_certificate_key /etc/ssl/mcp.kunde.intern.key;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_buffering off;            # Streaming/SSE durchreichen
        proxy_read_timeout 3600s;       # lange MCP-Sessions
    }
}
```
- **Zertifikat:** öffentliches (Let's Encrypt, wenn der Name auflösbar/erreichbar ist)
  oder eines aus der **internen Unternehmens-CA**. Bei interner CA muss das Stammzertifikat
  auf den Arbeitsplätzen vertrauenswürdig sein (sonst lehnen Browser/Client die Verbindung ab).
- **Windows:** als Reverse-Proxy eignet sich **IIS mit ARR/URL Rewrite** oder ebenfalls Caddy.
- Ergebnis: Endpunkt `https://mcp.kunde.intern/mcp`, Web-Login `https://mcp.kunde.intern/login`.

## 6. Firewall
- Nach außen (an die Arbeitsplätze) nur **443/TLS** des Proxys öffnen.
- Den MCP-Port `8000` **nicht** ins Netz exponieren (nur `127.0.0.1`).
- Vom Server zum SAP-Host muss **50000** (Service Layer) erreichbar sein.

## 7. Arbeitsplätze anbinden (ohne lokale Installation)
Jeder Nutzer trägt nur die **zentrale URL** im LLM-Client ein — sonst nichts.

**Claude Desktop, Variante A (empfohlen bei TLS):** Einstellungen → *Connectors* →
*Custom connector hinzufügen* → URL `https://mcp.kunde.intern/mcp`. Da die Instanz jetzt
über HTTPS läuft, akzeptiert Claude Desktop die URL direkt — **kein Node.js/keine Bridge**
nötig.

**Claude Desktop, Variante B (Bridge, falls gewünscht):**
`%APPDATA%\Claude\claude_desktop_config.json`:
```json
{
  "mcpServers": {
    "sapb1-mcp": {
      "command": "npx",
      "args": ["mcp-remote", "https://mcp.kunde.intern/mcp"]
    }
  }
}
```

**Andere Clients** (Cline, Continue, Cursor, eigene Agents): streamable-HTTP-fähige direkt
auf `https://mcp.kunde.intern/mcp`; nur-stdio-Clients über die `mcp-remote`-Bridge.

> **Tipp — Rollout:** Die `claude_desktop_config.json` lässt sich per GPO/Intune zentral
> verteilen, damit alle Arbeitsplätze automatisch dieselbe Anbindung erhalten.

## 8. Login pro Nutzer & Seats
- Im Client **`connect`** aufrufen → der Server liefert eine Browser-Login-URL
  (`https://mcp.kunde.intern/login?t=…`) → der Nutzer meldet sich dort mit seinen
  **eigenen** SAP-Zugangsdaten an und wählt die CompanyDB. Die Zugangsdaten gelangen nie
  in den Chat. Danach `connect(ticket="…")` und die SAP-Tools stehen bereit.
- **Seats:** Die zentrale Instanz zählt distinkte SAP-Nutzer über alle Arbeitsplätze.
  Die Anzahl ist durch `max_seats` der Lizenz begrenzt → siehe [lizenz.md](lizenz.md).

## 9. Betrieb & Updates
- **Update:** Binary austauschen und den Dienst neu starten (`systemctl restart sapb1-mcp`
  bzw. `nssm restart sapb1-mcp`); `versino.key` bleibt liegen. Abo-Erneuerung erfolgt
  automatisch über Phone-Home → [lizenz.md](lizenz.md).
- **Logs:** der Dienst loggt nach stdout/journald (Linux) bzw. ins NSSM-Log (Windows) —
  Startmeldung `server.per_user_start`, Warnungen bei abgeschalteter TLS-Prüfung etc.

## Troubleshooting
Häufige Fälle (`license.refused`, Verbindungs-/Login-Probleme) in
[troubleshooting.md](troubleshooting.md).
