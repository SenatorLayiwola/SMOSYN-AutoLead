# SMOSYN AutoLead
### AI-Powered Lead Generation, Outreach & Reply Detection System for UK SMEs
**Built by [SMOSYN Digitals](https://smosyn.co.uk) — Wales, UK**

---

## The Story Behind This Build

My original outreach workflow was two separate things stitched together.

Auto Lead Source pulled businesses from SerpAPI and enriched them with Hunter.io. Auto Lead Outreach sent emails. They worked independently. That was the problem.

There was no qualification layer between discovery and outreach. Every lead that passed a clean domain check got emailed — regardless of fit, urgency, or likelihood to convert. I was burning Outlook sends on cold leads that had no business being in the sequence.

And when a lead replied? I had no automated way to detect it. I was checking the inbox manually.

**AutoLead is the rebuild.** Three workflows. One unified pipeline. AI qualification sits between discovery and outreach. A reply detector runs every hour so nothing gets missed.

---

## What Is SMOSYN AutoLead?

A production-grade AI lead generation and outreach system built on n8n. It handles the complete prospecting lifecycle — from automated business discovery and AI-scored qualification, through a three-touch personalised email sequence, to automated reply detection and Telegram notification.

> Sending emails to everyone is not outreach. It's noise. Qualification changes the economics entirely.

---

## What Changed From the Original

| Area | Original | AutoLead v1 |
|------|-----------|-------------|
| Lead sourcing | SerpAPI only | SerpAPI (7 signals) + industry targeting sheet |
| Qualification | None — clean domain = contact | AI scoring 1–10 (Hot / Warm / Cold) |
| Email personalisation | Generic AI prompt | Pain points, urgency, best service fit from qualifier |
| Outreach sequence | 3-touch but no dedup | 3-touch + 14-day company dedup guard |
| Reply detection | Manual inbox check | Automated hourly Outlook scan + Merge matching |
| Notifications | None | Telegram alert on every reply |
| Node naming | Generic | Descriptive and client-safe |
| Documentation | None | Full Master + Client docs |

---

## System Architecture

```
[ WF-01: Lead Extractor ]
  Schedule Trigger
  → Get Industry Targets (Google Sheets)
  → IF Active
  → SerpAPI (7 signals per business)
  → Split Out
  → Clean & Extract
  → IF Clean Domain
  → Hunter.io (decision-maker email lookup)
  → Split Email
  → IF Email True
  → OpenAI GPT-4o (AI Qualifier — score 1–10)
  → Switch: Hot (8–10) / Warm (5–7) / Cold (1–4)
  → Google Sheets Append

              ↓ leads stored in Google Sheets

[ WF-02: Lead Outreach ]
  Schedule Trigger (Daily 9AM)
  → Config Set (sender info + company knowledge)
  → Get Qualified Leads
  → IF Ready to Contact + Not Contacted
  → Check 14-day company dedup
  → Build Email Context
  → OpenAI: First Email
  → Outlook: Send
  → Update Sheet
  → Wait 3 Days
  → OpenAI: Follow-Up 1
  → Outlook: Send
  → Update Sheet
  → Wait 7 Days
  → OpenAI: Final Email
  → Outlook: Send
  → Update Sheet

              ↓ running in parallel

[ WF-03: Reply Detector ]
  Hourly Trigger
  → Check Inbox (Outlook — get all emails)
  → Look Up Leads (Google Sheets — get all rows)
  → Merge (match by email address, keepMatches mode)
  → IF Not Already Replied (Replied = FALSE)
  → Update Reply Column
  → Telegram: Alert with sender, company, subject, preview
```

---

## Workflows

| File | Workflow | Purpose |
|------|----------|---------|
| `WF01_Lead_Extractor_CLEAN.json` | Lead Extractor | SerpAPI → Hunter → AI Qualifier → Hot/Warm/Cold routing |
| `WF02_Lead_Outreach_CLEAN.json` | Lead Outreach | 3-touch personalised email sequence with Wait nodes |
| `WF03_Reply_Detector_CLEAN.json` | Reply Detector | Hourly Outlook scan. Merge match. Telegram alert on reply. |

---

## Tech Stack

- **Automation:** n8n (self-hosted)
- **AI:** OpenAI GPT-4o
- **Lead Discovery:** SerpAPI
- **Email Enrichment:** Hunter.io
- **Outreach:** Microsoft Outlook
- **Data Layer:** Google Sheets (31-column lead schema)
- **Notifications:** Telegram Bot

---

## Key Features

- Industry-targeted prospecting — configure targets in a sheet, no workflow edits needed
- AI qualifies every lead before outreach — no wasted sends on poor-fit businesses
- Scores include: urgency, pain points, best service fit, AI reasoning
- Hot leads (8–10) flagged Ready to Contact immediately
- 14-day company-level deduplication — same company never receives two emails within a fortnight
- Three-touch sequence stops automatically the moment a lead replies
- Reply Detector runs every hour using Merge node matching — not polling
- Telegram alert includes sender, company name, subject line, and message preview
- 31-column Google Sheets schema — complete audit trail from discovery to reply

---

## How to Use

### Prerequisites
- n8n instance (self-hosted or cloud)
- OpenAI API key
- SerpAPI key
- Hunter.io API key
- Microsoft Outlook (OAuth2 connected in n8n)
- Google Sheets + Google Drive OAuth2
- Telegram Bot token

### Import a Workflow
1. Open your n8n instance
2. Click **Add Workflow** → **Import from file**
3. Select the `.json` file
4. Replace all placeholder values (see table below)
5. Connect your credentials
6. Activate

### Placeholders to Replace

| Placeholder | What to Replace With |
|-------------|----------------------|
| `YOUR_GOOGLE_SHEET_ID` | Google Sheet ID from the URL |
| `YOUR_TELEGRAM_CHAT_ID` | Your Telegram Chat ID (message @userinfobot) |
| `YOUR_TELEGRAM_CREDENTIAL_HASH` | Telegram credential ID in n8n credentials panel |
| `YOUR_OUTLOOK_FOLDER_ID` | Outlook Inbox folder ID (from Outlook node output) |
| `YOUR_SENDER_EMAIL` | Your Outlook email address |
| `YOUR_FULL_NAME` | Your name for email signatures |
| `YOUR_PHONE_NUMBER` | Your contact number |
| `YOUR_SYSTEM_PROMPT` | Your AI qualifier instructions — internal IP, not for GitHub |
| `YOUR_USER_PROMPT` | Your outreach prompt — internal IP, not for GitHub |
| `YOUR_COMPANY_KNOWLEDGE_BASE` | Your services, positioning, and USPs for AI context |

---

## Cost Optimisation

Every component is replaceable without touching the workflow logic:

| Component | Current | Alternative |
|-----------|---------|-------------|
| Lead Discovery | SerpAPI | ScraperAPI, Bright Data, Google Custom Search |
| Email Enrichment | Hunter.io | Apollo.io, Clearbit, Snov.io, Lusha |
| AI Qualification | OpenAI GPT-4o | Claude (Anthropic), Gemini, Llama (self-hosted) |
| Outreach | Microsoft Outlook | Gmail, SendGrid, Mailgun, any SMTP |
| Data Layer | Google Sheets | Airtable, Supabase, Notion, PostgreSQL |
| Notifications | Telegram | Slack, WhatsApp Business, Email |

---

## Hard Lessons From the Build

Things that cost me time — so they don't cost you yours:

**1. The Merge node is not a filter — it is a matcher**
Set joinMode to `keepMatches` and combine by `email` field. Without this, Merge returns everything or nothing. This is the entire logic behind WF-03.

**2. Check If Already Replied must sit on the TRUE branch**
The IF node (Replied = FALSE) routes matched leads. The Update node must connect to the TRUE output. Connecting to FALSE means replied leads get double-alerted and the column never updates.

**3. Look Up Leads fetches ALL rows with no filter — intentionally**
WF-03 does not pre-filter the sheet. It fetches everything and lets Merge do the matching. Pre-filtering by email before the Merge breaks the match logic entirely.

**4. Telegram Parse Mode must be None**
Any dynamic expression with special characters (angle brackets, asterisks, underscores) in an email subject or preview will throw a 400 error if Parse Mode is set to Markdown or HTML. Set it to blank. Always.

**5. 14-day dedup is company-level, not contact-level**
The dedup check in WF-02 looks at company domain, not individual email address. This prevents two contacts at the same business receiving outreach in the same fortnight — critical for professional credibility.

**6. Hunter.io returns an array — always split before IF**
Hunter returns `emails[]`. Without a Split node after Hunter, the IF Email True check runs against the parent object, not individual results. It will always fail.

---

## Roadmap

| Version | Scope |
|---------|-------|
| v1.0 | Auto Lead Source + Auto Lead Outreach — two separate workflows, no qualifier |
| v1.1 | Current — 3 unified workflows, AI qualifier, reply detector, Telegram alerts |
| v1.2 | Supabase migration, multi-tenant support, lead scoring dashboard |
| v2.0 | Revenue OS integration — AutoLead feeds live event operator pipeline |

---

## Documentation

Full Master Technical Documentation and Client Overview available on request.
Contact: [smosyn.co.uk](https://smosyn.co.uk)

---

## License

MIT — free to use and adapt. Attribution appreciated.

---

*Built with purpose. Documented with care. Designed to scale.*
