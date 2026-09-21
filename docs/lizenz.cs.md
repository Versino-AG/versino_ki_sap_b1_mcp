<!-- translation-of: lizenz.md@d11b4e7994d7 -->

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
| Edice | Čte | Nástroje nad rámec přihlášení a `sap_help` |
|---|---|---|
| BASIC | jen kmenová data (obchodní partneři, položky, …), omezená rychlost dotazů | `sap_query_odata`, `sap_fuzzy_search` |
| PRO | všechna data včetně dokladů, vyšší rychlost | navíc `sap_curated_query` (dodávané reporty `AI_*`), `sap_semantic_query`, `sap_attachment`, `sap_deploy_queries`, `sap_create`, `sap_update` |
| ENTERPRISE | všechna data, bez omezení rychlosti | navíc `sap_delete`, `sap_action` |

Nástroj, který by edice nikdy nemohla spustit, se **vůbec nenabízí** — asistent
ho nevidí, a nemůže tedy navrhnout něco, co licence nepokrývá. Nástroje čtoucí
libovolné tabulky (kurátorované reporty `AI_*`, pohledy semantické vrstvy,
přílohy) proto vyžadují edici s plným přístupem ke čtení; u BASIC se dodávané
reporty ani nenasazují, protože by tam nešly spustit. `sap_help` u každého
chybějícího nástroje uvádí důvod (`edition`, `read_scope`, `operation_mode`),
takže podpora zodpoví otázku „proč to neumí X" z jednoho místa. Nezávisle na
edici odebere `SAP_OPERATION_MODE=READ_ONLY` zápisové nástroje →
[konfiguration.cs.md](konfiguration.cs.md).

## Phone-Home (ochrana proti manipulaci, kontrola platnosti a obnova předplatného)
**Ve výchozím nastavení aktivní:** instalace v pravidelných intervalech vytváří
podepsané spojení s licenčním serverem. To slouží dvěma účelům:

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

> **Endpoint musí být `https`.** Požadavek phone-home nese číslo zákazníka a počet
> seatů, proto je prosté `http` na vzdálený host odmítnuto — nešifrovaně jsou
> povoleny jen `127.0.0.1`, `::1` a `localhost`, pro vývoj. Vlastní licenční server
> tedy potřebuje TLS; pokud `SAP_LICENSE_VALIDATION_URL` míří na adresu `http://` na
> jiném stroji, selže to, místo aby zákaznická data tiše odešla v otevřené podobě.
> Adresa pro enrollment se odvozuje z adresáře této URL — cestu (`…/validate`) proto
> ponechte beze změny.

> **Minimální verze 2.3.5.** Zapečený token je další pole v podepsané licenci.
> Starší verze programu toto pole neznají a takový klíč odmítnou — s ním
> nenaběhnou. Pokud dostanete nový `versino.key` a stále používáte verzi před
> 2.3.5, **nejprve aktualizujte software a teprve poté vyměňte klíč**. Váš starý
> `versino.key` platí až do svého data vypršení, takže jej můžete vždy vrátit.

**Provoz v kontejneru:** identita instalace se ve výchozím stavu zapisuje jako
`install_identity.json` do pracovního adresáře. Pokud kontejner běží s
`read_only: true` nebo v pomíjivém adresáři, nastavte `SAP_INSTALL_IDENTITY_PATH`
na **trvalou** cestu (volume) — jinak se při každém startu vytvoří nový klíčový
pár a s ním nová instalace:
```ini
SAP_INSTALL_IDENTITY_PATH=/data/install_identity.json
SAP_LICENSE_CACHE_FILE=/data/versino.renewed
```

Zda registrace proběhla, oznámí `sapb1-mcp doctor` (kontrola `enrollment`).

**Air-gapped / bez phone-home:** V prostředí bez internetu neprobíhá ani online
kontrola platnosti, ani automatické obnovení — dodaný token platí beze změny
až do svého data expirace a novou `versino.key` dostanete včas předem. Pokud phone-home nechcete vůbec, může jej Versino vypnout; pak dostáváte klíče bez registračního znaku (a tedy bez automatického prodloužení).

## Seaty
`max_seats` omezuje počet **distinktních** SAP uživatelů. Je-li potřeba víc,
navýšit přes licenční portál — nová `versino.key` nahradí starou.

Dotazy k licenci: **support@versino.de**
