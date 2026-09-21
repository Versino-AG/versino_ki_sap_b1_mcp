<!-- translation-of: troubleshooting.md@9ae7271704d5 -->

# Řešení problémů

> 🌐 [English](troubleshooting.md) · [Deutsch](troubleshooting.de.md) · **Česky**

## Která verze běží?
Čtyři cesty, všechny se stejným číslem:

```
sapb1-mcp.exe --version                 # na příkazové řádce
curl http://<host>:8000/version         # z monitoringu nebo skriptu
```

Startovní protokol ji uvádí jako pole `version` v řádku `server.per_user_start`
a `sap_help` ji hlásí v bloku `instance` — hodí se, když máte před sebou
asistenta, ale ne stroj.

`/version` odpovídá bez přihlášení a neuvádí nic kromě názvu a čísla. Je-li
nastaveno `SAP_ALLOWED_CLIENTS`, odpoví jen adresám uvedeným tam. **Není** to
health check: říká, která verze je nainstalovaná, ne zda je server v pořádku.

## Kde jsou soubory protokolu
Dva soubory s omezenou velikostí v `%APPDATA%\Versino\sapb1-mcp\` (na Linuxu
`~/.config/Versino/sapb1-mcp/`):
- `sapb1-mcp.log` — varování a chyby běžícího serveru.
- `sapb1-mcp-audit.log` — stopa přihlášení (kdo, odkud, kdy, úspěch či ne).
  Vedena zvlášť a s větším rozpočtem, aby záplava běžných varování nevytlačila
  doklady, které potřebuje podpora nebo analýza incidentu — a naopak.

**Jako služba Windows** leží soubory místo toho pod instalací:
`<složka instalace>\logs\Versino\sapb1-mcp\`. Služba běží jako LocalSystem,
jehož `%APPDATA%` je složka uvnitř `C:\Windows`, kterou nikoho nenapadne
otevřít — proto ji instalátor nasměruje vedle instalace.

## `license.refused` při startu
Nenalezena platná licence. Zkontrolovat:
- leží `versino.key` **vedle** binárky? (nebo na ni ukazuje `SAP_LICENSE_FILE`?)
- není klíč prošlý? (nový přes [licenční portál](https://aishop.versino.de))

Detaily: [lizenz.cs.md](lizenz.cs.md).

## Nahrání přílohy selže s chybou SAP `-43`
`-43` je interní chyba SAP typu *cesta / složka*. Při nahrání přílohy znamená, že
**Service Layer** nemohl zapisovat do složky příloh — nejde o problém souboru
(stejný soubor selhává opakovaně). Zkontrolujte v SAP Business One:
*Administrace → Inicializace systému → Obecná nastavení → Cesta → Složka příloh*:
- cesta musí existovat **z pohledu hostitele Service Layeru** (UNC cesta jako
  `\\fileserver\B1_Prilohy`, ne písmeno jednotky uživatelského PC),
- účet služby Service Layer potřebuje **právo zápisu**,
- ve složce nesmí už ležet soubor se **stejným názvem** — jinak zkuste jiný
  `file_name`.
Po opravě na straně SAP prostě nahrajte znovu. Pozadí: [anhaenge.cs.md](anhaenge.cs.md).

## Varování Windows Defenderu / SmartScreenu
Pokud binárka ještě není podepsaná, může Windows zobrazit falešný poplach.
- „Další informace" → „Přesto spustit", resp. soubor povolit v Defenderu.
- Pro produkční distribuci je plánované code-signing — pak varování odpadne.

## Klient se k serveru nepřipojí
- Zkontrolovat host/port a cestu **`/mcp`** v URL.
- Při přístupu z jiných počítačů: výchozí bind je už `0.0.0.0` (zkontrolovat, že
  není nastavené omezující `SAP_BIND_HOSTS`/`--host`), uvolnit **firewall**
  pro daný port a v URL klienta použít **interní IP/DNS** serveru.
- Claude Desktop preferuje **HTTPS**; pro holé `http://` použít `mcp-remote` bridge
  (viz [installation-windows.cs.md](installation-windows.cs.md)).

## Připojení k SAP selhává
- Je `SAP_BASE_URL` správně? (`https://<host>:50000/b1s/v2/`)
- self-signed SL certifikát → `SAP_ALLOW_SELF_SIGNED_CERT=true`.
- správný `SAP_AUTH_MODE` (`basic` vs. `bearer`)?
- je zvolená CompanyDB obsažená v `SAP_DATABASES`?

## Chybí zápisové nástroje / jsou odmítány
- Nastavit `SAP_OPERATION_MODE=READ_WRITE` (výchozí je `READ_ONLY`). V režimu
  čtení se zápisové nástroje vůbec neregistrují, asistent je tedy nevidí. Po
  změně režimu **server restartujte** a LLM klienta znovu připojte — seznam
  nástrojů si ukládá do mezipaměti.
