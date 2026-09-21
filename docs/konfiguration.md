# Configuration (`.env`)

> 🌐 **English** · [Deutsch](konfiguration.de.md) · [Česky](konfiguration.cs.md)

The server reads its instance configuration from a `.env` next to the binary
(or via `--env-file`). It contains **no end-user credentials** — only the
instance settings. Every user signs in themselves at runtime.

## Minimal configuration
Comments belong on their **own line** — **never** behind the value on the same
line (inline comments can corrupt the value depending on the parser).
```ini
# Service Layer URL of your SAP B1 instance
SAP_BASE_URL=https://your-sap-host:50000/b1s/v2/

# Selectable CompanyDBs (one or more, comma-separated)
SAP_DATABASES=SBO_YourCompany

# Auth mode: "basic" (user/password) or "bearer" (browser SSO with active
# IAM) — setup see sso-keycloak.md
SAP_AUTH_MODE=basic

# Access mode: READ_ONLY or READ_WRITE
SAP_OPERATION_MODE=READ_ONLY

# Only with a self-signed SL certificate
SAP_ALLOW_SELF_SIGNED_CERT=true

# Public address of this instance — needed for the browser login (Claude Desktop).
# Local: http://127.0.0.1:8000 (port must match the server start); on the network
# the HTTPS URL.
SAP_PUBLIC_URL=http://127.0.0.1:8000

# Phone-home / automatic subscription renewal runs automatically (the enrollment
# token is baked into versino.key — nothing needed here). Override only for
# test/staging:
# SAP_ENROLLMENT_TOKEN=<override-token>
```

## Service Layer
| Variable | Default | Meaning |
|---|---|---|
| `SAP_BASE_URL` | – | Service Layer URL, e.g. `https://host:50000/b1s/v2/` |
| `SAP_DATABASES` | – | selectable CompanyDBs (comma list) |
| `SAP_OPERATION_MODE` | `READ_ONLY` | `READ_ONLY` or `READ_WRITE`. In `READ_ONLY` the write tools (`sap_create`, `sap_update`, `sap_delete`, `sap_action`) are **not offered at all**; `sap_attachment` stays for `info`/`download` and refuses uploads, and `sap_deploy_queries` keeps its dry run (only the real rollout is refused). Restart the server after changing the mode (the LLM client re-reads the tool list on reconnect). Which tools exist also depends on the license edition → [lizenz.md](lizenz.md) |
| `SAP_ALLOW_SELF_SIGNED_CERT` | `false` | allow a self-signed SL certificate |
| `SAP_MAX_PAGE_SIZE` | `200` | max. rows per page |
| `SAP_MAX_CONCURRENT_REQUESTS` | `10` | parallel SL requests |
| `SAP_TIMEOUT_SECONDS` | `60` | HTTP timeout for Service Layer requests |
| `SAP_IDLE_LOGOUT_SECONDS` | `1500` | idle time after which a user session is signed out automatically |
| `SAP_SESSION_MAX_SECONDS` | `28800` | absolute lifetime of a user session (8 h) on top of the idle logout; afterwards the assistant is told to run `connect` again. `0` = unlimited |
| `SAP_PHONE_HOME_INTERVAL_SECONDS` | `3600` | interval of the license phone-home check |
| `SAP_DB_SERVER_TYPE` | _auto_ | override the database type (`HANA` / `MSSQL`); normally detected automatically — set only if detection is wrong |
| `SAP_AUTO_DEPLOY_QUERIES` | `true` | deploy the bundled reports on first connect per CompanyDB |

### Bundled reports
The server deploys its ready-made reports (`AI_*` queries in `SQLQueries`) on
the **first connect** per CompanyDB by itself — once per server run.
Prerequisites: `SAP_OPERATION_MODE=READ_WRITE`, a B1 user allowed to create
queries, and an edition that can run the reports. In read-only mode
(`READ_ONLY`) this is skipped, and with a master-data edition (BASIC) it is
skipped as well — `sap_curated_query` is not offered there, so the reports would
sit unusable in your `SQLQueries`. Everything else works unchanged. Self-written `AI_*` queries are never
overwritten. If needed, trigger the deployment in the chat
(`sap_deploy_queries`) or disable it with `SAP_AUTO_DEPLOY_QUERIES=false`.

