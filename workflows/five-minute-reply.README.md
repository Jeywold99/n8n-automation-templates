# 5-Minute Reply

A minimal n8n automation that replies to every new lead within seconds. One webhook catches leads from a website form and from Calendly, normalizes them into one lead object, then in strict order: logs to a Google Sheet, sends a warm human-sounding Gmail reply, and posts a Slack alert to #leads so a real person jumps in.

**Flow:** Webhook (instant) -> Normalize Lead (Code) -> Log to Sheet -> Gmail reply -> Slack alert. A separate Error Trigger posts a Slack failure alert.

Every external call (Sheets, Gmail, Slack) has **Continue On Fail** enabled, so a CRM hiccup never blocks the reply and a Gmail hiccup never blocks the Slack alert.

---

## 1. Prerequisites

- An n8n instance on **1.x**, reachable from the internet (webhooks need a public URL; n8n Cloud works out of the box, self-hosted needs a public domain or a tunnel like ngrok for testing).
- A Google Sheet for leads.
- A Gmail account to send from.
- A Slack workspace with a **#leads** channel and a bot invited to it.

### The Google Sheet

Create a sheet with a tab named **`Leads`** and this exact header row (row 1):

| name | email | source | received_at |
|------|-------|--------|-------------|

---

## 2. Credentials to configure

Create these in **n8n > Credentials**, then open each node and select them (imported nodes show as "not set" until you do).

| Node(s) | n8n credential type | What it needs |
|---------|---------------------|---------------|
| Log Lead to Sheet | **Google Sheets OAuth2 API** (`googleSheetsOAuth2Api`) | Google OAuth client with Sheets access |
| Send Instant Reply | **Gmail OAuth2** (`gmailOAuth2`) | Google OAuth client with Gmail send scope |
| Slack Alert #leads, Slack Failure Alert | **Slack API** (`slackApi`) | A Slack bot token (`xoxb-...`) with `chat:write` scope; invite the bot to #leads with `/invite @yourbot` |

> Slack bot quick setup: api.slack.com/apps > Create New App > From scratch > OAuth & Permissions > add `chat:write` (and `chat:write.public` to skip the invite) > Install to Workspace > copy the Bot User OAuth Token into the n8n Slack API credential.

---

## 3. Config / environment values

| Env var | Example | Used for |
|---------|---------|----------|
| `LEADS_SHEET_ID` | `1AbC...xyz` | Google Sheet ID (from the sheet URL between `/d/` and `/edit`) |
| `SLACK_LEADS_CHANNEL` | `leads` | Slack channel name for alerts (no `#`) |
| `FROM_NAME` | `Jane` | Signature name in the reply email |

**Docker example:**

```bash
docker run -it --rm \
  -e LEADS_SHEET_ID="your_sheet_id" \
  -e SLACK_LEADS_CHANNEL="leads" \
  -e FROM_NAME="Jane" \
  -p 5678:5678 docker.n8n.io/n8nio/n8n
```

On **n8n Cloud**, set these under **Variables/Environment**, or replace `{{ $env.X }}` in the nodes with literal values if your plan lacks env vars.

The email copy (both versions: booking confirmation and form acknowledgment) lives inside the **Normalize Lead** Code node, clearly commented. Edit it there; nothing else needs to change.

---

## 4. How to import

1. In n8n: **Workflows > Import from File** > choose `five-minute-reply.workflow.json`.
2. Open each node marked "not set" and select your credential (section 2).
3. Confirm the env vars from section 3 are set.
4. **Enable failure alerts:** Workflow Settings (top-right menu) > **Error Workflow** > select **5-Minute Reply** (itself).
5. Save, then toggle the workflow **Active** (webhooks only listen on their production URL when the workflow is active).

---

## 5. How to grab the webhook URLs

Open the **Webhook New Lead** node. n8n shows two URLs:

- **Test URL** — `https://<your-n8n>/webhook-test/new-lead`. Only live while you click **Execute Workflow** (or "Listen for test event") in the editor; catches ONE request, then shows you the data in the editor. Use this while wiring things up.
- **Production URL** — `https://<your-n8n>/webhook/new-lead`. Live 24/7 whenever the workflow is Active. This is the URL you give your form and Calendly.

Both accept **POST** with a JSON body. The webhook responds `200` immediately (Respond: Immediately), so forms never hang waiting.

---

## 6. Sample payloads + end-to-end test

### 6a. Fire a test form lead

PowerShell (Windows):

