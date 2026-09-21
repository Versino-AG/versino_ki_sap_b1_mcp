<!-- translation-of: installation-zentral.md@a7edd0be3160 -->

# SAP-B1-MCP centrální provoz (jedna instance pro všechna pracoviště)

> 🌐 [English](installation-zentral.md) · [Deutsch](installation-zentral.de.md) · **Česky**

Místo instalace binárky na každém pracovišti zvlášť běží **jedna** instance centrálně
na serveru v síti zákazníka. Všichni uživatelé se na ni připojují svým LLM klientem přes
síť — na pracovišti není potřeba **žádná** `.exe`, žádný `versino.key` ani žádný `.env`,
stačí zadat centrální URL do klienta.

Per-user charakter zůstává zachován: každý se při připojení přihlašuje **svými vlastními**
SAP B1 přihlašovacími údaji, seaty počítají distinktní SAP uživatele napříč všemi
pracovišti.

```
  Pracoviště A ─┐
  Pracoviště B ─┼─ HTTPS ─▶ [ Reverse-Proxy (TLS) ]
  Pracoviště C ─┘                   │  http://127.0.0.1:8000
                                    ▼
                          [ sapb1-mcp (služba) ] ─ HTTPS:50000 ─▶ SAP B1 Service Layer
                            + versino.key + .env
```

## 1. Výběr serveru
- Trvale běžící Windows nebo Linux server (VM nebo fyzický), dostupný v LAN/VPN uživatelů.
- Umístit **blízko SAP Service Layeru**: server potřebuje přístup na
  `https://<sap-host>:50000/b1s/v2/` (pozor na latenci/firewall).
- Doporučení: reverse-proxy a MCP server na **stejném** stroji — MCP server pak
  naslouchá jen lokálně (`127.0.0.1:8000`), navenek je vidět výhradně TLS proxy.

## 2. Základní instalace
Binárku, `versino.key` a `.env` umístit do **jedné** složky — přesně jak je popsáno v
[installation-windows.cs.md](installation-windows.cs.md), resp. [installation-linux.cs.md](installation-linux.cs.md).
Jen **jednou** na serveru, ne pro každé pracoviště.

## 3. Centrální `.env`
Oproti nastavení pro jedno pracoviště jsou pro centrální provoz důležitá tři nastavení.
Komentáře vždy na **vlastní řádek** (ne za hodnotu):
```ini
# URL Service Layeru vaší instance SAP B1
SAP_BASE_URL=https://vas-sap-host:50000/b1s/v2/

# všechny vybíratelné CompanyDB (oddělené čárkou)
SAP_DATABASES=SBO_FirmaA,SBO_FirmaB

# "basic" nebo "bearer" (browser SSO přes Keycloak)
SAP_AUTH_MODE=basic

# READ_WRITE jen pokud chcete povolit zápis
SAP_OPERATION_MODE=READ_ONLY

# --- obzvlášť důležité pro centrální provoz ---
# Přihlášení POUZE přes prohlížeč/dialog, nikdy jako argument v chatu
SAP_DISABLE_INLINE_LOGIN=true

# veřejná HTTPS URL této instance (ta od TLS proxy) — pro web-login
SAP_PUBLIC_URL=https://mcp.zakaznik.intern

# Phone-Home / automatické obnovení předplatného běží automaticky — enrollment token
# je zapečený ve versino.key, sem se nic nezadává. Jen při vědomě air-gapped provozu
# (bez internetu) k phone-home nedochází.
```
- **`SAP_PUBLIC_URL`** je URL, na které uživatelé server dosáhnou (ta od TLS proxy).
  **Musí být `https://`** — `http://` je povoleno jen pro `127.0.0.1`.
  Přes ni server sestavuje login URL prohlížeče (`<SAP_PUBLIC_URL>/login#t=…`).
- **`SAP_DISABLE_INLINE_LOGIN=true`** vynucuje, že SAP přihlašovací údaje se zadávají
  výhradně přes web-login a nikdy se nedostanou do kontextu LLM — u vícenávštěvnického
  provozu důrazně doporučeno.
- Kompletní seznam voleb: [konfiguration.cs.md](konfiguration.cs.md).

## 4. Trvalý provoz jako služba
Aby instance přežila restarty a běžela bez přihlášeného uživatele.

**Linux (systemd)** — `/etc/systemd/system/sapb1-mcp.service`:
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

