# LeadRadar

An n8n automation that discovers, qualifies, and AI-scores local SMB leads from Google Maps — fully hands-off.

Feed it a list of keywords. It finds real local businesses, scrapes their websites, runs Gemini AI analysis on each one, and delivers a scored lead sheet with outreach angles, pain points, and the best offer to pitch.

---

## What It Does

1. Reads search keywords (e.g. `"plumber in Austin"`) from a Google Sheet
2. Queries Google Maps via SerpAPI and pulls back local business listings
3. Applies quality filters — reviews, rating, and real website presence
4. Scrapes each qualifying business's website with Firecrawl
5. Sends all data to **Gemini 2.5 Flash** for deep opportunity analysis
6. Scores every lead on automation need, urgency, and business quality
7. Writes the enriched lead record back to Google Sheets
8. Marks each keyword as processed and loops to the next one

---

## Pipeline Flow

```
Trigger (Manual / Schedule)
  └── Get Keyword Rows (Google Sheets)
        └── Filter Unprocessed Keywords
              └── Loop Through Keywords
                    ├── Store Keyword Context
                    │     └── SerpAPI Google Maps Search
                    │           └── Flatten Results
                    │                 └── Filter Businesses (reviews > 50, rating 3.5–4.7)
                    │                       └── Remove Duplicates (by website URL)
                    │                             └── Loop Through Businesses
                    │                                   ├── Check for Real Website
                    │                                   │     ├── [has website] → Firecrawl Scrape → Combine Data
                    │                                   │     └── [no website]  → Build Gemini Request directly
                    │                                   │                               └── Gemini 2.5 Flash Analysis
                    │                                   │                                     └── Parse & Score Lead
                    │                                   │                                           └── Store Lead Intelligence (Google Sheets)
                    └── Mark Keyword as Processed → Wait → Next Keyword
```

---

## Setup

### Prerequisites

- [n8n](https://n8n.io) — self-hosted or cloud
- A Google account with access to Google Sheets
- API keys for three services (see below)

### 1. API Keys

| Service | Purpose | Where to get it |
|---|---|---|
| [SerpAPI](https://serpapi.com) | Google Maps business search | serpapi.com → Dashboard |
| [Firecrawl](https://firecrawl.dev) | Website content scraping | firecrawl.dev → API Keys |
| [Google Gemini](https://aistudio.google.com) | AI opportunity analysis | Google AI Studio → Get API Key |

### 2. Google Sheets Setup

Create a Google Sheet with **two tabs**:

#### Tab 1: `keywords`

| Column | Description |
|---|---|
| `id` | Unique row ID (e.g. `1`, `2`, `3`) |
| `keyword` | Search query (e.g. `dentist in Chicago`) |
| `city` | City name |
| `area` | Neighbourhood or district |
| `niche` | Business category label |
| `processed` | Leave blank — the workflow sets this to `yes` after processing |

#### Tab 2: `store lead intelligence`

The workflow writes to this sheet automatically. Create it with these columns:

`place_id` · `business_name` · `keyword` · `keyword_row_number` · `niche` · `city` · `area` · `address` · `website` · `phone` · `google_maps_link` · `rating` · `reviews` · `category` · `automation_score` · `urgency_score` · `lead_score` · `automation_need_score` · `ai_summary` · `issues_count` · `high_severity_issues` · `automation_opportunities` · `pain_points` · `outreach_angle` · `recommended_solution` · `best_offer` · `decision_maker` · `contacted` · `status` · `scraped_at`

### 3. Import Into n8n

1. Download `SMB Opportunity Intelligence Pipeline (1).json`
2. In n8n, go to **Workflows → Import from File**
3. Open the workflow and replace every placeholder:

| Placeholder | Replace with |
|---|---|
| `YOUR_SERPAPI_KEY` | Your SerpAPI key |
| `YOUR_FIRECRAWL_API_KEY` | Your Firecrawl key (keep the `Bearer ` prefix) |
| `YOUR_GEMINI_API_KEY` | Your Gemini API key |
| `YOUR_GOOGLE_SHEET_ID` | The ID from your Google Sheet URL |
| `YOUR_GOOGLE_SHEETS_CREDENTIAL_ID` | Your n8n Google Sheets credential ID |

4. Connect a **Google Sheets OAuth2** credential in the three Sheets nodes
5. Activate the workflow

---

## Lead Scoring

Each business gets three scores:

### Lead Score (0–100)
Composite of business quality (40%) and automation need (60%).

**Business quality factors:**
- Review count: up to 30 points (`reviews / 10`, capped at 30)
- Rating quality: up to 40 points (sweet spot around 4.0–4.5)
- Real website: 20 points (excluded if social profile only)

**Automation need factors:**
- Number of detected issues (up to 40 pts)
- High-severity issue count (up to 30 pts)
- AI confidence score (up to 30 pts)

### Automation Score (0–100)
Gemini's direct assessment of how much automation opportunity exists at this business.

### Urgency Score (0–100)
Gemini's assessment of how urgently the business needs help — useful for prioritising outreach order.

---

## Business Filtering Criteria

Businesses pass through two filter stages before reaching the AI:

**Stage 1 — Quality filter:**
- More than 50 Google reviews
- Rating between 3.5 and 4.7 (established but imperfect — prime for improvement)
- Has a website listed on Google Maps

**Stage 2 — Real website check:**
Excluded if the website is one of: `instagram.com`, `facebook.com`, `zomato`, `swiggy`, `linkedin.com`, `twitter.com`, `x.com`

Businesses with no real website still proceed to AI analysis — Gemini flags weak digital presence as an opportunity.

---

## AI Analysis

Gemini 2.5 Flash analyses each business and returns a structured JSON with:

- `issues[]` — list of detected problems, each with severity, suggested solution, business impact, and confidence score
- `automation_score` — overall automation opportunity rating
- `urgency_score` — outreach urgency rating
- `outreach_angle` — recommended hook for cold outreach
- `best_offer_to_pitch` — the specific service or solution most likely to resonate
- `likely_decision_maker` — who to contact at this business
- `ai_opportunity_summary` — one-paragraph narrative on the opportunity

The model is prompted to think like an automation agency, AI consultant, SaaS founder, and cold outreach strategist simultaneously.

---

## Scheduling

The workflow includes a **Schedule Trigger** (disabled by default) set to run every 24 hours. Enable it to run LeadRadar automatically each day — it will pick up any new unprocessed keywords added to the sheet overnight.

---

## Tech Stack

- **n8n** — workflow automation
- **SerpAPI** — Google Maps data
- **Firecrawl** — website scraping
- **Google Gemini 2.5 Flash** — AI analysis
- **Google Sheets** — input keywords + output leads
