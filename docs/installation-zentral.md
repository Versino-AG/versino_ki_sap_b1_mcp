# Running SAP B1 MCP centrally (one instance for all workstations)

> 🌐 **English** · [Deutsch](installation-zentral.de.md) · [Česky](installation-zentral.cs.md)

Instead of setting up the binary on every workstation individually, **one**
instance runs centrally on a server in the customer network. All users connect
their LLM client to it over the network — the workstation needs **no** `.exe`,
no `versino.key` and no `.env`, just the central URL entered in the client.

The per-user character is preserved: everyone signs in on connect with their
**own** SAP B1 credentials; seats count distinct SAP users across all
workstations.

```
  Workstation A ─┐
  Workstation B ─┼─ HTTPS ─▶ [ reverse proxy (TLS) ]
  Workstation C ─┘                   │  http://127.0.0.1:8000
                                     ▼
                           [ sapb1-mcp (service) ] ─ HTTPS:50000 ─▶ SAP B1 Service Layer
                             + versino.key + .env
```

## 1. Choose a server
- A permanently running Windows or Linux server (VM or physical), reachable in
  the users' LAN/VPN.
- Place it **close to the SAP Service Layer**: the server needs access to
  `https://<sap-host>:50000/b1s/v2/` (mind latency/firewall).
- Recommendation: reverse proxy and MCP server on the **same** machine — the MCP
  server then listens locally only (`127.0.0.1:8000`); only the TLS proxy is
  visible to the outside.

## 2. Base installation
Put binary, `versino.key` and `.env` into **one** folder — exactly as described
in [installation-windows.md](installation-windows.md) or
[installation-linux.md](installation-linux.md). Only **once** on the server, not
per workstation.

