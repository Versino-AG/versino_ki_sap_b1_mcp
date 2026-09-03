# SSO via Keycloak (OIDC) — setup guide

> 🌐 **English** · [Deutsch](sso-keycloak.de.md) · [Česky](sso-keycloak.cs.md)

Only needed when your SAP B1 uses the **Authentication Server (SLD/IAM,
"Keycloak")** and you want **real SSO**. If you sign in classically straight at
the Service Layer, keep `SAP_AUTH_MODE=basic`
(→ [konfiguration.md](konfiguration.md)) and this document is irrelevant.

In both variants the user signs in **in the browser directly at Keycloak**
(Authorization Code + PKCE) — an existing SSO session, MFA and federated logins
(AD/SAML) apply, and the **password never reaches the MCP**. A
"direct access grant" (ROPC) is **not** required on the client.

## 1. The SSO mode `bearer` — two tiers

SSO always runs via `SAP_AUTH_MODE=bearer`: after the browser login the token is
sent with **every** Service Layer call (`Authorization: Bearer` +
`X-b1-companyid`). This is exactly the path the SAP guide "Identity and
Authentication Management in SAP Business One" describes (ch. 6.5.8 and 6.9).

The difference is **who issues the tokens**:

| Variant | Token issuer | Minimum version | When |
|---|---|---|---|
| **A — SAP Authentication Server** (the normal case) | the `sapb1` realm of your SAP B1; client from the Extension Single Sign-On Manager (`b1-ext-…`) | SAP B1 10.0 **FP 2208** | whenever the sign-in runs via the SAP Authentication Server — **also when an external provider sits behind it** (AD, Entra ID, Okta, SAP IAS): the SLD creates a broker address for it in the `sapb1` realm (`…/auth/realms/sapb1/broker/b1-<alias>/endpoint` — in the guide for AD FS on p. 20, Entra ID p. 31, Okta p. 39); the user signs in at the external provider, but the token is issued by the `sapb1` realm |
| **B — your own identity provider issues the tokens itself** | your own IdP instance, registered as a trusted provider in the Extension SSO Manager; SAP calls this **principal propagation** | see note below | only when your application landscape should issue the tokens itself instead of fetching them from the SAP Authentication Server |

**In practice variant A is almost always the right one.** The version table of
the SAP guide (ch. 6.9) lists — even for the scenario "one or more external
identity providers are active" — the path "register the extension client id and
connect with the access token" **from FP 2208**: if your AD/Entra/Okta is
activated as an identity provider in the SLD, the user signs in there — but the
**token for the Service Layer still comes from the SAP Authentication Server**.
That is variant A.

> **Note on variant B:** the IAM guide describes principal propagation only
> conceptually (ch. 6.10) and refers to SAP's own document
> **"Principal Propagation for SAP Business One" (security guide, version 1.0 –
> 2025-03-25)**. The setup steps are there (see step 4 below). That document
> **states no minimum version either** — the commonly repeated "from FP 2411"
> is thus not backed by any of the three SAP sources. Clarify your feature
> package with us up front.

Both variants need SLD access (port 40000) for resolving the CompanyID.

> **Important — what active IAM changes for `basic`:** bound users no longer
> sign in with their **B1 user code** but with the **Authentication Server's
> credentials**. The guide phrases this with a condition that is easy to miss
> (ch. 6.1, p. 140): *"If you **only** enabled the identity provider SAP
> Business One Authentication Server, the DIAPI and Service Layer login
> interfaces allow you to use the SAP Business One Authentication Server user
> and password to login."*
>
> It follows:
> - **Only** the Authentication Server active as identity provider → `basic`
>   stays usable. The user name is the Authentication Server's user code; by
>   default it equals the IdP user name (ch. 3.2.4).
> - An **additional external** identity provider active (AD FS, Entra ID, Okta,
>   SAP IAS) → for those users `bearer` is the way. The version table in
>   ch. 6.9 lists for that scenario from FP 2305 **only** the access-token
>   path; the user/password option disappears there. Such users always sign in
>   with their **e-mail address** — with an external provider that is the only
>   permitted identifier (ch. 3.2.1), and the e-mail domain controls which
>   sign-in page they are routed to (ch. 3.1.1 and 5.1.5).
> - ⚠️ **Open point with a mixed landscape:** the guide explicitly allows the
>   classic login only when **exclusively** the Authentication Server is
>   active. With both active, it is not documented for **any** user that
>   `basic` keeps working — not even for those bound to the Authentication
>   Server. In that constellation we align on the approach together.
> - Users with **two-factor authentication** cannot use the classic path
>   either. The guide documents that for the DTW command mode (ch. 7) and does
>   not list it explicitly for the Service Layer.

