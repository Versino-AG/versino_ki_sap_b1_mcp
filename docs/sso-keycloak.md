# SSO-Anbindung über Keycloak — OIDC Authorization Code (Kurzanleitung)

Nur nötig, wenn euer SAP B1 den **Authentication Server (SLD/IAM mit Keycloak)** nutzt und
ihr **echtes SSO** wollt. Der Ablauf:
**`connect` → Browser-Login bei Keycloak → JWT → Service-Layer-`/Login` → fertig.**
Meldet ihr euch klassisch direkt am Service Layer an, bleibt `SAP_AUTH_MODE=basic` (→
[konfiguration.md](konfiguration.md)) — dieser Abschnitt ist dann irrelevant.

Der MCP nutzt den **OIDC Authorization-Code-Flow mit PKCE** und danach **Principal
Propagation**: jeder Service-Layer-Aufruf trägt `Authorization: Bearer <JWT>` +
`X-b1-companyid: <CompanyID>` (kein `/Login`, kein Cookie). Vorteile: bestehende
Keycloak-SSO-Session, MFA und Verbund-Logins (AD/SAML) greifen, und das **Passwort erreicht
den MCP nie**. Es wird **kein** „Direct Access Grant" (ROPC) am Client benötigt. Die
**CompanyID** für den Header liest der MCP automatisch aus der SLD (`CurrentUserInfo`).

## Voraussetzungen (SAP-/SLD-seitig)
- SAP B1 Authentication Server (Keycloak) mit dem Realm **`sapb1`**.
- Ein OAuth-Client für Web-SSO — genau das, was der **SSO Extension Manager** erzeugt
  (`b1-ext-web-…`, Client-ID + Secret). **Kein** Keycloak-Admin nötig.
- Am Client muss die **Redirect-URI** `<SAP_PUBLIC_URL>/callback` hinterlegt sein
  (im SSO Extension Manager eintragen). Ohne sie weist Keycloak den Redirect ab.

## `.env`
Kommentare jeweils in eine **eigene Zeile** (nicht hinter den Wert):
```ini
SAP_AUTH_MODE=oidc

# Öffentliche URL dieser MCP-Instanz — der Browser wird nach dem Login auf
# <SAP_PUBLIC_URL>/callback zurückgeleitet. PFLICHT bei oidc.
SAP_PUBLIC_URL=https://mcp.euer-host.de

SAP_KEYCLOAK_TOKEN_URL=https://<auth-host>:40020/auth/realms/sapb1/protocol/openid-connect/token
SAP_KEYCLOAK_CLIENT_ID=<Client-ID aus dem SSO Extension Manager>
SAP_KEYCLOAK_CLIENT_SECRET=<Client-Secret>

# optional — die Authorize-URL wird sonst aus der Token-URL abgeleitet
# (…/openid-connect/token → …/openid-connect/auth)
# SAP_KEYCLOAK_AUTHORIZE_URL=https://<auth-host>:40020/auth/realms/sapb1/protocol/openid-connect/auth

# optional — SLD für die CompanyID-Auflösung (X-b1-companyid). Wird sonst aus dem
# Keycloak-Host mit Port 40000 abgeleitet.
# SAP_SLD_URL=https://<auth-host>:40000

# optional — Default 'openid' (entspricht den offiziellen SAP-B1-Cloud-OIDC-Beispielen).
# Nur setzen, falls euer Client zusätzliche Scopes verlangt.
# SAP_KEYCLOAK_SCOPE=openid

# falls Auth-Server und/oder SL selbstsigniert
SAP_ALLOW_SELF_SIGNED_CERT=true
```
- `SAP_KEYCLOAK_TOKEN_URL` **muss `https://`** sein; Host/Port (typisch `40020`) und Realm
  (`sapb1`) gemäß eurer SLD-Konfiguration anpassen.
- `SAP_PUBLIC_URL` ist bei `oidc` **Pflicht** (Redirect-Ziel). Lokal ist auch
  `http://127.0.0.1:8000` zulässig (Loopback) — dann `.../callback` am Client hinterlegen.
- `SAP_ALLOW_SELF_SIGNED_CERT` gilt **gemeinsam** für Service Layer **und** Keycloak.

## Ablauf für Anwender
Im Client `connect` aufrufen → es kommt eine **Browser-Login-URL** zurück. Der Nutzer meldet
sich (einmalig; bestehende SSO-Session greift) im Browser bei Keycloak an. Danach im Client
`connect(ticket="…")` aufrufen — fertig. Zugangsdaten gehen **nur** an Keycloak, nie an den
MCP oder in den Chat.

## Testen
Server neu starten, `connect` aufrufen, der URL folgen, anmelden, `connect(ticket=…)`.
Klappt der Zugriff, ist die OIDC-Kette korrekt.

## Troubleshooting
| Fehler | Ursache / Lösung |
|---|---|
| `invalid_client` | falsche `SAP_KEYCLOAK_CLIENT_ID`/`..._SECRET` |
| `invalid_redirect_uri` / Redirect abgewiesen | `<SAP_PUBLIC_URL>/callback` ist am Client nicht als Redirect-URI hinterlegt (SSO Extension Manager) |
| Realm-/404-Fehler | falsche `SAP_KEYCLOAK_TOKEN_URL` (Host/Port/Realm prüfen) |
| Zertifikatsfehler | selbstsigniert → `SAP_ALLOW_SELF_SIGNED_CERT=true` |
| Login geht, aber kein SL-Zugriff | Token-Scope/Audience am Client prüfen (ggf. einen client-spezifischen Scope wie `B1.ServiceLayer` am Client hinterlegen) |

Weitere Hilfe: **support@versino.de**.
