# SEO Automation Workflow (n8n)

## Description

An n8n-based automation that analyzes Google search results for a given keyword, scrapes competitor websites via ScraperAPI, and generates optimized meta content (H1, Meta Title, Meta Description) using AI.

---

## Architecture

### Workflow Nodes

| Node | Purpose |
|------|---------|
| `Get row(s) in sheet` | Reads input data (Keyword, GEO, Language) from Google Sheets |
| `Code in JavaScript3` | Deduplication — skips rows where Result column is already filled |
| `HTTP Request (SerpAPI)` | Performs Google search with GEO targeting, retrieves TOP-10 results |
| `Code in JavaScript` | Filters TOP-3 affiliate sites, excludes irrelevant domains |
| `Split Out` | Splits competitor array into individual items for sequential processing |
| `HTTP Request1 (ScraperAPI)` | Scrapes each competitor website with anti-block proxy |
| `Code in JavaScript1` | Parses HTML, extracts H1, Meta Title, Meta Description |
| `Aggregate Competitors` | Groups all competitor data by keyword |
| `Basic LLM Chain (Groq)` | Generates optimized meta via llama-3.3-70b |
| `Code in JavaScript2` | Parses AI JSON response, applies Title Case and length validation |
| `Create a document` | Creates a Google Doc named `{Keyword}-{GEO}` |
| `Update a document` | Fills the document with competitor data and SEO content |
| `HTTP Request3` | Applies Heading 1 style to the document title |
| `HTTP Request2` | Sets Commenter access on the document |
| `Update row in sheet` | Writes the document link to the Result column |

---

## Deduplication Logic

- Before processing, the workflow checks the Result column in Google Sheets
- Rows with an existing link are skipped automatically
- Only rows with an empty Result field are processed
- If all rows are already processed, the workflow stops automatically

> **Important:** to regenerate an already processed row — first clear the link from the Result column, then run the workflow again.

---

## Scraping Logic

- Search is performed via SerpAPI with parameters `q=Keyword` and `gl=GEO`
- The first 3 relevant sites from TOP-10 are selected
- Blacklisted domains: google, youtube, wikipedia, apple, trustpilot, instagram, twitter, reddit and other irrelevant sources
- An additional topic filter is applied for gambling-related keywords
- Each site is scraped via ScraperAPI with `render=true&premium=true` to handle JS rendering and anti-bot protection
- Batching: 1 request every 5 seconds to avoid rate limit (429)
- On scraping failure (403/500) — the competitor is skipped, processing continues

### Known Scraping Limitations

Some sites cannot be scraped even with premium proxies:
- SPA sites (Angular/React) — H1 is rendered via JavaScript and not present in raw HTML (shown as `—`)
- Heavily protected domains — return 403 and are skipped automatically
- This is not critical: AI generates content based on available Meta Title and Meta Description

---

## AI Generation Logic

- Model: `llama-3.3-70b-versatile` via Groq API (free tier)
- Prompt includes all competitor data and strict generation rules:
  - H1 and Meta Title must start with the Keyword
  - Meta Title: 40–60 characters
  - Meta Description: 140–160 characters
  - No emojis, no stop-words (Discover, Thrilling, Enjoy, Excitement)
  - Generated strictly in the language specified in the Language column
  - Only one year (2026), no repetition
  - No language mixing

---

## Requirements

### Services & API Keys

| Service | Purpose | Link |
|---------|---------|------|
| SerpAPI | Google search | [serpapi.com](https://serpapi.com) |
| ScraperAPI | Website scraping | [scraperapi.com](https://scraperapi.com) |
| Groq | AI generation (free) | [console.groq.com](https://console.groq.com) |
| Google Account | Sheets, Docs, Drive | [google.com](https://google.com) |

> **ScraperAPI note:** the `premium=true` parameter is used. The `ultra_premium=true` parameter requires a higher-tier plan and will cause a 403 error on the basic plan.

---

## Setup Guide

### 1. Import the Workflow

1. Download the workflow JSON file
2. In n8n go to menu → **Import from file**
3. Upload the JSON

### 2. Configure Credentials

Set up the following connections in n8n:

- **Google Sheets account** — OAuth2 for Google Sheets
- **Google Docs account** — OAuth2 for Google Docs
- **Google Drive account** — OAuth2 for Google Drive
- **Groq account** — API key from console.groq.com
- **Query Auth account** (SerpAPI) — SerpAPI key
- **Query Auth account 2** (ScraperAPI) — ScraperAPI key

> **Important:** Google OAuth tokens expire periodically. If the workflow fails with `authorization grant... expired` — reconnect the Google credential (Reconnect button).

### 3. Configure the Spreadsheet

In the `Get row(s) in sheet` node, set your Google Spreadsheet URL.

Spreadsheet structure:

| Keyword | GEO | Language | Result |
|---------|-----|----------|--------|
| casino en ligne | FR | fr | (auto-filled) |
| aviator | BR | pt | (auto-filled) |
| 1win | IN | en | (auto-filled) |
| novibet | IE | en | (auto-filled) |

### 4. Run

1. Fill in the spreadsheet with keywords (leave the Result column empty)
2. In n8n click **Execute Workflow**
3. Once complete, Google Doc links will appear in the Result column

---

## Output Document Structure

Each Google Doc contains:

```
Analysis for [Keyword] - [GEO]    ← Heading 1

COMPETITOR REPORTS

Competitor 1
Position: X
URL: https://...
Title: ...
H1: ...
Meta Title: ...
Meta Description: ...

(up to 3 competitors)

OPTIMIZED SEO CONTENT
H1: ...
Meta Title: ...
Meta Description: ...
```

---

## Access to Results

- All created documents are shared via link with **Commenter** access
- Links are automatically written to the Result column of the source spreadsheet

---

## Troubleshooting

| Error | Cause | Solution |
|-------|-------|----------|
| `authorization grant... expired` | Google OAuth token expired | Reconnect the Google credential |
| `Authorization failed` (Groq) | Groq API key expired | Generate a new key at console.groq.com |
| 403 from ScraperAPI | Site is protected or plan limit reached | Not critical, competitor is skipped |
| 429 Too Many Requests | ScraperAPI rate limit | Increase Batch Interval to 5000 ms |
| Empty document | All competitors were blocked | Clear the Result link and re-run |
| H1 = `—` | Site renders H1 via JavaScript | Not critical, AI uses Meta Title instead |
