# The Ghost Chaser

Recovers deals that went quiet. Every morning at 8:00 AM it scans your proposals sheet for quotes with no reply after 3 days and drops a warm, ready-to-send follow-up **draft** into your Gmail drafts folder. You review, hit send, done. Nothing is emailed automatically.

**Flow:** Schedule (8 AM) -> Read Proposals sheet -> Filter (status "sent" AND sent 3+ days ago AND not chased in last 3 days) -> Create Gmail draft -> stamp `last_followup` = today. Error Trigger sends a Telegram alert on failure.

Replied, won, or lost proposals are never chased. The `last_followup` stamp prevents duplicate drafts on consecutive days.

---

## 1. Prerequisites

- n8n 1.x, a Google account, a Gmail account, a Telegram bot + chat ID.
- A Google Sheet with a tab named **`Proposals`** and this header row:

| client_name | email | proposal_no | amount | sent_date | status | last_followup |
|---|---|---|---|---|---|---|

- `sent_date` as `YYYY-MM-DD`. `status` is one of `sent` / `replied` / `won` / `lost` (only `sent` gets chased; case-insensitive). `last_followup` starts blank; the bot fills it.

## 2. Credentials (exact n8n credential types)

| Node(s) | Credential type |
|---|---|
| Read Proposals, Stamp Last Followup | **Google Sheets OAuth2 API** (`googleSheetsOAuth2Api`) |
| Create Gmail Draft | **Gmail OAuth2** (`gmailOAuth2`) |
| Telegram Failure Alert | **Telegram API** (`telegramApi`) — bot token from @BotFather |

After import, open each node marked "not set" and select your credential.

## 3. Config / env values

| Env var | Example | Used for |
|---|---|---|
| `PROPOSALS_SHEET_ID` | `1AbC...xyz` | Sheet ID from the URL between `/d/` and `/edit` |
| `TELEGRAM_CHAT_ID` | `123456789` | Failure alerts |
| `FROM_NAME` | `Jane` | Signature in the draft |

Docker: pass with `-e`. n8n Cloud: set under Variables/Environment, or replace `{{ $env.X }}` with literal values. The tab name `Proposals` and the "3 days" thresholds are set in the nodes (Filter node conditions) if you want to change them.

## 4. Import

1. **Workflows > Import from File** > `ghost-chaser.workflow.json`.
2. Select credentials on the 4 external nodes.
3. Set env vars.
4. **Workflow Settings > Error Workflow > The Ghost Chaser** (itself) — required for the Telegram alert.
5. Save, toggle **Active** for the daily 8 AM run.

## 5. Test with sample data

Add these rows (use your own email):

| client_name | email | proposal_no | amount | sent_date | status | last_followup |
|---|---|---|---|---|---|---|
| Ghost Test | you@x.com | PROP-001 | $3,000 | 2026-01-01 | sent | |
| Replied Test | you@x.com | PROP-002 | $500 | 2026-01-01 | replied | |
| Fresh Test | you@x.com | PROP-003 | $900 | (today's date) | sent | |

Click **Execute Workflow**. Expected: exactly **one draft** appears in your Gmail Drafts folder (PROP-001 only; PROP-002 is replied, PROP-003 is too fresh), and PROP-001's `last_followup` = today in the sheet. Run it again immediately: **no new draft** (already chased within 3 days). Test the failure alert by temporarily breaking `PROPOSALS_SHEET_ID` and disabling Continue On Fail on Read Proposals; expect a Telegram ⚠️.

## 6. SWAP POINTS

- **SWAP POINT 1 — Pipeline source (Sheets -> HubSpot / Pipedrive):** replace **Read Proposals** with your CRM's "get deals" node filtered to the proposal-sent stage, mapped to the 7 field names above; replace **Stamp Last Followup** with a CRM deal-property update.
- **SWAP POINT 2 — Email (Gmail -> Outlook / SMTP):** replace **Create Gmail Draft**. Keep it as a draft for human review, or switch the Gmail node's resource to Message > Send for fully automatic follow-ups.

## 7. Notes

- Continue On Fail is on for Sheets and Gmail, so one bad row never stops the batch; a failed draft may still get its row stamped (acceptable for a demo — add an IF after Gmail for strict accuracy).
- The showcase math: change a row's `status` to `replied` and it drops out of the chase instantly.