```powershell
Invoke-RestMethod -Method Post -Uri "https://<your-n8n>/webhook/new-lead" `
  -ContentType "application/json" `
  -Body '{"name":"Sam Tester","email":"your.own+form@gmail.com","message":"Hi! Do you have coaching slots in August?","source":"website_form"}'
```

curl (anywhere):

```bash
curl -X POST "https://<your-n8n>/webhook/new-lead" \
  -H "Content-Type: application/json" \
  -d '{"name":"Sam Tester","email":"your.own+form@gmail.com","message":"Hi! Do you have coaching slots in August?","source":"website_form"}'
```

**Expected within seconds:** a new row in the `Leads` tab; an email in your inbox with subject "Got your message, Sam" acknowledging the note; a Slack post in #leads with name, email, source `website_form`, time received, and the message.

### 6b. Fire a test Calendly lead

```bash
curl -X POST "https://<your-n8n>/webhook/new-lead" \
  -H "Content-Type: application/json" \
  -d '{"name":"Sam Tester","email":"your.own+cal@gmail.com","event_time":"2026-07-15 14:00"}'
```

**Expected:** a new row with source `calendly`; an email with subject "You're booked, Sam!" confirming the 2026-07-15 14:00 time; a Slack post including the booked time.

### 6c. Test the failure alert

Temporarily set `LEADS_SHEET_ID` to garbage AND disable Continue On Fail on the Sheets node, fire a test lead, and confirm the ⚠️ Slack failure alert arrives. Restore both afterward.

---

## 7. Wiring the real sources

### Website form

Point your form's submit action (or your form tool's webhook/integration setting) at the **Production URL** as a POST with JSON fields `name`, `email`, `message`, `source`. Minimal HTML example:

```html
<script>
async function submitLead(e) {
  e.preventDefault();
  const f = e.target;
  await fetch("https://<your-n8n>/webhook/new-lead", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      name: f.name.value,
      email: f.email.value,
      message: f.message.value,
      source: "website_form"
    })
  });
  f.reset(); alert("Thanks! Check your inbox.");
}
</script>
```

Form builders (Typeform, Tally, Jotform, Webflow) all have a "webhook" or "integrations" setting; paste the Production URL there. If their field names differ, adjust the mapping at the top of the **Normalize Lead** node.

### Calendly

Calendly webhook subscriptions require a **paid Calendly plan** (Standard or higher):

1. Calendly > Integrations > **Webhooks** > Create subscription (or via their API).
2. URL: your **Production URL**. Event: `invitee.created`.
3. Real Calendly payloads nest data under `payload` (`payload.name`, `payload.email`, `payload.scheduled_event.start_time`). The **Normalize Lead** node already handles this shape as well as the simplified test payload, so no changes needed.
4. Test by booking a real slot on your own Calendly link with a test email.

On the free Calendly plan, use the simplified payload via Zapier/Make as a bridge, or just test with the curl command in 6b.

---

## 8. SWAP POINTS (demo -> real tools)

Each has a labeled sticky note in the workflow.

- **SWAP POINT 1 - Lead sources:** point any form/booking tool at the one webhook URL. Different payload field names? Edit only the top of **Normalize Lead**.
- **SWAP POINT 2 - CRM (Google Sheets -> HubSpot / Pipedrive):** replace **Log Lead to Sheet** with HubSpot "Create/Update Contact" or Pipedrive "Create Person", mapping name, email, source, received_at. Keep Continue On Fail ON so the reply is never blocked.
- **SWAP POINT 3 - Email (Gmail -> Outlook / SMTP / SendGrid):** replace **Send Instant Reply**; keep the three expressions pulling email, email_subject, email_body from Normalize Lead.
- **SWAP POINT 4 - Alert (Slack -> Telegram):** replace both Slack nodes with Telegram nodes (credential type **Telegram API**, chatId = your chat, same text).

---

## 9. Notes and limits (kept intentionally simple)

- **Order is guaranteed:** log -> reply -> alert, in one straight line. With Continue On Fail on, a failed step passes through so the later steps still run.
- **No email in the payload?** The Gmail node will fail for that lead, but Continue On Fail means the Slack alert still fires, so a human can chase it manually.
- **No deduplication:** the same person submitting twice gets two replies. Fine for a demo; add a Sheets lookup on email if you need dedupe.
- The webhook is unauthenticated (fine for a demo). For production, add a secret header check: Webhook node > Options > add Header Auth, and configure your form/Calendly bridge to send it.
