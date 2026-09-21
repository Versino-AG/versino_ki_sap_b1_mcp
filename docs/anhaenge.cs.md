<!-- translation-of: anhaenge.md@e0898831ba2e -->
# Přílohy: jak se soubory dostanou do SAP

> 🌐 [English](anhaenge.md) · [Deutsch](anhaenge.de.md) · **Česky**

Asistent umí ukládat soubory (PDF, obrázky, …) jako přílohy SAP (`Attachments2`)
a propojit je s dokladem. **Bajty souboru nikdy neprocházejí chatem**: jazykový
model nedokáže spolehlivě reprodukovat desítky tisíc znaků base64, proto předává
jen *odkaz*. K dispozici jsou tři cesty.

## 1. Nahrání v prohlížeči (výchozí — funguje s Claude Desktop)

1. Požádejte asistenta o připojení souboru. Zavolá `sap_attachment(op="upload_request")`
   a dá vám odkaz jako `https://mcp.example.com/upload#u=…`.
2. Otevřete odkaz a přetáhněte soubor (nebo ho vyberte). Stránka potvrdí přijetí a
   zobrazí identifikátor.
3. Zpět v chatu asistent dokončí `sap_attachment(op="upload", upload_id=…)` a na
   přání nastaví `AttachmentEntry` dokladu pomocí `sap_update`.

Předpoklady: je nastaveno `SAP_PUBLIC_URL` (stejná adresa jako pro přihlášení v
prohlížeči), edice přihlášeného uživatele přílohy povoluje a instance běží s
`SAP_OPERATION_MODE=READ_WRITE`. Odkazy vyprší po 15 minutách a platí jednou;
nahrání je vázáno na chatovou relaci, která o něj požádala.

## 2. Z URL (`source_url`)

Server si soubor stáhne sám — např. z fileserveru nebo sdílení SharePoint. Tato
cesta je **vypnutá**, dokud nepovolíte hostitele:

```ini
SAP_ATTACHMENT_URL_ALLOWLIST=files.example.com,*.sharepoint.com
```

Pravidla: pouze HTTPS, hostitel musí být na seznamu (`*.domain` pokrývá
subdomény), adresy v privátních sítích se odmítají, pokud není povolena přímo IP,
přesměrování se nesledují, velikost je omezena.

## 3. Z cesty na serveru (`file_path`)

Pro lokálně nainstalovaný server nebo připojené síťové sdílení. **Vypnuto**, dokud
nenastavíte základní adresář; soubory musí ležet pod ním (symlinky ven se
odmítají):

```ini
SAP_ATTACHMENT_DIR=C:\sapb1-mcp\attachments
```

## Limity

| Proměnná | Výchozí | Význam |
|---|---|---|
| `SAP_UPLOAD_MAX_MB` | `25` | maximální velikost souboru pro všechny tři cesty |
| `SAP_ATTACHMENT_URL_ALLOWLIST` | prázdné (vypnuto) | hostitelé, ze kterých smí `source_url` stahovat |
| `SAP_ATTACHMENT_DIR` | prázdné (vypnuto) | adresář, ze kterého smí `file_path` číst |

Drobné soubory (do 32 KB) lze stále předat inline jako `content_base64`; nad tím
nástroj odmítne a odkáže na tři cesty.

## Předpoklad v SAP

Service Layer ukládá přílohy do **složky příloh** nakonfigurované v SAP Business
One (*Administrace → Inicializace systému → Obecná nastavení → Cesta → Složka
příloh*). Pokud chybí nebo do ní Service Layer nemůže zapisovat, nahrání selže s
chybou SAP — jde o nastavení na straně SAP, ne o volbu serveru.