## Authentication
| Variable | Meaning |
|---|---|
| `SAP_AUTH_MODE` | `basic` (user/password straight to the SL) or `bearer` (browser SSO with PKCE, token on every call — from FP 2208 with tokens of the SAP Authentication Server; if your own identity provider issues the tokens itself, see the note in sso-keycloak.md) |
| `SAP_DISABLE_INLINE_LOGIN` | login only via dialog/web UI, never as a chat argument. **Default `true` as soon as `SAP_PUBLIC_URL` is set** (network install); `false` only locally or as an explicit opt-in (`doctor` warns) |
| `SAP_ALLOWED_CLIENTS` | addresses/networks (IPs/CIDRs, comma-separated) that may reach the server at all. Empty (default) = everyone who reaches the port. **Not authentication** — sign-in stays user/password/database; this only removes the open internet. A typo aborts start-up naming the entry |
| `SAP_TICKET_SESSION_MAX_SECONDS` | absolute lifetime of a session addressed by a browser-login handle, in seconds (default `3600`, `0` = off). That handle travelled through the chat transcript and is passed on every call, so it is bounded more tightly than `SAP_SESSION_MAX_SECONDS`; the shorter of the two applies. Users then sign in again via `connect` |
| `SAP_TRUSTED_PROXIES` | reverse-proxy addresses (IPs/CIDRs, comma-separated) whose `X-Forwarded-For` the server trusts for the login throttle and the audit log — e.g. `127.0.0.1` when nginx runs on the same machine. Empty = the socket peer counts as the client (behind a proxy that would be the proxy itself, so the whole office shares one counter). A catch-all entry (`0.0.0.0/0`, `::/0`) is refused at start-up: trusting everyone would let any client claim a fresh address per request and turn the throttle off |
| `SAP_LOGIN_MAX_FAILURES` | failed sign-ins per client address within `SAP_LOGIN_WINDOW_SECONDS` before the address is paused (default `10`); the pause starts at 30 s and doubles up to 15 min, a successful sign-in resets it |
| `SAP_LOGIN_WINDOW_SECONDS` | window for `SAP_LOGIN_MAX_FAILURES` (default `900`) |
| `SAP_TICKET_ISSUE_PER_MINUTE` | global cap on browser-login links minted per minute (default `60`) — protects the unauthenticated path against floods |
| `SAP_TLS_CERT_FILE` | server certificate (PEM) for inbound HTTPS — together with `SAP_TLS_KEY_FILE`; otherwise HTTP |
| `SAP_TLS_KEY_FILE` | private key (PEM) for inbound HTTPS |
| `SAP_TLS_KEY_PASSWORD` | password for an encrypted TLS key (optional) |
| `SAP_TLS_MIN_VERSION` | minimum TLS version for inbound HTTPS: `1.2` (default) or `1.3` |
| `SAP_TLS_CIPHERS` | pin an OpenSSL cipher string (TLS 1.2 only; empty = safe defaults) |
| `SAP_TLS_CLIENT_CA_FILE` | client CA (PEM) → requires client certificates (mTLS) |
| `SAP_BIND_HOSTS` | bind addresses, comma-separated (default: all interfaces/`0.0.0.0`; `--host` on the command line wins) |
| `SAP_LANG` | language of all server-side texts (exe startup, `doctor`, web login incl. result/error messages, auth errors): `de` (default), `en`, `cs`. The guided installer writes the language chosen in its dialog here. The assistant's chat replies automatically follow the user's language |
| `SAP_PUBLIC_URL` | public HTTPS URL of the instance (web-login fallback; **required with `bearer`** — redirect target `…/callback`) |
| `SAP_UPLOAD_MAX_MB` | max. file size per attachment upload in MB (default `25`; browser upload, `source_url`, `file_path`) — see [anhaenge.md](anhaenge.md) |
| `SAP_ATTACHMENT_URL_ALLOWLIST` | hosts `sap_attachment` may fetch a `source_url` from, comma-separated (`host` or `*.domain`; empty = off) |
| `SAP_ATTACHMENT_DIR` | directory below which `sap_attachment` may read a `file_path` (empty = off) |

### Only with `SAP_AUTH_MODE=bearer`

Browser SSO with PKCE — the password never reaches the MCP. Required are
`SAP_PUBLIC_URL` plus the client data from the Extension Single Sign-On Manager:

