# Invoice Nag Bot

A minimal n8n automation that emails friendly payment reminders for overdue, unpaid invoices once a day, then records that the reminder was sent.

**Flow:** Schedule (9:00 AM daily) -> Read Google Sheet -> Filter (not paid AND overdue) -> Send Gmail reminder -> Update sheet (`reminders_sent` +1, `last_reminded` = today). A separate Error Trigger sends a Telegram alert if the run fails.

Paid invoices, and invoices not yet due, are dropped by the filter and are never contacted.

---

## 1. Prerequisites

- An n8n instance on **1.x** (cloud or self-hosted).
- A Google account with a Google Sheet of invoices.
- A Gmail account to send from.
- A Telegram bot + your chat ID (for failure alerts).

### The Google Sheet

Create a sheet with a tab named **`Invoices`** and this exact header row (row 1):

| client_name | email | invoice_no | amount | due_date | status | reminders_sent | last_reminded |
|-------------|-------|-----------|--------|----------|--------|----------------|---------------|

Notes:
- `due_date` must be a real date, ideally `YYYY-MM-DD` (e.g. `2026-07-01`).
- `status` uses `paid` for settled invoices (any other value, e.g. `unpaid`/`overdue`/blank, is treated as unpaid). Matching is case-insensitive.
- `reminders_sent` should be a number (blank is treated as 0).
- `last_reminded` can start blank; the bot fills it.

---

## 2. Credentials to configure

Create these in **n8n > Credentials**, then open each node in the workflow and select them (the imported nodes point at placeholder IDs and will show as "not set" until you pick yours).

| Node(s) | n8n credential type | What it needs |
|---------|---------------------|---------------|
| Read Invoices, Mark Reminder Sent | **Google Sheets OAuth2 API** (`googleSheetsOAuth2Api`) | Google OAuth client with Sheets access |
| Send Reminder Email | **Gmail OAuth2** (`gmailOAuth2`) | Google OAuth client with Gmail send scope |
| Telegram Failure Alert | **Telegram API** (`telegramApi`) | Bot token from @BotFather |

> Google Sheets and Gmail can reuse the same Google Cloud OAuth client, but they are **two separate n8n credentials** with different scopes. Create both.

---

## 3. Config / environment values

Set these as environment variables on your n8n instance (the workflow reads them, so no secrets or IDs are hardcoded):

| Env var | Example | Used for |
|---------|---------|----------|
| `INVOICE_SHEET_ID` | `1AbC...xyz` | The Google Sheet ID (the long string in the sheet URL between `/d/` and `/edit`) |
| `TELEGRAM_CHAT_ID` | `123456789` | Where failure alerts are sent |
| `FROM_NAME` | `Jane at Acme Co` | Signature name at the bottom of the reminder email |

**Docker example:**

```bash
docker run -it --rm \
  -e INVOICE_SHEET_ID="your_sheet_id" \
  -e TELEGRAM_CHAT_ID="your_chat_id" \
  -e FROM_NAME="Jane at Acme Co" \
  -p 5678:5678 docker.n8n.io/n8nio/n8n
```

On **n8n Cloud**, set these under **Variables/Environment** (or replace `{{ $env.X }}` in the nodes with literal values if your plan lacks env vars).

The tab name `Invoices` is set directly in the two Google Sheets nodes. Change it there if your tab has a different name.

---

## 4. How to import

1. In n8n: **Workflows > Import from File** (or the `...` menu > **Import from File**).
2. Choose `invoice-nag-bot.workflow.json`.
3. Open each node marked "not set" and select your credential (see section 2).
4. Confirm the env vars from section 3 are set.
5. **Enable failure alerts:** open **Workflow Settings** (top-right menu) > **Error Workflow** > select **Invoice Nag Bot** (itself). Without this, the Telegram alert will not fire.
6. Save.

---

## 5. How to test with sample data

1. Add a couple of test rows to the `Invoices` tab, for example:

   | client_name | email | invoice_no | amount | due_date | status | reminders_sent | last_reminded |
   |-------------|-------|-----------|--------|----------|--------|----------------|---------------|
   | Test Overdue | your.own+test@gmail.com | INV-001 | $250.00 | 2020-01-01 | unpaid | 0 | |
   | Test Paid | your.own+test@gmail.com | INV-002 | $99.00 | 2020-01-01 | paid | 0 | |
   | Test Future | your.own+test@gmail.com | INV-003 | $50.00 | 2099-01-01 | unpaid | 0 | |

   Use an inbox you control as the `email` so the test emails come to you.

2. Click **Execute Workflow** (manual run, no need to wait for 9 AM).
3. Expected result:
   - You receive **one** email (for `INV-001` only).
   - `INV-002` (paid) and `INV-003` (not yet due) are skipped.
   - In the sheet, `INV-001` now shows `reminders_sent = 1` and `last_reminded` = today.
4. **Test the failure alert:** temporarily break something (e.g. set `INVOICE_SHEET_ID` to a bad value) and run again. You should get a Telegram message from the Error Trigger. Restore the value afterward.
5. When happy, toggle the workflow **Active** so the 9:00 AM schedule runs it daily.

---

## 6. SWAP POINTS (demo -> real tools)

The workflow has labeled sticky notes at each swap point.

**SWAP POINT 1 - Data source (Google Sheets -> QuickBooks / Xero / Wave):**
- Replace the **Read Invoices** and **Mark Reminder Sent** nodes with your invoicing tool's nodes (or HTTP Request to their API).
- On read, map the tool's fields to these item keys: `client_name, email, invoice_no, amount, due_date, status, reminders_sent, last_reminded`. As long as those keys exist on each item, the filter and email need no changes.
- On write-back, instead of updating a sheet row, update the invoice/contact record in your tool (or store the "last reminded" state wherever you track it).

**SWAP POINT 2 - Email provider (Gmail -> Outlook / SMTP / SendGrid / Mailgun):**
- Replace the **Send Reminder Email** node with your provider's send node.
- Keep `sendTo` = `{{ $json.email }}`, the same subject, and the same warm, no-em-dash body copy.

---

## 7. Notes and limits (kept intentionally simple)

- **Continue On Fail** is enabled on the two Sheets nodes and the Gmail node, so one bad row (e.g. an invalid email) will not stop the rest of the batch.
- Because of that, if a single email send fails, the sheet row may still get `reminders_sent` incremented. For a demo this is acceptable; if you need strict accuracy, add an IF node after Gmail to only update the sheet when the send succeeded.
- The bot does **not** de-duplicate by day. If you run it twice in one day, overdue rows get reminded twice. Run it once daily (the schedule handles this) or add a check on `last_reminded` if needed.
- `due_date` equal to *today* is treated as **not yet overdue** (filter uses strictly "before today").
