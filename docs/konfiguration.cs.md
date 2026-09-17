<!-- translation-of: konfiguration.md@07bf91c8df20 -->

# Konfigurace (`.env`)

> 🌐 [English](konfiguration.md) · [Deutsch](konfiguration.de.md) · **Česky**

Server čte konfiguraci instance ze souboru `.env` vedle binárky
(nebo přes `--env-file`). Nepatří sem **žádné přihlašovací údaje koncových
uživatelů** — jen nastavení instance. Každý uživatel se přihlašuje sám za běhu.

## Minimální konfigurace
Komentáře patří na **vlastní řádek** — **nikdy** za hodnotu na stejný řádek
(inline komentáře mohou podle parseru hodnotu zkomolit).
```ini
# URL Service Layeru vaší instance SAP B1
SAP_BASE_URL=https://vas-sap-host:50000/b1s/v2/

# Vybíratelné CompanyDB (jedna nebo více, oddělené čárkou)
SAP_DATABASES=SBO_VaseFirma

# Přihlašovací režim: "basic" (uživatel/heslo) nebo "bearer" (browser SSO při
# aktivním IAM) — nastavení viz sso-keycloak.cs.md
SAP_AUTH_MODE=basic

# Přístupový režim: READ_ONLY nebo READ_WRITE
SAP_OPERATION_MODE=READ_ONLY

# Jen při self-signed SL certifikátu
SAP_ALLOW_SELF_SIGNED_CERT=true

# Veřejná adresa této instance — potřebná pro browser-login (Claude Desktop).
# Lokálně: http://127.0.0.1:8000 (port musí sedět se spuštěním serveru); v síti HTTPS URL.
SAP_PUBLIC_URL=http://127.0.0.1:8000

# Phone-Home / automatické obnovení předplatného běží automaticky (enrollment token
# je zapečený ve versino.key — sem se nic nezadává). Override jen pro test/staging:
# SAP_ENROLLMENT_TOKEN=<override-token>
```

## Service Layer
| Proměnná | Výchozí | Význam |
|---|---|---|
| `SAP_BASE_URL` | – | URL Service Layeru, např. `https://host:50000/b1s/v2/` |
| `SAP_DATABASES` | – | Vybíratelné CompanyDB (seznam oddělený čárkou) |
| `SAP_OPERATION_MODE` | `READ_ONLY` | `READ_ONLY` nebo `READ_WRITE`. V režimu `READ_ONLY` se zápisové nástroje (`sap_create`, `sap_update`, `sap_delete`, `sap_action`, `sap_deploy_queries`) **vůbec nenabízejí**; `sap_attachment` zůstává pro `info`/`download` a nahrávání odmítá. Po změně režimu server restartujte (LLM klient načte seznam nástrojů znovu při dalším připojení) |
| `SAP_ALLOW_SELF_SIGNED_CERT` | `false` | povolit self-signed SL certifikát |
| `SAP_MAX_PAGE_SIZE` | `200` | max. řádků na stránku |
| `SAP_MAX_CONCURRENT_REQUESTS` | `10` | paralelní SL požadavky |
| `SAP_TIMEOUT_SECONDS` | `60` | HTTP timeout požadavků na Service Layer |
| `SAP_IDLE_LOGOUT_SECONDS` | `1500` | nečinnost, po které se uživatelská relace automaticky odhlásí |
| `SAP_SESSION_MAX_SECONDS` | `28800` | absolutní délka uživatelské relace (8 h) navíc k odhlášení při nečinnosti; poté je asistent vyzván znovu zavolat `connect`. `0` = bez omezení |
| `SAP_PHONE_HOME_INTERVAL_SECONDS` | `3600` | interval licenční kontroly phone-home |
| `SAP_DB_SERVER_TYPE` | _auto_ | přepsat typ databáze (`HANA` / `MSSQL`); běžně rozpoznán automaticky — nastavte jen při chybné detekci |
| `SAP_AUTO_DEPLOY_QUERIES` | `true` | nasadit dodávané reporty při prvním připojení dané CompanyDB |

### Dodávané reporty
Server při **prvním připojení** dané CompanyDB sám nasadí své hotové reporty
(dotazy `AI_*` v `SQLQueries`) — jednou za běh serveru. Předpoklady:
`SAP_OPERATION_MODE=READ_WRITE` a B1 uživatel, který smí vytvářet dotazy. V režimu
čtení (`READ_ONLY`) se to přeskočí; reporty pak chybí, vše ostatní funguje beze
změny. Vlastní napsané dotazy `AI_*` se nikdy nepřepíší. V případě potřeby lze
nasazení v chatu cíleně spustit (`sap_deploy_queries`) nebo vypnout přes
`SAP_AUTO_DEPLOY_QUERIES=false`.

