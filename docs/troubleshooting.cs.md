<!-- translation-of: troubleshooting.md@5dfefad6971e -->

# Řešení problémů

> 🌐 [English](troubleshooting.md) · [Deutsch](troubleshooting.de.md) · **Česky**

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

## Přihlašovací stránka hlásí, že odkaz neobsahuje ticket
Odkaz nese ticket za znakem `#` (`…/login#t=…`). Některá chatovací rozhraní tuto
část při zobrazení odkazu odříznou: otevřete odkaz přesně tak, jak byl vydán,
nebo požádejte asistenta přes `connect` o nový. Také obnovení stránky po
přihlášení fragment ztratí (záměrně) — vyžádejte nový odkaz.

## `npx` / Node.js nenalezen (bridge)
`mcp-remote` bridge vyžaduje **Node.js**. Nainstalovat Node LTS z
[nodejs.org](https://nodejs.org), restartovat klienta. Tip: v konfiguraci klienta
nahradit `npx` za `cmd /c npx` (Windows), pokud příkaz není nalezen.

## Linux: binárka nestartuje
- je nastavená spustitelnost? `chmod +x sapb1-mcp`
- předpoklad je 64-bit Linux s glibc (x86-64).

Problémy přetrvávají? **support@versino.de** (prosím s výřezem logu a číslem verze).
