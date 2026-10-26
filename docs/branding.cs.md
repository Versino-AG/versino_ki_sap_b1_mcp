<!-- translation-of: branding.md@962430aacd95 -->
# Vlastní vzhled stránek v prohlížeči

> 🌐 [English](branding.md) · [Deutsch](branding.de.md) · **Česky**

Stránky, které vaši uživatelé vidí v prohlížeči (přihlášení, výběr databáze u SSO,
potvrzení po přihlášení, nahrání souboru), mohou nést vaše logo, barvy a texty.
Měníte je soubory vedle serveru — bez nové verze, bez kódu.

## Složka s brandingem

Vedle binárky vytvořte složku `branding`:

```
<instalační složka>\
  sapb1-mcp.exe
  .env
  versino.key
  branding\
    branding.json      <- barvy, texty, patička (vše volitelné)
    logo.png           <- nebo logo.jpg / logo.webp / logo.svg
```

Jiné umístění lze nastavit v `.env`:

```
SAP_BRANDING_DIR=D:\sapb1-mcp\branding
```

Složka se čte **při startu**. Změny se projeví po restartu. Složka uvedená
v `SAP_BRANDING_DIR`, která neexistuje, start zastaví, aby překlep nezůstal
bez povšimnutí. Hotový příklad je složka `branding-example` v repozitáři ke
stažení: zkopírujte ji, přejmenujte na `branding` a upravte.

## branding.json

Všechna pole jsou volitelná. Text napsaný v jednom jazyce se zobrazí v každém
jazyce bez vlastního textu (nejprve jazyk z `SAP_LANG`, pak angličtina, němčina,
čeština).

```json
{
  "colors": { "accent": "#0a6f8c", "accent2": "#0fa3b8", "bg": "#0a1622", "bg2": "#0d2233" },
  "eyebrow": { "en": "Example Ltd", "cs": "Vzor s.r.o." },
  "title": { "en": "SAP assistant sign-in", "cs": "Přihlášení k asistentovi SAP" },
  "subtitle": { "en": "Sign in with your SAP user.", "cs": "Přihlaste se svým uživatelem SAP." },
  "footer": { "en": "Questions: it@example.com", "cs": "Dotazy: it@example.com" },
  "page_title": "Example – SAP assistant",
  "support_url": "https://intranet.example.com/sap-help"
}
```

| Pole | Účinek | Pravidla |
|---|---|---|
| `colors.accent`, `colors.accent2` | barva zvýraznění a její partner v přechodu (horní pruh, tlačítko, značka) | `#rgb` nebo `#rrggbb` |
| `colors.bg`, `colors.bg2` | pozadí stránky (přechod) | `#rgb` nebo `#rrggbb` |
| `eyebrow` | malý řádek nad nadpisem na každé stránce (výchozí „SAP Business One“) | do 80 znaků na jazyk |
| `title` | nadpis přihlašovací stránky | do 80 znaků |
| `subtitle` | řádek pod nadpisem přihlašovací stránky | do 200 znaků |
| `footer` | řádek pod kartou na každé stránce | do 200 znaků |
| `page_title` | titulek karty prohlížeče u přihlašovací a nahrávací stránky | do 80 znaků |
| `support_url` | odkaz „Nápověda a kontakt“ v patičce, otevře novou kartu | adresa `https://`, do 200 znaků |

Jazyky jsou `en`, `de` a `cs`. Výběr databáze, potvrzení a nahrávací stránka si
ponechají vlastní nadpisy, protože uživateli říkají, co se stalo. Texty se
zobrazují jako prostý text: HTML nefunguje a zobrazí se tak, jak je napsáno.

Barvy se nekontrolují na kontrast. Po změně se na stránku jednou podívejte.

## Logo

Jeden soubor s pevným názvem: `logo.png`, `logo.jpg`, `logo.webp` nebo `logo.svg`.
Je-li jich více, platí první v tomto pořadí. Nejvýše 512 KB. Obsah musí odpovídat
příponě. Logo nahradí značku databáze v kartě a zobrazí se ve výšce 42 pixelů.
Nejlépe působí široké logo.

## Témata pro partnery

`SAP_WEB_THEME` přepne na úplný vzhled, který je součástí binárky:

| Hodnota | Vzhled |
|---|---|
| `versino` | výchozí (tmavé pozadí, karta) |
| `cloudiax` | vzhled Cloudiax: bílé pozadí, logo nad kartou, vlastní písmo |

Složka s brandingem působí nad tématem: její barvy, texty a logo mají přednost.
S tématem má obvykle smysl jen logo a texty; vlastní barvy pozadí nahradí pozadí
tématu a mohou se s jeho vzhledem tlouct.

## Jazyk stránek

Stránky se řídí jazykem prohlížeče (`en`, `de` nebo `cs`). Prohlížeč v jiném
jazyce dostane jazyk z `SAP_LANG`. Zprávy v chatu a v logu zůstávají v `SAP_LANG`.

## Kontrola před startem

```
sapb1-mcp doctor
```

Řádek `branding` ukáže téma, složku a co z ní platí, nebo chybu se souborem
a polem. Start serveru provede stejnou kontrolu a skončí se stejnou zprávou.

| Zpráva | Příčina |
|---|---|
| `branding.json: invalid JSON in line 4` | poblíž tohoto řádku chybí čárka nebo uvozovka |
| `colors.accent: … hex colour` | barva není zapsána jako `#rrggbb` |
| `title: text for 'de' is longer than 80 characters` | zkraťte text |
| `titel: Extra inputs are not permitted` | překlep v názvu pole |
| `logo.png: the content does not match the extension (image/png)` | soubor jen přejmenován, ne převeden |
| `SAP_BRANDING_DIR=…: the folder does not exist` | cesta v `.env` je chybná |

## Za reverzní proxy

Logo a písma se doručují pod `/branding/`. Pokud vaše proxy přeposílá jen vybrané
cesty, doplňte `/branding/` (viz installation-zentral.cs.md).
