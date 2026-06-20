# SSO-/ROPC-Anbindung über Keycloak (Kurzanleitung)

Nur nötig, wenn euer SAP B1 den **Authentication Server (SLD/IAM mit Keycloak)** nutzt.
Dann läuft die Anmeldung: **Nutzer-Login → Keycloak → JWT → Service-Layer-`/Login`**.
Meldet ihr euch klassisch direkt am Service Layer an, bleibt `SAP_AUTH_MODE=basic` (→
[konfiguration.md](konfiguration.md)) — dieser Abschnitt ist dann irrelevant.

## Voraussetzungen (SAP-/SLD-seitig)
- SAP B1 Authentication Server (Keycloak) mit dem Realm **`sapb1`**.
- Ein **vorregistrierter Service-Layer-OAuth-Client** (Client-ID + Secret) aus dem **SLD**,
  bei dem **„Direct Access Grants" (ROPC) aktiviert** ist. Diese Werte stammen aus der
  SAP-Einrichtung — sie sind **instanzweit** und **nicht** die Zugangsdaten eines Endnutzers.

## `.env`
```ini
SAP_AUTH_MODE=ropc
SAP_KEYCLOAK_TOKEN_URL=https://<auth-host>:40020/auth/realms/sapb1/protocol/openid-connect/token
SAP_KEYCLOAK_CLIENT_ID=<Client-ID aus dem SLD>
SAP_KEYCLOAK_CLIENT_SECRET=<Client-Secret>

# optional
SAP_KEYCLOAK_SCOPE=openid profile email B1.ServiceLayer   # Default (B1.ServiceLayer ist Pflicht)
SAP_ALLOW_SELF_SIGNED_CERT=true                            # falls Auth-Server UND/ODER SL selbstsigniert
```
- `SAP_KEYCLOAK_TOKEN_URL` **muss `https://`** sein; Host/Port (typisch `40020`) und Realm
  (`sapb1`) gemäß eurer SLD-Konfiguration anpassen.
- `SAP_ALLOW_SELF_SIGNED_CERT` gilt **gemeinsam** für Service Layer **und** Keycloak.
- Alles Übrige (`SAP_BASE_URL`, `SAP_DATABASES`, `SAP_OPERATION_MODE`, Lizenz …) bleibt wie
  gehabt.

## Was sich für Anwender ändert
**Nichts an der Bedienung.** Der Login läuft weiter über Dialog/Web-Login; der Nutzer gibt
seine **eigenen** Zugangsdaten ein — diese gehen jetzt nur an Keycloak (statt direkt an den
Service Layer). Client-ID/Secret sind reine Instanz-Konfiguration und tauchen nie beim
Nutzer auf.

## Testen
Server neu starten und im Client `connect` aufrufen (→ [erste-schritte.md](erste-schritte.md)).
Klappt die Anmeldung, ist die ROPC-Kette korrekt.

## Troubleshooting
| Fehler | Ursache / Lösung |
|---|---|
| `invalid_client` | falsche `SAP_KEYCLOAK_CLIENT_ID`/`..._SECRET` |
| `unauthorized_client` / Grant nicht erlaubt | **Direct Access Grants** für den Client im Keycloak nicht aktiviert |
| Realm-/404-Fehler | falsche `SAP_KEYCLOAK_TOKEN_URL` (Host/Port/Realm prüfen) |
| Zertifikatsfehler | selbstsigniert → `SAP_ALLOW_SELF_SIGNED_CERT=true` |
| Login geht, aber kein SL-Zugriff | `scope` ohne `B1.ServiceLayer` |

Weitere Hilfe: **support@versino.de**.
