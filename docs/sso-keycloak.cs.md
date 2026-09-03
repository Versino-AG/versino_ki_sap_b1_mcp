<!-- translation-of: sso-keycloak.md@300fb6dda9a4 -->

# Připojení SSO přes Keycloak (OIDC) — návod k nastavení

> 🌐 [English](sso-keycloak.md) · [Deutsch](sso-keycloak.de.md) · **Česky**

Potřeba jen tehdy, když váš SAP B1 využívá **Authentication Server (SLD/IAM,
„Keycloak")** a chcete **skutečné SSO**. Pokud se přihlašujete klasicky přímo
na Service Layer, zůstává `SAP_AUTH_MODE=basic` (→ [konfiguration.cs.md](konfiguration.cs.md))
a tento dokument je irelevantní.

V obou variantách platí: uživatel se přihlašuje **v prohlížeči přímo u
Keycloaku** (Authorization Code + PKCE) — funguje stávající SSO session, MFA i
federované přihlášení (AD/SAML), a **heslo se k MCP nikdy nedostane**.
„Direct Access Grant" (ROPC) na klientovi **není** potřeba.

## 1. SSO režim `bearer` — dvě úrovně

SSO vždy běží přes `SAP_AUTH_MODE=bearer`: po přihlášení v prohlížeči jde token
s **každým** voláním Service Layeru (`Authorization: Bearer` +
`X-b1-companyid`). Přesně tuhle cestu popisuje SAP příručka „Identity and
Authentication Management in SAP Business One" (kap. 6.5.8 a 6.9).

Rozlišuje se, **kdo tokeny vydává**:

| Varianta | Vydavatel tokenu | Minimální verze | Kdy |
|---|---|---|---|
| **A — SAP Authentication Server** (běžný případ) | realm `sapb1` vašeho SAP B1; klient z Extension Single Sign-On Manageru (`b1-ext-…`) | SAP B1 10.0 **FP 2208** | vždy, když přihlášení běží přes SAP Authentication Server — **i tehdy, když za ním stojí externí provider** (AD, Entra ID, Okta, SAP IAS): SLD pro něj v realmu `sapb1` vytvoří broker adresu (`…/auth/realms/sapb1/broker/b1-<Alias>/endpoint` — v příručce pro AD FS na str. 20, Entra ID str. 31, Okta str. 39), uživatel se přihlásí u externího provideru, ale token vydává realm `sapb1` |
| **B — vlastní Identity Provider vydává tokeny sám** | vaše vlastní IdP instance, registrovaná v Extension SSO Manageru jako důvěryhodný provider; SAP tomu říká **Principal Propagation** | viz poznámka níže | jen když má tokeny vydávat vaše vlastní aplikační krajina, místo aby se pro ně chodilo na SAP Authentication Server |

**V praxi je téměř vždy správně varianta A.** Tabulka verzí v SAP příručce
(kap. 6.9) uvádí i pro scénář „je aktivní jeden nebo více externích Identity
Providerů" cestu „zaregistrovat extension client ID a napojit se na access
token" **od FP 2208**: pokud je váš AD/Entra/Okta aktivovaný v SLD jako
Identity Provider, uživatel se přihlásí tam — ale **token pro Service Layer
pořád vydává SAP Authentication Server**. To je varianta A.

> **Poznámka k variantě B:** IAM příručka popisuje Principal Propagation jen
> koncepčně (kap. 6.10) a odkazuje na samostatný SAP dokument
> **„Principal Propagation for SAP Business One" (Security Guide, verze 1.0 –
> 2025-03-25)**. Tam jsou kroky k nastavení (viz krok 4 níže).
> **Minimální verzi neuvádí ani tento dokument** — rozšířený údaj „od FP 2411"
> tak není doložený v žádném ze tří SAP zdrojů. Váš Feature Package si proto
> předem ujasněte s námi.

Obě varianty potřebují přístup k SLD (port 40000) pro zjištění CompanyID.