## Autentizace
| Proměnná | Význam |
|---|---|
| `SAP_AUTH_MODE` | `basic` (uživatel/heslo přímo na SL) nebo `bearer` (browser SSO s PKCE, token při každém volání — od FP 2208 s tokeny SAP Authentication Serveru; pokud tokeny vydává vlastní Identity Provider, viz poznámka v sso-keycloak.cs.md) |
| `SAP_DISABLE_INLINE_LOGIN` | přihlášení jen přes dialog/web UI, nikdy jako argument v chatu. **Výchozí `true`, jakmile je nastaveno `SAP_PUBLIC_URL`** (síťová instalace); `false` jen lokálně nebo jako vědomé opt-in (`doctor` varuje) |
| `SAP_TRUSTED_PROXIES` | adresy reverse proxy (IP/CIDR, oddělené čárkou), jejichž `X-Forwarded-For` server důvěřuje pro omezení přihlášení a audit log — např. `127.0.0.1`, když nginx běží na stejném stroji. Prázdné = za klienta se bere socket peer (za proxy by to byla proxy sama, celá kancelář by pak sdílela jedno počítadlo) |
| `SAP_LOGIN_MAX_FAILURES` | neúspěšná přihlášení z jedné adresy během `SAP_LOGIN_WINDOW_SECONDS`, než je adresa pozastavena (výchozí `10`); pauza začíná na 30 s a zdvojnásobuje se až do 15 min, úspěšné přihlášení ji vynuluje |
| `SAP_LOGIN_WINDOW_SECONDS` | okno pro `SAP_LOGIN_MAX_FAILURES` (výchozí `900`) |
| `SAP_TICKET_ISSUE_PER_MINUTE` | globální strop přihlašovacích odkazů pro prohlížeč za minutu (výchozí `60`) — chrání neautentizovanou cestu před záplavou |
| `SAP_TLS_CERT_FILE` | serverový certifikát (PEM) pro příchozí HTTPS — společně se `SAP_TLS_KEY_FILE`; jinak HTTP |
| `SAP_TLS_KEY_FILE` | privátní klíč (PEM) pro příchozí HTTPS |
| `SAP_TLS_KEY_PASSWORD` | heslo šifrovaného TLS klíče (volitelné) |
| `SAP_TLS_MIN_VERSION` | minimální verze TLS pro příchozí HTTPS: `1.2` (výchozí) nebo `1.3` |
| `SAP_TLS_CIPHERS` | pevný OpenSSL cipher string (jen TLS 1.2; prázdné = bezpečné výchozí) |
| `SAP_TLS_CLIENT_CA_FILE` | klientská CA (PEM) → vynutí klientské certifikáty (mTLS) |
| `SAP_BIND_HOSTS` | bind adresy, oddělené čárkou (výchozí: všechna rozhraní/`0.0.0.0`; `--host` na příkazové řádce má přednost) |
| `SAP_LANG` | jazyk všech textů na straně serveru (start exe, `doctor`, web login včetně výsledkových/chybových hlášení, chyby ověření): `de` (výchozí), `en`, `cs`. Řízený instalátor sem zapíše jazyk zvolený v jeho dialogu. Odpovědi asistenta v chatu automaticky sledují jazyk uživatele |
| `SAP_PUBLIC_URL` | veřejná HTTPS URL instance (fallback web-login; **povinné při `bearer`** — cíl přesměrování `…/callback`) |
| `SAP_UPLOAD_MAX_MB` | max. velikost souboru pro nahrání přílohy v MB (výchozí `25`; nahrání v prohlížeči, `source_url`, `file_path`) — viz [anhaenge.cs.md](anhaenge.cs.md) |
| `SAP_ATTACHMENT_URL_ALLOWLIST` | hostitelé, ze kterých smí `sap_attachment` stáhnout `source_url`, oddělené čárkou (`host` nebo `*.domain`; prázdné = vypnuto) |
| `SAP_ATTACHMENT_DIR` | adresář, pod kterým smí `sap_attachment` číst `file_path` (prázdné = vypnuto) |

### Jen při `SAP_AUTH_MODE=bearer`

Browser SSO s PKCE — heslo se k MCP nikdy nedostane. Povinné jsou
`SAP_PUBLIC_URL` a údaje klienta z Extension Single Sign-On Manageru:

