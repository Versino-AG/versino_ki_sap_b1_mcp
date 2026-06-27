<!-- Auto-generiert aus dist-content/ im Quell-Repo. Bitte NICHT direkt hier editieren. -->

# Versino KI SAP B1 MCP — Builds & Dokumentation

Offizielles Bezugs-Repository für den **Versino-KI-SAP-Business-One-MCP-Server** der Versino AG:
fertige Binaries, Installationsanleitungen und Betriebsdoku.

Der Server stellt SAP Business One (Service Layer) als **MCP-Server** bereit, sodass
LLM-Clients wie Claude Desktop direkt mit eurem SAP B1 arbeiten können. Er läuft
**per-user**: jeder Nutzer meldet sich mit den **eigenen** SAP-Zugangsdaten an —
Credentials gelangen nie in den Chat-/LLM-Kontext.

## Download

Die aktuelle Version liegt unter **[Releases](../../releases/latest)**:

| Plattform | Datei | Empfehlung |
|-----------|-------|------------|
| **Windows — geführter Installer** | `sapb1-mcp-setup-<version>.exe` | **empfohlen** |
| Windows (64-bit) — Binary | `sapb1-mcp-<version>-windows-x64.exe` | manuell/Anpassung |
| Linux (64-bit) — Binary | `sapb1-mcp-<version>-linux-x64` | manuell/Anpassung |

Dazu benötigt ihr eure Lizenzdatei **`versino.key`** (kommt per E-Mail bzw. über das
[Lizenzportal](https://aishop.versino.de)). Siehe [docs/lizenz.md](docs/lizenz.md).

## Schnellstart

### Empfohlen (Windows): geführter Installer

1. `sapb1-mcp-setup-<version>.exe` aus den [Releases](../../releases/latest) laden und starten.
2. Dem Assistenten folgen: `versino.key` wählen, SL-URL + CompanyDBs angeben, Optionen
   setzen — der Installer lädt die Programmdatei, schreibt die `.env`, kopiert die Lizenz
   und richtet auf Wunsch einen Windows-Dienst sowie die Claude-Desktop-Anbindung ein.
   → [docs/installer.md](docs/installer.md)
3. Als Anwender loslegen → [docs/erste-schritte.md](docs/erste-schritte.md).

### Manuell (Windows/Linux): Binary selbst aufsetzen

Für eigene Verzeichnisse, eigenen Dienst-Wrapper, Reverse-Proxy oder Linux:

1. Binary + `versino.key` + `.env` in **einen** Ordner legen.
2. `.env` konfigurieren → [docs/konfiguration.md](docs/konfiguration.md).
3. Starten und an den LLM-Client anbinden:
   - **Windows:** [docs/installation-windows.md](docs/installation-windows.md)
   - **Linux:** [docs/installation-linux.md](docs/installation-linux.md)
   - **Zentral (eine Instanz für alle Arbeitsplätze):** [docs/installation-zentral.md](docs/installation-zentral.md)
4. Als Anwender loslegen → [docs/erste-schritte.md](docs/erste-schritte.md).

## Systemvoraussetzungen

- 64-bit Windows 10/11 / Windows Server 2019+ **oder** 64-bit Linux (glibc, x86-64).
- Netzwerkzugriff auf euren SAP-B1-**Service-Layer** (`https://<host>:50000/b1s/v2/`).
- Gültige `versino.key` (Edition BASIC / PRO / ENTERPRISE).
- Für Nur-stdio-Clients: Node.js (für die `mcp-remote`-Bridge).

## Dokumentation

- [Erste Schritte (für Anwender)](docs/erste-schritte.md) — Nutzung im Chat nach dem Setup
- [**Geführte Installation (Windows, empfohlen)**](docs/installer.md)
- [Installation Windows (manuell)](docs/installation-windows.md)
- [Installation Linux (manuell)](docs/installation-linux.md)
- [Zentrale Installation (eine Instanz für alle Arbeitsplätze)](docs/installation-zentral.md)
- [Konfiguration (`.env`)](docs/konfiguration.md)
- [SSO/ROPC über Keycloak (Kurzanleitung)](docs/sso-keycloak.md)
- [Lizenz (`versino.key`, Erneuerung, Seats)](docs/lizenz.md)
- [Troubleshooting](docs/troubleshooting.md)

## Support

Fragen, Lizenzen, Störungen: **support@versino.de**

## Rechtliches

Es gelten die **[Allgemeinen Geschäftsbedingungen](https://aishop.versino.de/agb)**.

---
© Versino AG. Nutzung gemäß den [AGB](https://aishop.versino.de/agb) und der
Lizenzvereinbarung. Das Binary ist ohne gültige `versino.key` nicht lauffähig.
