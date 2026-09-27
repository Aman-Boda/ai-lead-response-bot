# AI Lead Response Bot

An n8n automation that reads incoming sales leads, uses AI to judge and respond to them, and logs every one automatically — built end-to-end as a self-hosted project to learn production-style workflow automation.

## What it does

1. A webhook — protected by a shared-secret header, so it can't be triggered by anyone who doesn't have it — captures a new lead (name, email, company, message) from any form or lead-capture tool
2. A duplicate check silently drops repeat submissions from the same email within a 5-minute window, so a double form-submit doesn't send two replies
3. Google Gemini reads the message, detects its language, scores the lead 1–10, and drafts a short, personalized reply — written in whatever language the lead wrote in
4. The workflow branches on that score:
   - **Qualified** → the AI's reply is emailed straight to the lead
   - **Not qualified** → the internal team is alerted on three channels at once: email, Slack, and WhatsApp
5. Every lead — qualified or not — is logged to a Google Sheet automatically, acting as a lightweight CRM
6. Every external call (AI, email, Sheets) retries automatically on failure; if something still fails after retrying, an alert email fires so it's never silently dropped

## Architecture

```
Webhook (secured) → Normalize Data → Check Duplicate ──duplicate──→ stopped, no action
                                            │
                                        not a duplicate
                                            ↓
                                     AI Draft (Gemini, multilingual) → Parse Output → Qualified?
                                                                                        ├── Yes → Send Reply to Lead
                                                                                        └── No  → Email + Slack + WhatsApp, all at once
                                                    (in parallel) → Log to Google Sheet

Every external node (AI / email / Sheets / Slack / WhatsApp) retries on failure →
still failing after retries → routes to an "Alert Me" email instead of failing silently
```

## Tech stack

- **[n8n](https://n8n.io)** — self-hosted workflow engine (the automation runtime)
- **Google Gemini API** — reads each lead, detects language, and generates a structured JSON response (score + reply text)
- **Gmail (SMTP)** — sends the AI-drafted reply and internal notifications
- **Google Sheets API (OAuth2)** — logs every lead as a running record
- **Slack Incoming Webhooks** — real-time team alerts for leads needing review
- **Twilio WhatsApp API** — a second alert channel, built and tested against Twilio's sandbox

## Technical challenges solved

This wasn't just wiring nodes together — a few real production issues came up during testing, and fixing them properly (not just patching around them) was most of the actual learning:

- **OAuth2 setup for self-hosted n8n**: self-hosted instances can't use a managed "Sign in with Google" flow, so this required creating a real Google Cloud project, OAuth consent screen, and client credentials from scratch.
- **Race conditions on concurrent writes**: firing multiple leads at once caused the Google Sheets "append row" step to overwrite itself, since concurrent executions all read the same "next empty row" before any of them wrote. Fixed by serializing production executions (`N8N_CONCURRENCY_PRODUCTION_LIMIT=1`), trading a small amount of throughput for data integrity.
- **Free-tier API rate limits**: Gemini's free tier caps requests per minute *and* per day. Solved with automatic retry-with-backoff on the HTTP Request node, so a temporary "service overloaded" response recovers on its own instead of failing the whole run.
- **Parsing AI output safely**: the AI is asked to return structured JSON inside a text response. A Code node parses this with a try/catch fallback, so one malformed AI response can't break the whole execution.
- **A boolean that quietly became a string — and a "fix" that made it worse**: adding duplicate-detection meant checking a `true`/`false` flag in an IF node. It threw a type error ("expected boolean, got string"), so the obvious fix was enabling n8n's built-in "convert types where required" option. That actually made things *worse* — every lead started getting flagged as a duplicate, including brand-new ones. The reason: JavaScript's type coercion treats **any non-empty string as truthy**, including the literal text `"false"`. So the auto-conversion was silently turning `"false"` into `true`. The real fix wasn't more automatic conversion — it was removing the ambiguity entirely, by making both sides of the comparison explicit strings instead of relying on implicit type coercion at all. Good reminder that "make it convert more aggressively" isn't always the right instinct when a type error shows up.
- **Locking down the webhook**: by default, anyone who discovers a webhook URL can POST arbitrary data to it — burning API quota, triggering fake emails, or spamming the log. Fixed with n8n's built-in Header Auth option on the webhook node, so only requests carrying a shared secret token are accepted.
- **Multi-language support turned out to need zero new nodes**: rather than adding a separate language-detection step, the AI prompt itself was updated with one instruction — reply in the same language the lead wrote in. Gemini already understands language; the fix was asking it correctly, not adding more infrastructure.
- **A silently empty request body**: the Slack alert kept failing with `invalid_payload`, even after multiple rewrites of the expression. The actual cause, found by capturing the real outbound request with a request-inspector tool, was that the node was sending a completely empty body — 0 bytes — regardless of what the field appeared to contain. No amount of fixing the *expression* was ever going to solve a body that wasn't being sent at all. Deleting the node and rebuilding it from scratch resolved it instantly, confirming the corruption was in the node's saved state, not the logic. Lesson: when a field's *visible* content looks correct but behavior stays broken, verify what's actually on the wire before continuing to edit the field.
- **Third-party example IDs aren't universal**: an early attempt to use a WhatsApp Content Template SID pulled from Twilio's own documentation failed, because that ID was illustrative example text, not a real, reusable identifier — every Twilio account has its own unique templates. Good reminder to treat doc examples as illustrations of *shape*, not literal values to copy in.
- **Knowing when to stop**: WhatsApp's production template requires per-business content, submitted for Meta's approval — not something worth building speculatively before a real client needs it. The integration's plumbing (auth, request format, sandbox connection) is fully built and tested; the account-specific template is deliberately left as the one step done at client activation, not before.

## Setup

1. Import `workflow.json` into your n8n instance
2. Create these credentials:
   - Header Auth credential for the Gemini API key
   - SMTP credential for sending email
   - Google Sheets OAuth2 credential
3. Update the placeholder sender/recipient emails in the Email Send nodes
4. Point your lead source (a website form, Typeform, etc.) at the workflow's Production webhook URL

## What I'd build next

- A real, Meta-approved WhatsApp template for production client use (the plumbing is done; this is the one deliberately deferred, client-specific piece)
- Always-on hosting on a real server instead of a local machine, so it runs independent of any one computer
- A lightweight dashboard over the Google Sheet log

---

Built as a hands-on project to learn n8n, API integration, and workflow automation from the ground up — including the debugging, not just the happy path.