| Proměnná | Význam |
|---|---|
| `SAP_IDP_CLIENT_ID` | Client-ID registrovaného Web App klienta (`b1-ext-…`) |
| `SAP_IDP_CLIENT_SECRET` | příslušný Client-Secret (zobrazí se jen jednou) |
| `SAP_SLD_URL` | adresa SLD (např. `https://host:40000`) — u varianty A **nutná** pro zjištění CompanyID; zároveň automaticky zjistí všechny IdP endpointy |
| `SAP_COMPANY_IDS` | jen **varianta B** (vlastní Identity Provider): CompanyID poznamenaná při tenant-bindingu, pro každou databázi, formát `DB:ID`, více oddělených čárkou (`SBO_PROD:1,SBO_TEST:2`). Musí pokrýt **všechny** databáze ze `SAP_DATABASES`. Nahrazuje dotaz na SLD — port 40000 pak není potřeba |
| `SAP_IDP_TOKEN_URL` | token endpoint — potřeba jen pokud nemá proběhnout automatické zjištění přes `SAP_SLD_URL` |
| `SAP_IDP_AUTHORIZE_URL` | volitelné; odvozuje se z token URL. **Povinné**, pokud Identity Provider není Keycloak (Entra ID, Okta) |
| `SAP_IDP_SCOPE` | volitelné, výchozí `openid`. Rozšířené scopy nastavit jen, pokud jsou klientovi přiřazené |
| `SAP_IDP_END_SESSION_URL` / `SAP_IDP_JWKS_URL` | volitelné (odhlášení/centrální logout); rovněž součástí automatického zjištění |
| `SAP_IDP_ISSUER` / `SAP_IDP_REVOCATION_URL` | volitelné (vydavatel tokenu / odvolání tokenu); běžně se zjistí automaticky přes `SAP_SLD_URL` — nastavte jen při nutnosti přepsat automatiku |

Dřívější názvy `SAP_KEYCLOAK_*` stále platí (stejný význam).
Redirect URI `<SAP_PUBLIC_URL>/callback` musí být uložená u klienta (Extension
Single Sign-On Manager). Kompletní návod včetně kroků na straně SAP:
[sso-keycloak.cs.md](sso-keycloak.cs.md).

## Licence
| Proměnná | Význam |
|---|---|
| `SAP_LICENSE_FILE` | cesta k `versino.key` — **není potřeba**, pokud soubor leží vedle binárky (auto-discovery) |
| `SAP_LICENSE` | licenční token přímo (alternativa k souboru) |
| `SAP_LICENSE_CACHE_FILE` | cesta pro obnovené tokeny (tichá cache obnovy); výchozí `versino.renewed` vedle licence/binárky |
| `SAP_INSTALL_IDENTITY_PATH` | cesta identity instalace (`install_identity.json`); výchozí relativně k pracovnímu adresáři — v kontejnerech (read-only rootfs) nastavte trvalou cestu |
| `SAP_TIME_ANCHOR_PATH` | monotonní časová kotva proti vrácení hodin (hardening pro air-gapped provoz); bez cesty vypnuto |
| `SAP_ENROLLMENT_TOKEN` | phone-home/auto-renewal — **obvykle není potřeba** (token je zapečený ve `versino.key`). Jen jako override pro test/staging |

Phone-Home / automatické obnovení předplatného (běžný případ): viz [lizenz.cs.md](lizenz.cs.md).

## Bezpečnostní upozornění
- `SAP_OPERATION_MODE=READ_ONLY` jako výchozí; `READ_WRITE` jen pokud je zápis
  skutečně žádoucí. Instance jen pro čtení zápisové nástroje LLM klientovi vůbec
  nezobrazí, asistent se je tedy ani nemůže pokusit použít.
- `SAP_DISABLE_INLINE_LOGIN=true` zajistí, že se SAP přihlašovací údaje nikdy
  nedostanou do kontextu LLM (přihlášení jen přes dialog/web UI). U síťových
  instalací (nastavené `SAP_PUBLIC_URL`) je to výchozí.
- Přihlášení v prohlížeči je zabezpečené už ve výchozím stavu: přihlašovací odkaz
  nese ticket ve **fragmentu** URL (`…/login#t=…`, nikdy v logu serveru ani
  proxy); po `connect(ticket=…)` je ticket vyřazen a vrácena nová hodnota relace;
  neúspěšné pokusy jsou omezovány podle adresy a každý pokus se zapisuje do audit
  logu (uživatel, CompanyDB, zdrojová adresa, výsledek — nikdy heslo); relace končí
  po `SAP_SESSION_MAX_SECONDS`.
- Za reverse proxy nastavte `SAP_TRUSTED_PROXIES`, aby omezení vidělo skutečné
  adresy klientů, a doplňte limity na straně proxy z
  [installation-zentral.cs.md](installation-zentral.cs.md).
- Pro síťový provoz zapnout TLS (nativně v MCP, nebo reverse proxy).

## Logování

Varování a chyby se vždy zapisují do `%APPDATA%\Versino\sapb1-mcp\sapb1-mcp.log`
(JSON řádky, limit 1 MB — nejstarší záznamy se automaticky odstraňují).
Cesta je pevná, aby ji podpora vždy znala.