## 2. SAP-side prerequisites — in this order

Executing role: **B1 administrator / landscape administrator**. With hosted
systems (e.g. Cloudiax) possibly together with the provider (SLD access, ports).

> **Two admin UIs — check first which one you have.** SAP documents IAM in two
> guides and the paths differ:
> - **On-premise / hosted:** *SLD Control Center*, typically
>   `https://<sap-host>:40000/ControlCenter`. The steps below are written for
>   this.
> - **SAP Business One Cloud:** *Cloud Control Center*. There the areas are
>   called *System Configuration → Identity Providers* and *Customer Management
>   → Customers → Customer Details → Identity Providers*, and the SLD and
>   authentication-service addresses are configured there — so they are not
>   necessarily `<host>:40000` / `<host>:40020`.
>   ⚠️ Additionally, *System Configuration → Global Settings →
>   **Enable Third Party Identity Provider** = On* must be set there **once**;
>   only then does the *Add* button for identity providers appear. That
>   requires a registered software repository for **FP 2405 or higher**.
>
> Everything else (client registration in the Extension Single Sign-On Manager,
> user binding, token transport) is identical in both variants.

1. **Check IAM** — Control Center (see box), tab *Identity Providers*: at least
   one identity provider must be **Active** (SAP Business One Authentication
   Server, Active Directory Domain Services or an external OIDC provider).
   ⚠️ Bind **all** users before activating (step 2) — afterwards bound users
   sign in with the identity provider's credentials, no longer with the B1 user
   code (see box above).
2. **Bind users** — Control Center, tab *Users*: select every user who should
   use the MCP → **Bind** → assign server, company database(s) and B1 user
   code. Without this binding the sign-in fails.
3. **Create the OAuth client** — **SAP Business One Extension Single Sign-On
   Manager** (installed with the Extension Manager) → *Extensions* →
   **Register**:
   - **Client type: "Web App"** (yields client id *and* client secret).
   - **Redirect URI:** `<SAP_PUBLIC_URL>/callback`
     (e.g. `https://mcp.your-host.example/callback`; for a local test
     additionally `http://127.0.0.1:8000/callback`). The guide also allows
     **wildcards** here (e.g. `https://mcp.your-host.example/*`) — we still
     recommend the full URL so only the intended return path is valid.
   - ⚠️ The **client secret is shown only once** — save it immediately.
   - Optional (for central sign-out, from FP 2508): also register the
     back-channel-logout URL `<SAP_PUBLIC_URL>/backchannel-logout`.