## 3. Central `.env`
Compared to the single-workstation setup, three settings matter for central
operation. Comments each on their **own line** (not behind the value):
```ini
# Service Layer URL of your SAP B1 instance
SAP_BASE_URL=https://your-sap-host:50000/b1s/v2/

# all selectable CompanyDBs (comma-separated)
SAP_DATABASES=SBO_CompanyA,SBO_CompanyB

# "basic" or "bearer" (browser SSO via Keycloak)
SAP_AUTH_MODE=basic

# READ_WRITE only when write access is wanted
SAP_OPERATION_MODE=READ_ONLY

# --- especially relevant centrally ---
# login ONLY via browser/dialog, never as a chat argument
SAP_DISABLE_INLINE_LOGIN=true

# public HTTPS URL of the instance (the TLS proxy's) — for the web login
SAP_PUBLIC_URL=https://mcp.customer.internal

# Phone-home / automatic subscription renewal runs automatically — the
# enrollment token is baked into versino.key, NOTHING to enter here. Only in a
# deliberately air-gapped setup (no internet) no phone-home happens.
```
- **`SAP_PUBLIC_URL`** is the URL under which users reach the server (the TLS
  proxy's). It **must be `https://`** — `http://` is allowed for `127.0.0.1`
  only. The server builds the browser-login URL
  (`<SAP_PUBLIC_URL>/login#t=…`) from it.
- **`SAP_DISABLE_INLINE_LOGIN=true`** enforces that SAP credentials are captured
  exclusively via the web login and never enter the LLM context — strongly
  recommended in multi-user operation.
- Full option list: [konfiguration.md](konfiguration.md).

## 4. Run permanently as a service
So the instance survives reboots and runs without a signed-in user.

**Linux (systemd)** — `/etc/systemd/system/sapb1-mcp.service`:
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

**Windows (service via NSSM)** — [nssm.cc](https://nssm.cc):
```powershell
nssm install sapb1-mcp "C:\sapb1-mcp\sapb1-mcp.exe" "--env-file" "C:\sapb1-mcp\.env" "--port" "8000"
nssm set sapb1-mcp AppDirectory "C:\sapb1-mcp"
nssm start sapb1-mcp
```
(The server binds all interfaces by default. If the reverse proxy sits on the
same machine, set `SAP_BIND_HOSTS=127.0.0.1` in the `.env` so the port is not
additionally open to the outside; if the proxy is on another host, open the port
internally only.)

## 5. TLS for network operation (required)
On the network, login tickets and sign-ins travel the wire — so TLS **must** be
on. There are two ways:

### Variant A — TLS directly in the MCP (native, no proxy)
The MCP server terminates HTTPS itself. In the `.env`:
```ini
SAP_TLS_CERT_FILE=/etc/ssl/mcp.crt      # certificate/chain (PEM)
SAP_TLS_KEY_FILE=/etc/ssl/mcp.key       # private key (PEM)
# SAP_TLS_KEY_PASSWORD=...               # only with an encrypted key
SAP_PUBLIC_URL=https://mcp.customer.internal:8000

# hardening (data center):
# SAP_TLS_MIN_VERSION=1.2                 # minimum TLS version (1.2 default, 1.3 possible)
# SAP_TLS_CIPHERS=...                     # pin an OpenSSL cipher string (TLS 1.2 only)
# SAP_TLS_CLIENT_CA_FILE=/etc/ssl/ca.pem # requires client certificates (mTLS)
```
Without these variables it stays HTTP. Clearly separate from
`SAP_ALLOW_SELF_SIGNED_CERT` (which applies **outbound** to SAP/Keycloak). The
server builds the TLS context hardened: enforced minimum version, server cipher
preference, compression/renegotiation off; optionally pinned ciphers and
**mTLS** (client-certificate requirement via `SAP_TLS_CLIENT_CA_FILE`). An
incomplete TLS config (only cert/key, missing file, invalid minimum version) is
reported clearly at startup. The endpoint is then directly
`https://<host>:<port>/mcp` — open the firewall accordingly (section 6).

### Variant B — TLS reverse proxy (in front)
The proxy terminates HTTPS and forwards **all** paths (`/mcp`, `/login`,
`/api/login`) to the local server. Important: **long read timeout** and **no
response buffering** (Streamable HTTP / server-sent events). Sensible when a
central proxy exists anyway or automatic certificates (e.g. Caddy/Let's
Encrypt) are wanted.

**Caddy** (simplest variant incl. automatic certificate) — `Caddyfile`:
```
mcp.customer.internal {
    reverse_proxy 127.0.0.1:8000 {
        flush_interval -1          # do not buffer SSE/streaming
    }
}
```

**nginx** — server block:
```nginx
server {
    listen 443 ssl;
    server_name mcp.customer.internal;
    ssl_certificate     /etc/ssl/mcp.customer.internal.crt;
    ssl_certificate_key /etc/ssl/mcp.customer.internal.key;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_buffering off;            # pass streaming/SSE through
        proxy_read_timeout 3600s;       # long MCP sessions
    }
}
```
- **Certificate:** a public one (Let's Encrypt, if the name is
  resolvable/reachable) or one from the **internal company CA**. With an
  internal CA the root certificate must be trusted on the workstations
  (otherwise browsers/clients refuse the connection).
- **Windows:** as a reverse proxy, **IIS with ARR/URL Rewrite** or Caddy works.
- Result: endpoint `https://mcp.customer.internal/mcp`, web login
  `https://mcp.customer.internal/login`.

### Hardening the proxy (recommended)
The server throttles failed sign-ins per client address and logs every attempt.
Behind a proxy it only sees the proxy's address unless you tell it whom to trust:

- In the `.env`: `SAP_TRUSTED_PROXIES=127.0.0.1` (the proxy's address as the
  server sees it). Then `X-Forwarded-For` is honoured — one counter per real
  client instead of one for the whole office.
- nginx: forward the client address and add a request limit on the sign-in
  paths — it stops floods before they reach the server:
  ```nginx
  limit_req_zone $binary_remote_addr zone=sapb1_login:10m rate=10r/m;
  server {
      # … TLS as above …
      add_header Strict-Transport-Security "max-age=31536000" always;
      location / {
          proxy_pass http://127.0.0.1:8000;
          proxy_http_version 1.1;
          proxy_set_header Host $host;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_buffering off;
          proxy_read_timeout 3600s;
      }
      location ~ ^/(api/login|login)$ {
          limit_req zone=sapb1_login burst=20 nodelay;
          proxy_pass http://127.0.0.1:8000;
          proxy_set_header Host $host;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      }
  }
  ```
- Restrict who can reach the proxy at all: an IP allow-list (`allow`/`deny`) or
  VPN-only exposure. The server's own gate is the SAP login; the network
  boundary is yours.
- What the server adds on top: after `SAP_LOGIN_MAX_FAILURES` failed attempts an
  address is paused (30 s, doubling to 15 min), browser-login links are capped
  per minute, every attempt lands in the audit log, and sessions end after
  `SAP_SESSION_MAX_SECONDS` (8 h) — see [konfiguration.md](konfiguration.md).

## 6. Firewall
- To the outside (towards the workstations) open only **443/TLS** of the proxy.
- Do **not** expose the MCP port `8000` to the network (only `127.0.0.1`).
- From the server, **50000** (Service Layer) must be reachable on the SAP host.

## 7. Connecting workstations (no local installation)
Each user enters only the **central URL** in the LLM client — nothing else.

**Claude Desktop, option A (recommended with TLS):** Settings → *Connectors* →
*Add custom connector* → URL `https://mcp.customer.internal/mcp`. Since the
instance runs over HTTPS, Claude Desktop accepts the URL directly — **no
Node.js/bridge** needed.

**Claude Desktop, option B (bridge, if preferred):**
`%APPDATA%\Claude\claude_desktop_config.json`:
```json
{
  "mcpServers": {
    "sapb1-mcp": {
      "command": "npx",
      "args": ["mcp-remote", "https://mcp.customer.internal/mcp"]
    }
  }
}
```

**Other clients** (Cline, Continue, Cursor, custom agents): streamable-HTTP-
capable ones directly at `https://mcp.customer.internal/mcp`; stdio-only clients
via the `mcp-remote` bridge.

> **Tip — rollout:** the `claude_desktop_config.json` can be distributed
> centrally via GPO/Intune so all workstations get the same connection
> automatically.

## 8. Per-user login & seats
- Call **`connect`** in the client → the server returns a browser-login URL
  (`https://mcp.customer.internal/login#t=…`) → the user signs in there with
  their **own** SAP credentials and picks the CompanyDB. The credentials never
  enter the chat. Then `connect(ticket="…")` — its result returns a **new**
  `ticket` value (the browser ticket is retired); that value accompanies every
  further SAP tool call and the SAP tools are ready.
- **Seats:** the central instance counts distinct SAP users across all
  workstations. The number is limited by the license's `max_seats` → see
  [lizenz.md](lizenz.md).

## 9. Operation & updates
- **Update:** swap the binary and restart the service
  (`systemctl restart sapb1-mcp` or `nssm restart sapb1-mcp`); `versino.key`
  stays in place. Subscription renewal happens automatically via phone-home →
  [lizenz.md](lizenz.md).
- **Logs:** the service logs to stdout/journald (Linux) or the NSSM log
  (Windows) — startup message `server.per_user_start`, warnings when TLS
  verification is off, etc.

## Troubleshooting
Common cases (`license.refused`, connection/login problems) in
[troubleshooting.md](troubleshooting.md).
