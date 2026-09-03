<!-- translation-of: installation-linux.md@825665be4899 -->

# Provoz SAP-B1-MCP na Linuxu (manuálně)

> 🌐 [English](installation-linux.md) · [Deutsch](installation-linux.de.md) · **Česky**

> Pro **Windows** existuje **řízený instalátor**, který automatizuje většinu
> nastavení ([installer.cs.md](installer.cs.md)). Na Linuxu se nastavuje manuálně —
> tento návod.

Návod pro Linux distribuci (`sapb1-mcp-<verze>-linux-x64`). Server běží jako
**per-user HTTP server** (Streamable HTTP) — jedna instance obsluhuje více uživatelů;
každý se při připojení přihlašuje **svými vlastními** SAP B1 přihlašovacími údaji.

## 1. Stažení a umístění
Z [Releases](../../releases/latest) stáhnout `sapb1-mcp-<verze>-linux-x64` a umístit do
**jedné** složky (např. `/opt/sapb1-mcp/`):
- binárka (pro zjednodušení přejmenovat: `mv sapb1-mcp-*-linux-x64 sapb1-mcp`)
- `versino.key` — vaše licence (nalezena **automaticky vedle binárky**)
- `.env` — konfigurace (→ [konfiguration.cs.md](konfiguration.cs.md))

Nastavit spustitelnost:
```bash
chmod +x sapb1-mcp
```

## 2. `.env` (minimální)
Komentáře vždy na **vlastní řádek** (ne za hodnotu — inline komentáře mohou podle
parseru hodnotu zkomolit):
```ini
# URL Service Layeru vaší instance SAP B1
SAP_BASE_URL=https://vas-sap-host:50000/b1s/v2/

# Vybíratelné CompanyDB (jedna nebo více, oddělené čárkou)
SAP_DATABASES=SBO_VaseFirma

# Přihlašovací režim: "basic" nebo "bearer" (browser SSO přes Keycloak)
SAP_AUTH_MODE=basic

# Přístupový režim: READ_ONLY nebo READ_WRITE
SAP_OPERATION_MODE=READ_ONLY

# Jen při self-signed SL certifikátu
SAP_ALLOW_SELF_SIGNED_CERT=true

# Veřejná adresa této instance — potřebná pro browser-login (jinak "connect"
# nevrátí login link). Lokálně: http://127.0.0.1:8000 (port musí sedět se
# spuštěním serveru). V síťovém provozu HTTPS URL (viz installation-zentral.cs.md).
SAP_PUBLIC_URL=http://127.0.0.1:8000

# Phone-Home / automatické obnovení předplatného běží automaticky — enrollment token
# je zapečený ve versino.key, sem se nic nezadává. Jen při vědomě air-gapped provozu
# (bez internetu) k phone-home nedochází.
```
Kompletní seznam voleb: [konfiguration.cs.md](konfiguration.cs.md).

## 3. Spuštění serveru
```bash
cd /opt/sapb1-mcp
./sapb1-mcp --env-file .env --port 8000
```
Úspěch: log ukáže `server.per_user_start` a `Uvicorn running on http://127.0.0.1:8000`.
MCP endpoint je pak **`http://127.0.0.1:8000/mcp`**.

- **Výchozí stav:** server se váže na **všechna rozhraní** (`0.0.0.0`) — v URL použít
  interní IP/DNS, port uvolnit ve firewallu. Pro síťový provoz je doporučeno **TLS**
  (reverse-proxy).
- **Jen lokálně:** v `.env` nastavit `SAP_BIND_HOSTS=127.0.0.1` (možný seznam oddělený
  čárkou); explicitní `--host` má přednost.

> Pro **centrální** instanci obsluhující všechna pracoviště (bez instalace na každém
> pracovišti) viz [installation-zentral.cs.md](installation-zentral.cs.md).

## 4. Jako systemd služba (volitelné)
`/etc/systemd/system/sapb1-mcp.service`:
```ini
[Unit]
Description=SAP B1 MCP Server
After=network-online.target

[Service]
WorkingDirectory=/opt/sapb1-mcp
ExecStart=/opt/sapb1-mcp/sapb1-mcp --env-file /opt/sapb1-mcp/.env --port 8000
Restart=on-failure
User=sapb1mcp

[Install]
WantedBy=multi-user.target
```
```bash
sudo systemctl enable --now sapb1-mcp
```

## 5. Připojení k LLM klientům
- **Klienti podporující streamable HTTP** (Claude Desktop Custom Connector, Cline,
  Continue, Cursor …): míří na `http://<host>:8000/mcp`.
- **Jen-stdio klienti:** bridge `npx mcp-remote http://<host>:8000/mcp` (vyžaduje Node.js).

Kroky pro připojení Claude Desktop jsou shodné s Windows —
viz [installation-windows.cs.md](installation-windows.cs.md), oddíl 4.

## 6. První použití
V klientovi zavolat nástroj **`connect`** → SAP přihlášení (vstupní dialog/browser-login);
přihlašovací údaje se nikdy nedostanou do chatu/kontextu LLM. Poté jsou SAP nástroje
připravené (`sap_query_odata`, `sap_help`, `sap_create`, …).

## Poznámky
- Předpoklad: 64-bit Linux s glibc (x86-64).
- **Provoz v kontejneru (Docker):** na vyžádání — **support@versino.de**.
- Časté případy (`license.refused`, problémy s připojením) v [troubleshooting.cs.md](troubleshooting.cs.md).
