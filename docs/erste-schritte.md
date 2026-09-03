# Getting started (for end users)

> 🌐 **English** · [Deutsch](erste-schritte.de.md) · [Česky](erste-schritte.cs.md)

This guide is for **end users**: the server is already set up and your LLM
client (e.g. Claude Desktop) is connected to it. It only covers how you use
SAP Business One in the chat. You need **no** technical knowledge.

## 1. Connect to SAP
Just write in the chat in plain language, e.g.:

> **Connect me to SAP.**

The assistant then gives you a **sign-in link** (browser). Depending on how your
system is set up, you will see:

- **The assistant's sign-in page** (classic login): enter your
  **SAP B1 user + password** there and choose the **database (company)**.
- **Single sign-on:** with several companies a small page
  **"Choose database"** appears first — pick the company, click
  **"Continue to sign-in"**. After that (or directly, if there is only one
  company) you land on your **company's central sign-in page**: sign in there
  with your **e-mail address** — like with other company applications; often
  you are already signed in and it is just one click. The assistant **never**
  asks for user or password.

Tip: you can also name the company right in the chat
(*"Connect me to SAP, database XY"*) — the chooser page is then skipped.
Afterwards switch back to the chat and briefly write **"done"**.

> 🔒 **Important:** never type your SAP password **directly into the chat** —
> only on the sign-in page in the browser. The assistant never asks for the
> password in the chat.

The sign-in lasts for the current session. You work with **your** SAP
permissions — you only see and change what your SAP user may anyway.

## 2. Asking questions (reading)
Once connected, just ask in plain language. Examples:

- *"Show me the last 10 quotations."*
- *"Which open invoices does the customer Mustermann GmbH have?"*
- *"Find the item 'Pump 24V'."*
- *"How many orders were there this month, grouped by customer?"*
- *"Which companies/databases can I choose?"*

The assistant fetches the data live from SAP and answers in the chat. Feel free
to follow up ("… and of those only the ones above €1,000") — it refines the
result.

## 3. Creating or changing data (write permissions only)
Whether you may **write** depends on the setup (edition + activation) and your
SAP permissions. If enabled, things like this work:

- *"Create a new business partner 'Beispiel AG' as a customer."*
- *"Set the delivery date of order 1234 to next Friday."*

> ⚠️ Changes take effect **directly in the real SAP**. Review what the assistant
> proposes before you confirm. If no write tools are available, the instance is
> in **read-only mode** or your edition does not allow writing.

## 4. Switching company/database
Several CompanyDBs? Say e.g. *"Disconnect"* and then *"Connect me to SAP,
database XY"*. With single sign-on your browser session usually still exists —
the second sign-in is then just one click. A session is always connected to
**one** database.

## 5. Tips
- **Precise questions** get better answers: name the period, company, field.
- The assistant knows the SAP data model — you can also ask *"Which fields does
  a business partner have?"* or *"Explain the table OCRD to me."*
- If something does not work (no sign-in link, page not responding), check
  [troubleshooting.md](troubleshooting.md) or contact your IT.

## Help
Technical problems or activation questions: your IT or **support@versino.de**.
