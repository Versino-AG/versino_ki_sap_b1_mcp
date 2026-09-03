<!-- translation-of: lizenz.md@57b1daffa357 -->

# Licence (`versino.key`)

> 🌐 [English](lizenz.md) · [Deutsch](lizenz.de.md) · **Česky**

SAP-B1-MCP server je zpoplatněný licencí. Licence je **podepsaný offline
soubor** (`versino.key`) — bez platné licence se server nespustí (fail-closed).

## Získání licence
Po nákupu, resp. přes **[licenční portál](https://aishop.versino.de)**, dostanete
`versino.key` (i e-mailem). Obsahuje:
- **Edici**: BASIC / PRO / ENTERPRISE (určuje dostupné nástroje),
- **max. seaty**: počet distinktních SAP uživatelů,
- **datum expirace**.

## Použití licence
Umístit `versino.key` **vedle binárku** (stejná složka jako `sapb1-mcp.exe`
resp. `sapb1-mcp`). Najde se automaticky — `SAP_LICENSE_FILE` **není** potřeba
nastavovat. Alternativně zadat explicitní cestu přes `SAP_LICENSE_FILE`.

## Edice (stručný přehled)
| Edice | Rozsah |
|---|---|
| BASIC | čtení kmenových dat, omezená rychlost dotazů |
| PRO | čtení + zápis (`sap_create`, `sap_update`) |
| ENTERPRISE | plný rozsah nástrojů včetně `sap_delete`, `sap_action` |

## Phone-Home (ochrana proti manipulaci, kontrola platnosti a obnova předplatného)
Volitelné a ve výchozím stavu **vypnuté**. Když ho zapnete, instalace v pevném
intervalu naváže podepsané spojení s licenčním serverem. To plní dva účely:

- **Kontrola platnosti/blokace (ochrana proti manipulaci):** Server potvrzuje, že
  licence je stále platná. Licence **zablokovaná** na straně serveru (např. při
  zneužití nebo neuhrazené platbě) se tak rozpozná — nová připojení jsou pak odmítnuta.
- **Automatické obnovení předplatného:** Při měsíčním/ročním vyúčtování server
  znovu podepíše token s novým datem expirace a doručí ho **stejným** kanálem;
  instalace ho sama převezme — **žádná ruční výměna klíče** za každé zúčtovací období.

Přenášejí se přitom jen metadata (`customer_id`, `edition`, `version`, počet
seatů) — žádná obchodní data.

Aktivace: **nic se nezadává** — enrollment token je **zapečený** (podepsaný) ve
vaší `versino.key` a validation URL je pevně zabudovaná v produktu. Jakmile
`versino.key` leží vedle binárky, instalace se při prvním spuštění sama
zaregistruje a obnovení přebírá automaticky. Volitelně lze nastavit
zapisovatelnou cestu pro renewal-cache (výchozí: `versino.renewed` vedle
binárky):
```ini
# volitelné: zapisovatelná cesta → převezme obnovený token
SAP_LICENSE_CACHE_FILE=versino.renewed
```
(Jen pro test/staging: `SAP_ENROLLMENT_TOKEN` přepíše vestavěný token,
`SAP_LICENSE_VALIDATION_URL` vestavěný endpoint.)

**Air-gapped / bez phone-home:** V prostředí bez internetu neprobíhá ani online
kontrola platnosti, ani automatické obnovení — dodaný token platí beze změny
až do svého data expirace a novou `versino.key` dostanete včas předem.

## Seaty
`max_seats` omezuje počet **distinktních** SAP uživatelů. Je-li potřeba víc,
navýšit přes licenční portál — nová `versino.key` nahradí starou.

Dotazy k licenci: **support@versino.de**
