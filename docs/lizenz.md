# License (`versino.key`)

> 🌐 **English** · [Deutsch](lizenz.de.md) · [Česky](lizenz.cs.md)

The SAP B1 MCP server requires a license. The license is a **signed offline
file** (`versino.key`) — without a valid license the server does not start
(fail-closed).

## Obtaining a license
After purchase, or via the **[license portal](https://aishop.versino.de)**, you
receive the `versino.key` (also by e-mail). It contains:
- **Edition**: BASIC / PRO / ENTERPRISE (determines the available tools),
- **max. seats**: number of distinct SAP users,
- **expiry date**.

## Applying the license
Place the `versino.key` **next to the binary** (same folder as `sapb1-mcp.exe`
or `sapb1-mcp`). It is found automatically — `SAP_LICENSE_FILE` does **not**
need to be set. Alternatively, point `SAP_LICENSE_FILE` at an explicit path.

## Editions (short overview)
| Edition | Reads | Tools offered on top of sign-in and `sap_help` |
|---|---|---|
| BASIC | master data only (business partners, items, …), limited query rate | `sap_query_odata`, `sap_fuzzy_search` |
| PRO | all data incl. documents, higher rate | plus `sap_curated_query` (the bundled `AI_*` reports), `sap_semantic_query`, `sap_attachment`, `sap_deploy_queries`, `sap_create`, `sap_update` |
| ENTERPRISE | all data, no rate limit | plus `sap_delete`, `sap_action` |

A tool an edition could never execute is **not offered at all** — the assistant
never sees it, so it cannot propose something your license does not cover. Tools
that read arbitrary tables (the curated `AI_*` reports, semantic-layer views,
attachments) therefore need an edition with full read access; with BASIC the
bundled reports are not deployed either, because they could not be run.
`sap_help` lists the reason per missing tool (`edition`, `read_scope`,
`operation_mode`), so support can answer "why can't it do X" from one place.
Independently of the edition, `SAP_OPERATION_MODE=READ_ONLY` removes the write
tools → [konfiguration.md](konfiguration.md).

## Phone-home (tamper protection, validity check & subscription renewal)
**Active by default:** the installation contacts the license server at a fixed
interval over a signed connection. That serves two purposes:

- **Validity/revocation check (tamper protection):** the server confirms the
  license is still valid. A license **revoked** server-side (e.g. abuse or
  payment failure) is detected this way — new connections are then refused.
- **Automatic subscription renewal:** with monthly/yearly billing the server
  re-signs the token with a new expiry date and delivers it over the **same**
  channel; the installation adopts it by itself — **no manual key swap** per
  billing period.

Only metadata is transmitted (`customer_id`, `edition`, `version`, seat count) —
no business data.

Activation: **nothing to configure** — the enrollment token is **baked into**
your `versino.key` (signed), and the validation URL is built into the product.
As soon as the `versino.key` sits next to the binary, the installation enrolls
itself on first start and adopts renewals automatically. Optionally set a
writable renewal-cache path (default: `versino.renewed` next to the binary):
```ini
# optional: writable path → adopts renewed tokens
SAP_LICENSE_CACHE_FILE=versino.renewed
```
(Test/staging only: `SAP_ENROLLMENT_TOKEN` overrides the built-in token,
`SAP_LICENSE_VALIDATION_URL` the built-in endpoint.)

> **The endpoint must be `https`.** The phone-home request carries the customer id
> and the seat count, so plain `http` to a remote host is refused — only
> `127.0.0.1`, `::1` and `localhost` may be plain, for development. A self-hosted
> licence server therefore needs TLS; pointing `SAP_LICENSE_VALIDATION_URL` at an
> `http://` address on another machine fails instead of silently sending customer
> data in the clear. The enrolment address is derived from the directory of this
> URL, so keep the path (`…/validate`) intact.

> **Minimum version 2.3.5.** The baked-in token is an additional field in the
> signed license. Older program versions do not know that field and reject such a
> key — they will not start with it. If you receive a new `versino.key` while
> still running a version before 2.3.5, **update the software FIRST and only then
> swap the key**. Your old `versino.key` stays valid until its own expiry date, so
> you can always put it back.

**Container operation:** the installation identity is written to
`install_identity.json` in the working directory by default. If the container runs
with `read_only: true` or on an ephemeral directory, point
`SAP_INSTALL_IDENTITY_PATH` at a **persistent** path (a volume) — otherwise a new
keypair is created on every start, and with it a new installation:
```ini
SAP_INSTALL_IDENTITY_PATH=/data/install_identity.json
SAP_LICENSE_CACHE_FILE=/data/versino.renewed
```

Whether enrollment succeeded is reported by `sapb1-mcp doctor` (check
`enrollment`).

**Air-gapped / without phone-home:** in an environment without internet there is
neither an online validity check nor automatic renewal — the delivered token
stays valid unchanged until its expiry date, and you receive a new
`versino.key` in good time before that. If you do not want phone-home at all,
Versino can switch it off for you; you then receive keys without the enrollment
claim (and without automatic renewal).

## Seats
`max_seats` limits the number of **distinct** SAP users. If you need more,
upgrade via the license portal — the new `versino.key` replaces the old one.

License questions: **support@versino.de**
