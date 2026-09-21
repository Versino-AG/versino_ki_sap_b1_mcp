<!-- translation-of: lizenz.md@d11b4e7994d7 -->
# Lizenz (`versino.key`)

> 🌐 [English](lizenz.md) · **Deutsch** · [Česky](lizenz.cs.md)

Der SAP-B1-MCP-Server ist lizenzpflichtig. Die Lizenz ist eine **signierte
Offline-Datei** (`versino.key`) — ohne gültige Lizenz startet der Server nicht
(fail-closed).

## Lizenz beziehen
Nach dem Kauf bzw. über das **[Lizenzportal](https://aishop.versino.de)** erhaltet ihr
die `versino.key` (auch per E-Mail). Sie enthält:
- **Edition**: BASIC / PRO / ENTERPRISE (bestimmt die verfügbaren Tools),
- **max. Seats**: Anzahl distinkter SAP-Nutzer,
- **Ablaufdatum**.

## Lizenz anwenden
Die `versino.key` **neben das Binary** legen (gleicher Ordner wie `sapb1-mcp.exe`
bzw. `sapb1-mcp`). Sie wird automatisch gefunden — `SAP_LICENSE_FILE` muss **nicht**
gesetzt werden. Alternativ einen expliziten Pfad via `SAP_LICENSE_FILE` angeben.

## Editionen (Kurzüberblick)
| Edition | Liest | Tools zusätzlich zur Anmeldung und `sap_help` |
|---|---|---|
| BASIC | nur Stammdaten (Geschäftspartner, Artikel, …), begrenzte Query-Rate | `sap_query_odata`, `sap_fuzzy_search` |
| PRO | alle Daten inkl. Belege, höhere Rate | zusätzlich `sap_curated_query` (die mitgelieferten `AI_*`-Auswertungen), `sap_semantic_query`, `sap_attachment`, `sap_deploy_queries`, `sap_create`, `sap_update` |
| ENTERPRISE | alle Daten, ohne Ratenbegrenzung | zusätzlich `sap_delete`, `sap_action` |

Ein Tool, das eine Edition nie ausführen könnte, wird **gar nicht angeboten** —
der Assistent sieht es nicht und kann deshalb nichts vorschlagen, was die Lizenz
nicht abdeckt. Tools, die beliebige Tabellen lesen (die kuratierten
`AI_*`-Auswertungen, Semantic-Layer-Views, Anhänge), brauchen daher eine Edition
mit vollem Lesezugriff; mit BASIC werden die mitgelieferten Auswertungen auch
nicht ausgebracht, weil sie dort nicht laufen könnten. `sap_help` nennt je
fehlendem Tool den Grund (`edition`, `read_scope`, `operation_mode`), damit der
Support die Frage „warum kann es X nicht" an einer Stelle beantworten kann.
Unabhängig von der Edition entfernt `SAP_OPERATION_MODE=READ_ONLY` die
Schreib-Tools → [konfiguration.de.md](konfiguration.de.md).

## Phone-Home (Manipulationsschutz, Gültigkeitsprüfung & Abo-Erneuerung)
**Standardmäßig aktiv:** die Installation baut in einem festen Intervall eine
signierte Verbindung zum Lizenzserver auf. Das erfüllt zwei Zwecke:

- **Gültigkeits-/Sperrprüfung (Manipulationsschutz):** Der Server bestätigt, dass die
  Lizenz weiterhin gültig ist. Eine serverseitig **gesperrte** Lizenz (z. B. bei Missbrauch
  oder Zahlungsausfall) wird so erkannt — neue Verbindungen werden dann abgelehnt.
- **Automatische Abo-Erneuerung:** Bei monatlicher/jährlicher Abrechnung re-signiert der
  Server den Token mit neuem Ablaufdatum und liefert ihn über **denselben** Kanal aus; die
  Installation übernimmt ihn selbst — **kein manueller Key-Tausch** pro Abrechnungszeitraum.

Übertragen werden dabei nur Metadaten (`customer_id`, `edition`, `version`, Seat-Anzahl) —
keine Geschäftsdaten.

Aktivierung: **nichts einzutragen** — der Enrollment-Token ist in eure `versino.key`
**eingebacken** (signiert), und die Validation-URL ist fest ins Produkt eingebaut.
Sobald die `versino.key` neben dem Binary liegt, registriert sich die Installation
beim ersten Start selbst und übernimmt Verlängerungen automatisch. Optional lässt
sich ein beschreibbarer Renewal-Cache-Pfad setzen (Default: `versino.renewed` neben
dem Binary):
```ini
# optional: beschreibbarer Pfad → übernimmt erneuerte Token
SAP_LICENSE_CACHE_FILE=versino.renewed
```
(Nur für Test/Staging: `SAP_ENROLLMENT_TOKEN` überschreibt den eingebauten Token,
`SAP_LICENSE_VALIDATION_URL` den eingebauten Endpoint.)

> **Der Endpoint muss `https` sein.** Die Phone-Home-Anfrage trägt Kundennummer und
> Seat-Zahl, deshalb wird einfaches `http` gegen einen entfernten Host abgewiesen —
> unverschlüsselt erlaubt sind nur `127.0.0.1`, `::1` und `localhost`, für die
> Entwicklung. Ein selbst betriebener Lizenzserver braucht also TLS; zeigt
> `SAP_LICENSE_VALIDATION_URL` auf eine `http://`-Adresse auf einer anderen Maschine,
> scheitert das, statt Kundendaten still im Klartext zu senden. Die
> Enrollment-Adresse wird aus dem Verzeichnis dieser URL abgeleitet — den Pfad
> (`…/validate`) daher unverändert lassen.

> **Mindestversion 2.3.5.** Der eingebackene Token ist ein zusätzliches Feld in
> der signierten Lizenz. Ältere Programmversionen kennen dieses Feld nicht und
> lehnen so einen Schlüssel ab — sie starten damit nicht. Bekommt ihr eine neue
> `versino.key` und fahrt noch eine Version vor 2.3.5, **erst die Software
> aktualisieren, dann den Schlüssel tauschen**. Die alte `versino.key` bleibt bis
> zu ihrem Ablaufdatum gültig, ihr könnt sie also jederzeit wieder einlegen.

**Container-Betrieb:** Die Installations-Identität landet standardmäßig als
`install_identity.json` im Arbeitsverzeichnis. Läuft der Container mit
`read_only: true` oder auf einem flüchtigen Verzeichnis, zeigt
`SAP_INSTALL_IDENTITY_PATH` auf einen **persistenten** Pfad (Volume) — sonst
entsteht bei jedem Start ein neues Schlüsselpaar und damit eine neue Installation:
```ini
SAP_INSTALL_IDENTITY_PATH=/data/install_identity.json
SAP_LICENSE_CACHE_FILE=/data/versino.renewed
```

Ob die Registrierung geklappt hat, sagt euch `sapb1-mcp doctor` (Prüfpunkt
`enrollment`).

**Air-gapped / ohne Phone-Home:** In einer Umgebung ohne Internet gibt es weder
Online-Gültigkeitsprüfung noch automatische Erneuerung — der ausgelieferte Token
gilt unverändert bis zu seinem Ablaufdatum, und ihr erhaltet rechtzeitig vorher
eine neue `versino.key`. Wer das Phone-Home gar nicht möchte, kann es über Versino
abschalten lassen; ihr bekommt dann Schlüssel ohne Registrierungsmerkmal (und
damit ohne automatische Erneuerung).

## Seats
`max_seats` begrenzt die Anzahl **distinkter** SAP-Nutzer. Werden mehr benötigt, über
das Lizenzportal aufstocken — die neue `versino.key` ersetzt die alte.

Fragen zur Lizenz: **support@versino.de**
