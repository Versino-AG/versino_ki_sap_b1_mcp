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
| `SAP_OPERATION_MODE` | `READ_ONLY` | `READ_ONLY` or `READ_WRITE` (write tools) |
| `SAP_ALLOW_SELF_SIGNED_CERT` | `false` | allow a self-signed SL certificate |
| `SAP_MAX_PAGE_SIZE` | `200` | max. rows per page |
| `SAP_MAX_CONCURRENT_REQUESTS` | `10` | parallel SL requests |
| `SAP_TIMEOUT_SECONDS` | `60` | HTTP timeout for Service Layer requests |
| `SAP_IDLE_LOGOUT_SECONDS` | `1500` | idle time after which a user session is signed out automatically |
| `SAP_PHONE_HOME_INTERVAL_SECONDS` | `3600` | interval of the license phone-home check |
| `SAP_DB_SERVER_TYPE` | _auto_ | override the database type (`HANA` / `MSSQL`); normally detected automatically — set only if detection is wrong |
| `SAP_AUTO_DEPLOY_QUERIES` | `true` | deploy the bundled reports on first connect per CompanyDB |

### Bundled reports
The server deploys its ready-made reports (`AI_*` queries in `SQLQueries`) on
the **first connect** per CompanyDB by itself — once per server run.
Prerequisites: `SAP_OPERATION_MODE=READ_WRITE` and a B1 user allowed to create
queries. In read-only mode (`READ_ONLY`) this is skipped; the reports are then
missing, everything else works unchanged. Self-written `AI_*` queries are never
overwritten. If needed, trigger the deployment in the chat
(`sap_deploy_queries`) or disable it with `SAP_AUTO_DEPLOY_QUERIES=false`.

## Authentication
| Variable | Meaning |
|---|---|
| `SAP_AUTH_MODE` | `basic` (user/password straight to the SL) or `bearer` (browser SSO with PKCE, token on every call — from FP 2208 with tokens of the SAP Authentication Server; if your own identity provider issues the tokens itself, see the note in sso-keycloak.md) |
| `SAP_DISABLE_INLINE_LOGIN` | `true` recommended: login only via dialog/web UI, never as a chat argument |
| `SAP_TLS_CERT_FILE` | server certificate (PEM) for inbound HTTPS — together with `SAP_TLS_KEY_FILE`; otherwise HTTP |
| `SAP_TLS_KEY_FILE` | private key (PEM) for inbound HTTPS |
| `SAP_TLS_KEY_PASSWORD` | password for an encrypted TLS key (optional) |
| `SAP_TLS_MIN_VERSION` | minimum TLS version for inbound HTTPS: `1.2` (default) or `1.3` |
| `SAP_TLS_CIPHERS` | pin an OpenSSL cipher string (TLS 1.2 only; empty = safe defaults) |
| `SAP_TLS_CLIENT_CA_FILE` | client CA (PEM) → requires client certificates (mTLS) |
| `SAP_BIND_HOSTS` | bind addresses, comma-separated (default: all interfaces/`0.0.0.0`; `--host` on the command line wins) |
| `SAP_LANG` | language of all server-side texts (exe startup, `doctor`, web login incl. result/error messages, auth errors): `de` (default), `en`, `cs`. The guided installer writes the language chosen in its dialog here. The assistant's chat replies automatically follow the user's language |
| `SAP_PUBLIC_URL` | public HTTPS URL of the instance (web-login fallback; **required with `bearer`** — redirect target `…/callback`) |

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
| `SAP_ENROLLMENT_TOKEN` | phone-home/auto-renewal — **normally not needed** (the token is baked into `versino.key`). Only as an override for test/staging |

Phone-home / automatic subscription renewal (the normal case): see
[lizenz.md](lizenz.md).

## Security notes
- `SAP_OPERATION_MODE=READ_ONLY` as the default; `READ_WRITE` only when write
  access is really wanted.
- `SAP_DISABLE_INLINE_LOGIN=true` makes sure SAP credentials never enter the
  LLM context (sign-in only via dialog/web UI).
- For network operation put TLS in front (native or reverse proxy).

## Logging

Warnings and errors always land in `%APPDATA%\Versino\sapb1-mcp\sapb1-mcp.log`
(JSON lines, 1 MB cap — the oldest entries are pruned automatically).
The path is fixed so support always knows where to look.
