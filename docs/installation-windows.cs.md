<!-- translation-of: installation-windows.md@b1c629aea651 -->

# Připojení SAP-B1-MCP k Claude Desktop a dalším LLM (Windows, manuálně)

> 🌐 [English](installation-windows.md) · [Deutsch](installation-windows.de.md) · **Česky**

> **Rychleji to jde s řízeným instalátorem** — ten za vás udělá kroky níže
> (stažení, zápis `.env`, uložení licence, ověření připojení, služba, Claude config):
> [installer.cs.md](installer.cs.md). Tento návod popisuje **manuální** nastavení pro
> vlastní adresáře, vlastní service-wrapper nebo jemnější úpravy.

Návod pro Windows distribuci (`sapb1-mcp.exe`). Server běží jako **per-user HTTP server**
(Streamable HTTP) — jedna instance obsluhuje více uživatelů; každý se při připojení
přihlašuje **svými vlastními** SAP B1 přihlašovacími údaji.

## 1. Stažení a umístění
Z [Releases](../../releases/latest) stáhnout `sapb1-mcp-<verze>-windows-x64.exe`.
Pro zjednodušení přejmenovat na `sapb1-mcp.exe` a umístit spolu s ostatními soubory do
**jedné** složky (např. `C:\sapb1-mcp\`):
- `sapb1-mcp.exe` — server
- `versino.key` — vaše licence (nalezena **automaticky vedle .exe**, není potřeba cesta)
- `.env` — konfigurace (→ [konfiguration.cs.md](konfiguration.cs.md))

## 2. Konfigurace `.env` (minimální)
**Důležité:** komentáře vždy pište na **vlastní řádek** — **ne** za hodnotu na stejný
řádek (inline komentáře mohou podle parseru hodnotu zkomolit).

```ini
# URL Service Layeru vaší instance SAP B1
SAP_BASE_URL=https://vas-sap-host:50000/b1s/v2/

# Vybíratelné CompanyDB (jedna nebo více, oddělené čárkou)
SAP_DATABASES=SBO_VaseFirma

# Přihlašovací režim: "basic" (uživatel/heslo) nebo "bearer" (browser SSO přes Keycloak)
SAP_AUTH_MODE=basic

# Přístupový režim: READ_ONLY nebo READ_WRITE
SAP_OPERATION_MODE=READ_ONLY

# Jen při self-signed SL certifikátu
SAP_ALLOW_SELF_SIGNED_CERT=true

# Veřejná adresa této instance — potřebná pro browser-login v Claude
# Desktop (jinak "connect" nevrátí login link). Lokálně/stejný stroj:
# http://127.0.0.1:8000 — port MUSÍ sedět se spuštěním serveru (oddíl 3).
# V síťovém provozu místo toho HTTPS URL (viz installation-zentral.cs.md).
SAP_PUBLIC_URL=http://127.0.0.1:8000

# Phone-Home / automatické obnovení předplatného běží automaticky — enrollment token
# je zapečený ve versino.key, sem se nic nezadává. Jen při vědomě air-gapped provozu
# (bez internetu) k phone-home nedochází.
```
Licence se čerpá z `versino.key` vedle .exe — `SAP_LICENSE_FILE` **není** potřeba
nastavovat. Kompletní seznam voleb: [konfiguration.cs.md](konfiguration.cs.md).

## 3. Spuštění serveru
```powershell
cd C:\sapb1-mcp
.\sapb1-mcp.exe --env-file .env --port 8000
```
Úspěch: log ukáže `server.per_user_start` a `Uvicorn running on http://127.0.0.1:8000`.
MCP endpoint je pak **`http://127.0.0.1:8000/mcp`**.

- **Výchozí stav:** server se váže na **všechna rozhraní** (`0.0.0.0`) — jiné počítače
  v síti se k němu dostanou přímo přes `http://<interní-IP/DNS>:8000/mcp`; port uvolnit
  ve **Windows Firewall**. Pro síťový provoz je doporučeno **TLS** (reverse-proxy).
- **Jen lokálně:** v `.env` nastavit `SAP_BIND_HOSTS=127.0.0.1` (možný seznam oddělený
  čárkou, např. `127.0.0.1,192.168.1.10`); explicitní `--host` má přednost.

> Pro **centrální** instanci obsluhující všechna pracoviště (bez instalace na každém
> pracovišti) viz [installation-zentral.cs.md](installation-zentral.cs.md).

## 4. Připojení k Claude Desktop
**Varianta A — Custom Connector (URL):** Nastavení → *Connectors* →
*Přidat vlastní connector* → URL `http://127.0.0.1:8000/mcp` (resp. interní
HTTPS URL). *Poznámka:* Claude Desktop preferuje **HTTPS** — pro holé `http://`
použijte variantu B.

**Varianta B — bridge přes konfigurační soubor** (funguje i s `http://`;
vyžaduje Node.js): `%APPDATA%\Claude\claude_desktop_config.json`:
```json
{
  "mcpServers": {
    "sapb1-mcp": {
      "command": "npx",
      "args": ["mcp-remote", "http://127.0.0.1:8000/mcp"]
    }
  }
}
```
Poté restartovat Claude Desktop.

## 5. Ostatní MCP klienti (Cline, Continue, Cursor, vlastní agenti …)
- **Klienti podporující streamable HTTP:** míří přímo na `http://<host>:8000/mcp`.
- **Jen-stdio klienti:** stejný `mcp-remote` bridge jako výše (`npx mcp-remote <url>`).

## 6. První použití
V klientovi zavolat nástroj **`connect`**. Server vyžádá SAP přihlášení
(vstupní dialog klienta, resp. browser-login) — **přihlašovací údaje se nikdy
nedostanou do chatu/kontextu LLM**. Poté jsou SAP nástroje připravené
(`sap_query_odata`, `sap_help`, `sap_create`, …); `list_databases` zobrazí CompanyDB.

## Řešení problémů
Časté případy (varování Defenderu, `license.refused`, problémy s připojením) najdete v
[troubleshooting.cs.md](troubleshooting.cs.md).
