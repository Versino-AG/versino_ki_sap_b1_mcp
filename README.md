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

| Plattform | Datei |
|-----------|-------|
| Windows (64-bit) | `sapb1-mcp-<version>-windows-x64.exe` |
| Linux (64-bit)   | `sapb1-mcp-<version>-linux-x64` |

Dazu benötigt ihr eure Lizenzdatei **`versino.key`** (kommt per E-Mail bzw. über das
[Lizenzportal](https://aishop.versino.de)). Siehe [docs/lizenz.md](docs/lizenz.md).

## Schnellstart

1. Binary + `versino.key` + `.env` in **einen** Ordner legen.
2. `.env` konfigurieren → [docs/konfiguration.md](docs/konfiguration.md).
3. Starten und an den LLM-Client anbinden:
   - **Windows:** [docs/installation-windows.md](docs/installation-windows.md)
   - **Linux:** [docs/installation-linux.md](docs/installation-linux.md)

## Systemvoraussetzungen

- 64-bit Windows 10/11 / Windows Server 2019+ **oder** 64-bit Linux (glibc, x86-64).
- Netzwerkzugriff auf euren SAP-B1-**Service-Layer** (`https://<host>:50000/b1s/v2/`).
- Gültige `versino.key` (Edition BASIC / PRO / ENTERPRISE).
- Für Nur-stdio-Clients: Node.js (für die `mcp-remote`-Bridge).

## Dokumentation

- [Installation Windows](docs/installation-windows.md)
- [Installation Linux](docs/installation-linux.md)
- [Konfiguration (`.env`)](docs/konfiguration.md)
- [Lizenz (`versino.key`, Erneuerung, Seats)](docs/lizenz.md)
- [Troubleshooting](docs/troubleshooting.md)

## Support

Fragen, Lizenzen, Störungen: **support@versino.de**

---
© Versino AG. Nutzung gemäß Lizenzvereinbarung. Das Binary ist ohne gültige
`versino.key` nicht lauffähig.
