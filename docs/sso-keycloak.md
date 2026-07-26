# SSO-Anbindung über Keycloak (OIDC) — Einrichtungsanleitung

Nur nötig, wenn euer SAP B1 den **Authentication Server (SLD/IAM, „Keycloak")** nutzt
und ihr **echtes SSO** wollt. Meldet ihr euch klassisch direkt am Service Layer an,
bleibt `SAP_AUTH_MODE=basic` (→ [konfiguration.md](konfiguration.md)) und dieses
Dokument ist irrelevant.

In beiden Varianten gilt: Der Nutzer meldet sich **im Browser direkt bei
Keycloak** an (Authorization Code + PKCE) — bestehende SSO-Session, MFA und
Verbund-Logins (AD/SAML) greifen, und das **Passwort erreicht den MCP nie**. Ein
„Direct Access Grant" (ROPC) wird am Client **nicht** benötigt.

## 1. Der SSO-Modus `bearer` — zwei Ausbaustufen

SSO läuft immer über `SAP_AUTH_MODE=bearer`: Nach dem Browser-Login geht das
Token bei **jedem** Service-Layer-Aufruf mit (`Authorization: Bearer` +
`X-b1-companyid`). Genau diesen Weg beschreibt der SAP-Leitfaden „Identity and
Authentication Management in SAP Business One" (Kap. 6.5.8 und 6.9).

Unterschieden wird, **wer die Tokens ausstellt**:

| Variante | Token-Aussteller | Mindestversion | Wann |
|---|---|---|---|
| **A — SAP-Authentication-Server** (der Regelfall) | der `sapb1`-Realm eures SAP B1; Client aus dem Extension Single Sign-On Manager (`b1-ext-…`) | SAP B1 10.0 **FP 2208** | immer, wenn die Anmeldung über den SAP-Authentication-Server läuft — **auch dann, wenn dahinter ein externer Provider steht** (AD, Entra ID, Okta, SAP IAS): das SLD legt für ihn eine Broker-Adresse im `sapb1`-Realm an (`…/auth/realms/sapb1/broker/b1-<Alias>/endpoint` — im Leitfaden für AD FS auf S. 20, Entra ID S. 31, Okta S. 39), der Anwender meldet sich beim externen Provider an, das Token stellt aber der `sapb1`-Realm aus |
| **B — eigener Identity Provider stellt die Tokens selbst aus** | eure eigene IdP-Instanz, im Extension SSO Manager als vertrauenswürdiger Provider registriert; SAP nennt das **Principal Propagation** | siehe Hinweis unten | nur, wenn eure Anwendungslandschaft die Tokens selbst ausstellen soll, statt sie beim SAP-Authentication-Server zu holen |

**In der Praxis ist fast immer Variante A richtig.** Die Versionstabelle des
SAP-Leitfadens (Kap. 6.9) führt selbst für das Szenario „ein oder mehrere
externe Identity Provider sind aktiv" den Weg „Extension-Client-ID registrieren
und mit dem Access Token verbinden" **ab FP 2208** auf: Ist euer AD/Entra/Okta
im SLD als Identity Provider aktiviert, meldet sich der Anwender dort an — das
**Token für den Service Layer kommt aber weiterhin vom
SAP-Authentication-Server**. Das ist Variante A.

> **Hinweis zu Variante B:** Der IAM-Leitfaden beschreibt Principal Propagation
> nur konzeptionell (Kap. 6.10) und verweist auf das eigene SAP-Dokument
> **„Principal Propagation for SAP Business One" (Security Guide, Version 1.0 –
> 2025-03-25)**. Dort stehen die Einrichtungsschritte (siehe Schritt 4 unten).
> Eine **Mindestversion nennt auch dieses Dokument nicht** — die verbreitete
> Angabe „ab FP 2411" ist damit in keiner der drei SAP-Quellen belegt. Klärt euer
> Feature Package deshalb vorab mit uns ab.

Beide Varianten brauchen SLD-Zugriff (Port 40000) für die Auflösung der
CompanyID.

