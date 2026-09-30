# What's new

> 🌐 **English** · [Deutsch](neuigkeiten.de.md) · [Česky](neuigkeiten.cs.md)

The most important changes per version, in plain words. The running version is
shown by `sapb1-mcp --version`, and the assistant knows it too (ask *"Which
version is running?"*).

## 3.8.2 — 2026-09-30

- **Adapt reports in the SAP Query Manager.** Copy a shipped report there under
  the same name, add the field you need (e.g. a UDF), save — the assistant uses
  your version. New reports work the same way → [konfiguration.md](konfiguration.md).
- **Remove document lines safely.** *"Remove line 3 from quotation 4711"* now
  works on open quotations, orders and purchase documents, including sales BOMs
  as a whole → [erste-schritte.md](erste-schritte.md).
- **A second attachment no longer replaces the first**, and one upload link
  takes several files → [anhaenge.md](anhaenge.md).
- **A seat becomes free again** after 30 minutes of inactivity, even if the
  client was simply closed → [lizenz.md](lizenz.md).
- **Serial numbers in stock per warehouse** as a new report; changes to
  documents no longer fail with a false "changed since read" conflict.

## 3.8.1 — 2026-09-21

- **Security hardening** throughout sign-in, uploads and the licence check.
- **The running version is visible**: `--version`, the start message and the
  assistant's help.
- **Browser sign-in sessions end after one hour** (instead of eight) — sign in
  again with `connect`.
- **`SAP_ALLOWED_CLIENTS`** limits which addresses reach the server at all
  → [konfiguration.md](konfiguration.md).
- **Clock protection for the licence is on by default**; the anchor file sits
  next to `versino.key` → [lizenz.md](lizenz.md).
