# Attachments: how files get into SAP

> 🌐 **English** · [Deutsch](anhaenge.de.md) · [Česky](anhaenge.cs.md)

The assistant can store files (PDF, images, …) as SAP attachments (`Attachments2`)
and link them to a document. **The file bytes never travel through the chat**: a
language model cannot reliably reproduce tens of thousands of base64 characters,
so it only passes a *reference*. Three ways are available.

## 1. Upload in the browser (default — works with Claude Desktop)

1. Ask the assistant to attach a file. It calls `sap_attachment(op="upload_request")`
   and gives you a link like `https://mcp.example.com/upload?u=…`.
2. Open the link, drop the file (or choose it). The page confirms and shows the id.
3. Back in the chat the assistant finishes with `sap_attachment(op="upload", upload_id=…)`
   and, if requested, sets the document's `AttachmentEntry` via `sap_update`.

Prerequisites: `SAP_PUBLIC_URL` is set (the same address used for the browser
login), the connected user's edition allows attachments, and the instance runs
`SAP_OPERATION_MODE=READ_WRITE`. Links expire after 15 minutes and can be used
once; the upload is bound to the chat session that requested it.

## 2. From a URL (`source_url`)

The server downloads the file itself — e.g. from a file server or SharePoint
share. This is **off** until you allowlist hosts:

```ini
SAP_ATTACHMENT_URL_ALLOWLIST=files.example.com,*.sharepoint.com
```

Rules: HTTPS only, host must be on the list (`*.domain` covers subdomains),
addresses inside private networks are refused unless you allowlist the literal
IP, redirects are not followed, size is capped.

## 3. From a path on the server (`file_path`)

For a locally installed server or a mounted network share. **Off** until you set
the base directory; files must lie below it (symlinks out of it are refused):

```ini
SAP_ATTACHMENT_DIR=C:\sapb1-mcp\attachments
```

## Limits

| Variable | Default | Meaning |
|---|---|---|
| `SAP_UPLOAD_MAX_MB` | `25` | maximum size per file for all three ways |
| `SAP_ATTACHMENT_URL_ALLOWLIST` | empty (off) | hosts `source_url` may fetch from |
| `SAP_ATTACHMENT_DIR` | empty (off) | directory `file_path` may read from |

Tiny files (up to 32 KB) may still be passed inline as `content_base64`; above
that the tool refuses and points to the three ways.

## SAP prerequisite

The Service Layer stores attachments in the **attachment folder** configured in
SAP Business One (*Administration → System Initialization → General Settings →
Path → Attachments Folder*). If it is missing or not writable for the Service
Layer, uploads fail with an SAP error — that is an SAP-side setting, not a
server option.
