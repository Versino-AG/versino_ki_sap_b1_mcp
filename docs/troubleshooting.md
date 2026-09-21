# Troubleshooting

> 🌐 **English** · [Deutsch](troubleshooting.de.md) · [Česky](troubleshooting.cs.md)

## Which version is running?
Four ways, all reporting the same number:

```
sapb1-mcp.exe --version                 # on the command line
curl http://<host>:8000/version         # from a monitor or a script
```

The start-up log carries it as the field `version` in the line
`server.per_user_start`, and `sap_help` reports it in its `instance` block —
useful when you have the assistant in front of you but not the machine.

`/version` answers without a sign-in and says nothing beyond the name and the
number. If `SAP_ALLOWED_CLIENTS` is set, it answers only the addresses listed
there. It is **not** a health check: it says which version is installed, not
whether the server is healthy.

## Where the log files are
Two size-capped files under `%APPDATA%\Versino\sapb1-mcp\` (on Linux
`~/.config/Versino/sapb1-mcp/`):
- `sapb1-mcp.log` — warnings and errors of the running server.
- `sapb1-mcp-audit.log` — the sign-in trail (who, from where, when, success or
  failure). Kept separately and with a larger budget so a flood of ordinary
  warnings cannot prune away the evidence a support case or an incident review
  needs — and the other way round.

**Running as a Windows service** the files live under the installation instead:
`<installation folder>\logs\Versino\sapb1-mcp\`. The service runs as
LocalSystem, whose `%APPDATA%` is a folder inside `C:\Windows` that nobody would
think to open, so the installer points it next to the installation.

## `license.refused` at startup
No valid license found. Check:
- is `versino.key` **next to** the binary? (or does `SAP_LICENSE_FILE` point to it?)
- is the key not expired? (get a new one via the [license portal](https://aishop.versino.de))

Details: [lizenz.md](lizenz.md).

## Attachment upload fails with SAP error `-43`
`-43` is SAP's internal *path / folder* error. On an attachment upload it means the
**Service Layer** could not write into the attachment folder — it is not a problem
with the file itself (the same file fails again and again). Check in SAP Business
One: *Administration → System Initialization → General Settings → Path →
Attachments Folder*:
- the path must exist **as seen from the Service Layer host** (a UNC path such as
  `\\fileserver\B1_Attachments`, not a drive letter of a user's PC),
- the Service Layer service account needs **write** permission there,
- a file with the **same file name** must not already exist in the folder — retry
  with a different `file_name` if it does.
After the SAP-side fix, simply upload again. Background: [anhaenge.md](anhaenge.md).

## Windows Defender / SmartScreen warning
If the binary is not signed yet, Windows may show a false positive.
- "More info" → "Run anyway", or allow the file in Defender.
- Code signing is planned for the production delivery — the warning then disappears.

## Client cannot reach the server
- Check host/port and the **`/mcp`** path in the URL.
- Access from other machines: the default bind is already `0.0.0.0` (check that no
  restricting `SAP_BIND_HOSTS`/`--host` is set), open the **firewall**
  for the port and use the server's **internal IP/DNS** in the client URL.
- Claude Desktop prefers **HTTPS**; for plain `http://` use the `mcp-remote` bridge
  (see [installation-windows.md](installation-windows.md)).

## Connection to SAP fails
- `SAP_BASE_URL` correct? (`https://<host>:50000/b1s/v2/`)
- self-signed SL certificate → `SAP_ALLOW_SELF_SIGNED_CERT=true`.
- right `SAP_AUTH_MODE` (`basic` vs. `bearer`)?
- is the chosen CompanyDB listed in `SAP_DATABASES`?

## Write tools missing / refused
- Set `SAP_OPERATION_MODE=READ_WRITE` (default is `READ_ONLY`). In read-only
  mode the write tools are not registered at all, so the assistant does not
  see them. After changing the mode **restart the server** and reconnect the
  LLM client — it caches the tool list.
- Attachment upload refused with "READ_ONLY: uploading attachments writes to
  SAP" → same switch; `info`/`download` keep working in read-only mode.
- Write/delete tools are additionally bound to the **edition** (PRO/ENTERPRISE),
  see [lizenz.md](lizenz.md).
- **"… is part of this instance's own configuration"** — the write tools
  deliberately cannot touch the server's own control surfaces: `SQLQueries`,
  `SQLViews`, `Users`, `UserPermissionTree`, `UserObjectsMD`, `UserTablesMD`,
  `UserFieldsMD` and `B1Sessions`. That is not a permission problem — your SAP
  user may well be allowed to. It is a boundary of the chat interface: an
  assistant that can rewrite a curated report could present manipulated figures
  as a vetted one. Use the SAP client for those, and `sap_deploy_queries` to roll
  out curated reports. Everything else stays writable.

## A tool is missing entirely (not just refused)
The server only offers tools this installation can actually run, so the
assistant never proposes something your license or settings do not cover. Three
gates decide it, and `sap_help` names the one that applies per tool under
`instance.tools_hidden`:

- `edition` — the license edition does not include it → [lizenz.md](lizenz.md).
- `read_scope` — the edition reads master data only, while the tool reads
  arbitrary tables (`sap_curated_query`, `sap_semantic_query`, `sap_attachment`,
  and therefore `sap_deploy_queries`).
- `operation_mode` — `SAP_OPERATION_MODE=READ_ONLY` (see above).

After a license or mode change, restart the server and reconnect the client — it
caches the tool list.

