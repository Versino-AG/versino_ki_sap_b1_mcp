# Guided installation (Windows, recommended)

> 🌐 **English** · [Deutsch](installer.de.md) · [Česky](installer.cs.md)

The **guided Windows installer** sets up the SAP B1 MCP server completely in a
few steps. It takes the manual steps off your hands that you would otherwise do
by hand:

- download and place the program file
- write the `.env` by hand (right keys, no typos, correct encoding)
- copy `versino.key` to the right place
- check Service Layer reachability
- optionally set up a Windows service
- optionally configure Claude Desktop

> If you want to set the server up **manually** or tune it further (custom
> directories, your own service wrapper, reverse proxy, Linux), use the manual
> guides instead: [installation-windows.md](installation-windows.md) /
> [installation-linux.md](installation-linux.md).

## Prerequisites

- 64-bit Windows 10/11 or Windows Server 2019+
- your **license key** (by e-mail or from the
  [license portal](https://aishop.versino.de)) — as text to paste **or** as a
  `versino.key` file; both work. See [lizenz.md](lizenz.md)
- network access to your SAP B1 **Service Layer** (`https://<host>:50000/b1s/v2/`)
- internet access during the installation (the installer downloads the program
  file from the release)

## Download

Get the setup from the [releases](../../releases/latest):

| File | Purpose |
|-------|-------|
| `sapb1-mcp-setup-<version>.exe` | guided installer |

The setup is **signed by Versino AG** (Authenticode) — Windows shows
"Versino AG" as the publisher. The actual server program file is downloaded by
the installer during installation, matching the version.

## The installation flow

Start the setup and follow the wizard. It asks in order:

1. **Terms** — confirm the [terms and conditions](https://aishop.versino.de/agb).
2. **Target folder** — where to install (default:
   `%LOCALAPPDATA%\Versino\sapb1-mcp`, e.g.
   `C:\Users\<user>\AppData\Local\Versino\sapb1-mcp`). The installation runs
   without administrator rights; only the optional service setup asks once via
   UAC.
3. **License key** — **paste** the key delivered by Versino into the field or
   load it from a file via **"Choose versino.key …"** (both end up in the same
   field). The installer downloads the program file, writes the key as
   `versino.key` into the target folder and validates it **immediately offline**
   (signature + expiry against the embedded key). If it is invalid or expired,
   the flow stops. The enrollment token (for renewal) is inside — you need to
   enter **nothing** else by hand.
4. **Service Layer connection** — enter your SL URL
   (`https://<host>:50000/b1s/v2/`). The installer checks the server's
   **reachability**. If the Service Layer (as usual) demands a sign-in, that
   counts **as reachable** — only true unreachability (wrong address,
   network/firewall, certificate problem) is reported.
5. **CompanyDB(s)** — the selectable databases (comma-separated). Optionally you
   can enter a **test login** (user/password) here; the installer verifies DB
   name and access with it. These test credentials are **not stored**.
6. **Reachability** — port (default `8000`) and optionally the public URL.
   Locally the port suffices; on the network the public HTTPS URL (reverse
   proxy, see [installation-zentral.md](installation-zentral.md)).
7. **Options**:
   - **Allow write access** (`READ_WRITE` instead of read-only)
   - **Accept self-signed SL certificate**
   - **Set up as a Windows service** — the server then runs automatically
     (even without a signed-in user)
   - **Add the Claude Desktop configuration automatically** (on by default)

## After the installation

The target folder then contains:

- `sapb1-mcp.exe` — the server
- `versino.key` — your license
- `.env` — the pre-filled configuration from your inputs

Depending on the chosen options additionally:

- a **Windows service** `SAPB1-MCP` that starts the server automatically
- the **Claude Desktop entry** is set (existing entries are kept) —
  restart Claude Desktop afterwards

If **no** service was chosen, start the server as described in
[installation-windows.md](installation-windows.md), section 3.

## First use

Call the **`connect`** tool in the LLM client → SAP sign-in (input dialog or
browser login). The **credentials never enter the chat/LLM context**. After that
the SAP tools are available; `list_databases` shows the CompanyDBs. See also
[erste-schritte.md](erste-schritte.md).

## Troubleshooting

Common cases (Defender warning, `license.refused`, connection problems) are in
[troubleshooting.md](troubleshooting.md).