> **Wichtig — was sich mit aktivem IAM für `basic` ändert:** Gebundene Benutzer
> melden sich nicht mehr mit ihrem **B1-Benutzercode**, sondern mit den
> **Zugangsdaten des Authentication Servers** an. Der Leitfaden formuliert das mit
> einer Bedingung, die man leicht überliest (Kap. 6.1, S. 140): *„If you **only**
> enabled the identity provider SAP Business One Authentication Server, the DIAPI
> and Service Layer login interfaces allow you to use the SAP Business One
> Authentication Server user and password to login."*
>
> Daraus folgt:
> - **Nur** der Authentication Server als Identity Provider aktiv → `basic` bleibt
>   nutzbar. Der Benutzername ist der Benutzercode des Authentication Servers;
>   voreingestellt ist er identisch mit dem IdP-Benutzernamen (Kap. 3.2.4).
> - **Zusätzlich ein externer** Identity Provider aktiv (AD FS, Entra ID, Okta,
>   SAP IAS) → für diese Benutzer ist `bearer` der Weg. Die Versionstabelle in
>   Kap. 6.9 führt für dieses Szenario ab FP 2305 **nur noch** den
>   Access-Token-Weg auf, die Benutzer/Kennwort-Option fällt dort weg. Solche
>   Benutzer melden sich grundsätzlich mit ihrer **E-Mail-Adresse** an — bei
>   einem externen Provider ist das die einzig zulässige Kennung (Kap. 3.2.1),
>   und die E-Mail-Domäne steuert, auf welche Anmeldeseite geleitet wird
>   (Kap. 3.1.1 und 5.1.5).
> - ⚠️ **Offener Punkt bei gemischter Landschaft:** Der Leitfaden erlaubt den
>   klassischen Login ausdrücklich nur, wenn **ausschließlich** der
>   Authentication Server aktiv ist. Sind beide aktiv, ist für **keinen**
>   Benutzer belegt, dass `basic` weiter funktioniert — auch nicht für die am
>   Authentication Server gebundenen. In dieser Konstellation stimmen wir das
>   Vorgehen gemeinsam ab.
> - Benutzer mit **Zwei-Faktor-Authentifizierung** können den klassischen Weg
>   ebenfalls nicht nutzen. Der Leitfaden belegt das für den
>   DTW-Kommandomodus (Kap. 7) und führt es für den Service Layer nicht
>   ausdrücklich auf.

## 2. Voraussetzungen SAP-seitig — in dieser Reihenfolge

Ausführende Rolle: **B1-Administrator / Landscape-Administrator**. Bei gehosteten
Systemen (z. B. Cloudiax) ggf. gemeinsam mit dem Provider (SLD-Zugang, Ports).

> **Zwei Verwaltungsoberflächen — prüft zuerst, welche ihr habt.** SAP
> dokumentiert IAM in zwei Leitfäden, und die Wege unterscheiden sich:
> - **On-Premise / gehostet:** *SLD Control Center*, typisch
>   `https://<sap-host>:40000/ControlCenter`. Die Schritte unten sind so
>   beschrieben.
> - **SAP Business One Cloud:** *Cloud Control Center*. Dort heißen die Bereiche
>   *System Configuration → Identity Providers* bzw. *Customer Management →
>   Customers → Customer Details → Identity Providers*, und die SLD- und
>   Authentication-Service-Adressen werden dort konfiguriert — sie sind also nicht
>   zwangsläufig `<host>:40000` / `<host>:40020`.
>   ⚠️ Zusätzlich muss dort **einmalig** *System Configuration → Global Settings →
>   **Enable Third Party Identity Provider** = On* gesetzt werden; erst danach
>   erscheint der *Add*-Button für Identity Provider. Voraussetzung dafür ist ein
>   registriertes Software-Repository für **FP 2405 oder höher**.
>
> Alles Übrige (Client-Registrierung im Extension Single Sign-On Manager,
> Benutzerbindung, Token-Transport) ist in beiden Varianten identisch.

1. **IAM prüfen** — Control Center (siehe Kasten), Reiter *Identity Providers*:
   Mindestens ein Identity Provider muss **Active** sein (SAP Business One
   Authentication Server, Active Directory Domain Services oder ein externer
   OIDC-Provider).
   ⚠️ Vor dem Aktivieren **alle** Benutzer binden (Schritt 2) — danach melden sich
   gebundene Benutzer mit den Zugangsdaten des Identity Providers an, nicht mehr
   mit dem B1-Benutzercode (siehe Kasten oben).
2. **Benutzer binden** — Control Center, Reiter *Users*: Jeden Anwender, der den
   MCP nutzen soll, markieren → **Bind** → Server, Company-Datenbank(en) und
   B1-Benutzercode zuordnen. Ohne diese Bindung schlägt die Anmeldung fehl.
