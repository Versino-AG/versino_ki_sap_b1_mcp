<!-- translation-of: troubleshooting.md@87c72bdf8599 -->

# Řešení problémů

> 🌐 [English](troubleshooting.md) · [Deutsch](troubleshooting.de.md) · **Česky**

## `license.refused` při startu
Nenalezena platná licence. Zkontrolovat:
- leží `versino.key` **vedle** binárky? (nebo na ni ukazuje `SAP_LICENSE_FILE`?)
- není klíč prošlý? (nový přes [licenční portál](https://aishop.versino.de))

Detaily: [lizenz.cs.md](lizenz.cs.md).

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
- Nastavit `SAP_OPERATION_MODE=READ_WRITE` (výchozí je `READ_ONLY`).
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

## `npx` / Node.js nenalezen (bridge)
`mcp-remote` bridge vyžaduje **Node.js**. Nainstalovat Node LTS z
[nodejs.org](https://nodejs.org), restartovat klienta. Tip: v konfiguraci klienta
nahradit `npx` za `cmd /c npx` (Windows), pokud příkaz není nalezen.

## Linux: binárka nestartuje
- je nastavená spustitelnost? `chmod +x sapb1-mcp`
- předpoklad je 64-bit Linux s glibc (x86-64).

Problémy přetrvávají? **support@versino.de** (prosím s výřezem logu a číslem verze).