| Variable | Meaning |
|---|---|
| `SAP_IDP_CLIENT_ID` | client id of the registered web-app client (`b1-ext-…`) |
| `SAP_IDP_CLIENT_SECRET` | its client secret (shown only once) |
| `SAP_SLD_URL` | SLD address (e.g. `https://host:40000`) — **required** for CompanyID resolution in variant A; also discovers all IdP endpoints automatically |
| `SAP_COMPANY_IDS` | **Variant B** only (own identity provider): the CompanyID per database noted during tenant binding, format `DB:ID`, several comma-separated (`SBO_PROD:1,SBO_TEST:2`). Must cover **all** databases from `SAP_DATABASES`. Replaces the SLD lookup — port 40000 is then not needed |
| `SAP_IDP_TOKEN_URL` | token endpoint — only needed when no automatic discovery via `SAP_SLD_URL` should happen |
| `SAP_IDP_AUTHORIZE_URL` | optional; derived from the token URL. **Required** when the identity provider is not a Keycloak (Entra ID, Okta) |
| `SAP_IDP_SCOPE` | optional, default `openid`. Extended scopes only if assigned to the client |
| `SAP_IDP_END_SESSION_URL` / `SAP_IDP_JWKS_URL` | optional (sign-out/central logout); also come from the automatic discovery |
| `SAP_IDP_ISSUER` / `SAP_IDP_REVOCATION_URL` | optional (token issuer / token revocation); normally discovered automatically via `SAP_SLD_URL` — set only when the auto-discovery must be overridden |

The former `SAP_KEYCLOAK_*` names keep working (same meaning).
The redirect URI `<SAP_PUBLIC_URL>/callback` must be registered on the client
(Extension Single Sign-On Manager). Full guide incl. the SAP-side steps:
[sso-keycloak.md](sso-keycloak.md).

## License
| Variable | Meaning |
|---|---|
| `SAP_LICENSE_FILE` | path to the `versino.key` — **not needed** when the file sits next to the binary (auto-discovery) |
| `SAP_LICENSE` | license token inline (alternative to the file) |
| `SAP_LICENSE_CACHE_FILE` | path for renewed tokens (silent renewal cache); default `versino.renewed` next to the license/binary |
| `SAP_INSTALL_IDENTITY_PATH` | path of the installation identity (`install_identity.json`); default relative to the working directory — set a persistent path in containers (read-only rootfs) |
| `SAP_TIME_ANCHOR_PATH` | monotonic time anchor against clock roll-back. **On by default** since 3.8.1: the anchor file sits next to `versino.key`. Set a path to move it, or set it to an empty value to switch it off (read-only containers) |
| `SAP_ENROLLMENT_TOKEN` | phone-home/auto-renewal — **normally not needed** (the token is baked into `versino.key`). Only as an override for test/staging |
| `SAP_LICENSE_VALIDATION_URL` | override for the built-in validation endpoint (test/staging). **Must be `https`** — the request carries the customer id and the seat count, so plain `http` to a remote host is refused; only `127.0.0.1`, `::1` and `localhost` may be plain. The enrolment address is derived from this URL's directory, so keep the path intact |

Phone-home / automatic subscription renewal (the normal case): see
[lizenz.md](lizenz.md).

## Security notes
- `SAP_OPERATION_MODE=READ_ONLY` as the default; `READ_WRITE` only when write
  access is really wanted. A read-only instance does not even show the write
  tools to the LLM client, so the assistant cannot try them.
- `SAP_DISABLE_INLINE_LOGIN=true` makes sure SAP credentials never enter the
  LLM context (sign-in only via dialog/web UI). It is the default on network
  installs (`SAP_PUBLIC_URL` set).
- The browser login is hardened out of the box: the sign-in link carries the
  ticket in the URL **fragment** (`…/login#t=…`, never in server or proxy logs);
  after `connect(ticket=…)` the ticket is retired and a fresh session value is
  returned; failed sign-ins are throttled per address and every attempt is
  written to the audit log (user, CompanyDB, source address, outcome — never
  the password); sessions end after `SAP_SESSION_MAX_SECONDS` (8 h), and a session
  addressed by a browser-login handle after `SAP_TICKET_SESSION_MAX_SECONDS` (1 h) —
  the shorter limit wins.
- Behind a reverse proxy set `SAP_TRUSTED_PROXIES` so the throttle sees real
  client addresses, and add the proxy-side limits from
  [installation-zentral.md](installation-zentral.md).
- `sapb1-mcp doctor` reads the test password from **`SAP_DOCTOR_PASSWORD`**, not
  from `--password`: a password on the command line is visible in the process list
  and, with command-line auditing on, in the Windows event log, where it outlives the
  installation. The flag still works but warns. The variable is meant for the single
  `doctor` call (the installer sets it for that child process only) — it does not
  belong in the `.env`.
- For network operation put TLS in front (native or reverse proxy).

## Logging

Warnings and errors always land in `%APPDATA%\Versino\sapb1-mcp\sapb1-mcp.log`
(JSON lines, 1 MB cap — the oldest entries are pruned automatically).
The path is fixed so support always knows where to look.
