# The Pocket Briefing

Walk into every sales call like you spent an hour prepping. About 30 minutes before each meeting on your Google Calendar, a Telegram brief lands on your phone: who the guest is, their company, how they found you, your notes, and the last 5 emails you exchanged.

**Flow:** Schedule (every 15 min) -> Google Calendar (meetings starting in 30–45 min) -> extract the external guest -> look them up in the CRM sheet by email -> pull last 5 Gmail messages with them -> build one brief -> send via Telegram. Error Trigger sends a Telegram failure alert.

The 15-minute poll with a 30–45 minute lookahead window means every meeting falls into exactly one window: **one brief per meeting, no duplicates**. Meetings without an external guest (focus blocks, solo events) are skipped silently.

---

## 1. Prerequisites

- n8n 1.x, a Google account (Calendar + Sheets + Gmail), a Telegram bot + your chat ID.
- A Google Sheet with a tab named **`Leads`** (your demo CRM):

| name | email | company | source | notes |
|---|---|---|---|---|
| Maria Lopez | maria@acme.com | Acme Co | webinar signup | Wants the team package, budget approved |

The lookup matches the meeting guest's email against the `email` column.

## 2. Credentials (exact n8n credential types)

| Node(s) | Credential type |
|---|---|
| Get Upcoming Meetings | **Google Calendar OAuth2 API** (`googleCalendarOAuth2Api`) |
| Lookup Lead in CRM | **Google Sheets OAuth2 API** (`googleSheetsOAuth2Api`) |
| Fetch Email History | **Gmail OAuth2** (`gmailOAuth2`) |
| Send Briefing, Telegram Failure Alert | **Telegram API** (`telegramApi`) — bot token from @BotFather |

All three Google credentials can reuse one Google Cloud OAuth client, but they are three separate n8n credentials with different scopes.

> Telegram chat ID: message your bot once, then open `https://api.telegram.org/bot<TOKEN>/getUpdates` and copy `message.chat.id`.

## 3. Config / env values

| Env var | Example | Used for |
|---|---|---|
| `BRIEFING_CALENDAR` | `primary` | Which calendar to watch (`primary` = your main one, or a calendar's email address) |
| `CRM_SHEET_ID` | `1AbC...xyz` | The Leads sheet ID |
| `TELEGRAM_CHAT_ID` | `123456789` | Where briefs and failure alerts go |

The 30–45 minute window lives in the **Get Upcoming Meetings** node (`timeMin` / `timeMax`); the poll rate in **Every 15 Minutes**. Keep window length = poll interval so briefs never duplicate.

## 4. Import

1. **Workflows > Import from File** > `pocket-briefing.workflow.json`.
2. Select credentials on the 5 external nodes.
3. Set the 3 env vars.
4. **Workflow Settings > Error Workflow > The Pocket Briefing** (itself).
5. Save, toggle **Active**.

## 5. Test end to end

1. Add a row to the `Leads` tab for a test guest — use a second email address you control (e.g. `you+guest@gmail.com` won't work for Calendar invites; use a real second account or a colleague).
2. Send that address an email or two from your Gmail so there's history to find.
3. Create a Google Calendar event starting **~35 minutes from now** and invite the test guest.
4. Click **Execute Workflow** (manual run — no need to wait for the schedule).
5. Expected: a Telegram message like:

```
📋 Briefing: Intro call
🕐 When: 2026-07-09T15:35:00+08:00
👤 Who: Maria Lopez (maria@acme.com)
🏢 Company: Acme Co
🔎 How they found you: webinar signup
📝 Notes: Wants the team package, budget approved
✉️ Recent emails:
• Re: pricing question - Thanks for sending that over, I'll review with...
```

6. Unknown-guest case: invite an address that's NOT in the sheet — the brief still arrives with "Company: not in CRM".
7. Failure alert: temporarily break `CRM_SHEET_ID` **and** disable Continue On Fail on the Sheets node, run, expect the ⚠️ Telegram alert. Restore both.

## 6. SWAP POINTS

- **SWAP POINT 1 — Calendar (Google -> Outlook / Calendly):** replace **Get Upcoming Meetings** with Microsoft Outlook "Get Many events" using the same 30–45 min window, or a Calendly booking webhook (then the brief sends at booking time instead).
- **SWAP POINT 2 — CRM (Sheets -> HubSpot / Pipedrive):** replace **Lookup Lead in CRM** with HubSpot "Search Contact" / Pipedrive "Search Persons" by the guest email; map to `name, email, company, source, notes`.
- **SWAP POINT 3 — Email history (Gmail -> Outlook):** replace **Fetch Email History** with Outlook "Get Many messages" filtered by the guest address, or your CRM's email timeline.
- **SWAP POINT 4 — Delivery (Telegram -> Slack DM / SMS):** replace **Send Briefing**; the finished text is in `{{ $json.text }}`.

## 7. Notes and limits (kept intentionally simple)

- Handles the common case of **one meeting per 15-minute window** cleanly. Two meetings starting in the same window will produce briefs but may cross-mix email history; rare enough to ignore for a demo (fix: wrap the lookup chain in a Loop Over Items node).
- Only the **first external guest** is briefed. Multi-guest meetings get one brief for the first non-you attendee.
- CRM lookup, Gmail search, and Calendar all have Continue On Fail + Always Output Data where it matters, so a missing lead or empty inbox degrades the brief instead of killing it.
- Times shown come straight from Google Calendar (includes timezone offset). Check your n8n `GENERIC_TIMEZONE` if the window seems off.
