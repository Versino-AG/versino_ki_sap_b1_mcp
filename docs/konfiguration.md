# Konfiguration (`.env`)

Der Server liest seine Instanz-Konfiguration aus einer `.env` neben dem Binary
(oder via `--env-file`). Hier stehen **keine Endnutzer-Zugangsdaten** — nur die
Instanz-Einstellungen. Jeder Nutzer meldet sich zur Laufzeit selbst an.

## Minimalkonfiguration
Kommentare gehören in eine **eigene Zeile** — **nie** hinter den Wert in dieselbe Zeile
(Inline-Kommentare können je nach Parser den Wert verfälschen).
```ini
# Service-Layer-URL eurer SAP-B1-Instanz
SAP_BASE_URL=https://ihr-sap-host:50000/b1s/v2/

# Wählbare CompanyDBs (eine oder mehrere, kommagetrennt)
SAP_DATABASES=SBO_IhreFirma

# Anmeldemodus: "basic" (User/Passwort) oder "bearer" (Browser-SSO bei aktivem
# IAM) — Einrichtung siehe sso-keycloak.md
SAP_AUTH_MODE=basic

# Zugriffsmodus: READ_ONLY oder READ_WRITE
SAP_OPERATION_MODE=READ_ONLY

# Nur bei selbstsigniertem SL-Zertifikat
SAP_ALLOW_SELF_SIGNED_CERT=true

# Öffentliche Adresse dieser Instanz — nötig für den Browser-Login (Claude Desktop).
# Lokal: http://127.0.0.1:8000 (Port muss zum Serverstart passen); im Netz die HTTPS-URL.
SAP_PUBLIC_URL=http://127.0.0.1:8000

# Phone-Home / automatische Abo-Erneuerung läuft automatisch (Enrollment-Token ist
# in der versino.key eingebacken — hier nichts nötig). Override nur für Test/Staging:
# SAP_ENROLLMENT_TOKEN=<override-token>
```

## Service Layer
| Variable | Default | Bedeutung |
|---|---|---|
| `SAP_BASE_URL` | – | Service-Layer-URL, z. B. `https://host:50000/b1s/v2/` |
| `SAP_DATABASES` | – | Wählbare CompanyDBs (Komma-Liste) |
| `SAP_OPERATION_MODE` | `READ_ONLY` | `READ_ONLY` oder `READ_WRITE` (Schreib-Tools) |
| `SAP_ALLOW_SELF_SIGNED_CERT` | `false` | selbstsigniertes SL-Zertifikat zulassen |
| `SAP_MAX_PAGE_SIZE` | `200` | max. Zeilen pro Seite |
| `SAP_MAX_CONCURRENT_REQUESTS` | `10` | parallele SL-Requests |

## Authentifizierung
| Variable | Bedeutung |
|---|---|
| `SAP_AUTH_MODE` | `basic` (User/Passwort direkt an SL) oder `bearer` (Browser-SSO mit PKCE, Token bei jedem Aufruf — ab FP 2208 mit Tokens des SAP-Authentication-Servers; stellt ein eigener Identity Provider die Tokens selbst aus, siehe Hinweis in sso-keycloak.md) |
| `SAP_DISABLE_INLINE_LOGIN` | `true` empfohlen: Login nur via Dialog/Web-UI, nie als Chat-Argument |
| `SAP_BIND_HOSTS` | Bind-Adressen, kommagetrennt (Default: alle Interfaces/`0.0.0.0`; `--host` auf der Kommandozeile gewinnt) |
| `SAP_PUBLIC_URL` | öffentliche HTTPS-URL der Instanz (Web-Login-Fallback; **Pflicht bei `bearer`** — Redirect-Ziel `…/callback`) |

### Nur bei `SAP_AUTH_MODE=bearer`

Browser-SSO mit PKCE — das Passwort erreicht den MCP nie. Pflicht sind
`SAP_PUBLIC_URL` sowie die Client-Daten aus dem Extension Single Sign-On Manager:

| Variable | Bedeutung |
|---|---|
| `SAP_IDP_CLIENT_ID` | Client-ID des registrierten Web-App-Clients (`b1-ext-…`) |
| `SAP_IDP_CLIENT_SECRET` | zugehöriges Client-Secret (wird nur einmal angezeigt) |
| `SAP_SLD_URL` | SLD-Adresse (z. B. `https://host:40000`) — bei Variante A für die CompanyID-Auflösung **erforderlich**; ermittelt zugleich alle IdP-Endpunkte automatisch |
| `SAP_COMPANY_IDS` | Nur **Variante B** (eigener Identity Provider): die beim Tenant-Binding notierte CompanyID je Datenbank, Format `DB:ID`, mehrere komma-getrennt (`SBO_PROD:1,SBO_TEST:2`). Muss **alle** Datenbanken aus `SAP_DATABASES` abdecken. Ersetzt die SLD-Abfrage — dann ist Port 40000 nicht nötig |
| `SAP_IDP_TOKEN_URL` | Token-Endpunkt — nur nötig, wenn keine automatische Ermittlung über `SAP_SLD_URL` erfolgen soll |
| `SAP_IDP_AUTHORIZE_URL` | optional; wird aus der Token-URL abgeleitet. **Pflicht**, wenn der Identity Provider kein Keycloak ist (Entra ID, Okta) |
| `SAP_IDP_SCOPE` | optional, Default `openid`. Erweiterte Scopes nur, wenn dem Client zugewiesen |
| `SAP_IDP_END_SESSION_URL` / `SAP_IDP_JWKS_URL` | optional (Abmeldung/zentrales Logout); kommen ebenfalls aus der automatischen Ermittlung |

Die früheren Namen `SAP_KEYCLOAK_*` gelten weiterhin (gleiche Bedeutung).
Die Redirect-URI `<SAP_PUBLIC_URL>/callback` muss am Client (Extension Single
Sign-On Manager) hinterlegt sein. Vollständige Anleitung inklusive der
SAP-seitigen Schritte: [sso-keycloak.md](sso-keycloak.md).

## Lizenz
| Variable | Bedeutung |
|---|---|
| `SAP_LICENSE_FILE` | Pfad zur `versino.key` — **nicht nötig**, wenn die Datei neben dem Binary liegt (Auto-Discovery) |
| `SAP_LICENSE` | Lizenz-Token direkt (Alternative zur Datei) |
| `SAP_ENROLLMENT_TOKEN` | Phone-Home/Auto-Renewal — **normalerweise nicht nötig** (Token ist in der `versino.key` eingebacken). Nur als Override für Test/Staging |

Phone-Home / automatische Abo-Erneuerung (Normalfall): siehe [lizenz.md](lizenz.md).

## Sicherheitshinweise
- `SAP_OPERATION_MODE=READ_ONLY` als Standard; `READ_WRITE` nur, wenn Schreibzugriff
  wirklich gewünscht ist.
- `SAP_DISABLE_INLINE_LOGIN=true` stellt sicher, dass SAP-Credentials nie in den
  LLM-Kontext geraten (Anmeldung nur über Dialog/Web-UI).
- Für Netzwerkbetrieb TLS (Reverse-Proxy) vorschalten.

## Logging

Warnungen und Fehler landen immer in `%APPDATA%\Versino\sapb1-mcp\sapb1-mcp.log`
(JSON-Zeilen, 1-MB-Kappe — älteste Einträge werden automatisch entfernt).
Der Pfad ist fest, damit der Support ihn immer kennt.
