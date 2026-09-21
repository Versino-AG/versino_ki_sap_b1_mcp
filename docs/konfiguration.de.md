<!-- translation-of: konfiguration.md@79126fc98aea -->
# Konfiguration (`.env`)

> 🌐 [English](konfiguration.md) · **Deutsch** · [Česky](konfiguration.cs.md)

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
# IAM) — Einrichtung siehe sso-keycloak.de.md
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
| `SAP_OPERATION_MODE` | `READ_ONLY` | `READ_ONLY` oder `READ_WRITE`. Im `READ_ONLY` werden die Schreib-Tools (`sap_create`, `sap_update`, `sap_delete`, `sap_action`) **gar nicht angeboten**; `sap_attachment` bleibt für `info`/`download` und lehnt Uploads ab, und `sap_deploy_queries` behält seinen Probelauf (nur das echte Ausbringen wird abgelehnt). Nach einem Moduswechsel den Server neu starten (der LLM-Client liest die Tool-Liste beim Neuverbinden neu). Welche Tools es gibt, hängt zusätzlich von der Lizenz-Edition ab → [lizenz.de.md](lizenz.de.md) |
| `SAP_ALLOW_SELF_SIGNED_CERT` | `false` | selbstsigniertes SL-Zertifikat zulassen |
| `SAP_MAX_PAGE_SIZE` | `200` | max. Zeilen pro Seite |
| `SAP_MAX_CONCURRENT_REQUESTS` | `10` | parallele SL-Requests |
| `SAP_TIMEOUT_SECONDS` | `60` | HTTP-Timeout für Service-Layer-Requests |
| `SAP_IDLE_LOGOUT_SECONDS` | `1500` | Leerlauf, nach dem eine Nutzer-Session automatisch abgemeldet wird |
| `SAP_SESSION_MAX_SECONDS` | `28800` | absolute Lebensdauer einer Nutzer-Session (8 h) zusätzlich zum Leerlauf-Logout; danach wird der Assistent aufgefordert, `connect` erneut aufzurufen. `0` = unbegrenzt |
| `SAP_PHONE_HOME_INTERVAL_SECONDS` | `3600` | Intervall der Lizenz-Phone-Home-Prüfung |
| `SAP_DB_SERVER_TYPE` | _auto_ | Datenbanktyp übersteuern (`HANA` / `MSSQL`); normalerweise automatisch erkannt — nur bei fehlerhafter Erkennung setzen |
| `SAP_AUTO_DEPLOY_QUERIES` | `true` | mitgelieferte Auswertungen beim ersten Verbinden je CompanyDB ausbringen |

### Mitgelieferte Auswertungen
Der Server bringt seine fertigen Auswertungen (`AI_*`-Abfragen in
`SQLQueries`) beim **ersten Verbinden** je CompanyDB selbst aus — einmal pro
Serverlauf. Voraussetzungen: `SAP_OPERATION_MODE=READ_WRITE`, ein
B1-Benutzer, der Abfragen anlegen darf, und eine Edition, welche die
Auswertungen ausführen kann. Im Lesebetrieb (`READ_ONLY`) wird das übersprungen,
und mit einer Stammdaten-Edition (BASIC) ebenfalls — dort gibt es
`sap_curated_query` nicht, die Auswertungen lägen also unbenutzbar in euren
`SQLQueries`. Alles andere funktioniert unverändert. Selbst geschriebene `AI_*`-Abfragen werden nie überschrieben.
Bei Bedarf lässt sich die Ausbringung im Chat gezielt anstoßen
(`sap_deploy_queries`) oder mit `SAP_AUTO_DEPLOY_QUERIES=false` abschalten.