**Windows (služba přes NSSM)** — [nssm.cc](https://nssm.cc):
```powershell
nssm install sapb1-mcp "C:\sapb1-mcp\sapb1-mcp.exe" "--env-file" "C:\sapb1-mcp\.env" "--port" "8000"
nssm set sapb1-mcp AppDirectory "C:\sapb1-mcp"
nssm start sapb1-mcp
```
(Server ve výchozím stavu naslouchá na všech rozhraních. Pokud reverse-proxy běží na
stejném stroji, nastavte v `.env` `SAP_BIND_HOSTS=127.0.0.1`, aby port nebyl navíc
otevřený navenek; pokud proxy běží na jiném hostu, port uvolněte jen interně.)

## 5. TLS pro síťový provoz (povinné)
V síti běží login tickety a přihlášení po drátě — proto **musí** být TLS zapnuté.
Jsou dvě cesty:

### Varianta A — TLS přímo v MCP (nativně, bez proxy)
MCP server terminuje HTTPS sám. V `.env`:
```ini
SAP_TLS_CERT_FILE=/etc/ssl/mcp.crt      # certifikát/řetěz (PEM)
SAP_TLS_KEY_FILE=/etc/ssl/mcp.key       # privátní klíč (PEM)
# SAP_TLS_KEY_PASSWORD=...               # jen u šifrovaného klíče
SAP_PUBLIC_URL=https://mcp.zakaznik.intern:8000

# zpřísnění (datové centrum):
# SAP_TLS_MIN_VERSION=1.2                 # minimální verze TLS (výchozí 1.2, možné 1.3)
# SAP_TLS_CIPHERS=...                     # pevný OpenSSL cipher string (jen TLS 1.2)
# SAP_TLS_CLIENT_CA_FILE=/etc/ssl/ca.pem # vynutí klientské certifikáty (mTLS)
```
Bez těchto proměnných zůstává HTTP. Jasně odděleno od
`SAP_ALLOW_SELF_SIGNED_CERT` (to platí **odchozím** směrem k SAP/Keycloaku).
Server staví TLS kontext zpřísněně: vynucená minimální verze, server cipher
preference, komprese/renegociace vypnuté; volitelně pevné ciphery a **mTLS**
(povinnost klientského certifikátu přes `SAP_TLS_CLIENT_CA_FILE`). Neúplnou
TLS konfiguraci (jen cert/klíč, chybějící soubor, neplatná minimální verze)
ohlásí start srozumitelně. Endpoint je pak přímo `https://<host>:<port>/mcp` —
firewall otevřete odpovídajícím způsobem (sekce 6).

### Varianta B — TLS reverse proxy (předřazená)
Proxy terminuje HTTPS a přeposílá **všechny** cesty (`/mcp`, `/login`, `/api/login`) na
lokální server. Důležité: **dlouhý read timeout** a **žádné bufferování odpovědi**
(Streamable HTTP / Server-Sent Events). Vhodné, pokud centrální proxy stejně
existuje nebo jsou žádoucí automatické certifikáty (např. Caddy/Let's Encrypt).

**Caddy** (nejjednodušší varianta včetně automatického certifikátu) — `Caddyfile`:
```
mcp.zakaznik.intern {
    reverse_proxy 127.0.0.1:8000 {
        flush_interval -1          # nebufferovat SSE/streaming
    }
}
```

**nginx** — server block:
```nginx
server {
    listen 443 ssl;
    server_name mcp.zakaznik.intern;
    ssl_certificate     /etc/ssl/mcp.zakaznik.intern.crt;
    ssl_certificate_key /etc/ssl/mcp.zakaznik.intern.key;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_buffering off;            # propustit streaming/SSE
        proxy_read_timeout 3600s;       # dlouhé MCP relace
    }
}
```
- **Certifikát:** veřejný (Let's Encrypt, pokud je jméno rozlišitelné/dostupné)
  nebo z **interní firemní CA**. Při interní CA musí být kořenový certifikát důvěryhodný
  na pracovištích (jinak prohlížeč/klient spojení odmítne).
- **Windows:** jako reverse-proxy se hodí **IIS s ARR/URL Rewrite**, případně také Caddy.
- Výsledek: endpoint `https://mcp.zakaznik.intern/mcp`, web-login `https://mcp.zakaznik.intern/login`.

### Zabezpečení proxy (doporučeno)
Server omezuje neúspěšná přihlášení podle adresy klienta a každý pokus loguje. Za
proxy vidí jen adresu proxy, dokud mu neřeknete, komu smí důvěřovat:

- V `.env`: `SAP_TRUSTED_PROXIES=127.0.0.1` (adresa proxy, jak ji vidí server).
  Pak platí `X-Forwarded-For` — jedno počítadlo na skutečného klienta místo
  jednoho pro celou kancelář.
- nginx: předávat adresu klienta a přidat limit požadavků na přihlašovací cesty
  — zastaví záplavu dřív, než dorazí k serveru:
  ```nginx
  limit_req_zone $binary_remote_addr zone=sapb1_login:10m rate=10r/m;
  server {
      # … TLS jako výše …
      add_header Strict-Transport-Security "max-age=31536000" always;
      location / {
          proxy_pass http://127.0.0.1:8000;
          proxy_http_version 1.1;
          proxy_set_header Host $host;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_buffering off;
          proxy_read_timeout 3600s;
      }
      location ~ ^/(api/login|login)$ {
          limit_req zone=sapb1_login burst=20 nodelay;
          proxy_pass http://127.0.0.1:8000;
          proxy_set_header Host $host;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      }
  }
  ```
- Omezte, kdo se k proxy vůbec dostane: IP allow-list (`allow`/`deny`) nebo jen
  přes VPN. Vlastní bránou serveru je přihlášení do SAP; hranice sítě je na vás.
- Co server dělá navíc: po `SAP_LOGIN_MAX_FAILURES` neúspěšných pokusech je adresa
  pozastavena (30 s, zdvojnásobuje se až do 15 min), přihlašovací odkazy jsou
  omezeny za minutu, každý pokus se zapisuje do audit logu a relace končí po
  `SAP_SESSION_MAX_SECONDS` (8 h) — viz [konfiguration.cs.md](konfiguration.cs.md).

## 6. Firewall
- Navenek (na pracoviště) otevřít jen **443/TLS** proxy.
- MCP port `8000` **neexponovat** do sítě (jen `127.0.0.1`).
- Ze serveru na SAP host musí být dostupný **50000** (Service Layer).
- Nastavte `SAP_ALLOWED_CLIENTS` na sítě, které smějí server oslovit, např.
  `SAP_ALLOWED_CLIENTS=10.0.0.0/8` (za proxy: adresu proxy). Každá jiná adresa
  je odmítnuta s 403 ještě před spuštěním jakékoli trasy.

> **Síťová hranice je na vás.** Přihlášení probíhá uživatelem, heslem a
> databází — samotný MCP endpoint záměrně nenese transportní autentizaci (kdo
> chce přístup přes token, použije `SAP_AUTH_MODE=bearer`). Znamená to: kdo
> dosáhne na port, může zkusit přihlášení se SAP údaji, chráněné jen omezovačem
> přihlášení. Za firewallem, s `SAP_ALLOWED_CLIENTS` a TLS jde o zvládnuté
> riziko, na rozhraní otevřeném do internetu nikoli. Server při startu hlasitě
> varuje, když naslouchá na všech rozhraních bez TLS a bez allowlistu.

## 7. Připojení pracovišť (bez lokální instalace)
Každý uživatel do LLM klienta zadá jen **centrální URL** — nic jiného.

**Claude Desktop, varianta A (doporučeno při TLS):** Nastavení → *Connectors* →
*Přidat vlastní connector* → URL `https://mcp.zakaznik.intern/mcp`. Protože instance teď
běží přes HTTPS, Claude Desktop URL přijme přímo — **není potřeba Node.js/bridge**.

**Claude Desktop, varianta B (bridge, pokud je požadován):**
`%APPDATA%\Claude\claude_desktop_config.json`:
```json
{
  "mcpServers": {
    "sapb1-mcp": {
      "command": "npx",
      "args": ["mcp-remote", "https://mcp.zakaznik.intern/mcp"]
    }
  }
}
```

**Ostatní klienti** (Cline, Continue, Cursor, vlastní agenti): klienti podporující
streamable HTTP míří přímo na `https://mcp.zakaznik.intern/mcp`; jen-stdio klienti přes
`mcp-remote` bridge.

> **Tip — rollout:** `claude_desktop_config.json` lze distribuovat centrálně přes
> GPO/Intune, aby všechna pracoviště automaticky dostala stejné připojení.

## 8. Přihlášení uživatele a seaty
- V klientovi zavolat **`connect`** → server vrátí URL pro přihlášení v prohlížeči
  (`https://mcp.zakaznik.intern/login#t=…`) → uživatel se tam přihlásí svými
  **vlastními** SAP přihlašovacími údaji a vybere CompanyDB. Přihlašovací údaje se
  nikdy nedostanou do chatu. Poté `connect(ticket="…")` — jeho výsledek vrátí **novou**
  hodnotu `ticket` (ticket z prohlížeče je vyřazen); tato hodnota doprovází každé další
  volání SAP nástroje a SAP nástroje jsou připravené.
- **Seaty:** centrální instance počítá distinktní SAP uživatele napříč všemi pracovišti.
  Počet je omezen `max_seats` licence → viz [lizenz.cs.md](lizenz.cs.md).

## 9. Provoz a aktualizace
- **Aktualizace:** vyměnit binárku a restartovat službu (`systemctl restart sapb1-mcp`
  resp. `nssm restart sapb1-mcp`); `versino.key` zůstává na místě. Obnovení předplatného
  probíhá automaticky přes phone-home → [lizenz.cs.md](lizenz.cs.md).
- **Logy:** služba loguje do stdout/journald (Linux) resp. do NSSM logu (Windows) —
  úvodní zpráva `server.per_user_start`, varování při vypnuté TLS kontrole atd.

## Řešení problémů
Časté případy (`license.refused`, problémy s připojením/přihlášením) v
[troubleshooting.cs.md](troubleshooting.cs.md).