> **Důležité — co se s aktivním IAM mění pro `basic`:** navázaní uživatelé se
> nepřihlašují už svým **B1 uživatelským kódem**, ale **přihlašovacími údaji
> Authentication Serveru**. Příručka to formuluje podmínkou, která se dá
> snadno přehlédnout (kap. 6.1, str. 140), doslova (anglicky, jde o citaci):
> *„If you **only** enabled the identity provider SAP Business One
> Authentication Server, the DIAPI and Service Layer login interfaces allow
> you to use the SAP Business One Authentication Server user and password to
> login."* (Volně přeloženo: „Pokud jste aktivovali **pouze** identity
> providera SAP Business One Authentication Server, přihlašovací rozhraní
> DIAPI a Service Layer umožňují přihlášení uživatelem a heslem SAP Business
> One Authentication Serveru.")
>
> Z toho plyne:
> - **Jen** Authentication Server aktivní jako Identity Provider → `basic`
>   zůstává použitelné. Uživatelské jméno je uživatelský kód Authentication
>   Serveru; ve výchozím stavu je shodný s uživatelským jménem u IdP (kap. 3.2.4).
> - **Navíc aktivní externí** Identity Provider (AD FS, Entra ID, Okta,
>   SAP IAS) → pro tyto uživatele je cestou `bearer`. Tabulka verzí v kap. 6.9
>   pro tento scénář od FP 2305 uvádí **už jen** cestu přes access token,
>   možnost uživatel/heslo tam odpadá. Takoví uživatelé se zásadně přihlašují
>   svou **e-mailovou adresou** — u externího provideru je to jediný
>   přípustný identifikátor (kap. 3.2.1) a e-mailová doména řídí, na jakou
>   přihlašovací stránku se uživatel přesměruje (kap. 3.1.1 a 5.1.5).
> - ⚠️ **Otevřená otázka u smíšené krajiny:** příručka výslovně povoluje
>   klasické přihlášení jen tehdy, když je aktivní **výhradně** Authentication
>   Server. Jsou-li aktivní oba, není pro **žádného** uživatele doloženo, že
>   `basic` dál funguje — ani pro ty navázané na Authentication Server. V této
>   konstelaci postup ladíme společně.
> - Uživatelé s **dvoufaktorovou autentizací** nemohou klasickou cestu
>   rovněž použít. Příručka to dokládá pro příkazový režim DTW (kap. 7), pro
>   Service Layer to výslovně neuvádí.

## 2. Předpoklady na straně SAP — v tomto pořadí

Provádějící role: **B1 administrátor / Landscape administrátor**. U hostovaných
systémů (např. Cloudiax) případně společně s providerem (přístup do SLD, porty).

> **Dvě administrační rozhraní — nejdřív zjistěte, které máte.** SAP
> dokumentuje IAM ve dvou příručkách a postupy se liší:
> - **On-premise / hostováno:** *SLD Control Center*, typicky
>   `https://<sap-host>:40000/ControlCenter`. Kroky níže jsou popsané takto.
> - **SAP Business One Cloud:** *Cloud Control Center*. Tam se sekce jmenují
>   *System Configuration → Identity Providers*, resp. *Customer Management →
>   Customers → Customer Details → Identity Providers*, a adresy SLD a
>   Authentication Service se konfigurují tam — nejsou tedy nutně
>   `<host>:40000` / `<host>:40020`.
>   ⚠️ Navíc je tam potřeba **jednorázově** nastavit *System Configuration →
>   Global Settings → **Enable Third Party Identity Provider** = On*; teprve
>   pak se objeví tlačítko *Add* pro Identity Providery. Podmínkou je
>   registrovaný software repository pro **FP 2405 nebo vyšší**.
>
> Všechno ostatní (registrace klienta v Extension Single Sign-On Manageru,
> navázání uživatele, přenos tokenu) je v obou variantách shodné.

1. **Ověřit IAM** — Control Center (viz rámeček výše), záložka *Identity
   Providers*: alespoň jeden Identity Provider musí být **Active** (SAP
   Business One Authentication Server, Active Directory Domain Services nebo
   externí OIDC provider).
   ⚠️ Před aktivací navázat **všechny** uživatele (krok 2) — poté se navázaní
   uživatelé přihlašují přihlašovacími údaji Identity Providera, ne už
   B1 uživatelským kódem (viz rámeček výše).
