# The Morning Numbers

One clean Slack summary every morning at 8: yesterday's ad spend, leads, and cost per lead by source, plus which deals moved. Kills the daily copy-paste hour and surfaces a broken ad in one day instead of three weeks.

**Flow:** Schedule (8 AM) -> read `AdNumbers` tab -> read `Deals` tab -> Code node computes per-source spend/leads/CPL + totals + deal movements for yesterday -> post to Slack. Error Trigger posts a Slack failure alert.

The demo uses **Google Sheets tabs standing in for Meta Ads, Google Ads, and your CRM** so a coach can run it in 10 minutes with zero ad-platform setup. Section 6 shows how to wire the real sources.

---

## 1. Prerequisites

- n8n 1.x, a Google account, a Slack workspace + bot (scopes `chat:write`, `chat:write.public`).
- A Google Sheet with **two tabs**:

**Tab `AdNumbers`** (one row per source per day):

| date | source | spend | leads |
|---|---|---|---|
| 2026-07-08 | meta | 180.50 | 12 |
| 2026-07-08 | google | 95.00 | 4 |

**Tab `Deals`** (one row per stage move):

| date | deal_name | from_stage | to_stage |
|---|---|---|---|
| 2026-07-08 | Acme Co | Proposal | Won |

Dates as `YYYY-MM-DD`. The workflow only reads rows dated **yesterday**.

## 2. Credentials (exact n8n credential types)

| Node(s) | Credential type |
|---|---|
| Read Ad Numbers, Read Deals | **Google Sheets OAuth2 API** (`googleSheetsOAuth2Api`) |
| Post Daily Numbers, Slack Failure Alert | **Slack API** (`slackApi`) |

## 3. Config / env values

| Env var | Example | Used for |
|---|---|---|
| `NUMBERS_SHEET_ID` | `1AbC...xyz` | The Google Sheet ID |
| `SLACK_NUMBERS_CHANNEL` | `daily-numbers` | Slack channel (no `#`) |

Tab names (`AdNumbers`, `Deals`) and the 8:00 AM time are set in the nodes.

## 4. Import

1. **Workflows > Import from File** > `morning-numbers.workflow.json`.
2. Select credentials on the 4 external nodes.
3. Set the 2 env vars.
4. **Workflow Settings > Error Workflow > The Morning Numbers** (itself).
5. Save, toggle **Active**.

## 5. Test with sample data

1. Add rows to both tabs **dated yesterday** (copy the tables above, adjust the date).
2. Click **Execute Workflow**.
3. Expected Slack message in your channel:

```
☀️ Morning Numbers, 2026-07-08

Ad performance:
  • meta: $180.50 spend, 12 leads, CPL $15.04
  • google: $95.00 spend, 4 leads, CPL $23.75

Totals: $275.50 spend | 16 leads | CPL $17.22

Deals moved (1):
  • Acme Co: Proposal -> Won
```

4. Delete yesterday's ad rows and run again — the summary says "No ad data found for yesterday. Check the sources!" (that line IS the broken-ad early warning).
5. Failure alert: break `NUMBERS_SHEET_ID` and disable Continue On Fail on Read Ad Numbers, run, expect ⚠️ in Slack. Restore after.

## 6. SWAP POINTS (demo tabs -> real sources)

Everything downstream of the reads only needs items shaped `{ date, source, spend, leads }` and `{ date, deal_name, from_stage, to_stage }`.

- **SWAP POINT 1 — Meta Ads:** replace/augment **Read Ad Numbers** with an HTTP Request to `https://graph.facebook.com/v21.0/act_<AD_ACCOUNT_ID>/insights?date_preset=yesterday&fields=spend,actions` (credential: **Facebook Graph API**), then a small Code node mapping the response to `{ date, source: 'meta', spend, leads }`.
- **SWAP POINT 1 — Google Ads:** add the built-in **Google Ads** node (credential: **Google Ads OAuth2 API**), report for yesterday, map to `{ date, source: 'google', spend, leads }`. Merge both sources with a Merge node (Append) before Build Summary.
- **SWAP POINT 2 — CRM:** replace **Read Deals** with HubSpot ("Recently Updated Deals") or Pipedrive, mapped to the 4 deal fields.
- **SWAP POINT 3 — Output:** swap Slack for Telegram / Teams / a Gmail digest; the finished text is in `{{ $json.text }}`.

## 7. Notes

- Both reads have Continue On Fail + Always Output Data, so if one source is down you still get a summary from the other (with the "no data" warning) instead of silence.
- "No deals moved yesterday" and "No ad data" are normal messages, not errors.
- Runs at 8:00 AM server time — check your n8n instance timezone (`GENERIC_TIMEZONE`).
