# Running SAP B1 MCP on Linux (manual)

> 🌐 **English** · [Deutsch](installation-linux.de.md) · [Česky](installation-linux.cs.md)

> For **Windows** there is a **guided installer** that automates most of the
> setup ([installer.md](installer.md)). On Linux the setup is manual — this
> guide.

Guide for the Linux delivery (`sapb1-mcp-<version>-linux-x64`). The server runs
as a **per-user HTTP server** (Streamable HTTP) — one instance serves several
users; each signs in on connect with their **own** SAP B1 credentials.

## 1. Download & place
Download `sapb1-mcp-<version>-linux-x64` from the
[releases](../../releases/latest) and put it into **one** folder
(e.g. `/opt/sapb1-mcp/`):
- the binary (rename for convenience: `mv sapb1-mcp-*-linux-x64 sapb1-mcp`)
- `versino.key` — your license (found **automatically next to the binary**)
- `.env` — configuration (→ [konfiguration.md](konfiguration.md))

Make it executable:
```bash
chmod +x sapb1-mcp
```

## 2. `.env` (minimal)
Always put comments on their **own line** (not behind the value — inline
comments can corrupt the value depending on the parser):
```ini
# Service Layer URL of your SAP B1 instance
SAP_BASE_URL=https://your-sap-host:50000/b1s/v2/

# Selectable CompanyDBs (one or more, comma-separated)
SAP_DATABASES=SBO_YourCompany

# Auth mode: "basic" or "bearer" (browser SSO via Keycloak)
SAP_AUTH_MODE=basic

# Access mode: READ_ONLY or READ_WRITE
SAP_OPERATION_MODE=READ_ONLY

# Only with a self-signed SL certificate
SAP_ALLOW_SELF_SIGNED_CERT=true

# Public address of this instance — needed for the browser login (otherwise
# "connect" returns no sign-in link). Local: http://127.0.0.1:8000 (port must
# match the server start). On the network the HTTPS URL (see
# installation-zentral.md).
SAP_PUBLIC_URL=http://127.0.0.1:8000

# Phone-home / automatic subscription renewal runs automatically — the
# enrollment token is baked into versino.key, NOTHING to enter here. Only in a
# deliberately air-gapped setup (no internet) no phone-home happens.
```
Full option list: [konfiguration.md](konfiguration.md).

## 3. Start the server
```bash
cd /opt/sapb1-mcp
./sapb1-mcp --env-file .env --port 8000
```
Success: the log shows `server.per_user_start` and
`Uvicorn running on http://127.0.0.1:8000`. The MCP endpoint is then
**`http://127.0.0.1:8000/mcp`**.

- **Default:** the server binds **all interfaces** (`0.0.0.0`) — use the
  internal IP/DNS in the URL, open the port in the firewall. For network
  operation **TLS** (reverse proxy) is recommended.
- **Local only:** set `SAP_BIND_HOSTS=127.0.0.1` in the `.env` (comma-separated
  list possible); an explicit `--host` wins.

> For a **central** instance serving all workstations (no per-workstation
> install) see [installation-zentral.md](installation-zentral.md).

## 4. As a systemd service (optional)
`/etc/systemd/system/sapb1-mcp.service`:
```ini
[Unit]
Description=SAP B1 MCP Server
After=network-online.target

[Service]
WorkingDirectory=/opt/sapb1-mcp
ExecStart=/opt/sapb1-mcp/sapb1-mcp --env-file /opt/sapb1-mcp/.env --port 8000
Restart=on-failure
User=sapb1mcp

[Install]
WantedBy=multi-user.target
```
```bash
sudo systemctl enable --now sapb1-mcp
```

## 5. Connecting LLM clients
- **Streamable-HTTP-capable clients** (Claude Desktop custom connector, Cline,
  Continue, Cursor …): point at `http://<host>:8000/mcp`.
- **stdio-only clients:** bridge `npx mcp-remote http://<host>:8000/mcp`
  (requires Node.js).

The Claude Desktop steps are identical to Windows —
see [installation-windows.md](installation-windows.md), section 4.

## 6. First use
Call the **`connect`** tool in the client → SAP sign-in (input dialog/browser
login); the credentials never enter the chat/LLM context. After that the SAP
tools are available (`sap_query_odata`, `sap_help`, `sap_create`, …).

## Notes
- Prerequisite: 64-bit Linux with glibc (x86-64).
- **Container operation (Docker):** on request — **support@versino.de**.
- Common cases (`license.refused`, connection problems) in
  [troubleshooting.md](troubleshooting.md).
