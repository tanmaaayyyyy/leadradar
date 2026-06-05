# OutreachForge

An n8n automation that takes high-scoring leads, hunts down real contact details via Apollo and Hunter.io, scrapes their website, and uses Gemini AI to write a hyper-personalized cold email — ready to send.

Designed to run after [LeadRadar](https://github.com/tanmaaayyyyy/leadradar). LeadRadar discovers and scores the leads. OutreachForge prepares the outreach.

---

## What It Does

1. Reads all leads from the LeadRadar Google Sheet
2. Filters to only high-quality, uncontacted leads (lead score > 60, automation score > 20)
3. For each lead, searches Apollo.io for the real decision-maker's contact details
4. Falls back to Hunter.io for email discovery by domain
5. Optionally triggers a LinkedIn lead-gen agent via a local webhook
6. Scrapes the business website with Firecrawl for live context
7. Sends all context to **Gemini 2.5 Flash** to write a short, specific, human-sounding cold email
8. Writes the contact details + personalized email to the `outreach enrichment` Google Sheet tab
9. Loops through all qualifying leads with a 10-second pause between each

---

## Pipeline Flow

```
Trigger (Manual / Schedule every 6h)
  └── Read Leads (Google Sheets — "store lead intelligence" tab)
        └── Filter High-Quality Leads
              (lead_score > 60, automation_score > 20, status = new, not contacted, no existing email)
              └── Loop Through Leads
                    └── Store Lead Context (extract domain, cache metadata)
                          ├── Apollo.io Contact Search (founder / owner / manager)
                          │     └── Wait for LinkedIn webhook callback (up to 3 min)
                          │           └── Extract Best Contact (highest confidence score)
                          │                 ├── Firecrawl Website Scrape
                          │                 │     └── Build Gemini Request
                          │                 │           └── Gemini 2.5 Flash — Write Cold Email
                          │                 └── Merge (contact + LinkedIn + email)
                          │                       └── Parse Outreach Response
                          │                             └── Update Lead in Sheets ("outreach enrichment" tab)
                          │                                   └── Wait 10s → Next Lead
                          └── Hunter.io Email Discovery (runs in parallel, currently disconnected)
```

---

## Setup

### Prerequisites

- [n8n](https://n8n.io) — self-hosted or cloud
- LeadRadar already set up and populated with scored leads in Google Sheets
- API keys for Apollo, Hunter, Firecrawl, and Gemini

### 1. API Keys

| Service | Purpose | Where to get it |
|---|---|---|
| [Apollo.io](https://apollo.io) | Find decision-maker contacts by company domain | Apollo → Settings → API Keys |
| [Hunter.io](https://hunter.io) | Email discovery by domain (fallback) | Hunter → Dashboard → API |
| [Firecrawl](https://firecrawl.dev) | Live website content scraping | firecrawl.dev → API Keys |
| [Google Gemini](https://aistudio.google.com) | AI cold email generation | Google AI Studio → Get API Key |

### 2. Google Sheets Setup

OutreachForge reads from and writes to the **same Google Sheet used by LeadRadar**. You need a third tab added:

#### Tab 3: `outreach enrichment`

Create this tab with the following columns:

`place_id` · `business_name` · `contact_name` · `contact_role` · `contact_email` · `linkedin_url` · `confidence` · `personalized_observation` · `personalized_email` · `outreach_status` · `time`

The workflow uses `place_id` as the unique key to match and update rows.

### 3. Import Into n8n

1. Download `Lead Enrichment + Outreach Preparation.json`
2. In n8n, go to **Workflows → Import from File**
3. Replace every placeholder value:

| Placeholder | Replace with |
|---|---|
| `YOUR_APOLLO_API_KEY` | Your Apollo.io API key |
| `YOUR_HUNTER_API_KEY` | Your Hunter.io API key |
| `YOUR_FIRECRAWL_API_KEY` | Your Firecrawl key (keep the `Bearer ` prefix) |
| `YOUR_GEMINI_API_KEY` | Your Gemini API key |
| `YOUR_GOOGLE_SHEET_ID` | The ID from your Google Sheet URL |
| `YOUR_GOOGLE_SHEETS_CREDENTIAL_ID` | Your n8n Google Sheets credential ID |
| `YOUR_LOCAL_N8N_HOST` | Your local n8n address (e.g. `127.0.0.1:5678`) |
| `YOUR_LOCAL_WEBHOOK_TOKEN` | Auth token for your local LinkedIn agent webhook |

4. Connect a **Google Sheets OAuth2** credential in the Sheets nodes
5. Activate the workflow

---

## Lead Filtering Criteria

A lead is picked up by OutreachForge only if it meets **all** of the following:

| Condition | Value |
|---|---|
| `lead_score` | > 60 |
| `automation_score` | > 20 |
| `status` | `new` or empty |
| `contacted` | not `yes` |
| `personalized_email` | not already written |
| `website` | present and not a social profile (Instagram, Facebook, Zomato, Swiggy) |

This ensures only genuinely strong, untouched leads reach the enrichment stage.

---

## Contact Discovery

OutreachForge runs two contact discovery methods:

### Apollo.io (primary)
Searches by company domain for people with titles: `founder`, `owner`, `operations manager`, `marketing manager`, `general manager`. Returns up to 5 results. The contact with the highest confidence score is selected.

### Hunter.io (secondary)
Searches by extracted domain, returns up to 2 email results. Currently wired in parallel but not connected to the main merge — can be enabled as a fallback if Apollo returns nothing.

### LinkedIn Agent (optional)
OutreachForge can trigger a locally running LinkedIn lead-gen agent via a webhook at `YOUR_LOCAL_N8N_HOST/hooks/wake`. The workflow waits up to 3 minutes for a callback. If no callback arrives, it continues with whatever Apollo returned. This is optional — remove the HTTP Request node if you don't have a LinkedIn agent running.

---

## AI Email Generation

Once contact and website data is collected, Gemini 2.5 Flash is prompted to write a cold email with strict rules:

- Under 130 words
- References **one specific thing** observed about the business (from live website content)
- Names **one specific pain point** with a realistic example of how it costs them
- Proposes **one concrete solution** (not vague "AI automation")
- Ends with a soft CTA: a 10-minute call ask
- Subject line: under 8 words, no spam triggers, no mention of AI or automation
- No buzzwords: leverage, streamline, revolutionize, cutting-edge, game-changer are banned
- Sounds like a human wrote it, not a marketing template

The email is written in the voice of the sender (configurable in the Build Gemini Request node) and starts with the contact's first name.

### Output fields written to Google Sheets

| Field | Description |
|---|---|
| `contact_name` | Name of the discovered decision-maker |
| `contact_role` | Their title (founder, owner, etc.) |
| `contact_email` | Verified or discovered email address |
| `linkedin_url` | LinkedIn profile URL if found |
| `confidence` | Apollo's confidence score for the contact |
| `personalized_observation` | The specific thing Gemini noticed about the business |
| `personalized_email` | The complete cold email body |
| `outreach_status` | Set to `outreach_ready` once complete |
| `time` | Timestamp of enrichment |

---

## Scheduling

A **Schedule Trigger** is included (disabled by default) set to run every 6 hours. Enable it to continuously pick up new high-quality leads as LeadRadar adds them throughout the day.

---

## Relationship to LeadRadar

OutreachForge is the second stage of a two-part pipeline:

| Stage | Workflow | What it does |
|---|---|---|
| 1 | **LeadRadar** | Discovers local SMBs, scores automation opportunities |
| 2 | **OutreachForge** | Enriches contacts, writes personalized cold emails |

Both workflows share the same Google Sheet. LeadRadar writes to `store lead intelligence`. OutreachForge reads from it and writes results to `outreach enrichment`.

---

## Tech Stack

- **n8n** — workflow automation
- **Apollo.io** — contact and email discovery
- **Hunter.io** — domain-based email lookup
- **Firecrawl** — live website scraping
- **Google Gemini 2.5 Flash** — AI cold email generation
- **Google Sheets** — lead storage and output
