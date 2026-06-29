# Geführte Installation (Windows, empfohlen)

Der **geführte Windows-Installer** richtet den SAP-B1-MCP-Server in wenigen Schritten
vollständig ein. Er nimmt euch die manuellen Schritte ab, die ihr sonst von Hand machen
müsstet:

- Programmdatei herunterladen und ablegen
- `.env` von Hand schreiben (richtige Schlüssel, keine Tippfehler, korrektes Encoding)
- `versino.key` an die richtige Stelle kopieren
- Service-Layer-Erreichbarkeit prüfen
- optional einen Windows-Dienst einrichten
- optional Claude Desktop konfigurieren

> Wer den Server **manuell** aufsetzen oder feiner anpassen möchte (eigene Verzeichnisse,
> eigener Dienst-Wrapper, Reverse-Proxy, Linux), nutzt stattdessen die manuellen
> Anleitungen: [installation-windows.md](installation-windows.md) /
> [installation-linux.md](installation-linux.md).

## Voraussetzungen

- 64-bit Windows 10/11 oder Windows Server 2019+
- euren **Lizenzschlüssel** (per E-Mail bzw. aus dem
  [Lizenzportal](https://aishop.versino.de)) — als Text zum Einfügen **oder** als
  Datei `versino.key`; beides funktioniert. Siehe [lizenz.md](lizenz.md)
- Netzwerkzugriff auf euren SAP-B1-**Service-Layer** (`https://<host>:50000/b1s/v2/`)
- Internetzugang während der Installation (der Installer lädt die Programmdatei aus dem
  Release)

## Herunterladen

Aus den [Releases](../../releases/latest) das Setup laden:

| Datei | Zweck |
|-------|-------|
| `sapb1-mcp-setup-<version>.exe` | geführter Installer |

Das Setup ist von der **Versino AG signiert** (Authenticode) — Windows zeigt „Versino AG"
als Herausgeber. Die eigentliche Server-Programmdatei lädt der Installer während der
Installation passend zur Version selbst herunter.

## Der Installationsablauf

Setup starten und dem Assistenten folgen. Er fragt der Reihe nach ab:

1. **AGB** — die [Allgemeinen Geschäftsbedingungen](https://aishop.versino.de/agb)
   bestätigen.
2. **Zielordner** — wohin installiert wird (Standard: `%LOCALAPPDATA%\Versino\sapb1-mcp`,
   z. B. `C:\Users\<Benutzer>\AppData\Local\Versino\sapb1-mcp`). Die Installation
   läuft ohne Administrator-Rechte; nur die optionale Dienst-Einrichtung fragt einmalig
   per UAC nach.
3. **Lizenzschlüssel** — euren von Versino gelieferten Schlüssel in das Feld **einfügen**
   oder per **„versino.key wählen …“** aus einer Datei laden (beides landet im selben
   Feld). Der Installer lädt die Programmdatei, schreibt den Schlüssel als `versino.key`
   in den Zielordner und prüft ihn **sofort offline** (Signatur + Ablauf gegen den
   eingebauten Schlüssel). Ist er ungültig oder abgelaufen, geht es nicht weiter. Der
   Enrollment-Token (für Erneuerung) steckt darin — ihr müsst **nichts** weiter von Hand
   eintragen.
4. **Service-Layer-Verbindung** — eure SL-URL eingeben
   (`https://<host>:50000/b1s/v2/`). Der Installer prüft die **Erreichbarkeit** des
   Servers. Verlangt der Service Layer (wie üblich) eine Anmeldung, gilt das **als
   erreichbar** — nur eine echte Nichterreichbarkeit (falsche Adresse, Netz/Firewall,
   Zertifikatsproblem) wird gemeldet.
5. **CompanyDB(s)** — die wählbaren Datenbanken (kommagetrennt). Optional könnt ihr hier
   einen **Test-Login** (Benutzer/Passwort) eingeben; der Installer prüft damit DB-Name
   und Zugang. Diese Test-Zugangsdaten werden **nicht gespeichert**.
6. **Erreichbarkeit** — Port (Standard `8000`) und optional die öffentliche URL. Lokal
   genügt der Port; im Netzbetrieb die öffentliche HTTPS-URL (Reverse-Proxy, siehe
   [installation-zentral.md](installation-zentral.md)).
7. **Optionen**:
   - **Schreibzugriff erlauben** (`READ_WRITE` statt nur Lesen)
   - **Selbstsigniertes SL-Zertifikat akzeptieren**
   - **Als Windows-Dienst einrichten** — der Server läuft dann automatisch (auch ohne
     angemeldeten Benutzer)
   - **Claude-Desktop-Konfiguration automatisch eintragen** (standardmäßig an)

## Nach der Installation

Im Zielordner liegen dann:

- `sapb1-mcp.exe` — der Server
- `versino.key` — eure Lizenz
- `.env` — die vorausgefüllte Konfiguration aus euren Eingaben

Je nach gewählten Optionen zusätzlich:

- ein **Windows-Dienst** `SAPB1-MCP`, der den Server automatisch startet
- der **Claude-Desktop-Eintrag** ist gesetzt (bestehende Einträge bleiben erhalten) —
  Claude Desktop danach neu starten

Wurde **kein** Dienst gewählt, startet ihr den Server wie in
[installation-windows.md](installation-windows.md), Abschnitt 3 beschrieben.

## Erste Nutzung

Im LLM-Client das Tool **`connect`** aufrufen → SAP-Anmeldung (Eingabedialog bzw.
Browser-Login). Die **Zugangsdaten gelangen nie in den Chat-/LLM-Kontext**. Danach stehen
die SAP-Tools bereit; `list_databases` zeigt die CompanyDBs. Siehe auch
[erste-schritte.md](erste-schritte.md).

## Troubleshooting

Häufige Fälle (Defender-Warnung, `license.refused`, Verbindungsprobleme) findet ihr in
[troubleshooting.md](troubleshooting.md).
