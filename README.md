# AI Lead Response Bot

An n8n automation that reads incoming sales leads, uses AI to judge and respond to them, and logs every one automatically — built end-to-end as a self-hosted project to learn production-style workflow automation.

## What it does

1. A webhook captures a new lead (name, email, company, message) from any form or lead-capture tool
2. Google Gemini reads the message, scores the lead 1–10, and drafts a short, personalized reply
3. The workflow branches on that score:
   - **Qualified** → the AI's reply is emailed straight to the lead
   - **Not qualified** → an internal email flags it for manual review instead
4. Every lead — qualified or not — is logged to a Google Sheet automatically, acting as a lightweight CRM

## Architecture

```
Webhook → Normalize Data → AI Draft (Gemini) → Parse Output → Qualified?
                                                                  ├── Yes → Send Reply to Lead
                                                                  └── No  → Notify Sales Team
                                    (in parallel) → Log to Google Sheet
```

## Tech stack

- **[n8n](https://n8n.io)** — self-hosted workflow engine (the automation runtime)
- **Google Gemini API** — reads each lead and generates a structured JSON response (score + reply text)
- **Gmail (SMTP)** — sends the AI-drafted reply and internal notifications
- **Google Sheets API (OAuth2)** — logs every lead as a running record

## Technical challenges solved

This wasn't just wiring nodes together — a few real production issues came up during testing, and fixing them properly (not just patching around them) was most of the actual learning:

- **OAuth2 setup for self-hosted n8n**: self-hosted instances can't use a managed "Sign in with Google" flow, so this required creating a real Google Cloud project, OAuth consent screen, and client credentials from scratch.
- **Race conditions on concurrent writes**: firing multiple leads at once caused the Google Sheets "append row" step to overwrite itself, since concurrent executions all read the same "next empty row" before any of them wrote. Fixed by serializing production executions (`N8N_CONCURRENCY_PRODUCTION_LIMIT=1`), trading a small amount of throughput for data integrity.
- **Free-tier API rate limits**: Gemini's free tier caps requests per minute *and* per day. Solved with automatic retry-with-backoff on the HTTP Request node, so a temporary "service overloaded" response recovers on its own instead of failing the whole run.
- **Parsing AI output safely**: the AI is asked to return structured JSON inside a text response. A Code node parses this with a try/catch fallback, so one malformed AI response can't break the whole execution.

## Setup

1. Import `workflow.json` into your n8n instance
2. Create these credentials:
   - Header Auth credential for the Gemini API key
   - SMTP credential for sending email
   - Google Sheets OAuth2 credential
3. Update the placeholder sender/recipient emails in the Email Send nodes
4. Point your lead source (a website form, Typeform, etc.) at the workflow's Production webhook URL

## What I'd build next

- Multi-language reply support (detect the lead's language and respond in kind)
- Slack/WhatsApp notifications alongside email
- A lightweight dashboard over the Google Sheet log

---

Built as a hands-on project to learn n8n, API integration, and workflow automation from the ground up — including the debugging, not just the happy path.
