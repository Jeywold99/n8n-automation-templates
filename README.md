# n8n Automation Templates

Six importable n8n workflows that remove common small-business busywork, plus two AI lead-response flows. Each workflow has its own README with the flow, setup steps and the sheet/table layout it expects.

| Workflow | What it does |
|---|---|
| [5-Minute Reply](workflows/five-minute-reply.README.md) | Answers every new website or Calendly lead within seconds: logs it to Google Sheets, sends a warm Gmail reply, and alerts Slack. |
| [Ghost Chaser](workflows/ghost-chaser.README.md) | Every morning, finds proposals with no reply after 3 days and drops a follow-up **draft** into Gmail for review. |
| [Invoice Nag Bot](workflows/invoice-nag-bot.README.md) | Sends a friendly daily reminder for overdue, unpaid invoices and records each reminder on the sheet. |
| [Morning Numbers](workflows/morning-numbers.README.md) | Posts one Slack summary at 8 AM: yesterday's ad spend, leads and cost per lead by source, plus deals that moved. |
| [Pocket Briefing](workflows/pocket-briefing.README.md) | 30 minutes before each sales call, sends a Telegram brief: who the guest is, CRM notes, and the last 5 emails with them. |
| [Approve-and-Post](workflows/approve-and-post.README.md) | Claude drafts a LinkedIn post from an Airtable idea, Slack shows Approve/Decline buttons, approved posts publish on schedule. |

Also included: `n8n-lead-capture-workflow.json` and `n8n-client-qualification-workflow.json`, which use Claude to qualify inbound leads and draft a personalized reply.

## Design rules used throughout

- **Every workflow has an Error Trigger** that alerts Slack or Telegram, so nothing fails silently.
- **Humans stay in the loop where it matters:** follow-ups land as drafts, posts need approval.
- **Filters are explicit** (paid invoices, recent follow-ups, and not-yet-due items are dropped before anything is sent).

## Import

In n8n: **Workflows → Import from File**, choose a `.workflow.json`, then connect your own credentials on each node (every credential ID in these files is a `REPLACE_WITH_YOUR_CRED_ID` placeholder).
