<!-- translation-of: erste-schritte.md@e26e7d2321e1 -->

# První kroky (pro uživatele)

> 🌐 [English](erste-schritte.md) · [Deutsch](erste-schritte.de.md) · **Česky**

Tento návod je pro **uživatele**: server je už nastavený a váš LLM klient
(např. Claude Desktop) je s ním propojený. Jde tu jen o to, jak používat
SAP Business One v chatu. Nepotřebujete **žádné** technické znalosti.

## 1. Připojení k SAP
V chatu napište prostě normální řečí, např.:

> **Připoj mě k SAP.**

Asistent vám na to vrátí **přihlašovací odkaz** (prohlížeč). Podle nastavení
vašeho systému tam uvidíte:

- **Přihlašovací stránku asistenta** (klasické přihlášení): tam zadáte svého
  **SAP B1 uživatele + heslo** a vyberete **databázi (firmu)**.
- **Single Sign-On:** Při více firmách se nejdřív zobrazí malá stránka
  **„Vybrat databázi"** — vyberte firmu, klikněte na **„Pokračovat na přihlášení"**.
  Poté (nebo rovnou, pokud je jen jedna firma) se dostanete na **centrální
  přihlašovací stránku vaší firmy**: tam se přihlásíte svou
  **e-mailovou adresou** — stejně jako u jiných firemních aplikací; často už
  jste přihlášení a stačí jeden klik. Uživatele ani heslo se vás asistent
  **nikdy** neptá.

Tip: Firmu můžete rovnou zmínit v chatu
(*„Připoj mě k SAP, databáze XY"*) — pak odpadne výběrová stránka.
Poté se vraťte zpět do chatu a napište krátce **„hotovo"**.

> 🔒 **Důležité:** Své SAP heslo **nikdy nepište přímo do chatu** — vždy jen na
> přihlašovací stránce v prohlížeči. Asistent se vás na heslo v chatu nikdy neptá.

Přihlášení platí pro běžící relaci. Pracujete se **svými** SAP oprávněními —
vidíte a měníte jen to, co váš SAP uživatel smí i jinak.

## 2. Kladení dotazů (čtení)
Jakmile jste připojeni, ptejte se prostě normální řečí. Příklady:

- *„Ukaž mi posledních 10 nabídek."*
- *„Jaké má otevřené faktury zákazník Novák s.r.o.?"*
- *„Najdi artikl ‚Čerpadlo 24V'."*
- *„Kolik bylo objednávek tento měsíc, seskupených podle zákazníka?"*

Asistent živě natáhne data ze SAP a odpoví v chatu. Klidně se doptávejte
(„… a z toho jen ty nad 1 000 Kč") — výsledek pak upřesní.

## 3. Vytváření nebo úprava dat (jen s právy na zápis)
Zda smíte **zapisovat**, závisí na nastavení (edice + odemčení) a vašich
SAP oprávněních. Pokud je to aktivované, jde třeba:

- *„Založ nového obchodního partnera ‚Příklad a.s.' jako zákazníka."*
- *„U zakázky 1234 nastav datum dodání na příští pátek."*

> ⚠️ Změny se projeví **přímo v reálném SAP**. Než potvrdíte, zkontrolujte, co
> asistent navrhuje. Pokud nejsou k dispozici žádné zápisové nástroje, je instance
> v **režimu jen pro čtení**, nebo vaše edice zápis nepovoluje.

## 4. Přepnutí firmy/databáze
Máte více CompanyDB? Řekněte např. *„Odpoj se"* a poté *„Připoj mě k SAP,
databáze XY"*. Při Single Sign-On vaše přihlášení v prohlížeči většinou pořád
platí — druhé přihlášení je pak jen jeden klik. Jedna relace je vždy
připojená k **jedné** databázi.

## 5. Tipy
- **Přesné dotazy** dávají lepší odpovědi: uveďte období, firmu, pole.
- Asistent zná datový model SAP — můžete se i zeptat *„Jaká pole má obchodní
  partner?"* nebo *„Vysvětli mi tabulku OCRD."*
- Když něco nefunguje (žádný login link, stránka nereaguje), podívejte se do
  [troubleshooting.cs.md](troubleshooting.cs.md) nebo se ozvěte svému IT.

## Pomoc
Technické problémy nebo dotazy k odemčení funkcí: vaše IT, resp. **support@versino.de**.
