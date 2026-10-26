# Branding the browser pages

> 🌐 **English** · [Deutsch](branding.de.md) · [Česky](branding.cs.md)

The pages your users see in the browser (login, database choice for SSO, the
confirmation after sign-in, file upload) can carry your logo, colours and texts.
You change them with files next to the server — no new version, no code.

## The branding folder

Put a folder named `branding` next to the binary:

```
<installation folder>\
  sapb1-mcp.exe
  .env
  versino.key
  branding\
    branding.json      <- colours, texts, footer (all optional)
    logo.png           <- or logo.jpg / logo.webp / logo.svg
```

A different place can be set in the `.env`:

```
SAP_BRANDING_DIR=D:\sapb1-mcp\branding
```

The folder is read **at start**. Changes take effect after a restart. A folder
named in `SAP_BRANDING_DIR` that does not exist stops the start, so a typo does
not go unnoticed. A ready-made example is the folder `branding-example` in the
download repository: copy it, rename it to `branding` and adjust it.

## branding.json

Every field is optional. A text you write in one language is shown in every
language that has no text of its own (the language from `SAP_LANG` first, then
English, German, Czech).

```json
{
  "colors": { "accent": "#0a6f8c", "accent2": "#0fa3b8", "bg": "#0a1622", "bg2": "#0d2233" },
  "eyebrow": { "en": "Example Ltd", "de": "Muster GmbH" },
  "title": { "en": "SAP assistant sign-in", "de": "Anmeldung SAP-Assistent" },
  "subtitle": { "en": "Sign in with your SAP user.", "de": "Mit Ihrem SAP-Benutzer anmelden." },
  "footer": { "en": "Questions: it@example.com", "de": "Fragen: it@muster.de" },
  "page_title": "Example – SAP assistant",
  "support_url": "https://intranet.example.com/sap-help"
}
```

| Field | Effect | Rules |
|---|---|---|
| `colors.accent`, `colors.accent2` | accent colour and its gradient partner (top bar, button, mark) | `#rgb` or `#rrggbb` |
| `colors.bg`, `colors.bg2` | page background (gradient) | `#rgb` or `#rrggbb` |
| `eyebrow` | small line above the heading on every page (default "SAP Business One") | up to 80 characters per language |
| `title` | heading of the login page | up to 80 characters |
| `subtitle` | line below the heading of the login page | up to 200 characters |
| `footer` | line below the card on every page | up to 200 characters |
| `page_title` | browser tab title of the login and upload pages | up to 80 characters |
| `support_url` | link "Help & contact" in the footer, opens in a new tab | `https://` address, up to 200 characters |

Languages are `en`, `de` and `cs`. The database choice, the confirmation and the
upload page keep their own headings, because they tell the user what happened.
Texts are shown as plain text: HTML does not work and is shown as typed.

Colours are not checked for contrast. Look at the page once after a change.

## Logo

One file with a fixed name: `logo.png`, `logo.jpg`, `logo.webp` or `logo.svg`.
If several are present, the first in this order counts. At most 512 KB. The
content must match the extension. The logo replaces the database mark in the
card and is shown at a height of 42 pixels. A wide logo works best.

## Themes for partners

`SAP_WEB_THEME` switches to a complete design that is part of the binary:

| Value | Look |
|---|---|
| `versino` | default (dark background, card) |
| `cloudiax` | Cloudiax design: white background, logo above the card, own font |

The branding folder works on top of a theme: its colours, texts and logo win.
With a theme, usually only logo and texts make sense; own background colours
replace the theme's background and can clash with its design.

## Page language

The pages follow the language of the browser (`en`, `de` or `cs`). A browser in
another language gets the language from `SAP_LANG`. Messages in the chat and in
the log stay in `SAP_LANG`.

## Check before the start

```
sapb1-mcp doctor
```

The line `branding` shows the theme, the folder and what took effect, or the
error with file and field. The server start runs the same check and stops with
the same message.

| Message | Cause |
|---|---|
| `branding.json: invalid JSON in line 4` | a comma or quote is missing near that line |
| `colors.accent: … hex colour` | a colour is not written as `#rrggbb` |
| `title: text for 'de' is longer than 80 characters` | shorten the text |
| `titel: Extra inputs are not permitted` | the field name is misspelt |
| `logo.png: the content does not match the extension (image/png)` | file renamed, not converted |
| `SAP_BRANDING_DIR=…: the folder does not exist` | path in the `.env` is wrong |

## Behind a reverse proxy

Logo and fonts are served under `/branding/`. If your proxy forwards only
selected paths, add `/branding/` (see installation-zentral.md).
