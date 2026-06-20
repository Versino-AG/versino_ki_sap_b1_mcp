# Erste Schritte (für Anwender)

Diese Anleitung ist für **Anwender**: Der Server ist bereits eingerichtet und euer
LLM-Client (z. B. Claude Desktop) ist mit ihm verbunden. Hier geht es nur darum, wie ihr
SAP Business One im Chat nutzt. Ihr braucht **keine** Technikkenntnisse.

## 1. Mit SAP verbinden
Schreibt im Chat einfach in normaler Sprache, z. B.:

> **Verbinde mich mit SAP.**

Daraufhin meldet sich der Assistent und gibt euch einen **Anmelde-Link** (Browser),
ungefähr so:
```
Bitte melde dich über diesen Link an: https://…/login?t=…
```

1. **Link anklicken** → eine Login-Seite öffnet sich im Browser.
2. Dort euren **SAP-B1-Benutzer + Passwort** eingeben und die **Datenbank (Firma)** wählen.
3. Nach „Login erfolgreich" zurück in den Chat wechseln und kurz **„fertig"** schreiben.

> 🔒 **Wichtig:** Tippt euer SAP-Passwort **nie direkt in den Chat** — immer nur auf der
> Login-Seite im Browser. Der Assistent fragt euch nie nach dem Passwort im Chat.

Die Anmeldung gilt für die laufende Sitzung. Ihr arbeitet mit **euren** SAP-Rechten —
ihr seht und ändert nur, was euer SAP-Benutzer ohnehin darf.

## 2. Fragen stellen (Lesen)
Sobald ihr verbunden seid, fragt einfach in normaler Sprache. Beispiele:

- *„Zeig mir die letzten 10 Angebote."*
- *„Welche offenen Rechnungen hat der Kunde Mustermann GmbH?"*
- *„Finde den Artikel ‚Pumpe 24V'."*
- *„Wie viele Aufträge gab es diesen Monat, gruppiert nach Kunde?"*
- *„Welche Firmen/Datenbanken kann ich auswählen?"*

Der Assistent holt die Daten live aus SAP und antwortet im Chat. Fragt ruhig nach
(„… und davon nur die über 1.000 €") — er verfeinert das Ergebnis.

## 3. Daten anlegen oder ändern (nur mit Schreibrechten)
Ob ihr **schreiben** dürft, hängt von der Einrichtung (Edition + Freischaltung) und euren
SAP-Rechten ab. Falls aktiviert, geht z. B.:

- *„Lege einen neuen Geschäftspartner ‚Beispiel AG' als Kunde an."*
- *„Setze beim Auftrag 1234 das Lieferdatum auf nächsten Freitag."*

> ⚠️ Änderungen wirken **direkt im echten SAP**. Prüft, was der Assistent vorschlägt, bevor
> ihr bestätigt. Sind keine Schreib-Tools verfügbar, ist die Instanz im **Nur-Lese-Modus**
> oder eure Edition erlaubt kein Schreiben.

## 4. Firma/Datenbank wechseln
Mehrere CompanyDBs? Sagt z. B. *„Trenne die Verbindung"* und danach *„Verbinde mich mit
SAP, Datenbank XY"* — oder ruft die Anmeldung erneut auf und wählt eine andere Datenbank.

## 5. Tipps
- **Präzise fragen** liefert bessere Antworten: Zeitraum, Firma, Feld nennen.
- Der Assistent kennt das SAP-Datenmodell — ihr könnt auch fragen *„Welche Felder hat ein
  Geschäftspartner?"* oder *„Erklär mir die Tabelle OCRD."*
- Funktioniert etwas nicht (kein Login-Link, Seite reagiert nicht), schaut in
  [troubleshooting.md](troubleshooting.md) oder meldet euch bei eurer IT.

## Hilfe
Technische Probleme oder Fragen zur Freischaltung: eure IT bzw. **support@versino.de**.
