# LeadRadar Pipeline

A fully automated, two-stage n8n pipeline for discovering local SMB leads and converting them into ready-to-send personalized cold emails — entirely hands-off.

---

## How It Works

```
Stage 1 — LeadRadar
  Scans Google Maps → filters businesses → AI scores automation opportunities
                              ↓
                      Google Sheets (shared)
                              ↓
Stage 2 — OutreachForge
  Finds real contacts → scrapes website → AI writes personalized cold email
```

Both workflows share a single Google Sheet. LeadRadar fills it with scored leads. OutreachForge reads from it and produces outreach-ready emails.

---

## Stage 1 — LeadRadar

> `leadradar/`

Takes a list of search keywords (e.g. `"dentist in Mumbai"`) and:

- Searches Google Maps via SerpAPI for local businesses
- Filters by quality: >50 reviews, rating 3.5–4.7, real website
- Scrapes each business's website with Firecrawl
- Sends data to Gemini 2.5 Flash for deep opportunity analysis
- Scores each lead on automation need, urgency, and business quality
- Writes enriched lead records to Google Sheets

**APIs used:** SerpAPI · Firecrawl · Google Gemini · Google Sheets

**Trigger:** Manual or scheduled every 24 hours

[View full LeadRadar setup →](leadradar/README.md)

---

## Stage 2 — OutreachForge

> `outreachforge/`

Picks up high-scoring leads from LeadRadar (lead score >60) and:

- Searches Apollo.io for the real decision-maker at each business
- Uses Hunter.io as a fallback for email discovery by domain
- Scrapes the business website again for live context
- Sends everything to Gemini 2.5 Flash to write a short, specific cold email
- Writes contact details + personalized email to a separate Google Sheet tab

**APIs used:** Apollo.io · Hunter.io · Firecrawl · Google Gemini · Google Sheets

**Trigger:** Manual or scheduled every 6 hours

[View full OutreachForge setup →](outreachforge/README.md)

---

## Google Sheets Structure

Both workflows share **one Google Sheet** with three tabs:

| Tab | Written by | Read by |
|---|---|---|
| `keywords` | You (manually add keywords) | LeadRadar |
| `store lead intelligence` | LeadRadar | OutreachForge |
| `outreach enrichment` | OutreachForge | You |

---

## Quick Start

### 1. Set up the Google Sheet

Create one Google Sheet with three tabs as described in each workflow's README. Copy the Sheet ID from the URL.

### 2. Get your API keys

| Key | Used by |
|---|---|
| SerpAPI | LeadRadar |
| Firecrawl | LeadRadar + OutreachForge |
| Google Gemini | LeadRadar + OutreachForge |
| Apollo.io | OutreachForge |
| Hunter.io | OutreachForge |

### 3. Import into n8n

- Import `leadradar/SMB Opportunity Intelligence Pipeline (1).json` first
- Import `outreachforge/Lead Enrichment + Outreach Preparation.json` second
- Fill in all API key placeholders in both workflows
- Connect your Google Sheets OAuth2 credential

### 4. Run

1. Add keywords to the `keywords` tab (with city, area, niche columns filled)
2. Run **LeadRadar** — it will populate `store lead intelligence` with scored leads
3. Run **OutreachForge** — it will enrich top leads and write cold emails to `outreach enrichment`

---

## Tech Stack

- **n8n** — workflow automation
- **SerpAPI** — Google Maps business data
- **Apollo.io** — contact and email discovery
- **Hunter.io** — domain email lookup
- **Firecrawl** — website scraping
- **Google Gemini 2.5 Flash** — AI analysis and email generation
- **Google Sheets** — data storage throughout the pipeline