4. **Variant B only (your own identity provider issues the tokens).**
   For variant A this step is skipped entirely. Source: SAP security guide
   "Principal Propagation for SAP Business One" (version 1.0), ch. 1.2/1.3.
   - **Prerequisites:** your identity provider is configured as a third-party
     IdP in the SLD; it issues access tokens in **JWT format** and has its
     **own discovery URL**; user identity is carried via the **e-mail
     address**, which must be **unique landscape-wide** ("one e-mail address
     exclusively represents one user only").
   - **Register the IdP:** *Extension Single Sign-On Manager → Principal
     Propagation → Identity Providers → Register*. Enter: **name**,
     **discovery endpoint** (`…/.well-known/openid-configuration`) and
     **identity claim name** — through the latter SAP reads the e-mail from
     the token. The claim must be present in the token and carry the user's
     correct e-mail.
   - **Bind the company:** *Principal Propagation → Tenants → Bind*. Bind only
     the company databases that should really be accessible — SAP explicitly
     calls both security-critical. **Note the company id shown there**; we need
     it for the configuration.
   - **Changes:** a registered IdP cannot be **edited** — delete and re-create
     (*Delete* in the same path).
5. **Network:** reachable from the MCP server must be port **50000** (Service
   Layer) and — with **variant A** — **40020** (Authentication Server) plus
   **40000** (SLD, for the CompanyID). The **users' browser** must reach the
   sign-in page: with variant A port **40020**, with variant B your own
   provider. **Variant B does not need port 40000** when the CompanyIDs are
   configured (see section 3). With hosted systems request the openings from
   the provider if needed.

## 3. `.env` — two ways

Comments each on their **own line** (not behind the value).

### Recommended: automatic endpoint discovery (SLD discovery)

With `SAP_SLD_URL` the server fetches the Keycloak endpoints (token, authorize,
logout, signing keys) itself at startup — fewer typos:

```ini
SAP_AUTH_MODE=bearer

# Public URL of this MCP instance — after the login the browser is redirected
# to <SAP_PUBLIC_URL>/callback. REQUIRED with bearer.
SAP_PUBLIC_URL=https://mcp.your-host.example

# SLD — all Keycloak endpoints are discovered from it automatically.
SAP_SLD_URL=https://<sap-host>:40000

SAP_IDP_CLIENT_ID=<client id from the Extension Single Sign-On Manager>
SAP_IDP_CLIENT_SECRET=<client secret>

# if Authentication Server and/or Service Layer are self-signed
SAP_ALLOW_SELF_SIGNED_CERT=true
```

### Alternative: specify the endpoints yourself

When port 40000 is unreachable or you want the values pinned:

```ini
SAP_AUTH_MODE=bearer
SAP_PUBLIC_URL=https://mcp.your-host.example
SAP_IDP_TOKEN_URL=https://<sap-host>:40020/auth/realms/sapb1/protocol/openid-connect/token
SAP_IDP_CLIENT_ID=<client id>
SAP_IDP_CLIENT_SECRET=<client secret>

# optional — otherwise derived from the token URL
# (…/openid-connect/token → …/openid-connect/auth). REQUIRED when your identity
# provider is not a Keycloak (e.g. Microsoft Entra ID, Okta).
# SAP_IDP_AUTHORIZE_URL=https://<sap-host>:40020/auth/realms/sapb1/protocol/openid-connect/auth

# optional — default 'openid' (like the official SAP samples). Set extended
# scopes only if they are assigned to the client, otherwise the login fails.
# SAP_IDP_SCOPE=openid

# SLD for the CompanyID resolution (X-b1-companyid) — required with bearer
SAP_SLD_URL=https://<sap-host>:40000
```

Notes:
- The variables were formerly named `SAP_KEYCLOAK_*`; that spelling keeps
  working. New and preferred is `SAP_IDP_*` (identical meaning).
- `SAP_IDP_TOKEN_URL` must be **`https://`**; adjust host/port (typically
  `40020`) and realm (`sapb1`) according to your SLD configuration.
- `SAP_PUBLIC_URL` is **required** with `bearer`. Locally
  `http://127.0.0.1:8000` is also allowed (loopback) — then register
  `…/callback` with exactly that address on the client.
- `SAP_ALLOW_SELF_SIGNED_CERT` applies **jointly** to Service Layer and Keycloak
  and is meant as a **transitional solution** — set up a valid certificate in
  the medium term and set the value back to `false`.
- For **variant B** a dedicated block applies — see below.

### Variant B: your own identity provider (principal propagation)

Here **your** identity provider issues the tokens. Two things differ from
variant A:

1. `SAP_IDP_TOKEN_URL` points at **your** provider, not at the SAP
   Authentication Server.
2. The **CompanyID is configured statically** instead of being queried from the
   SLD at runtime. You noted it when binding the company (step 4 above,
   *Principal Propagation → Tenants*). This mode therefore does not need **port
   40000 at all** — the SLD is no longer contacted.

```ini
SAP_AUTH_MODE=bearer
SAP_PUBLIC_URL=https://mcp.your-host.example

# your own identity provider
SAP_IDP_TOKEN_URL=https://idp.your-host.example/realms/<realm>/protocol/openid-connect/token
SAP_IDP_CLIENT_ID=<client id in YOUR provider>
SAP_IDP_CLIENT_SECRET=<client secret>

# REQUIRED with variant B: the CompanyID noted during tenant binding, per
# company database. Format: DB:ID, several comma-separated.
SAP_COMPANY_IDS=SBO_PROD:1,SBO_TEST:2

# No SAP_SLD_URL needed — the CompanyID is set above.
```

On `SAP_COMPANY_IDS`:
- **All** databases from `SAP_DATABASES` must be listed. If one is missing, the
  server refuses to start and names it — so nobody silently falls back to the
  SLD lookup.
- The value is sent unchanged as `X-b1-companyid`. Take it exactly as the
  Extension Single Sign-On Manager shows it.
- If your provider is **not** a Keycloak (e.g. Entra ID, Okta), additionally set
  `SAP_IDP_AUTHORIZE_URL` — the derivation only works for Keycloak paths.
- From **FP 2602** the Service Layer validates the `audience` claim. With
  variant B **your** provider must write the Service Layer client id into the
  audience (see note in section 7).

## 4. Reverse proxy

If the MCP runs behind a reverse proxy (HTTPS termination), **these paths** must
be forwarded to the MCP port — with **anonymous** access (no Windows
authentication, otherwise the connection fails with a 401):

```
/mcp   /login   /api/login   /callback   /backchannel-logout
```

Additionally: disable response buffering for `/mcp` and set generous timeouts
(the connection stays open). Details:
[installation-zentral.md](installation-zentral.md).

## 5. Flow for end users

**Important up front:** in the SSO modes the MCP **never accepts credentials** —
the sign-in always happens at the identity provider. The page at `/login`
therefore shows **only a database chooser** (with several company databases),
no user or password fields.

The flow, step by step:

1. **Call `connect`** — the user simply says in the chat *"Connect me to SAP"*.
   They get a **sign-in link** back.
   - *Several databases:* the link first opens the **database chooser**
     ("Choose database" with a dropdown and "Continue to sign-in") and after
     the choice redirects automatically to the identity provider's sign-in
     page.
   - *One database*, or the database already named in the chat (*"… database
     BRAGI_TEST"*): the link leads **directly** to the identity provider's
     sign-in page — the chooser is skipped.
2. **Sign in at the identity provider.** The chooser page redirects via
   JavaScript (no additional path needed on the reverse proxy — it stays
   `/login`). There the user signs in with their **e-mail address** and IdP
   password (an existing SSO session applies; MFA and federated logins work).
   Credentials **never** go to the MCP or into the chat.
3. **Back in the chat** call `connect(ticket="…")` once (or write "done", the
   client handles it) — connected.

**Several databases:** a session is always connected to **one** database.
Switching: *"Disconnect"* (`disconnect`), then reconnect — pick the other
database on the chooser page (or name it right in the chat); the SSO session in
the browser usually still exists, the second sign-in is then just one click.
Whether a user may use a database at all is decided by SAP: with variant A
their **user binding** must include the chosen database (otherwise: "no SLD
company binding found for …" — remedy: extend the binding in the SLD by that
company); with variant B the Service Layer refuses access when the permission
is missing. Within the connection the user's **own SAP permissions** always
apply.

**Sign-out:** `disconnect` ends the session and revokes the token at the
identity provider. If the back-channel-logout URL is registered (step 3), a
central sign-out at the identity provider also ends the MCP sessions
automatically.

## 6. Testing

Restart the server, call `connect`, follow the URL, sign in,
`connect(ticket=…)`, then a sample query. If that works, the chain is correct.
For a first test **without** a reverse proxy: set
`SAP_PUBLIC_URL=http://127.0.0.1:8000`, register that callback address on the
client and run the browser test directly on the server (remote desktop).

## 7. Troubleshooting

| Error / symptom | Cause / fix |
|---|---|
| `invalid_client` | wrong `SAP_IDP_CLIENT_ID` / `SAP_IDP_CLIENT_SECRET`, or the client is not (or no longer) registered |
| Redirect refused (`invalid_redirect_uri`) | the called callback URL is not registered on the client — compare protocol, host, port and path (Extension Single Sign-On Manager) |
| `invalid_scope` | the requested scope is not assigned to the client → reset `SAP_IDP_SCOPE` to `openid` or assign the scope on the client |
| Realm/404 error | wrong `SAP_IDP_TOKEN_URL` (check host/port/realm) — or simply use `SAP_SLD_URL` |
| "authorize_url not derivable" at startup | the token URL is not a Keycloak path (e.g. Entra ID/Okta) → set `SAP_IDP_AUTHORIZE_URL` explicitly |
| Certificate error | self-signed → temporarily `SAP_ALLOW_SELF_SIGNED_CERT=true` |
| Login works but access gets 401 | the Service Layer rejects the token: check the feature package (variant A from FP 2208) and the client registration in the Extension Single Sign-On Manager. **From FP 2602** the Service Layer additionally validates the `audience` claim (see note below the table) |
| Login works but access is refused (401/403) | the user is not bound (or bound to a different company DB) → check SLD *Users* (the guide names 401 for the unauthenticated case, ch. 6.8.2) |
| "No SLD company binding found" | the user binding is missing, or port 40000 is unreachable from the MCP server |
| `/login` shows only a database chooser, no login fields | correct — in the SSO modes you sign in at the identity provider, never at the MCP; the page only picks the company |
| Clicking "Continue to sign-in" does nothing | check the browser console. If there is a `Content Security Policy` message about `form-action`, a version before **3.2.1** is running — please update. Otherwise: is JavaScript enabled in the browser? The page needs it for the redirect (without JS a fallback applies that fails at the CSP with some identity providers) |
| Startup aborts: "SAP_COMPANY_IDS must cover every CompanyDB" | variant B: a database from `SAP_DATABASES` has no CompanyID. Add it — or remove `SAP_COMPANY_IDS` entirely if the SLD should resolve it |
| Startup aborts: "SAP_COMPANY_IDS names unknown CompanyDB" | typo in the database name — it must exactly match an entry from `SAP_DATABASES` |
| Access refused although the CompanyID is configured | check the value against the tenant binding in the Extension Single Sign-On Manager; it is sent unchanged as `X-b1-companyid` |

### Note on audience validation (from SAP B1 10.0 FP 2602)

From FP 2602 the Service Layer validates the `audience` claim on token access
(per the SAP guide initially introduced for single-page apps). If access yields
**401** despite a successful sign-in, the **Service Layer client id** must be
added as an audience value in the Keycloak client configuration (audience
mapper). The guide explicitly calls this mapper a **workaround for versions
before FP 2608** — from FP 2608 the mapping should apply without manual
intervention. For test/development purposes the validation can be disabled in
the Service Layer configuration `b1s.conf` with
`EnableAudienceValidation=false` (then restart the Service Layer service) — for
production the audience mapper is the right way.
Details in the SAP guide "Identity and Authentication Management in SAP
Business One", section "Configuring Audience in Keycloak".

### Go-live

The SAP-side steps (section 2) require rights in the SLD and the Extension
Single Sign-On Manager. We accompany the initial go-live together — please plan
a short time slot with your SAP administrators.

Further help: **support@versino.de**.