## Authentifizierung
| Variable | Bedeutung |
|---|---|
| `SAP_AUTH_MODE` | `basic` (User/Passwort direkt an SL) oder `bearer` (Browser-SSO mit PKCE, Token bei jedem Aufruf — ab FP 2208 mit Tokens des SAP-Authentication-Servers; stellt ein eigener Identity Provider die Tokens selbst aus, siehe Hinweis in sso-keycloak.de.md) |
| `SAP_DISABLE_INLINE_LOGIN` | Login nur via Dialog/Web-UI, nie als Chat-Argument. **Standard `true`, sobald `SAP_PUBLIC_URL` gesetzt ist** (Netzwerk-Installation); `false` nur lokal oder als bewusstes Opt-in (`doctor` warnt) |
| `SAP_ALLOWED_CLIENTS` | Adressen/Netze (IP/CIDR, kommagetrennt), die den Server überhaupt erreichen dürfen. Leer (Default) = jeder, der den Port erreicht. **Keine Anmeldung** — die bleibt Benutzer/Passwort/Datenbank; es entfernt nur das offene Internet. Ein Tippfehler bricht den Start ab und nennt den Eintrag |
| `SAP_TICKET_SESSION_MAX_SECONDS` | absolute Lebensdauer einer Sitzung, die über einen Browser-Login-Schlüssel angesprochen wird, in Sekunden (Default `3600`, `0` = aus). Dieser Schlüssel reiste durch den Chatverlauf und wird bei jedem Aufruf mitgegeben, ist also enger begrenzt als `SAP_SESSION_MAX_SECONDS`; es gilt die kürzere der beiden Grenzen. Nutzer melden sich danach über `connect` neu an |
| `SAP_TRUSTED_PROXIES` | Reverse-Proxy-Adressen (IPs/CIDRs, kommagetrennt), deren `X-Forwarded-For` der Server für die Login-Drossel und das Audit-Log vertraut — z. B. `127.0.0.1`, wenn nginx auf derselben Maschine läuft. Leer = der Socket-Peer gilt als Client (hinter einem Proxy wäre das der Proxy selbst, das ganze Büro teilt sich dann einen Zähler). Ein Alles-Eintrag (`0.0.0.0/0`, `::/0`) wird beim Start abgewiesen: Jedem zu vertrauen hieße, dass jeder Client pro Anfrage eine neue Adresse behaupten kann — die Drossel wäre aus |
| `SAP_LOGIN_MAX_FAILURES` | Fehlversuche pro Client-Adresse innerhalb von `SAP_LOGIN_WINDOW_SECONDS`, bevor die Adresse pausiert wird (Standard `10`); die Pause beginnt bei 30 s und verdoppelt sich bis 15 Min., eine erfolgreiche Anmeldung setzt zurück |
| `SAP_LOGIN_WINDOW_SECONDS` | Zeitfenster für `SAP_LOGIN_MAX_FAILURES` (Standard `900`) |
| `SAP_TICKET_ISSUE_PER_MINUTE` | globale Obergrenze für Browser-Login-Links pro Minute (Standard `60`) — schützt den unauthentifizierten Pfad vor Fluten |
| `SAP_TLS_CERT_FILE` | Server-Zertifikat (PEM) für eingehendes HTTPS — zusammen mit `SAP_TLS_KEY_FILE`; sonst HTTP |
| `SAP_TLS_KEY_FILE` | Privater Schlüssel (PEM) für eingehendes HTTPS |
| `SAP_TLS_KEY_PASSWORD` | Passwort für einen verschlüsselten TLS-Schlüssel (optional) |
| `SAP_TLS_MIN_VERSION` | Mindest-TLS-Version für eingehendes HTTPS: `1.2` (Default) oder `1.3` |
| `SAP_TLS_CIPHERS` | OpenSSL-Cipher-String pinnen (nur TLS 1.2; leer = sichere Defaults) |
| `SAP_TLS_CLIENT_CA_FILE` | Client-CA (PEM) → erzwingt Client-Zertifikate (mTLS) |
| `SAP_BIND_HOSTS` | Bind-Adressen, kommagetrennt (Default: alle Interfaces/`0.0.0.0`; `--host` auf der Kommandozeile gewinnt) |
| `SAP_LANG` | Sprache aller serverseitigen Texte (exe-Start, `doctor`, Web-Login inkl. Ergebnis-/Fehlermeldungen, Auth-Fehler): `de` (Default), `en`, `cs`. Der geführte Installer trägt hier die in seinem Dialog gewählte Sprache ein. Chat-Antworten des Assistenten folgen automatisch der Sprache des Nutzers |
| `SAP_PUBLIC_URL` | öffentliche HTTPS-URL der Instanz (Web-Login-Fallback; **Pflicht bei `bearer`** — Redirect-Ziel `…/callback`) |
| `SAP_UPLOAD_MAX_MB` | max. Dateigröße pro Anhang-Upload in MB (Default `25`; Browser-Upload, `source_url`, `file_path`) — siehe [anhaenge.de.md](anhaenge.de.md) |
| `SAP_ATTACHMENT_URL_ALLOWLIST` | Hosts, von denen `sap_attachment` eine `source_url` laden darf, kommagetrennt (`host` oder `*.domain`; leer = aus) |
| `SAP_ATTACHMENT_DIR` | Verzeichnis, unterhalb dessen `sap_attachment` einen `file_path` lesen darf (leer = aus) |

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
| `SAP_IDP_ISSUER` / `SAP_IDP_REVOCATION_URL` | optional (Token-Aussteller / Token-Widerruf); werden normalerweise automatisch über `SAP_SLD_URL` ermittelt — nur setzen, wenn die Auto-Ermittlung übersteuert werden muss |

Die früheren Namen `SAP_KEYCLOAK_*` gelten weiterhin (gleiche Bedeutung).
Die Redirect-URI `<SAP_PUBLIC_URL>/callback` muss am Client (Extension Single
Sign-On Manager) hinterlegt sein. Vollständige Anleitung inklusive der
SAP-seitigen Schritte: [sso-keycloak.de.md](sso-keycloak.de.md).