3. **OAuth-Client anlegen** — **SAP Business One Extension Single Sign-On Manager**
   (wird mit dem Extension Manager installiert) → *Extensions* → **Register**:
   - **Client Type: „Web App"** (liefert Client-ID *und* Client-Secret).
   - **Redirect URI:** `<SAP_PUBLIC_URL>/callback`
     (z. B. `https://mcp.euer-host.de/callback`; für einen lokalen Test zusätzlich
     `http://127.0.0.1:8000/callback`). Der Leitfaden erlaubt hier auch
     **Wildcards** (z. B. `https://mcp.euer-host.de/*`) — wir empfehlen dennoch
     die vollständige URL, damit nur der vorgesehene Rückweg gilt.
   - ⚠️ Das **Client-Secret wird nur einmal angezeigt** — sofort sichern.
   - Optional (für zentrales Abmelden, ab FP 2508): zusätzlich die Back-Channel-Logout-URL
     `<SAP_PUBLIC_URL>/backchannel-logout` hinterlegen.
4. **Nur bei Variante B (eigener Identity Provider stellt die Tokens aus).**
   Für Variante A entfällt dieser Schritt komplett. Quelle: SAP Security Guide
   „Principal Propagation for SAP Business One" (Version 1.0), Kap. 1.2/1.3.
   - **Voraussetzungen:** Euer Identity Provider ist im SLD als Drittanbieter-IdP
     konfiguriert; er stellt Access Tokens im **JWT-Format** aus und hat eine
     **eigene Discovery-URL**; die Benutzeridentität wird über die
     **E-Mail-Adresse** geführt, die **landschaftsweit eindeutig** sein muss
     („one e-mail address exclusively represents one user only").
   - **IdP registrieren:** *Extension Single Sign-On Manager → Principal
     Propagation → Identity Providers → Register*. Einzutragen sind: **Name**,
     **Discovery Endpoint** (`…/.well-known/openid-configuration`) und **Identity
     Claim Name** — über letzteren liest SAP die E-Mail aus dem Token. Der Claim
     muss im Token enthalten sein und die korrekte E-Mail des Anwenders tragen.
   - **Company binden:** *Principal Propagation → Tenants → Bind*. Nur die
     Company-Datenbanken binden, die wirklich zugänglich sein sollen — SAP nennt
     beides ausdrücklich sicherheitskritisch. **Die dort ausgegebene Company-ID
     notieren**, wir brauchen sie für die Konfiguration.
   - **Änderungen:** Ein registrierter IdP lässt sich **nicht** bearbeiten —
     löschen und neu anlegen (*Delete* im selben Pfad).
5. **Netzwerk:** Vom MCP-Server aus erreichbar sein müssen Port **50000**
   (Service Layer) und — bei **Variante A** — **40020** (Authentication
   Server) sowie **40000** (SLD, für die CompanyID). Der **Browser der
   Anwender** muss die Anmeldeseite erreichen: bei Variante A Port **40020**,
   bei Variante B euren eigenen Provider. **Variante B braucht Port 40000
   nicht**, wenn die CompanyIDs konfiguriert sind (siehe Abschnitt 3). Bei
   gehosteten Systemen ggf. Freigaben beim Provider beantragen.

## 3. `.env` — zwei Wege

Kommentare jeweils in eine **eigene Zeile** (nicht hinter den Wert).

### Empfohlen: automatische Endpoint-Ermittlung (SLD-Discovery)

Mit `SAP_SLD_URL` holt sich der Server die Keycloak-Endpunkte (Token, Authorize,
Logout, Signaturschlüssel) beim Start selbst — weniger Tippfehler:

```ini
SAP_AUTH_MODE=bearer

# Öffentliche URL dieser MCP-Instanz — der Browser wird nach dem Login auf
# <SAP_PUBLIC_URL>/callback zurückgeleitet. PFLICHT bei bearer.
SAP_PUBLIC_URL=https://mcp.euer-host.de

# SLD — daraus werden alle Keycloak-Endpunkte automatisch ermittelt.
SAP_SLD_URL=https://<sap-host>:40000

SAP_IDP_CLIENT_ID=<Client-ID aus dem Extension Single Sign-On Manager>
SAP_IDP_CLIENT_SECRET=<Client-Secret>

# falls Authentication Server und/oder Service Layer selbstsigniert sind
SAP_ALLOW_SELF_SIGNED_CERT=true
```

### Alternative: Endpunkte selbst angeben

Wenn Port 40000 nicht erreichbar ist oder ihr die Werte fest eintragen wollt:

```ini
SAP_AUTH_MODE=bearer
SAP_PUBLIC_URL=https://mcp.euer-host.de
SAP_IDP_TOKEN_URL=https://<sap-host>:40020/auth/realms/sapb1/protocol/openid-connect/token
SAP_IDP_CLIENT_ID=<Client-ID>
SAP_IDP_CLIENT_SECRET=<Client-Secret>

# optional — wird sonst aus der Token-URL abgeleitet
# (…/openid-connect/token → …/openid-connect/auth). PFLICHT, wenn euer Identity
# Provider kein Keycloak ist (z. B. Microsoft Entra ID, Okta).
# SAP_IDP_AUTHORIZE_URL=https://<sap-host>:40020/auth/realms/sapb1/protocol/openid-connect/auth

# optional — Default 'openid' (wie die offiziellen SAP-Beispiele). Erweiterte Scopes
# nur setzen, wenn sie dem Client zugewiesen sind, sonst schlägt der Login fehl.
# SAP_IDP_SCOPE=openid

# SLD für die CompanyID-Auflösung (X-b1-companyid) — bei bearer erforderlich
SAP_SLD_URL=https://<sap-host>:40000
```

Hinweise:
- Die Variablen hießen früher `SAP_KEYCLOAK_*`; diese Schreibweise funktioniert
  weiterhin. Neu und bevorzugt ist `SAP_IDP_*` (identische Bedeutung).
- `SAP_IDP_TOKEN_URL` muss **`https://`** sein; Host/Port (typisch `40020`) und
  Realm (`sapb1`) gemäß eurer SLD-Konfiguration anpassen.
- `SAP_PUBLIC_URL` ist bei `bearer` **Pflicht**. Lokal ist auch
  `http://127.0.0.1:8000` zulässig (Loopback) — dann `…/callback` mit genau
  dieser Adresse am Client hinterlegen.
- `SAP_ALLOW_SELF_SIGNED_CERT` gilt **gemeinsam** für Service Layer und Keycloak
  und ist als **Übergangslösung** gedacht — mittelfristig ein gültiges Zertifikat
  einrichten und den Wert auf `false` setzen.
- Für **Variante B** gilt ein eigener Block — siehe unten.

### Variante B: eigener Identity Provider (Principal Propagation)

Hier stellt **euer** Identity Provider die Tokens aus. Zwei Dinge sind anders als
bei Variante A:

1. `SAP_IDP_TOKEN_URL` zeigt auf **euren** Provider, nicht auf den
   SAP-Authentication-Server.
2. Die **CompanyID wird fest konfiguriert**, statt sie zur Laufzeit bei der SLD
   zu erfragen. Ihr habt sie beim Binden der Company notiert (Schritt 4 oben,
   *Principal Propagation → Tenants*). Damit braucht diese Betriebsart **Port
   40000 überhaupt nicht** — die SLD wird nicht mehr angesprochen.

```ini
SAP_AUTH_MODE=bearer
SAP_PUBLIC_URL=https://mcp.euer-host.de

# Euer eigener Identity Provider
SAP_IDP_TOKEN_URL=https://idp.euer-host.de/realms/<realm>/protocol/openid-connect/token
SAP_IDP_CLIENT_ID=<Client-ID in EUREM Provider>
SAP_IDP_CLIENT_SECRET=<Client-Secret>

# PFLICHT bei Variante B: die beim Tenant-Binding notierte CompanyID je
# Company-Datenbank. Format: DB:ID, mehrere komma-getrennt.
SAP_COMPANY_IDS=SBO_PROD:1,SBO_TEST:2

# Kein SAP_SLD_URL nötig — die CompanyID steht ja oben.
```

Zu `SAP_COMPANY_IDS`:
- Es müssen **alle** Datenbanken aus `SAP_DATABASES` aufgeführt sein. Fehlt eine,
  startet der Server nicht und nennt die fehlende — so fällt niemand unbemerkt auf
  die SLD-Abfrage zurück.
- Der Wert wird unverändert als `X-b1-companyid` gesendet. Übernimmt ihn genau so,
  wie der Extension Single Sign-On Manager ihn anzeigt.
- Ist euer Provider **kein** Keycloak (z. B. Entra ID, Okta), setzt zusätzlich
  `SAP_IDP_AUTHORIZE_URL` — die Ableitung funktioniert nur bei Keycloak-Pfaden.
- Ab **FP 2602** prüft der Service Layer den `audience`-Claim. Bei Variante B muss
  **euer** Provider die Service-Layer-Client-ID in die Audience schreiben (siehe
  Hinweis in Abschnitt 7).

## 4. Reverse Proxy

Läuft der MCP hinter einem Reverse Proxy (HTTPS-Terminierung), müssen **diese Pfade**
an den MCP-Port weitergeleitet werden — und zwar mit **anonymem** Zugriff (keine
Windows-Authentifizierung, sonst scheitert die Verbindung mit einem 401):

```
/mcp   /login   /api/login   /callback   /backchannel-logout
```

Zusätzlich: Antwort-Pufferung für `/mcp` abschalten und großzügige Timeouts setzen
(die Verbindung bleibt offen). Details: [installation-zentral.md](installation-zentral.md).

## 5. Ablauf für Anwender

**Wichtig vorab:** In den SSO-Modi nimmt der MCP **niemals Zugangsdaten**
entgegen — die Anmeldung passiert immer beim Identity Provider. Die Seite unter
`/login` zeigt deshalb **nur eine Datenbank-Auswahl** (bei mehreren
Company-Datenbanken), keine Benutzer- oder Passwortfelder.

Der Ablauf, Schritt für Schritt:

1. **`connect` aufrufen** — der Anwender sagt im Chat einfach *„Verbinde mich
   mit SAP"*. Er bekommt einen **Anmelde-Link** zurück.
   - *Mehrere Datenbanken:* Der Link öffnet zuerst die **Datenbank-Auswahl**
     („Datenbank wählen" mit Dropdown und „Weiter zur Anmeldung") und leitet
     nach der Wahl automatisch zur Anmeldeseite des Identity Providers weiter.
   - *Eine Datenbank* oder Datenbank schon im Chat genannt (*„… Datenbank
     BRAGI_TEST"*): Der Link führt **direkt** zur Anmeldeseite des Identity
     Providers — die Auswahl entfällt.
2. **Beim Identity Provider anmelden.** Die Auswahlseite leitet dazu per
   JavaScript weiter (kein zusätzlicher Pfad am Reverse Proxy nötig — es
   bleibt bei `/login`). Dort meldet sich der Anwender mit
   seiner **E-Mail-Adresse** und dem IdP-Kennwort an (eine bestehende
   SSO-Session greift; MFA und Verbund-Logins funktionieren). Zugangsdaten
   gehen **nie** an den MCP oder in den Chat.
3. **Zurück im Chat** einmal `connect(ticket="…")` aufrufen (bzw. „fertig"
   schreiben, der Client erledigt das) — verbunden.

**Mehrere Datenbanken:** Eine Sitzung ist immer mit **einer** Datenbank
verbunden. Wechseln: *„Trenne die Verbindung"* (`disconnect`), dann neu
verbinden — auf der Auswahlseite die andere Datenbank wählen (oder sie gleich
im Chat nennen); die SSO-Session im Browser besteht meist noch, der zweite
Login ist dann nur ein Klick. Ob ein Anwender eine Datenbank überhaupt
nutzen darf, entscheidet SAP: Bei Variante A muss sein **User-Binding** die
gewählte Datenbank umfassen (sonst: „Keine SLD-Company-Bindung für … gefunden" —
Abhilfe: Binding im SLD um diese Company erweitern); bei Variante B lehnt der
Service Layer den Zugriff ab, wenn die Berechtigung fehlt. Innerhalb der
Verbindung gelten immer die **eigenen SAP-Rechte** des Anwenders.

**Abmelden:** `disconnect` beendet die Session und widerruft das Token beim Identity
Provider. Ist die Back-Channel-Logout-URL hinterlegt (Schritt 3), beendet auch ein
zentrales Abmelden am Identity Provider die MCP-Sitzungen automatisch.

## 6. Testen

Server neu starten, `connect` aufrufen, der URL folgen, anmelden,
`connect(ticket=…)`, dann eine Beispielabfrage. Klappt das, ist die Kette korrekt.
Für einen ersten Test **ohne** Reverse Proxy: `SAP_PUBLIC_URL=http://127.0.0.1:8000`
setzen, diese Callback-Adresse am Client hinterlegen und den Browser-Test direkt auf
dem Server (Remotedesktop) durchführen.

## 7. Troubleshooting

| Fehler / Symptom | Ursache / Lösung |
|---|---|
| `invalid_client` | falsche `SAP_IDP_CLIENT_ID` / `SAP_IDP_CLIENT_SECRET`, oder Client nicht (mehr) registriert |
| Redirect abgewiesen (`invalid_redirect_uri`) | Die aufgerufene Callback-URL ist am Client nicht hinterlegt — Protokoll, Host, Port und Pfad vergleichen (Extension Single Sign-On Manager) |
| `invalid_scope` | angeforderter Scope ist dem Client nicht zugewiesen → `SAP_IDP_SCOPE` auf `openid` zurücksetzen oder den Scope am Client zuweisen |
| Realm-/404-Fehler | falsche `SAP_IDP_TOKEN_URL` (Host/Port/Realm prüfen) — oder einfach `SAP_SLD_URL` nutzen |
| „authorize_url nicht ableitbar" beim Start | Token-URL ist kein Keycloak-Pfad (z. B. Entra ID/Okta) → `SAP_IDP_AUTHORIZE_URL` explizit setzen |
| Zertifikatsfehler | selbstsigniert → vorübergehend `SAP_ALLOW_SELF_SIGNED_CERT=true` |
| Login klappt, Zugriff aber 401 | Token wird vom Service Layer abgelehnt: Feature Package prüfen (Variante A ab FP 2208) bzw. die Client-Registrierung im Extension Single Sign-On Manager. **Ab FP 2602** prüft der Service Layer zusätzlich den `audience`-Claim (siehe Hinweis unter der Tabelle) |
| Login klappt, Zugriff wird aber abgewiesen (401/403) | Benutzer ist nicht (oder auf eine andere Company-DB) gebunden → SLD *Users* prüfen (der Leitfaden nennt für den nicht authentifizierten Fall 401, Kap. 6.8.2) |
| „Keine SLD-Company-Bindung gefunden" | Benutzerbindung fehlt, oder Port 40000 ist vom MCP-Server nicht erreichbar |
| `/login` zeigt nur eine Datenbank-Auswahl, keine Anmeldefelder | korrekt — in den SSO-Modi meldet man sich beim Identity Provider an, nie beim MCP; die Seite wählt nur die Firma |
| Klick auf „Weiter zur Anmeldung" bewirkt nichts | Browser-Konsole prüfen. Ist dort eine `Content Security Policy`-Meldung zu `form-action`, läuft eine Version vor **3.2.1** — bitte aktualisieren. Ansonsten: JavaScript im Browser aktiviert? Die Seite braucht es für die Weiterleitung (ohne JS greift ein Fallback, der bei manchen Identity Providern an der CSP scheitert) |
| Start bricht ab: „SAP_COMPANY_IDS must cover every CompanyDB" | Variante B: eine Datenbank aus `SAP_DATABASES` hat keine CompanyID. Ergänzen — oder `SAP_COMPANY_IDS` ganz entfernen, wenn die SLD sie ermitteln soll |
| Start bricht ab: „SAP_COMPANY_IDS names unknown CompanyDB" | Tippfehler im Datenbanknamen — er muss genau einem Eintrag aus `SAP_DATABASES` entsprechen |
| Zugriff wird abgewiesen, obwohl die CompanyID konfiguriert ist | Wert gegen das Tenant-Binding im Extension Single Sign-On Manager prüfen; er wird unverändert als `X-b1-companyid` gesendet |

### Hinweis zur Audience-Prüfung (ab SAP B1 10.0 FP 2602)

Ab FP 2602 prüft der Service Layer bei Token-Zugriffen den `audience`-Claim
(laut SAP-Leitfaden zunächst für Single-Page-Apps eingeführt). Führt der
Zugriff trotz erfolgreicher Anmeldung zu **401**, muss die **Service-Layer-Client-ID**
in der Keycloak-Client-Konfiguration als Audience-Wert ergänzt werden
(Audience-Mapper). Der Leitfaden bezeichnet diesen Mapper ausdrücklich als
**Workaround für Versionen vor FP 2608** — ab FP 2608 sollte die Zuordnung ohne
manuellen Eingriff greifen.
Für Test-/Entwicklungszwecke lässt sich die Prüfung in der Service-Layer-Konfiguration
`b1s.conf` mit `EnableAudienceValidation=false` abschalten (danach Service-Layer-Dienst
neu starten) — für den Produktivbetrieb ist der Audience-Mapper der richtige Weg.
Details im SAP-Leitfaden „Identity and Authentication
Management in SAP Business One", Abschnitt „Configuring Audience in Keycloak".

### Inbetriebnahme

Die SAP-seitigen Schritte (Abschnitt 2) erfordern Rechte im SLD und im Extension
Single Sign-On Manager. Die Erstinbetriebnahme begleiten wir gemeinsam — bitte
plant dafür ein kurzes Zeitfenster mit euren SAP-Administratoren ein.

Weitere Hilfe: **support@versino.de**.
