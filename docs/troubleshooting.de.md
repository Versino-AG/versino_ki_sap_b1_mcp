<!-- translation-of: troubleshooting.md@5dfefad6971e -->
# Troubleshooting

> 🌐 [English](troubleshooting.md) · **Deutsch** · [Česky](troubleshooting.cs.md)

## `license.refused` beim Start
Keine gültige Lizenz gefunden. Prüfen:
- liegt `versino.key` **neben** dem Binary? (oder zeigt `SAP_LICENSE_FILE` darauf?)
- ist der Schlüssel nicht abgelaufen? (neue über das [Lizenzportal](https://aishop.versino.de))

Details: [lizenz.de.md](lizenz.de.md).

## Anhang-Upload scheitert mit SAP-Fehler `-43`
`-43` ist SAPs interner *Pfad-/Ordner*-Fehler. Beim Anhang-Upload heißt das: Der
**Service Layer** konnte nicht in den Anlagenordner schreiben — es liegt nicht an
der Datei (dieselbe Datei scheitert immer wieder). In SAP Business One prüfen:
*Administration → Systeminitialisierung → Allgemeine Einstellungen → Pfad →
Anlagenordner*:
- der Pfad muss **aus Sicht des Service-Layer-Hosts** existieren (UNC-Pfad wie
  `\\fileserver\B1_Anlagen`, kein Laufwerksbuchstabe eines Anwender-PCs),
- das Dienstkonto des Service Layers braucht dort **Schreibrechte**,
- im Ordner darf noch keine Datei mit **demselben Dateinamen** liegen — sonst mit
  anderem `file_name` erneut versuchen.
Nach der Korrektur auf SAP-Seite einfach erneut hochladen. Hintergrund:
[anhaenge.de.md](anhaenge.de.md).

## Windows-Defender / SmartScreen-Warnung
Ist das Binary noch nicht signiert, kann Windows einen Fehlalarm zeigen.
- „Weitere Informationen" → „Trotzdem ausführen", bzw. die Datei in Defender zulassen.
- Für die produktive Auslieferung ist Code-Signing vorgesehen — dann entfällt die Warnung.

## Client erreicht den Server nicht
- Host/Port und den **`/mcp`**-Pfad in der URL prüfen.
- Bei Zugriff von anderen Rechnern: Standard-Bind ist bereits `0.0.0.0` (prüfen, dass kein
  einschränkendes `SAP_BIND_HOSTS`/`--host` gesetzt ist), **Firewall**
  für den Port freigeben und die **interne IP/DNS** des Servers in der Client-URL verwenden.
- Claude Desktop bevorzugt **HTTPS**; für blankes `http://` die `mcp-remote`-Bridge nutzen
  (siehe [installation-windows.de.md](installation-windows.de.md)).

## Verbindung zu SAP schlägt fehl
- `SAP_BASE_URL` korrekt? (`https://<host>:50000/b1s/v2/`)
- selbstsigniertes SL-Zertifikat → `SAP_ALLOW_SELF_SIGNED_CERT=true`.
- richtiger `SAP_AUTH_MODE` (`basic` vs. `bearer`)?
- gewählte CompanyDB in `SAP_DATABASES` enthalten?

## Schreib-Tools fehlen / werden abgelehnt
- `SAP_OPERATION_MODE=READ_WRITE` setzen (Default ist `READ_ONLY`). Im Lesebetrieb
  werden die Schreib-Tools gar nicht registriert, der Assistent sieht sie also
  nicht. Nach dem Moduswechsel den **Server neu starten** und den LLM-Client neu
  verbinden — er cacht die Tool-Liste.
- Anhang-Upload abgelehnt mit „READ_ONLY: uploading attachments writes to SAP" →
  derselbe Schalter; `info`/`download` funktionieren auch im Lesebetrieb.
- Schreib-/Lösch-Tools sind zusätzlich an die **Edition** gebunden (PRO/ENTERPRISE),
  siehe [lizenz.de.md](lizenz.de.md).

## Windows: Fenster schließt sich sofort wieder
Meist ist der **Port schon belegt** (ein anderer Dienst lauscht auf `8000`; im Log
`WinError 10048` / „… nur jeweils einmal verwendet werden"). Lösung:
- Server auf einem **freien Port** starten: `sapb1-mcp.exe --port 8765` — und im Client
  dieselbe Portnummer verwenden (`…:8765/mcp`).
- Oder den belegenden Dienst beenden.
- Damit ihr die Fehlermeldung **seht** statt eines sofort schließenden Fensters: die `.exe`
  aus einem geöffneten Terminal (PowerShell) starten, nicht per Doppelklick.

## `connect` liefert keinen Login-Link (Web-Login)
Der Browser-Login-Fallback braucht die **öffentliche Adresse** der Instanz. In der `.env`
`SAP_PUBLIC_URL` setzen (im Netz `https://…`, lokal `http://127.0.0.1:8000`) und den Server
neu starten. Siehe [konfiguration.de.md](konfiguration.de.md) und
[installation-zentral.de.md](installation-zentral.de.md).

## Login-Seite reagiert beim Klick nicht / keine Bestätigung
Beim Klick auf „Anmelden" passiert nichts und es kommt keine Erfolgsseite — fast immer ist
der **SAP Service Layer nicht erreichbar** (die Anmeldung läuft ins Leere/Timeout):
- Ist der SAP-Host vom **Server** aus erreichbar? (oft **VPN** nötig, Firewall, Port `50000`).
  Test: `Test-NetConnection <sap-host> -Port 50000` (Windows) bzw. `nc -vz <sap-host> 50000`.
- `SAP_BASE_URL` korrekt (`https://<host>:50000/b1s/v2/`)?
- Tickets sind kurzlebig: zwischen Linköffnen und Login nicht zu lange warten.

## „Zu viele fehlgeschlagene Anmeldeversuche" / HTTP 429
Der Server pausiert eine Adresse nach `SAP_LOGIN_MAX_FAILURES` Fehlversuchen
(Standard 10 in 15 Min.): zuerst 30 s, verdoppelnd bis 15 Min.; eine erfolgreiche
Anmeldung hebt die Pause auf. Die Meldung nennt die Wartezeit.
- Hinter einem Reverse-Proxy ohne `SAP_TRUSTED_PROXIES` ist der **Proxy** die
  Adresse — die Tippfehler eines Kollegen pausieren das ganze Büro. Dann
  `SAP_TRUSTED_PROXIES=<Proxy-Adresse>` setzen (siehe
  [installation-zentral.de.md](installation-zentral.de.md)).
- „Ungewöhnlich viele Anmeldeanfragen … Browser-Anmeldungen pausiert": die
  globale Obergrenze `SAP_TICKET_ISSUE_PER_MINUTE` (Standard 60) wurde erreicht —
  eine Minute warten; nur bei sehr großen Installationen erhöhen.
- Jeder Versuch steht in der Log-Datei (`audit.auth.login` mit Nutzer, CompanyDB,
  Quelladresse, Ergebnis — nie das Passwort), damit der Support eine Meldung
  einer Zeile zuordnen kann.

## „Deine SAP-Sitzung hat ihre Höchstdauer erreicht"
Sitzungen enden nach `SAP_SESSION_MAX_SECONDS` (Standard 8 h) unabhängig von der
Aktivität; der Assistent wird aufgefordert, `connect` erneut aufzurufen — mehr ist
nicht nötig. `0` schaltet die Grenze ab.

## Login-Seite meldet, der Link enthalte kein Ticket
Der Link trägt das Ticket hinter dem `#` (`…/login#t=…`). Manche Chat-Oberflächen
schneiden diesen Teil beim Darstellen ab: den Link genau so öffnen, wie er
ausgegeben wurde, oder den Assistenten mit `connect` um einen neuen bitten. Auch
ein Neuladen der Seite nach der Anmeldung verliert das Fragment (absichtlich) —
neuen Link anfordern.

## `npx` / Node.js nicht gefunden (Bridge)
Die `mcp-remote`-Bridge benötigt **Node.js**. Node LTS von
[nodejs.org](https://nodejs.org) installieren, Client neu starten. Tipp: in der Client-Config
`npx` durch `cmd /c npx` ersetzen (Windows), falls der Befehl nicht gefunden wird.

## Linux: Binary startet nicht
- ausführbar gemacht? `chmod +x sapb1-mcp`
- 64-bit Linux mit glibc (x86-64) vorausgesetzt.

Weiterhin Probleme? **support@versino.de** (bitte mit Logausschnitt und Versionsnummer).

## Lizenz abgelaufen / Abo-Verlängerung
- Die exe holt eine fällige **Abo-Verlängerung automatisch beim Start** (sofern
  Internet + gültiges Abo). Zusätzlich gibt es **3 Tage Grace-Period** nach
  Ablauf, in denen der Server auch ohne erfolgreiches Phone-Home noch startet.
- Startet die exe nach Ablauf dauerhaft nicht mehr, prüfen: Internetzugang zum
  Lizenzserver vorhanden? Abo im Lizenzportal (aishop.versino.de) aktiv/bezahlt?
  Im Zweifel bei **support@versino.de** einen frischen `versino.key` anfordern.
