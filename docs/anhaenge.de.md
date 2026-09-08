<!-- translation-of: anhaenge.md@4ffbe64aff73 -->
# Anhänge: Wie Dateien nach SAP kommen

> 🌐 [English](anhaenge.md) · **Deutsch** · [Česky](anhaenge.cs.md)

Der Assistent kann Dateien (PDF, Bilder, …) als SAP-Anhänge (`Attachments2`)
speichern und mit einem Beleg verknüpfen. **Die Dateibytes laufen dabei nie durch
den Chat**: Ein Sprachmodell kann zehntausende Base64-Zeichen nicht zuverlässig
wiedergeben, deshalb übergibt es nur eine *Referenz*. Drei Wege stehen zur Wahl.

## 1. Upload im Browser (Standard — funktioniert mit Claude Desktop)

1. Bitte den Assistenten, eine Datei anzuhängen. Er ruft `sap_attachment(op="upload_request")`
   auf und gibt dir einen Link wie `https://mcp.example.com/upload?u=…`.
2. Öffne den Link, ziehe die Datei hinein (oder wähle sie aus). Die Seite bestätigt
   und zeigt die Kennung.
3. Zurück im Chat schließt der Assistent mit `sap_attachment(op="upload", upload_id=…)`
   ab und setzt auf Wunsch per `sap_update` das `AttachmentEntry` des Belegs.

Voraussetzungen: `SAP_PUBLIC_URL` ist gesetzt (dieselbe Adresse wie für den
Browser-Login), die Edition des angemeldeten Nutzers erlaubt Anhänge, und die
Instanz läuft mit `SAP_OPERATION_MODE=READ_WRITE`. Links verfallen nach 15 Minuten
und gelten einmal; der Upload ist an die anfragende Chat-Sitzung gebunden.

## 2. Von einer URL (`source_url`)

Der Server lädt die Datei selbst — z. B. von einem Fileserver oder einer
SharePoint-Freigabe. Dieser Weg ist **aus**, bis du Hosts freigibst:

```ini
SAP_ATTACHMENT_URL_ALLOWLIST=files.example.com,*.sharepoint.com
```

Regeln: nur HTTPS, der Host muss auf der Liste stehen (`*.domain` deckt
Subdomains ab), Adressen in privaten Netzen werden abgelehnt, sofern nicht die
IP selbst freigegeben ist, Weiterleitungen werden nicht gefolgt, die Größe ist
begrenzt.

## 3. Von einem Pfad auf dem Server (`file_path`)

Für einen lokal installierten Server oder eine eingebundene Netzfreigabe. **Aus**,
bis du das Basisverzeichnis setzt; Dateien müssen darunter liegen (Symlinks nach
außen werden abgelehnt):

```ini
SAP_ATTACHMENT_DIR=C:\sapb1-mcp\attachments
```

## Grenzen

| Variable | Default | Bedeutung |
|---|---|---|
| `SAP_UPLOAD_MAX_MB` | `25` | maximale Größe pro Datei für alle drei Wege |
| `SAP_ATTACHMENT_URL_ALLOWLIST` | leer (aus) | Hosts, von denen `source_url` laden darf |
| `SAP_ATTACHMENT_DIR` | leer (aus) | Verzeichnis, aus dem `file_path` lesen darf |

Winzige Dateien (bis 32 KB) dürfen weiterhin inline als `content_base64` kommen;
darüber lehnt das Tool ab und verweist auf die drei Wege.

## Voraussetzung in SAP

Der Service Layer legt Anhänge im **Anlagenordner** ab, der in SAP Business One
konfiguriert ist (*Administration → Systeminitialisierung → Allgemeine
Einstellungen → Pfad → Anlagenordner*). Fehlt er oder ist er für den Service
Layer nicht beschreibbar, scheitern Uploads mit einem SAP-Fehler — das ist eine
Einstellung auf SAP-Seite, keine Server-Option.