## Lizenz
| Variable | Bedeutung |
|---|---|
| `SAP_LICENSE_FILE` | Pfad zur `versino.key` — **nicht nötig**, wenn die Datei neben dem Binary liegt (Auto-Discovery) |
| `SAP_LICENSE` | Lizenz-Token direkt (Alternative zur Datei) |
| `SAP_LICENSE_CACHE_FILE` | Pfad für erneuerte Token (stiller Renewal-Cache); Default `versino.renewed` neben Lizenz/Binary |
| `SAP_INSTALL_IDENTITY_PATH` | Pfad der Installations-Identität (`install_identity.json`); Default relativ zum Arbeitsverzeichnis — in Containern (read-only Rootfs) einen persistenten Pfad setzen |
| `SAP_TIME_ANCHOR_PATH` | monotone Zeitmarke gegen Zurückstellen der Uhr. **Seit 3.8.1 standardmäßig aktiv**: Die Datei liegt neben `versino.key`. Ein Pfad verschiebt sie, ein leerer Wert schaltet sie ab (schreibgeschützte Container) |
| `SAP_ENROLLMENT_TOKEN` | Phone-Home/Auto-Renewal — **normalerweise nicht nötig** (Token ist in der `versino.key` eingebacken). Nur als Override für Test/Staging |
| `SAP_LICENSE_VALIDATION_URL` | Override für den eingebauten Validierungs-Endpoint (Test/Staging). **Muss `https` sein** — die Anfrage trägt Kundennummer und Seat-Zahl, einfaches `http` gegen einen entfernten Host wird abgewiesen; unverschlüsselt erlaubt sind nur `127.0.0.1`, `::1` und `localhost`. Die Enrollment-Adresse wird aus dem Verzeichnis dieser URL abgeleitet — den Pfad daher unverändert lassen |

Phone-Home / automatische Abo-Erneuerung (Normalfall): siehe [lizenz.de.md](lizenz.de.md).

## Sicherheitshinweise
- `SAP_OPERATION_MODE=READ_ONLY` als Standard; `READ_WRITE` nur, wenn Schreibzugriff
  wirklich gewünscht ist. Eine Nur-Lese-Instanz zeigt dem LLM-Client die
  Schreib-Tools gar nicht, der Assistent kann sie also nicht einmal versuchen.
- `SAP_DISABLE_INLINE_LOGIN=true` stellt sicher, dass SAP-Credentials nie in den
  LLM-Kontext geraten (Anmeldung nur über Dialog/Web-UI). Bei Netzwerk-Installationen
  (`SAP_PUBLIC_URL` gesetzt) ist das der Standard.
- Der Browser-Login ist ab Werk gehärtet: Der Anmelde-Link trägt das Ticket im
  URL-**Fragment** (`…/login#t=…`, nie in Server- oder Proxy-Logs); nach
  `connect(ticket=…)` wird das Ticket ausgemustert und ein frischer Sitzungswert
  zurückgegeben; Fehlversuche werden pro Adresse gedrosselt, und jeder Versuch
  landet im Audit-Log (Nutzer, CompanyDB, Quelladresse, Ergebnis — nie das
  Passwort); Sitzungen enden nach `SAP_SESSION_MAX_SECONDS` (8 h), eine über einen
  Browser-Login-Schlüssel angesprochene Sitzung nach `SAP_TICKET_SESSION_MAX_SECONDS`
  (1 h) — es gilt die kürzere Grenze.
- Hinter einem Reverse-Proxy `SAP_TRUSTED_PROXIES` setzen, damit die Drossel echte
  Client-Adressen sieht, und die Proxy-Limits aus
  [installation-zentral.de.md](installation-zentral.de.md) ergänzen.
- `sapb1-mcp doctor` liest das Testpasswort aus **`SAP_DOCTOR_PASSWORD`**, nicht aus
  `--password`: Ein Passwort auf der Kommandozeile steht in der Prozessliste und, bei
  aktivierter Kommandozeilen-Protokollierung, im Windows-Ereignisprotokoll, wo es die
  Installation überdauert. Der Schalter funktioniert weiter, warnt aber. Die Variable
  gilt dem einzelnen `doctor`-Aufruf (der Installer setzt sie nur für diesen
  Kindprozess) — in die `.env` gehört sie nicht.
- Für Netzwerkbetrieb TLS (Reverse-Proxy) vorschalten.

## Logging

Warnungen und Fehler landen immer in `%APPDATA%\Versino\sapb1-mcp\sapb1-mcp.log`
(JSON-Zeilen, 1-MB-Kappe — älteste Einträge werden automatisch entfernt).
Der Pfad ist fest, damit der Support ihn immer kennt.
