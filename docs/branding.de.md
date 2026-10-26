<!-- translation-of: branding.md@962430aacd95 -->
# Browserseiten im eigenen Design

> 🌐 [English](branding.md) · **Deutsch** · [Česky](branding.cs.md)

Die Seiten, die Ihre Benutzer im Browser sehen (Anmeldung, Datenbankwahl bei SSO,
die Bestätigung nach der Anmeldung, Datei-Upload), können Ihr Logo, Ihre Farben
und Ihre Texte tragen. Sie ändern sie mit Dateien neben dem Server — ohne neue
Version, ohne Code.

## Der Branding-Ordner

Legen Sie einen Ordner `branding` neben das Binary:

```
<Installationsordner>\
  sapb1-mcp.exe
  .env
  versino.key
  branding\
    branding.json      <- Farben, Texte, Fußzeile (alles optional)
    logo.png           <- oder logo.jpg / logo.webp / logo.svg
```

Ein anderer Ort lässt sich in der `.env` setzen:

```
SAP_BRANDING_DIR=D:\sapb1-mcp\branding
```

Der Ordner wird **beim Start** gelesen. Änderungen wirken nach einem Neustart. Ein
in `SAP_BRANDING_DIR` genannter Ordner, den es nicht gibt, verhindert den Start,
damit ein Tippfehler nicht unbemerkt bleibt. Ein fertiges Beispiel ist der Ordner
`branding-example` im Download-Repository: kopieren, in `branding` umbenennen und
anpassen.

## branding.json

Alle Felder sind optional. Ein Text, den Sie in einer Sprache schreiben, erscheint
in jeder Sprache ohne eigenen Text (zuerst die Sprache aus `SAP_LANG`, dann
Englisch, Deutsch, Tschechisch).

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

| Feld | Wirkung | Regeln |
|---|---|---|
| `colors.accent`, `colors.accent2` | Akzentfarbe und ihr Verlaufspartner (Leiste oben, Schaltfläche, Zeichen) | `#rgb` oder `#rrggbb` |
| `colors.bg`, `colors.bg2` | Seitenhintergrund (Verlauf) | `#rgb` oder `#rrggbb` |
| `eyebrow` | kleine Zeile über der Überschrift auf jeder Seite (Standard „SAP Business One“) | bis 80 Zeichen je Sprache |
| `title` | Überschrift der Anmeldeseite | bis 80 Zeichen |
| `subtitle` | Zeile unter der Überschrift der Anmeldeseite | bis 200 Zeichen |
| `footer` | Zeile unter der Karte auf jeder Seite | bis 200 Zeichen |
| `page_title` | Titel des Browser-Tabs von Anmelde- und Upload-Seite | bis 80 Zeichen |
| `support_url` | Link „Hilfe & Kontakt“ in der Fußzeile, öffnet einen neuen Tab | `https://`-Adresse, bis 200 Zeichen |

Sprachen sind `en`, `de` und `cs`. Datenbankwahl, Bestätigung und Upload-Seite
behalten ihre eigenen Überschriften, weil sie dem Benutzer sagen, was passiert ist.
Texte erscheinen als reiner Text: HTML wirkt nicht und wird so angezeigt, wie es
geschrieben ist.

Farben werden nicht auf Kontrast geprüft. Sehen Sie sich die Seite nach einer
Änderung einmal an.

## Logo

Eine Datei mit festem Namen: `logo.png`, `logo.jpg`, `logo.webp` oder `logo.svg`.
Liegen mehrere da, zählt die erste in dieser Reihenfolge. Höchstens 512 KB. Der
Inhalt muss zur Endung passen. Das Logo ersetzt das Datenbank-Zeichen in der Karte
und wird 42 Pixel hoch angezeigt. Ein breites Logo wirkt am besten.

## Themes für Partner

`SAP_WEB_THEME` schaltet auf ein vollständiges Design um, das Teil des Binarys ist:

| Wert | Aussehen |
|---|---|
| `versino` | Standard (dunkler Hintergrund, Karte) |
| `cloudiax` | Cloudiax-Design: weißer Hintergrund, Logo über der Karte, eigene Schrift |

Der Branding-Ordner wirkt über einem Theme: seine Farben, Texte und sein Logo gewinnen.
Mit einem Theme sind meist nur Logo und Texte sinnvoll; eigene Hintergrundfarben
ersetzen den Hintergrund des Themes und können sich mit dessen Design beißen.

## Seitensprache

Die Seiten folgen der Sprache des Browsers (`en`, `de` oder `cs`). Ein Browser in
einer anderen Sprache bekommt die Sprache aus `SAP_LANG`. Meldungen im Chat und im
Log bleiben bei `SAP_LANG`.

## Vor dem Start prüfen

```
sapb1-mcp doctor
```

Die Zeile `branding` zeigt Theme, Ordner und was davon wirkt, oder den Fehler mit
Datei und Feld. Der Serverstart macht dieselbe Prüfung und bricht mit derselben
Meldung ab.

| Meldung | Ursache |
|---|---|
| `branding.json: invalid JSON in line 4` | in der Nähe dieser Zeile fehlt ein Komma oder Anführungszeichen |
| `colors.accent: … hex colour` | eine Farbe ist nicht als `#rrggbb` geschrieben |
| `title: text for 'de' is longer than 80 characters` | Text kürzen |
| `titel: Extra inputs are not permitted` | Feldname falsch geschrieben |
| `logo.png: the content does not match the extension (image/png)` | Datei nur umbenannt, nicht umgewandelt |
| `SAP_BRANDING_DIR=…: the folder does not exist` | Pfad in der `.env` stimmt nicht |

## Hinter einem Reverse-Proxy

Logo und Schriften kommen unter `/branding/`. Leitet Ihr Proxy nur ausgewählte
Pfade weiter, ergänzen Sie `/branding/` (siehe installation-zentral.de.md).