## Windows: window closes again immediately
Usually the **port is already taken** (another service listens on `8000`; the log
shows `WinError 10048` / "… only be used once"). Fix:
- Start the server on a **free port**: `sapb1-mcp.exe --port 8765` — and use the
  same port number in the client (`…:8765/mcp`).
- Or stop the occupying service.
- To actually **see** the error message instead of an instantly closing window:
  start the `.exe` from an open terminal (PowerShell), not by double-click.

## `connect` returns no sign-in link (web login)
The browser-login fallback needs the instance's **public address**. Set
`SAP_PUBLIC_URL` in the `.env` (on the network `https://…`, locally
`http://127.0.0.1:8000`) and restart the server. See
[konfiguration.md](konfiguration.md) and
[installation-zentral.md](installation-zentral.md).

## Sign-in page does not react on click / no confirmation
Nothing happens on "Sign in" and no success page appears — almost always the
**SAP Service Layer is unreachable** (the sign-in runs into a void/timeout):
- Is the SAP host reachable from the **server**? (often **VPN** needed, firewall,
  port `50000`). Test: `Test-NetConnection <sap-host> -Port 50000` (Windows) or
  `nc -vz <sap-host> 50000`.
- `SAP_BASE_URL` correct (`https://<host>:50000/b1s/v2/`)?
- Tickets are short-lived: do not wait too long between opening the link and
  signing in.

## "Too many failed sign-in attempts" / HTTP 429
The server pauses an address after `SAP_LOGIN_MAX_FAILURES` failed sign-ins
(default 10 in 15 min): first 30 s, doubling up to 15 min; a successful sign-in
clears it. The message names the wait.
- Behind a reverse proxy without `SAP_TRUSTED_PROXIES` the **proxy** is the
  address — one colleague's typos pause the whole office. Set
  `SAP_TRUSTED_PROXIES=<proxy address>` (see
  [installation-zentral.md](installation-zentral.md)).
- "Unusually many sign-in requests … paused new browser logins": the global cap
  `SAP_TICKET_ISSUE_PER_MINUTE` (default 60) was hit — wait a minute; raise it
  only for very large installations.
- Every attempt is in the log file (`audit.auth.login` with user, CompanyDB,
  source address, outcome — never the password), so support can match a report
  to a line.

## "Your SAP session reached its maximum lifetime"
Sessions end after `SAP_SESSION_MAX_SECONDS` (default 8 h) regardless of
activity; the assistant is told to run `connect` again — that is all it takes.
`0` disables the limit.

A session addressed by a **browser-login key** is capped more tightly:
`SAP_TICKET_SESSION_MAX_SECONDS` (default 1 h). That key travelled through the chat
transcript and is sent with every call, so it expires sooner than a transport-bound
session — the shorter of the two limits wins. If users have to sign in again after
about an hour, this is the variable to look at, not `SAP_SESSION_MAX_SECONDS`.

## Sign-in page says the link has no ticket
The link carries the ticket after `#` (`…/login#t=…`). Some chat surfaces cut
the fragment when rendering a link: open the link exactly as written, or ask the
assistant for a new one with `connect`. Reloading the page after a sign-in also
loses the fragment (by design) — request a new link.

## HTTP 403 "This address is not in SAP_ALLOWED_CLIENTS"
`SAP_ALLOWED_CLIENTS` limits which addresses may reach the server at all (IP/CIDR,
comma-separated; empty = off, anyone who reaches the port). The check runs before
every route, so it covers the MCP endpoint too, not just the web pages. Add the
caller's address or network to the variable and restart. Behind a reverse proxy the
address the server sees is the **proxy**, not the end user — list the proxy there,
and use `SAP_TRUSTED_PROXIES` for the per-user login throttle. A typo aborts the
start and names the entry.

## HTTP 403 "Cross-site request refused"
The sign-in routes are reached by an MCP client or by the sign-in page this server
serves itself — never by another website. A browser that arrives from a foreign page
(`Sec-Fetch-Site: cross-site`/`same-site`, or an `Origin` that is neither the
request's own host nor `SAP_PUBLIC_URL`) is refused. MCP clients send neither header
and are unaffected. If this hits a legitimate setup, the usual cause is a reverse
proxy that rewrites `Host`: it must pass the public name through, or `SAP_PUBLIC_URL`
must match what the browser actually calls.

## "This sign-in ticket was already redeemed"
A browser-login ticket hands over the session value **once**. One repeat is still
answered (so a lost reply does not cost the sign-in); every further repeat gets this
message and no value. Use the `ticket` value the successful `connect` returned for
all further calls — it is a different value than the one in the link. If it is lost,
start a new browser login with `connect`.

## `npx` / Node.js not found (bridge)
The `mcp-remote` bridge requires **Node.js**. Install Node LTS from
[nodejs.org](https://nodejs.org), restart the client. Tip: in the client config
replace `npx` with `cmd /c npx` (Windows) if the command is not found.

## Linux: binary does not start
- made executable? `chmod +x sapb1-mcp`
- 64-bit Linux with glibc (x86-64) required.

Still stuck? **support@versino.de** (please include a log excerpt and the
version number).

## License expired / subscription renewal
- The exe fetches a due **subscription renewal automatically at startup** (given
  internet + an active subscription). In addition there is a **3-day grace
  period** after expiry during which the server still starts even without a
  successful phone-home.
- If the exe permanently refuses to start after expiry, check: internet access
  to the license server available? Subscription active/paid in the license
  portal (aishop.versino.de)? When in doubt request a fresh `versino.key` from
  **support@versino.de**.
