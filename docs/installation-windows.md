# Connecting SAP B1 MCP to Claude Desktop & other LLMs (Windows, manual)

> 🌐 **English** · [Deutsch](installation-windows.de.md) · [Česky](installation-windows.cs.md)

> **The guided installer is faster** — it takes the steps below off your hands
> (download, writing the `.env`, placing the license, connection check, service,
> Claude config): [installer.md](installer.md). This guide describes the
> **manual** setup for custom directories, your own service wrapper or finer
> tuning.

Guide for the Windows delivery (`sapb1-mcp.exe`). The server runs as a
**per-user HTTP server** (Streamable HTTP) — one instance serves several users;
each signs in on connect with their **own** SAP B1 credentials.

## 1. Download & place
Download `sapb1-mcp-<version>-windows-x64.exe` from the
[releases](../../releases/latest). For convenience rename it to `sapb1-mcp.exe`
and put it with the other files into **one** folder (e.g. `C:\sapb1-mcp\`):
- `sapb1-mcp.exe` — the server
- `versino.key` — your license (found **automatically next to the .exe**, no path needed)
- `.env` — configuration (→ [konfiguration.md](konfiguration.md))

## 2. Configure the `.env` (minimal)
**Important:** always put comments on their **own line** — **not** behind the
value on the same line (inline comments can corrupt the value depending on the
parser).

```ini
# Service Layer URL of your SAP B1 instance
SAP_BASE_URL=https://your-sap-host:50000/b1s/v2/

# Selectable CompanyDBs (one or more, comma-separated)
SAP_DATABASES=SBO_YourCompany

# Auth mode: "basic" (user/password) or "bearer" (browser SSO via Keycloak)
SAP_AUTH_MODE=basic

# Access mode: READ_ONLY or READ_WRITE
SAP_OPERATION_MODE=READ_ONLY

# Only with a self-signed SL certificate
SAP_ALLOW_SELF_SIGNED_CERT=true

# Public address of this instance — needed for the browser login in Claude
# Desktop (otherwise "connect" returns no sign-in link). Local/same machine:
# http://127.0.0.1:8000 — the port MUST match the server start (section 3).
# On the network use the HTTPS URL instead (see installation-zentral.md).
SAP_PUBLIC_URL=http://127.0.0.1:8000

# Phone-home / automatic subscription renewal runs automatically — the
# enrollment token is baked into versino.key, NOTHING to enter here. Only in a
# deliberately air-gapped setup (no internet) no phone-home happens.
```
The license is picked up via `versino.key` next to the .exe — `SAP_LICENSE_FILE`
does **not** need to be set. Full option list: [konfiguration.md](konfiguration.md).

## 3. Start the server
```powershell
cd C:\sapb1-mcp
.\sapb1-mcp.exe --env-file .env --port 8000
```
Success: the log shows `server.per_user_start` and
`Uvicorn running on http://127.0.0.1:8000`. The MCP endpoint is then
**`http://127.0.0.1:8000/mcp`**.

- **Default:** the server binds **all interfaces** (`0.0.0.0`) — other machines
  on the network reach it directly via `http://<internal-IP/DNS>:8000/mcp`; open
  the **Windows firewall** for the port. For network operation **TLS**
  (reverse proxy) is recommended.
- **Local only:** set `SAP_BIND_HOSTS=127.0.0.1` in the `.env` (comma-separated
  list possible, e.g. `127.0.0.1,192.168.1.10`); an explicit `--host` wins.

> For a **central** instance serving all workstations (no per-workstation
> install) see [installation-zentral.md](installation-zentral.md).

## 4. Connect to Claude Desktop
**Option A — custom connector (URL):** Settings → *Connectors* →
*Add custom connector* → URL `http://127.0.0.1:8000/mcp` (or the internal
HTTPS URL). *Note:* Claude Desktop prefers **HTTPS** — for plain `http://` use
option B.

**Option B — bridge via the config file** (also works with `http://`;
requires Node.js): `%APPDATA%\Claude\claude_desktop_config.json`:
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
Then restart Claude Desktop.

## 5. Other MCP clients (Cline, Continue, Cursor, custom agents …)
- **Streamable-HTTP-capable clients:** point directly at `http://<host>:8000/mcp`.
- **stdio-only clients:** the same `mcp-remote` bridge as above (`npx mcp-remote <url>`).

## 6. First use
Call the **`connect`** tool in the client. The server requests the SAP sign-in
(client input dialog or browser login) — the **credentials never enter the
chat/LLM context**. After that the SAP tools are available
(`sap_query_odata`, `sap_help`, `sap_create`, …); `list_databases` shows the
CompanyDBs.

## Troubleshooting
Common cases (Defender warning, `license.refused`, connection problems) are in
[troubleshooting.md](troubleshooting.md).
