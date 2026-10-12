# Approve-and-Post

Post on LinkedIn every day without ever opening LinkedIn. Dump a rough idea into Airtable; Claude drafts the post; Slack shows it with one-tap Approve/Decline buttons; approved posts publish to LinkedIn automatically each weekday at 9 AM.

**Two paths in one workflow:**
- **PATH A (drafting):** Airtable Trigger (new idea) -> Claude drafts -> Slack "Send and Wait for Approval" -> decision saved to Airtable (`approved` / `rejected`).
- **PATH B (publishing):** Schedule (weekdays 9 AM) -> fetch ONE oldest `approved` post -> publish to LinkedIn -> mark `posted`. Approve five posts on Sunday, they roll out Mon–Fri.

---

## 1. Prerequisites

- n8n 1.x (cloud or self-hosted).
- An **Airtable** base with a table (e.g. `Ideas`) with these fields:
  - `idea` (long text) — your rough idea / transcribed voice note
  - `status` (single line text) — leave blank on new rows; the workflow writes `approved` / `rejected` / `posted`
  - `draft` (long text) — filled by the workflow
  - `posted_at` (date) — filled by the workflow
  - `Created` (created time field) — **required**, the trigger polls on it
- A **Slack** workspace + bot (see 5-Minute Reply README for the 5-minute bot setup: scopes `chat:write` + `chat:write.public`).
- An **Anthropic API key** (console.anthropic.com) for drafting.
- A **LinkedIn** account (personal profile posting).
- Voice notes: use any voice-to-text (e.g. Airtable's mobile app dictation) to get the idea into the `idea` field — the workflow only needs text.

## 2. Credentials (exact n8n credential types)

| Node(s) | Credential type | Notes |
|---|---|---|
| New Idea in Airtable, Save Decision, Get Next Approved Post, Mark as Posted | **Airtable Personal Access Token** (`airtableTokenApi`) | Create at airtable.com/create/tokens with scopes `data.records:read`, `data.records:write` on your base |
| Draft Post with Claude | **Anthropic** (`anthropicApi`) | Paste your `sk-ant-...` API key |
| Approve in Slack, Slack Failure Alert | **Slack API** (`slackApi`) | Bot token `xoxb-...` |
| Publish to LinkedIn | **LinkedIn OAuth2 API** (`linkedInOAuth2Api`) | OAuth app with `w_member_social`; after connecting, **open the node and select your profile** in the Person field (it shows a placeholder until you do) |

## 3. Config / env values

| Env var | Example | Used for |
|---|---|---|
| `AIRTABLE_BASE_ID` | `appXXXXXXXXXXXXXX` | From your base URL (`airtable.com/appXXX.../tblYYY...`) |
| `AIRTABLE_TABLE_ID` | `tblYYYYYYYYYYYYYY` | Same URL, second segment |
| `SLACK_APPROVALS_CHANNEL` | `content-approvals` | Channel for approval requests + failure alerts (no `#`) |

The AI writing rules (hook, tone, no em dashes, max 3 hashtags) live in the **Draft Post with Claude** node's JSON body. The posting schedule (`0 9 * * 1-5`) is in the **Weekdays 9AM** node.

## 4. Import

1. **Workflows > Import from File** > `approve-and-post.workflow.json`.
2. Select credentials on every node marked "not set". **Open Publish to LinkedIn and pick your profile** in the Person field.
3. Set the 3 env vars.
4. **Workflow Settings > Error Workflow > Approve-and-Post** (itself).
5. Save and toggle **Active** (the Airtable trigger polls every minute; the schedule fires weekdays at 9).

## 5. Test end to end

1. Add a row to Airtable with `idea` = "clients ghost proposals because following up feels awkward, automation makes it polite and automatic" (leave `status` blank).
2. Within ~1 minute, a Slack message appears with the drafted post and **Approve / Decline** buttons.
3. Tap **Approve**. Check Airtable: the row now has `status = approved` and the `draft` filled in.
4. Don't want to wait for 9 AM? Open the workflow, click the **Weekdays 9AM** node, and **Execute step** to run PATH B now. The post publishes to LinkedIn and the row flips to `status = posted` with today's `posted_at`.
5. Test decline: add another idea, tap **Decline** — the row becomes `rejected` and is never published.
6. Failure alert: temporarily break `AIRTABLE_BASE_ID`, run PATH B manually, expect a ⚠️ in Slack. Restore it after.

## 6. SWAP POINTS

- **SWAP POINT 1 — Idea inbox (Airtable -> Notion / Sheets / Telegram bot):** replace the trigger; downstream only needs an `idea` text field and a record you can update.
- **SWAP POINT 2 — AI writer (Claude -> OpenAI / other):** replace the HTTP Request node; keep the fallback behavior (if drafting fails, the raw idea flows through so approval still works). Edit tone rules in the request body.
- **SWAP POINT 3 — Approval (Slack -> Telegram):** the Telegram node has the same "Send and Wait for Approval" operation; swap node, keep the message.
- **SWAP POINT 4 — Publishing (LinkedIn -> X / Facebook / Buffer):** replace **Publish to LinkedIn**. Keep Continue On Fail OFF on the publish node so a failed publish is never marked as posted.

## 7. Notes

- PATH A pauses (workflow "waiting" state) until someone taps a button — that's normal, not a hang.
- PATH B publishes at most one post per weekday; queue depth = how many you approve.
- If publishing fails, the run errors (by design), the Slack alert fires, and the post stays `approved` so it retries the next weekday.
