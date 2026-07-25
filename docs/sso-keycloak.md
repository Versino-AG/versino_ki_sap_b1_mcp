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
| **B — eigener Identity Provider stellt die Tokens selbst aus** | eure eigene IdP-Instanz; SAP nennt das **Principal Propagation** | siehe Hinweis unten | nur, wenn eure Anwendungslandschaft die Tokens selbst ausstellen soll, statt sie beim SAP-Authentication-Server zu holen |

**In der Praxis ist fast immer Variante A richtig.** Die Versionstabelle des
SAP-Leitfadens (Kap. 6.9) führt selbst für das Szenario „ein oder mehrere
externe Identity Provider sind aktiv" den Weg „Extension-Client-ID registrieren
und mit dem Access Token verbinden" **ab FP 2208** auf: Ist euer AD/Entra/Okta
im SLD als Identity Provider aktiviert, meldet sich der Anwender dort an — das
**Token für den Service Layer kommt aber weiterhin vom
SAP-Authentication-Server**. Das ist Variante A.

> **Hinweis zu Variante B:** Der Leitfaden beschreibt Principal Propagation nur
> konzeptionell (Kap. 6.10) und verweist für die Details auf ein separates
> SAP-Dokument („Principal Propagation for SAP Business One"). Er nennt dort
> **keine** Mindestversion und keine Einrichtungsschritte. Die verbreitete
> Angabe „ab FP 2411" ist über diesen Leitfaden **nicht** belegt. Wenn ihr
> Variante B braucht, klären wir Version und Vorgehen vorab mit SAP — planbar
> ist heute Variante A.

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
>   Authentication Server gebundenen. Wir testen das im Zweifel gemeinsam, statt
>   es zuzusagen.
> - Benutzer mit **Zwei-Faktor-Authentifizierung** können den klassischen Weg
>   ebenfalls nicht nutzen (belegt für den DTW-Kommandomodus, Kap. 7 — für den
>   Service Layer analog zu erwarten, von uns nicht live geprüft).

## 2. Voraussetzungen SAP-seitig — in dieser Reihenfolge

Ausführende Rolle: **B1-Administrator / Landscape-Administrator**. Bei gehosteten
Systemen (z. B. Cloudiax) ggf. gemeinsam mit dem Provider (SLD-Zugang, Ports).

1. **IAM prüfen** — SLD Control Center (`https://<sap-host>:40000/ControlCenter`),
   Reiter *Identity Providers*: Mindestens ein Identity Provider muss **Active**
   sein (SAP Business One Authentication Server, Active Directory Domain Services
   oder ein externer OIDC-Provider).
   ⚠️ Vor dem Aktivieren **alle** Benutzer binden (Schritt 2) — danach melden sich
   gebundene Benutzer mit den Zugangsdaten des Identity Providers an, nicht mehr
   mit dem B1-Benutzercode (siehe Kasten oben).
2. **Benutzer binden** — SLD Control Center, Reiter *Users*: Jeden Anwender, der den
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
4. **Nur bei Variante B (eigener Identity Provider stellt die Tokens aus):**
   Diese Einrichtung ist im SAP-Leitfaden **nicht** beschrieben — sie steht im
   separaten Dokument „Principal Propagation for SAP Business One“. Bitte vorab
   mit uns abstimmen; wir nennen euch dann die konkreten Schritte. Für Variante A
   entfällt dieser Schritt.
5. **Netzwerk:** Vom MCP-Server aus erreichbar sein müssen Port **50000**
   (Service Layer), **40020** (Authentication Server) und **40000** (SLD). Der **Browser der Anwender**
   muss Port **40020** erreichen (die Keycloak-Anmeldeseite). Bei gehosteten
   Systemen ggf. Freigaben beim Provider beantragen.

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
- Bei Variante B (eigener Identity Provider) zeigt `SAP_IDP_TOKEN_URL` auf euren
  Provider; `SAP_SLD_URL` muss dann zwingend gesetzt werden (es lässt sich nicht
  aus dem Provider-Host ableiten).

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

Im Client `connect` aufrufen → es kommt eine **Browser-Login-URL** zurück. Der Nutzer
meldet sich im Browser bei Keycloak an (bestehende SSO-Session greift). Danach im
Client einmal `connect(ticket="…")` aufrufen — fertig. Zugangsdaten gehen **nur** an
Keycloak, nie an den MCP oder in den Chat. Das Anmeldeformular des MCP (`/login`) ist
in den SSO-Modi bewusst deaktiviert.

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
| Anmeldeformular `/login` zeigt „Browser-SSO" | korrekt — in den SSO-Modi läuft die Anmeldung über `connect`, nicht über das Formular |

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

### Nicht abgedeckt

Der klassische Anmeldeweg (`basic`) und der SSO-Weg (`bearer`) sind gegen die
SAP-Vorgaben umgesetzt, aber SSO ist **noch nicht** auf einer
Produktivinstallation abgenommen — die Erstinbetriebnahme begleiten wir daher
gemeinsam. Bitte plant dafür ein kurzes Zeitfenster mit euren
SAP-Administratoren ein.

Weitere Hilfe: **support@versino.de**.
