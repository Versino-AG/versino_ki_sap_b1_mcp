<!-- Auto-generiert aus dist-content/ im Quell-Repo. Bitte NICHT direkt hier editieren. -->

# Versino KI SAP B1 MCP — Builds & Dokumentation

> 🌐 Die Dokumentation gibt es in drei Sprachen: **Deutsch** (dieses README, `docs/*.de.md`) · English (`docs/*.md`) · Česky (`docs/*.cs.md`). Jede Doku-Seite verlinkt oben ihre Sprachvarianten. Die Sprache der Server-Meldungen (Start, Web-Login, Fehler) stellt `SAP_LANG=de|en|cs` in der `.env` ein (Default `de`); passende `.env`-Vorlagen: `.env.example` (DE), `.env.en.example`, `.env.cs.example`.

Offizielles Bezugs-Repository für den **Versino-KI-SAP-Business-One-MCP-Server** der Versino AG:
fertige Binaries, Installationsanleitungen und Betriebsdoku.

Der Server stellt SAP Business One (Service Layer) als **MCP-Server** bereit, sodass
LLM-Clients wie Claude Desktop direkt mit eurem SAP B1 arbeiten können. Er läuft
**per-user**: jeder Nutzer meldet sich mit der **eigenen** SAP-Identität an —
klassisch mit Benutzer/Passwort oder per **Single Sign-On** (Browser-Anmeldung
beim Identity Provider, siehe [docs/sso-keycloak.de.md](docs/sso-keycloak.de.md)).
Zugangsdaten gelangen nie in den Chat-/LLM-Kontext.

## Download

Die aktuelle Version liegt unter **[Releases](../../releases/latest)**:

| Plattform | Datei | Empfehlung |
|-----------|-------|------------|
| **Windows — geführter Installer** | `sapb1-mcp-setup-<version>.exe` | **empfohlen** |
| Windows (64-bit) — Binary | `sapb1-mcp-<version>-windows-x64.exe` | manuell/Anpassung |
| Linux (64-bit) — Binary | `sapb1-mcp-<version>-linux-x64` | manuell/Anpassung |

Dazu benötigt ihr eure Lizenzdatei **`versino.key`** (kommt per E-Mail bzw. über das
[Lizenzportal](https://aishop.versino.de)). Siehe [docs/lizenz.de.md](docs/lizenz.de.md).

## Schnellstart

### Empfohlen (Windows): geführter Installer

1. `sapb1-mcp-setup-<version>.exe` aus den [Releases](../../releases/latest) laden und starten.
2. Dem Assistenten folgen: `versino.key` wählen, SL-URL + CompanyDBs angeben, Optionen
   setzen — der Installer lädt die Programmdatei, schreibt die `.env`, kopiert die Lizenz
   und richtet auf Wunsch einen Windows-Dienst sowie die Claude-Desktop-Anbindung ein.
   → [docs/installer.de.md](docs/installer.de.md)
3. Als Anwender loslegen → [docs/erste-schritte.de.md](docs/erste-schritte.de.md).

### Manuell (Windows/Linux): Binary selbst aufsetzen

Für eigene Verzeichnisse, eigenen Dienst-Wrapper, Reverse-Proxy oder Linux:

1. Binary + `versino.key` + `.env` in **einen** Ordner legen.
2. `.env` konfigurieren → [docs/konfiguration.de.md](docs/konfiguration.de.md).
3. Starten und an den LLM-Client anbinden:
   - **Windows:** [docs/installation-windows.de.md](docs/installation-windows.de.md)
   - **Linux:** [docs/installation-linux.de.md](docs/installation-linux.de.md)
   - **Zentral (eine Instanz für alle Arbeitsplätze):** [docs/installation-zentral.de.md](docs/installation-zentral.de.md)
4. Als Anwender loslegen → [docs/erste-schritte.de.md](docs/erste-schritte.de.md).

## Systemvoraussetzungen

- 64-bit Windows 10/11 / Windows Server 2019+ **oder** 64-bit Linux (glibc, x86-64).
- Netzwerkzugriff auf euren SAP-B1-**Service-Layer** (`https://<host>:50000/b1s/v2/`).
- Gültige `versino.key` (Edition BASIC / PRO / ENTERPRISE).
- Für Nur-stdio-Clients: Node.js (für die `mcp-remote`-Bridge).

## Dokumentation

- [Erste Schritte (für Anwender)](docs/erste-schritte.de.md) — Nutzung im Chat nach dem Setup
- [**Geführte Installation (Windows, empfohlen)**](docs/installer.de.md)
- [Installation Windows (manuell)](docs/installation-windows.de.md)
- [Installation Linux (manuell)](docs/installation-linux.de.md)
- [Zentrale Installation (eine Instanz für alle Arbeitsplätze)](docs/installation-zentral.de.md)
- [Konfiguration (`.env`)](docs/konfiguration.de.md)
- [SSO über Keycloak (OIDC) — Einrichtungsanleitung](docs/sso-keycloak.de.md)
- [Lizenz (`versino.key`, Erneuerung, Seats)](docs/lizenz.de.md)
- [Troubleshooting](docs/troubleshooting.de.md)

## Support

Fragen, Lizenzen, Störungen: **support@versino.de**

## Rechtliches

Es gelten die **[Allgemeinen Geschäftsbedingungen](https://aishop.versino.de/agb)**.

---
© Versino AG. Nutzung gemäß den [AGB](https://aishop.versino.de/agb) und der
Lizenzvereinbarung. Das Binary ist ohne gültige `versino.key` nicht lauffähig.