2. **Navázat uživatele** — Control Center, záložka *Users*: u každého uživatele,
   který má MCP používat, označit → **Bind** → přiřadit server, databázi/e
   firmy a B1 uživatelský kód. Bez tohoto navázání přihlášení selže.
3. **Vytvořit OAuth klienta** — **SAP Business One Extension Single Sign-On
   Manager** (instaluje se s Extension Managerem) → *Extensions* → **Register**:
   - **Client Type: „Web App"** (dodá Client-ID *i* Client-Secret).
   - **Redirect URI:** `<SAP_PUBLIC_URL>/callback`
     (např. `https://mcp.vas-host.cz/callback`; pro lokální test navíc
     `http://127.0.0.1:8000/callback`). Příručka tu povoluje i **wildcardy**
     (např. `https://mcp.vas-host.cz/*`) — přesto doporučujeme plnou URL, aby
     platila jen zamýšlená zpětná cesta.
   - ⚠️ **Client-Secret se zobrazí jen jednou** — hned si ho uložte.
   - Volitelně (pro centrální odhlášení, od FP 2508): navíc uložit URL
     back-channel logoutu `<SAP_PUBLIC_URL>/backchannel-logout`.
4. **Jen u varianty B (vlastní Identity Provider vydává tokeny).**
   U varianty A tento krok úplně odpadá. Zdroj: SAP Security Guide
   „Principal Propagation for SAP Business One" (verze 1.0), kap. 1.2/1.3.
   - **Předpoklady:** váš Identity Provider je v SLD nakonfigurovaný jako
     Identity Provider třetí strany; vydává access tokeny ve formátu **JWT** a
     má **vlastní discovery URL**; identita uživatele se vede přes
     **e-mailovou adresu**, která musí být **jedinečná v celé krajině**
     („one e-mail address exclusively represents one user only").
   - **Registrace IdP:** *Extension Single Sign-On Manager → Principal
     Propagation → Identity Providers → Register*. Zadává se: **Name**,
     **Discovery Endpoint** (`…/.well-known/openid-configuration`) a
     **Identity Claim Name** — přes něj SAP čte e-mail z tokenu. Claim musí
     být v tokenu obsažený a nést správný e-mail uživatele.
   - **Navázání company:** *Principal Propagation → Tenants → Bind*.
     Navázat jen ty databáze firem, které mají být skutečně přístupné — SAP
     označuje obojí výslovně jako bezpečnostně kritické. **Poznamenat si
     přidělené Company-ID**, potřebujeme ho pro konfiguraci.
   - **Změny:** registrovaného IdP **nelze** upravovat — smazat a založit znovu
     (*Delete* na stejné cestě).
5. **Síť:** z MCP serveru musí být dostupný port **50000** (Service Layer) a —
   u **varianty A** — **40020** (Authentication Server) a **40000** (SLD, pro
   CompanyID). **Prohlížeč uživatelů** musí dosáhnout na přihlašovací stránku:
   u varianty A port **40020**, u varianty B váš vlastní provider. **Varianta
   B port 40000 nepotřebuje**, pokud jsou nakonfigurovaná CompanyID (viz
   oddíl 3). U hostovaných systémů případně požádat providera o uvolnění.

## 3. `.env` — dvě cesty

Komentáře vždy na **vlastní řádek** (ne za hodnotu).

### Doporučeno: automatické zjištění endpointů (SLD discovery)

Pomocí `SAP_SLD_URL` si server při startu sám natáhne endpointy Keycloaku
(token, authorize, logout, podpisové klíče) — méně překlepů:

```ini
SAP_AUTH_MODE=bearer

# Veřejná URL této MCP instance — prohlížeč se po přihlášení přesměruje na
# <SAP_PUBLIC_URL>/callback. POVINNÉ při bearer.
SAP_PUBLIC_URL=https://mcp.vas-host.cz

# SLD — z něj se automaticky zjistí všechny endpointy Keycloaku.
SAP_SLD_URL=https://<sap-host>:40000

SAP_IDP_CLIENT_ID=<Client-ID z Extension Single Sign-On Manageru>
SAP_IDP_CLIENT_SECRET=<Client-Secret>

# pokud je Authentication Server a/nebo Service Layer self-signed
SAP_ALLOW_SELF_SIGNED_CERT=true
```

### Alternativa: zadat endpointy ručně

Pokud port 40000 není dostupný nebo chcete hodnoty zadat napevno:

```ini
SAP_AUTH_MODE=bearer
SAP_PUBLIC_URL=https://mcp.vas-host.cz
SAP_IDP_TOKEN_URL=https://<sap-host>:40020/auth/realms/sapb1/protocol/openid-connect/token
SAP_IDP_CLIENT_ID=<Client-ID>
SAP_IDP_CLIENT_SECRET=<Client-Secret>

# volitelné — jinak se odvodí z token URL
# (…/openid-connect/token → …/openid-connect/auth). POVINNÉ, pokud váš Identity
# Provider není Keycloak (např. Microsoft Entra ID, Okta).
# SAP_IDP_AUTHORIZE_URL=https://<sap-host>:40020/auth/realms/sapb1/protocol/openid-connect/auth

# volitelné — výchozí 'openid' (jako oficiální SAP příklady). Rozšířené scopy
# nastavit jen, pokud jsou klientovi přiřazené, jinak přihlášení selže.
# SAP_IDP_SCOPE=openid

# SLD pro zjištění CompanyID (X-b1-companyid) — při bearer nutné
SAP_SLD_URL=https://<sap-host>:40000
```

Poznámky:
- Proměnné se dřív jmenovaly `SAP_KEYCLOAK_*`; tento zápis stále funguje. Nové
  a preferované je `SAP_IDP_*` (stejný význam).
- `SAP_IDP_TOKEN_URL` musí být **`https://`**; host/port (typicky `40020`) a
  realm (`sapb1`) přizpůsobit podle vaší konfigurace SLD.
- `SAP_PUBLIC_URL` je při `bearer` **povinná**. Lokálně je přípustná i
  `http://127.0.0.1:8000` (loopback) — pak u klienta uložit `…/callback`
  přesně s touto adresou.
- `SAP_ALLOW_SELF_SIGNED_CERT` platí **společně** pro Service Layer i Keycloak
  a je myšlená jako **dočasné řešení** — ve střednědobém horizontu nastavit
  platný certifikát a hodnotu vrátit na `false`.
- Pro **variantu B** platí samostatný blok — viz níže.

### Varianta B: vlastní Identity Provider (Principal Propagation)

Tady tokeny vydává **váš** Identity Provider. Oproti variantě A jsou jiné dvě věci:

1. `SAP_IDP_TOKEN_URL` míří na **vašeho** providera, ne na SAP Authentication
   Server.
2. **CompanyID se konfiguruje napevno**, místo aby se za běhu dotazovala u
   SLD. Poznamenali jste si ji při navázání company (krok 4 výše,
   *Principal Propagation → Tenants*). Tento provozní režim tak port **40000
   vůbec nepotřebuje** — SLD se už neoslovuje.

```ini
SAP_AUTH_MODE=bearer
SAP_PUBLIC_URL=https://mcp.vas-host.cz

# Váš vlastní Identity Provider
SAP_IDP_TOKEN_URL=https://idp.vas-host.cz/realms/<realm>/protocol/openid-connect/token
SAP_IDP_CLIENT_ID=<Client-ID u VAŠEHO providera>
SAP_IDP_CLIENT_SECRET=<Client-Secret>

# POVINNÉ u varianty B: CompanyID poznamenaná při tenant-bindingu, pro
# každou databázi firmy. Formát: DB:ID, více oddělených čárkou.
SAP_COMPANY_IDS=SBO_PROD:1,SBO_TEST:2

# SAP_SLD_URL není potřeba — CompanyID je už uvedená výše.
```

K `SAP_COMPANY_IDS`:
- Musí být uvedené **všechny** databáze ze `SAP_DATABASES`. Chybí-li některá,
  server se nespustí a chybějící pojmenuje — nikdo tak nespadne nepozorovaně
  zpátky na dotaz do SLD.
- Hodnota se posílá beze změny jako `X-b1-companyid`. Převezměte ji přesně
  tak, jak ji zobrazuje Extension Single Sign-On Manager.
- Pokud váš provider **není** Keycloak (např. Entra ID, Okta), nastavte navíc
  `SAP_IDP_AUTHORIZE_URL` — automatické odvození funguje jen u cest Keycloaku.
- Od **FP 2602** Service Layer kontroluje claim `audience`. U varianty B musí
  **váš** provider zapsat Client-ID Service Layeru do audience (viz poznámka
  pod tabulkou v oddílu 7).

## 4. Reverse proxy

Pokud MCP běží za reverse proxy (terminace HTTPS), musí se na MCP port
přeposílat **tyto cesty** — a to s **anonymním** přístupem (žádná Windows
autentizace, jinak spojení selže s 401):

```
/mcp   /login   /api/login   /callback   /backchannel-logout
```

Navíc: vypnout bufferování odpovědi pro `/mcp` a nastavit velkorysé timeouty
(spojení zůstává otevřené). Detaily: [installation-zentral.cs.md](installation-zentral.cs.md).

## 5. Průběh pro uživatele

**Důležité předem:** v SSO režimech MCP **nikdy nepřebírá přihlašovací
údaje** — přihlášení vždy probíhá u Identity Providera. Stránka na `/login`
proto zobrazuje **jen výběr databáze** (při více databázích firem), žádná
pole pro uživatele nebo heslo.

Průběh krok za krokem:

1. **Zavolat `connect`** — uživatel v chatu prostě řekne *„Připoj mě k SAP"*.
   Dostane zpátky **přihlašovací odkaz**.
   - *Více databází:* odkaz nejdřív otevře **výběr databáze**
     („Vybrat databázi" s rozbalovacím seznamem a „Pokračovat na přihlášení")
     a po výběru automaticky přesměruje na přihlašovací stránku Identity Providera.
   - *Jedna databáze* nebo databáze už zmíněná v chatu (*„… databáze
     BRAGI_TEST"*): odkaz vede **přímo** na přihlašovací stránku Identity
     Providera — výběr odpadá.
2. **Přihlásit se u Identity Providera.** Výběrová stránka tam přesměruje přes
   JavaScript (žádná další cesta na reverse proxy není potřeba — zůstává se u
   `/login`). Tam se uživatel přihlásí svou **e-mailovou adresou** a heslem
   IdP (funguje stávající SSO session, MFA i federované přihlášení).
   Přihlašovací údaje **nikdy** nejdou k MCP ani do chatu.
3. **Zpátky v chatu** jednou zavolat `connect(ticket="…")` (resp. napsat
   „hotovo", klient to zařídí) — připojeno.

**Více databází:** jedna relace je vždy připojená k **jedné** databázi.
Přepnutí: *„Odpoj se"* (`disconnect`), pak znovu připojit — na výběrové
stránce zvolit jinou databázi (nebo ji rovnou zmínit v chatu); SSO session v
prohlížeči většinou pořád trvá, druhé přihlášení je pak jen jeden klik.
Zda uživatel smí danou databázi vůbec používat, rozhoduje SAP: u varianty A
musí jeho **navázání uživatele** danou databázi zahrnovat (jinak: „Keine
SLD-Company-Bindung für … gefunden" / nenalezeno navázání SLD company pro … —
náprava: rozšířit navázání v SLD o tuto company); u varianty B Service Layer
přístup odmítne, pokud oprávnění chybí. V rámci připojení vždy platí
**vlastní SAP oprávnění** uživatele.

**Odhlášení:** `disconnect` ukončí session a odvolá token u Identity
Providera. Je-li uložená URL back-channel logoutu (krok 3), ukončí centrální
odhlášení u Identity Providera automaticky i MCP session.

## 6. Testování

Restartovat server, zavolat `connect`, projít odkaz, přihlásit se,
`connect(ticket=…)`, pak zkusit příkladový dotaz. Funguje-li to, je řetězec
správně. Pro první test **bez** reverse proxy: nastavit
`SAP_PUBLIC_URL=http://127.0.0.1:8000`, tuto callback adresu uložit u klienta
a test v prohlížeči provést přímo na serveru (vzdálená plocha).

## 7. Řešení problémů

| Chyba / příznak | Příčina / řešení |
|---|---|
| `invalid_client` | špatné `SAP_IDP_CLIENT_ID` / `SAP_IDP_CLIENT_SECRET`, nebo klient (už) není registrovaný |
| Redirect odmítnut (`invalid_redirect_uri`) | volaná callback URL není uložená u klienta — porovnat protokol, host, port a cestu (Extension Single Sign-On Manager) |
| `invalid_scope` | požadovaný scope není klientovi přiřazený → vrátit `SAP_IDP_SCOPE` na `openid` nebo scope klientovi přiřadit |
| Chyba realmu/404 | špatná `SAP_IDP_TOKEN_URL` (zkontrolovat host/port/realm) — nebo jednoduše použít `SAP_SLD_URL` |
| „authorize_url nelze odvodit" při startu | token URL není cesta Keycloaku (např. Entra ID/Okta) → nastavit explicitně `SAP_IDP_AUTHORIZE_URL` |
| Chyba certifikátu | self-signed → dočasně `SAP_ALLOW_SELF_SIGNED_CERT=true` |
| Přihlášení projde, přístup ale 401 | token odmítá Service Layer: zkontrolovat Feature Package (varianta A od FP 2208), resp. registraci klienta v Extension Single Sign-On Manageru. **Od FP 2602** Service Layer navíc kontroluje claim `audience` (viz poznámka pod tabulkou) |
| Přihlášení projde, přístup je ale odmítnutý (401/403) | uživatel není navázaný (nebo je navázaný na jinou databázi firmy) → zkontrolovat SLD *Users* (příručka pro neautentizovaný případ uvádí 401, kap. 6.8.2) |
| „Nenalezeno navázání SLD company" | chybí navázání uživatele, nebo port 40000 není z MCP serveru dostupný |
| `/login` zobrazuje jen výběr databáze, žádná přihlašovací pole | správně — v SSO režimech se uživatel přihlašuje u Identity Providera, nikdy u MCP; stránka jen volí firmu |
| Klik na „Pokračovat na přihlášení" nic neudělá | zkontrolovat konzoli prohlížeče. Pokud je tam hlášení `Content Security Policy` k `form-action`, běží verze starší než **3.2.1** — aktualizujte. Jinak: je v prohlížeči zapnutý JavaScript? Stránka ho potřebuje pro přesměrování (bez JS se uplatní fallback, který u některých Identity Providerů naráží na CSP) |
| Start selže: „SAP_COMPANY_IDS must cover every CompanyDB" | varianta B: některá databáze ze `SAP_DATABASES` nemá CompanyID. Doplnit — nebo `SAP_COMPANY_IDS` úplně odstranit, má-li ji zjišťovat SLD |
| Start selže: „SAP_COMPANY_IDS names unknown CompanyDB" | překlep v názvu databáze — musí přesně odpovídat některé položce ze `SAP_DATABASES` |
| Přístup odmítnut, přestože je CompanyID nakonfigurovaná | zkontrolovat hodnotu proti tenant-bindingu v Extension Single Sign-On Manageru; posílá se beze změny jako `X-b1-companyid` |

### Poznámka ke kontrole audience (od SAP B1 10.0 FP 2602)

Od FP 2602 Service Layer při přístupu s tokenem kontroluje claim `audience`
(podle SAP příručky zavedeno nejdřív pro single-page aplikace). Pokud přístup
i přes úspěšné přihlášení skončí **401**, je potřeba do konfigurace klienta v
Keycloaku doplnit jako hodnotu audience **Client-ID Service Layeru**
(audience mapper). Příručka tento mapper výslovně označuje jako **workaround
pro verze před FP 2608** — od FP 2608 by mělo přiřazení fungovat bez ručního
zásahu.
Pro testovací/vývojové účely lze kontrolu vypnout v konfiguraci Service
Layeru `b1s.conf` pomocí `EnableAudienceValidation=false` (poté restartovat
službu Service Layeru) — pro produkční provoz je správnou cestou audience
mapper.
Detaily v SAP příručce „Identity and Authentication Management in SAP
Business One", sekce „Configuring Audience in Keycloak".

### Uvedení do provozu

Kroky na straně SAP (oddíl 2) vyžadují práva v SLD a v Extension Single
Sign-On Manageru. Prvotní uvedení do provozu doprovázíme společně — počítejte
prosím s krátkým časovým oknem s vašimi SAP administrátory.

Další pomoc: **support@versino.de**.
