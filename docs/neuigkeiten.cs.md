<!-- translation-of: neuigkeiten.md@378f9ffb6ea5 -->
# Novinky

> 🌐 [English](neuigkeiten.md) · [Deutsch](neuigkeiten.de.md) · **Česky**

Nejdůležitější změny v každé verzi, jednoduše. Běžící verzi ukáže
`sapb1-mcp --version` a asistent ji zná také (zeptejte se *„Jaká verze běží?"*).

## 3.8.4 — 2026-10-02

- **Reporty i v režimu čtení.** S `SAP_READ_ONLY_DEPLOY_QUERIES=true` instance v
  režimu `READ_ONLY` sama nasadí reporty — i vaše vlastní ze Správce dotazů.
  Zapisuje se jen do úložiště reportů; každá jiná změna v SAP zůstává zakázaná
  → [konfiguration.cs.md](konfiguration.cs.md).

## 3.8.3 — 2026-09-30

- **Odmítnutá licence je vysvětlena místo „Internal Server Error".** Když
  licenční server instalaci odmítne nebo jsou obsazena všechna místa, řekne to
  teď přihlašovací stránka. Pro nejčastější případ — instalace je registrovaná
  pro jiného zákazníka než nyní uložená licence — uvede řešení
  → [troubleshooting.cs.md](troubleshooting.cs.md).

## 3.8.2 — 2026-09-30

- **Úprava reportů ve Správci dotazů SAP.** Dodaný report tam zkopírujte pod
  stejným názvem, doplňte potřebné pole (např. UDF), uložte — asistent použije
  vaši verzi. Nové reporty fungují stejně → [konfiguration.cs.md](konfiguration.cs.md).
- **Bezpečné odebrání řádků dokladu.** *„Odeber řádek 3 z nabídky 4711"* teď
  funguje u otevřených nabídek, zakázek a nákupních dokladů, prodejní kusovník
  jako celek → [erste-schritte.cs.md](erste-schritte.cs.md).
- **Druhá příloha už nenahradí první** a jeden odkaz pro nahrání přijme více
  souborů → [anhaenge.cs.md](anhaenge.cs.md).
- **Místo se znovu uvolní** po 30 minutách nečinnosti, i když byl klient jen
  zavřen → [lizenz.cs.md](lizenz.cs.md).
- **Sériová čísla na skladě podle skladu** jako nový report; změny dokladů už
  neselhávají na falešném „od načtení změněno".

## 3.8.1 — 2026-09-21

- **Bezpečnostní zpevnění** přihlášení, nahrávání a kontroly licence.
- **Běžící verze je vidět**: `--version`, úvodní zpráva a nápověda asistenta.
- **Relace přes přihlášení v prohlížeči končí po jedné hodině** (místo osmi)
  — pak se znovu přihlaste přes `connect`.
- **`SAP_ALLOWED_CLIENTS`** omezuje, které adresy se k serveru vůbec dostanou
  → [konfiguration.cs.md](konfiguration.cs.md).
- **Ochrana hodin pro licenci je ve výchozím stavu zapnutá**; soubor kotvy leží
  vedle `versino.key` → [lizenz.cs.md](lizenz.cs.md).