- Nahrání přílohy odmítnuto s „READ_ONLY: uploading attachments writes to SAP" →
  stejný přepínač; `info`/`download` fungují i v režimu čtení.
- Zápisové/mazací nástroje jsou navíc vázané na **edici** (PRO/ENTERPRISE),
  viz [lizenz.cs.md](lizenz.cs.md).
- **„… patří ke konfiguraci této instance"** — zápisové nástroje záměrně nemohou
  sáhnout na řídicí plochy serveru: `SQLQueries`, `SQLViews`, `Users`,
  `UserPermissionTree`, `UserObjectsMD`, `UserTablesMD`, `UserFieldsMD` a
  `B1Sessions`. Nejde o problém oprávnění — váš uživatel SAP to klidně smí. Je to
  hranice chatovacího rozhraní: asistent, který umí přepsat kurátorovaný report,
  by mohl vydávat zmanipulovaná čísla za ověřená. Na to použijte klienta SAP, pro
  kurátorované reporty `sap_deploy_queries`. Vše ostatní zůstává zapisovatelné.

## Nástroj chybí úplně (není jen odmítnut)
Server nabízí jen nástroje, které tato instalace opravdu umí spustit — asistent
tak nenavrhne nic, co licence nebo nastavení nepokrývají. Rozhodují tři brány a
`sap_help` u každého nástroje uvádí tu platnou pod `instance.tools_hidden`:

- `edition` — edice licence ho neobsahuje → [lizenz.cs.md](lizenz.cs.md).
- `read_scope` — edice čte jen kmenová data, zatímco nástroj čte libovolné
  tabulky (`sap_curated_query`, `sap_semantic_query`, `sap_attachment`, a tím i
  `sap_deploy_queries`).
- `operation_mode` — `SAP_OPERATION_MODE=READ_ONLY` (viz výše).

Po změně licence nebo režimu server restartujte a klienta znovu připojte —
seznam nástrojů si ukládá do mezipaměti.

