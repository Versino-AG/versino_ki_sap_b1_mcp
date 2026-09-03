<!-- translation-of: installer.md@abf974d8ecf0 -->

# Řízená instalace (Windows, doporučeno)

> 🌐 [English](installer.md) · [Deutsch](installer.de.md) · **Česky**

**Řízený Windows instalátor** kompletně nastaví SAP-B1-MCP server v několika krocích.
Odebere vám manuální kroky, které byste jinak museli dělat ručně:

- stažení a umístění programového souboru
- ruční zápis `.env` (správné klíče, žádné překlepy, správné kódování)
- zkopírování `versino.key` na správné místo
- ověření dostupnosti Service Layeru
- volitelně nastavení Windows služby
- volitelně nastavení Claude Desktop

> Kdo chce server nastavit **manuálně** nebo jemněji přizpůsobit (vlastní adresáře,
> vlastní service-wrapper, reverse-proxy, Linux), použije místo toho manuální návody:
> [installation-windows.cs.md](installation-windows.cs.md) /
> [installation-linux.cs.md](installation-linux.cs.md).

## Předpoklady

- 64-bit Windows 10/11 nebo Windows Server 2019+
- váš **licenční klíč** (e-mailem, resp. z [licenčního portálu](https://aishop.versino.de))
  — jako text ke vložení **nebo** jako soubor `versino.key`; funguje obojí. Viz [lizenz.cs.md](lizenz.cs.md)
- síťový přístup na váš SAP B1 **Service Layer** (`https://<host>:50000/b1s/v2/`)
- přístup k internetu během instalace (instalátor stahuje programový soubor z release)

## Stažení

Z [Releases](../../releases/latest) stáhnout setup:

| Soubor | Účel |
|-------|-------|
| `sapb1-mcp-setup-<verze>.exe` | řízený instalátor |

Setup je **podepsaný společností Versino AG** (Authenticode) — Windows zobrazí jako
vydavatele „Versino AG". Samotný programový soubor serveru si instalátor stahuje během
instalace, ve verzi odpovídající vybranému releasu.

## Průběh instalace

Spustit setup a postupovat podle asistenta. Ten se postupně zeptá na:

1. **Obchodní podmínky** — potvrzení [Všeobecných obchodních podmínek](https://aishop.versino.de/agb).
2. **Cílová složka** — kam se instaluje (výchozí: `%LOCALAPPDATA%\Versino\sapb1-mcp`,
   např. `C:\Users\<Uživatel>\AppData\Local\Versino\sapb1-mcp`). Instalace probíhá bez
   administrátorských práv; UAC se zeptá jen jednou, a to pouze při volitelném nastavení
   služby.
3. **Licenční klíč** — svůj klíč dodaný Versinem **vložit** do pole nebo přes
   **„Vybrat versino.key…“** načíst ze souboru (obojí skončí ve stejném poli).
   Instalátor stáhne programový soubor, zapíše klíč jako `versino.key` do cílové
   složky a **okamžitě offline** ho ověří (podpis + platnost proti vestavěnému klíči).
   Pokud je neplatný nebo prošlý, nejde pokračovat dál. Enrollment token (pro obnovu)
   je v něm obsažený — nic dalšího se ručně nezadává.
4. **Připojení Service Layeru** — zadat vaši SL URL (`https://<host>:50000/b1s/v2/`).
   Instalátor ověří **dostupnost** serveru. Pokud Service Layer (jak je obvyklé)
   vyžaduje přihlášení, počítá se to **jako dostupný** — hlásí se jen skutečná
   nedostupnost (chybná adresa, síť/firewall, problém s certifikátem).
5. **CompanyDB** — vybíratelné databáze (oddělené čárkou). Volitelně zde můžete zadat
   **testovací přihlášení** (uživatel/heslo); instalátor jím ověří název DB a přístup.
   Tyto testovací přihlašovací údaje se **neukládají**.
6. **Dostupnost** — port (výchozí `8000`) a volitelně veřejná URL. Lokálně stačí port;
   v síťovém provozu veřejná HTTPS URL (reverse-proxy, viz
   [installation-zentral.cs.md](installation-zentral.cs.md)).
7. **Volby**:
   - **Povolit zápis** (`READ_WRITE` místo jen čtení)
   - **Akceptovat self-signed SL certifikát**
   - **Nastavit jako Windows službu** — server pak běží automaticky (i bez
     přihlášeného uživatele)
   - **Automaticky zapsat konfiguraci Claude Desktop** (ve výchozím stavu zapnuto)

## Po instalaci

V cílové složce pak leží:

- `sapb1-mcp.exe` — server
- `versino.key` — vaše licence
- `.env` — předvyplněná konfigurace podle vašich zadání

Podle zvolených voleb navíc:

- **Windows služba** `SAPB1-MCP`, která server automaticky spouští
- je nastavený **záznam v Claude Desktop** (stávající záznamy zůstávají zachovány) —
  poté restartovat Claude Desktop

Pokud **nebyla** zvolena služba, server spustíte jak je popsáno v
[installation-windows.cs.md](installation-windows.cs.md), oddíl 3.

## První použití

V LLM klientovi zavolat nástroj **`connect`** → SAP přihlášení (vstupní dialog, resp.
browser-login). **Přihlašovací údaje se nikdy nedostanou do chatu/kontextu LLM.** Poté
jsou SAP nástroje připravené; `list_databases` zobrazí CompanyDB. Viz také
[erste-schritte.cs.md](erste-schritte.cs.md).

## Řešení problémů

Časté případy (varování Defenderu, `license.refused`, problémy s připojením) najdete v
[troubleshooting.cs.md](troubleshooting.cs.md).
