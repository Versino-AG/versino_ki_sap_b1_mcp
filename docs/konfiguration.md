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

# Anmeldemodus: "basic" oder "ropc"
SAP_AUTH_MODE=basic

# Zugriffsmodus: READ_ONLY oder READ_WRITE
SAP_OPERATION_MODE=READ_ONLY

# Nur bei selbstsigniertem SL-Zertifikat
SAP_ALLOW_SELF_SIGNED_CERT=true

# Öffentliche Adresse dieser Instanz — nötig für den Browser-Login (Claude Desktop).
# Lokal: http://127.0.0.1:8000 (Port muss zum Serverstart passen); im Netz die HTTPS-URL.
SAP_PUBLIC_URL=http://127.0.0.1:8000
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
| `SAP_AUTH_MODE` | `basic` (User/Passwort direkt an SL) oder `ropc` (Keycloak/SSO) |
| `SAP_DISABLE_INLINE_LOGIN` | `true` empfohlen: Login nur via Dialog/Web-UI, nie als Chat-Argument |
| `SAP_PUBLIC_URL` | öffentliche HTTPS-URL der Instanz (für den Web-Login-Fallback) |

Nur bei `SAP_AUTH_MODE=ropc` (Instanz-Secret aus dem SLD, **kein** Endnutzer-Login):
`SAP_KEYCLOAK_TOKEN_URL`, `SAP_KEYCLOAK_CLIENT_ID`, `SAP_KEYCLOAK_CLIENT_SECRET` (optional
`SAP_KEYCLOAK_SCOPE`). Kurzanleitung: [sso-keycloak.md](sso-keycloak.md).

## Lizenz
| Variable | Bedeutung |
|---|---|
| `SAP_LICENSE_FILE` | Pfad zur `versino.key` — **nicht nötig**, wenn die Datei neben dem Binary liegt (Auto-Discovery) |
| `SAP_LICENSE` | Lizenz-Token direkt (Alternative zur Datei) |

Optionales Phone-Home / automatische Abo-Erneuerung: siehe [lizenz.md](lizenz.md).

## Sicherheitshinweise
- `SAP_OPERATION_MODE=READ_ONLY` als Standard; `READ_WRITE` nur, wenn Schreibzugriff
  wirklich gewünscht ist.
- `SAP_DISABLE_INLINE_LOGIN=true` stellt sicher, dass SAP-Credentials nie in den
  LLM-Kontext geraten (Anmeldung nur über Dialog/Web-UI).
- Für Netzwerkbetrieb TLS (Reverse-Proxy) vorschalten.
