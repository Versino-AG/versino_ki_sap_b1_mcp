<!-- translation-of: neuigkeiten.md@1f2abddb4424 -->
# Neuigkeiten

> 🌐 [English](neuigkeiten.md) · **Deutsch** · [Česky](neuigkeiten.cs.md)

Die wichtigsten Änderungen je Version, in einfachen Worten. Die laufende Version
zeigt `sapb1-mcp --version`, und der Assistent kennt sie auch (fragt *„Welche
Version läuft?"*).

## 3.8.5 — 2026-10-06

- **Ein zweiter Upload- oder Login-Link öffnet den zweiten Link.** Bisher konnte
  die Seite beim ersten stehen bleiben — im normalen Browser-Tab und vor allem im
  integrierten Browser von Claude Cowork. Jeder Link hat jetzt eine eigene
  Adresse.

## 3.8.4 — 2026-10-02

- **Auswertungen auch im Lesebetrieb.** Mit `SAP_READ_ONLY_DEPLOY_QUERIES=true`
  bringt eine `READ_ONLY`-Instanz die Auswertungen selbst aus — auch eure eigenen
  aus dem Abfrage-Manager. Beschrieben wird nur der Auswertungsspeicher; jede
  andere Änderung an SAP bleibt gesperrt → [konfiguration.de.md](konfiguration.de.md).

## 3.8.3 — 2026-09-30

- **Eine abgelehnte Lizenz wird erklärt statt „Internal Server Error".** Lehnt
  der Lizenzserver die Installation ab oder sind alle Plätze belegt, sagt die
  Anmeldeseite das jetzt. Für den häufigsten Fall — die Installation ist für
  einen anderen Kunden registriert als die jetzt hinterlegte Lizenz — nennt sie
  die Abhilfe → [troubleshooting.de.md](troubleshooting.de.md).

## 3.8.2 — 2026-09-30

- **Auswertungen im SAP-Abfrage-Manager anpassen.** Eine mitgelieferte
  Auswertung dort unter demselben Namen kopieren, das benötigte Feld ergänzen
  (etwa ein UDF), speichern — der Assistent nutzt eure Fassung. Neue
  Auswertungen gehen genauso → [konfiguration.de.md](konfiguration.de.md).
- **Belegzeilen sicher entfernen.** *„Entferne Position 3 aus dem Angebot 4711"*
  geht jetzt bei offenen Angeboten, Aufträgen und Einkaufsbelegen, eine
  Verkaufsstückliste als Ganzes → [erste-schritte.de.md](erste-schritte.de.md).
- **Ein zweiter Anhang ersetzt den ersten nicht mehr**, und ein Upload-Link nimmt
  mehrere Dateien → [anhaenge.de.md](anhaenge.de.md).
- **Ein Platz wird wieder frei** nach 30 Minuten ohne Aktivität, auch wenn der
  Client einfach geschlossen wurde → [lizenz.de.md](lizenz.de.md).
- **Seriennummern auf Lager je Lager** als neue Auswertung; Änderungen an
  Belegen scheitern nicht mehr an einem falschen „seit dem Lesen geändert".

## 3.8.1 — 2026-09-21

- **Sicherheitshärtung** bei Anmeldung, Uploads und Lizenzprüfung.
- **Die laufende Version ist sichtbar**: `--version`, die Startmeldung und die
  Hilfe des Assistenten.
- **Sitzungen über den Browser-Login enden nach einer Stunde** (statt nach acht)
  — dann erneut mit `connect` anmelden.
- **`SAP_ALLOWED_CLIENTS`** begrenzt, welche Adressen den Server überhaupt
  erreichen → [konfiguration.de.md](konfiguration.de.md).
- **Der Uhrenschutz der Lizenz ist standardmäßig an**; die Ankerdatei liegt neben
  `versino.key` → [lizenz.de.md](lizenz.de.md).