## Windows: okno se hned zase zavře
Většinou je **port už obsazený** (jiná služba naslouchá na `8000`; v logu
`WinError 10048` / „… lze použít jen jednou"). Řešení:
- spustit server na **volném portu**: `sapb1-mcp.exe --port 8765` — a v klientovi
  použít stejné číslo portu (`…:8765/mcp`).
- nebo ukončit službu, která port obsazuje.
- aby bylo chybové hlášení **vidět** místo okamžitě se zavírajícího okna: spustit
  `.exe` z otevřeného terminálu (PowerShell), ne dvojklikem.

## `connect` nevrátí login link (web-login)
Fallback browser-loginu potřebuje **veřejnou adresu** instance. V `.env`
nastavit `SAP_PUBLIC_URL` (v síti `https://…`, lokálně `http://127.0.0.1:8000`) a
server restartovat. Viz [konfiguration.cs.md](konfiguration.cs.md) a
[installation-zentral.cs.md](installation-zentral.cs.md).

## Přihlašovací stránka na klik nereaguje / žádné potvrzení
Po kliknutí na „Přihlásit" se nic neděje a nezobrazí se stránka úspěchu — téměř
vždy je **SAP Service Layer nedostupný** (přihlášení běží do prázdna/timeout):
- Je SAP host dostupný **ze serveru**? (často potřeba **VPN**, firewall, port `50000`).
  Test: `Test-NetConnection <sap-host> -Port 50000` (Windows) resp. `nc -vz <sap-host> 50000`.
- Je `SAP_BASE_URL` správně (`https://<host>:50000/b1s/v2/`)?
- Tickety mají krátkou životnost: mezi otevřením odkazu a přihlášením nečekat příliš dlouho.

## „Příliš mnoho neúspěšných pokusů o přihlášení" / HTTP 429
Server pozastaví adresu po `SAP_LOGIN_MAX_FAILURES` neúspěšných přihlášeních
(výchozí 10 za 15 min): nejprve 30 s, zdvojnásobuje se až do 15 min; úspěšné
přihlášení pauzu zruší. Zpráva uvádí dobu čekání.
- Za reverse proxy bez `SAP_TRUSTED_PROXIES` je adresou **proxy** — překlepy
  jednoho kolegy pozastaví celou kancelář. Nastavte
  `SAP_TRUSTED_PROXIES=<adresa proxy>` (viz
  [installation-zentral.cs.md](installation-zentral.cs.md)).
- „Nezvykle mnoho žádostí o přihlášení … pozastavil nová přihlášení": byl dosažen
  globální strop `SAP_TICKET_ISSUE_PER_MINUTE` (výchozí 60) — počkejte minutu;
  zvyšujte jen u velmi velkých instalací.
- Každý pokus je v log souboru (`audit.auth.login` s uživatelem, CompanyDB,
  zdrojovou adresou a výsledkem — nikdy heslo), takže podpora přiřadí hlášení
  k řádku.

## „Vaše relace SAP dosáhla maximální délky"
Relace končí po `SAP_SESSION_MAX_SECONDS` (výchozí 8 h) bez ohledu na aktivitu;
asistent je vyzván znovu zavolat `connect` — nic víc není třeba. `0` limit vypne.

Relace oslovená **klíčem z přihlášení přes prohlížeč** má přísnější limit:
`SAP_TICKET_SESSION_MAX_SECONDS` (výchozí 1 h). Tento klíč prošel historií chatu a
posílá se při každém volání, takže vyprší dříve než relace vázaná na přenos — platí
kratší z obou mezí. Pokud se uživatelé musí po zhruba hodině přihlásit znovu, jde
o tuto proměnnou, nikoli o `SAP_SESSION_MAX_SECONDS`.

## Přihlašovací stránka hlásí, že odkaz neobsahuje ticket
Odkaz nese ticket za znakem `#` (`…/login#t=…`). Některá chatovací rozhraní tuto
část při zobrazení odkazu odříznou: otevřete odkaz přesně tak, jak byl vydán,
nebo požádejte asistenta přes `connect` o nový. Také obnovení stránky po
přihlášení fragment ztratí (záměrně) — vyžádejte nový odkaz.

## HTTP 403 „This address is not in SAP_ALLOWED_CLIENTS"
`SAP_ALLOWED_CLIENTS` omezuje, které adresy smějí server vůbec oslovit (IP/CIDR,
oddělené čárkou; prázdné = vypnuto, tedy kdokoli, kdo dosáhne na port). Kontrola běží
před každou routou, pokrývá tedy i MCP endpoint, nejen webové stránky. Doplňte adresu
nebo síť volajícího a restartujte. Za reverzní proxy vidí server adresu **proxy**,
nikoli koncového uživatele — do proměnné patří proxy a pro přihlašovací brzdu se
použije `SAP_TRUSTED_PROXIES`. Překlep přeruší start a pojmenuje záznam.

## HTTP 403 „Cross-site request refused"
Přihlašovací routy oslovuje MCP klient nebo přihlašovací stránka, kterou tento server
sám vydává — nikdy cizí web. Prohlížeč přicházející z cizí stránky
(`Sec-Fetch-Site: cross-site`/`same-site`, nebo `Origin`, který není ani vlastní host
požadavku, ani `SAP_PUBLIC_URL`) je odmítnut. MCP klienti ani jednu hlavičku
neposílají a nejsou dotčeni. Pokud to potká legitimní instalaci, bývá příčinou
reverzní proxy přepisující `Host`: musí propustit veřejné jméno, nebo musí
`SAP_PUBLIC_URL` odpovídat tomu, co prohlížeč skutečně volá.

## „Tento přihlašovací ticket už byl uplatněn"
Ticket z přihlášení přes prohlížeč vydá hodnotu relace **jednou**. Jedno zopakování se
ještě obslouží (aby ztracená odpověď nestála přihlášení), každé další dostane tuto
zprávu a žádnou hodnotu. Pro všechna další volání použijte hodnotu `ticket` z
úspěšného `connect` — je to jiná hodnota než ta v odkazu. Pokud se ztratila, spusťte
přes `connect` nové přihlášení v prohlížeči.

## `npx` / Node.js nenalezen (bridge)
`mcp-remote` bridge vyžaduje **Node.js**. Nainstalovat Node LTS z
[nodejs.org](https://nodejs.org), restartovat klienta. Tip: v konfiguraci klienta
nahradit `npx` za `cmd /c npx` (Windows), pokud příkaz není nalezen.

## Linux: binárka nestartuje
- je nastavená spustitelnost? `chmod +x sapb1-mcp`
- předpoklad je 64-bit Linux s glibc (x86-64).

Problémy přetrvávají? **support@versino.de** (prosím s výřezem logu a číslem verze).

## Licence vypršela / obnovení předplatného
- Binárka si splatné **obnovení předplatného vyzvedne automaticky při startu**
  (je-li k dispozici internet + aktivní předplatné). Navíc je po vypršení
  **3denní ochranná lhůta**, během níž server nastartuje i bez úspěšného
  phone-home.
- Pokud binárka po vypršení trvale odmítá nastartovat, zkontrolujte: je k
  dispozici přístup k licenčnímu serveru přes internet? Je předplatné v
  licenčním portálu (aishop.versino.de) aktivní/zaplacené? V případě pochybností
  si vyžádejte čerstvý `versino.key` na **support@versino.de**.
