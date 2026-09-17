# Troubleshooting

> 🌐 **English** · [Deutsch](troubleshooting.de.md) · [Česky](troubleshooting.cs.md)

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

## Sign-in page says the link has no ticket
The link carries the ticket after `#` (`…/login#t=…`). Some chat surfaces cut
the fragment when rendering a link: open the link exactly as written, or ask the
assistant for a new one with `connect`. Reloading the page after a sign-in also
loses the fragment (by design) — request a new link.

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
