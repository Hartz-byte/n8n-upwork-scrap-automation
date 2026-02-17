# Automated Startup Outreach System (Local AI & n8n)

## Project Overview
This project is an automated workflow designed to identify newly registered Indian startups from news sources, enrich their data, and prepare personalized outreach messages. It is designed with a "Zero-Cost" architecture, utilizing self-hosted n8n and a local LLM (Ollama) to eliminate API costs for intelligence and processing.

## Architecture
The system follows a linear "Over-fetch, Filter, and Keep the Best" architecture to ensure high-quality data output despite the unpredictability of AI and public feeds.

1.  **Discovery:** Aggregates news from RSS feeds (Inc42, YourStory).
2.  **Buffering:** Fetches a larger batch (12 items) to account for filtering attrition (removing news publishers and invalid articles).
3.  **Extraction (Local AI):** Uses Ollama (Mistral model) to extract structured entity data from unstructured article text.
4.  **Sanitization:** Cleans raw LLM output to prevent JSON parsing errors.
5.  **Enrichment:** Queries Clearbit (Free Autocomplete API) to validate company domain names and logos.
6.  **Validation & Logic:** Applies strict filters to remove news publishers and incomplete records.
7.  **Storage:** Saves only valid, enriched leads to Airtable.

## Prerequisites & Setup

### 1. n8n (Self-Hosted)
The workflow runs on a local Docker container.
**Run Command:**
```bash
docker run -it --rm -p 5678:5678 -e N8N_HOST=localhost -e N8N_PORT=5678 -e N8N_PROTOCOL=http -v C:/n8n-data:/home/node/.n8n n8nio/n8n
```

### 2. Ollama (Local LLM)
Ensure Ollama is running locally and exposing the API.

Port: 11434

Model: Mistral (run `ollama run mistral` to pull)

Connection: The workflow connects to Ollama via HTTP Request/LangChain node (typically http://host.docker.internal:11434/ on Docker).

### Notes on Data Sources and Paid Services

- Free vs Paid: Paid APIs and commercial scraping/data-extraction services generally provide higher coverage, more reliable entity extraction, better deduplication, and lower false positives than free RSS-based approaches. Consider paid services if you need higher recall/precision or commercial SLAs.
- Current feeds: This workflow currently uses free RSS Read with two URLs:

	- https://inc42.com/feed/
	- https://yourstory.com/feed/


## Workflow Breakdown

1. Ingestion Layer

- RSS Read: Consumes feeds from major startup news outlets.
- Merge: Combines multiple feed sources into a single list.
- Code (Buffer): Slices the top 12 items to process. We intentionally process more than the required 5 to allow for the removal of invalid items (e.g., reports about Google or the news agency itself).

2. Processing Layer (The Loop)

- Loop Over Items: Processes articles one by one (Batch Size: 1) to manage local CPU/RAM usage for the LLM.
- Basic LLM Chain: Sends the article content to Ollama.
- Prompt Strategy: Instructs the AI to extract Company Name, Founder, Industry, Location, Stage, and draft outreach messages. Enforces strict JSON output.

Parser (Code Node):

- Extracts the JSON object from the LLM response.
- Crucial Fix: Sanitizes the string using regex `replace(/[^\\u0000-\\u0019]+/g, "")` to remove invisible control characters that frequently break `JSON.parse()`.
- Sets default values for `target_customer` (B2B) if missed by the AI.

3. Enrichment Layer

Clearbit (HTTP Request):

- Endpoint: `https://autocomplete.clearbit.com/v1/companies/suggest`
- Settings: "Always Output Data" is set to ON. This ensures the loop does not crash if Clearbit finds no results (returns an empty object).

4. Validation & Logic Layer

- Final Code: The core logic engine.
- Retrieval: Uses `$('Parser').last().json` to safely retrieve the LLM extraction from the previous step inside the loop.
- Blocklist: Checks `company_name` against a list of publishers (Inc42, YourStory, Entrackr, TechCrunch). If matched, `save_to_airtable` is set to `false`.
- Completeness: Checks for missing Founders.
- Defaults: Forces "Early Stage" and "B2B" if fields remain empty.
- Email Construction: Generates `hello@company.com` if a verified email is not found.
- Filter (IF Node): Checks if `save_to_airtable` is `true`. Only valid startups pass this gate; invalid items are discarded here.

5. Storage Layer

- Airtable: Creates a new record with the enriched, validated fields.

## Challenges & Solutions

1. JSON Parsing Crashes

Problem: The local LLM often outputted valid-looking JSON that contained invisible "bad control characters" (newlines or tabs inside strings), causing the workflow to crash at the parsing step.

Solution: Implemented a regex sanitization step in the Parser node before attempting to parse the object.

2. The "Diamond" Loop Issue

Problem: Branching the workflow to call Clearbit and then trying to merge back resulted in lost data contexts within the n8n loop (data mismatch).

Solution: Switched to a linear flow. Instead of merging, the downstream "Final Code" node references the upstream data using `$('NodeName').last().json`, which is the robust method for accessing active loop data.

3. Clearbit Empty Results

Problem: If Clearbit found no website, it returned an empty item, causing the loop to terminate early for that specific iteration.

Solution: Enabled "Always Output Data" in the Clearbit node and added logic in the Final Code to handle empty result sets gracefully without breaking the flow.

4. False Positives (News Publishers)

Problem: The AI would correctly identify "Inc42" as the subject of an article (e.g., "Inc42 announces new event"). This is technically correct but useless for lead generation.

Solution: Implemented a hardcoded blocklist in the Final Code validation step to reject any company name containing known publisher keywords.

## Filter Criteria

The system automatically rejects an item if:

- Company Name contains: "Inc42", "YourStory", "Entrackr", "Oracle", "Google", "TechCrunch", or "INVALID".
- Founder field is empty or generic ("Startup Team" logic handles defaults, but empty strings are flagged).
- JSON Parsing failed completely (marked as INVALID).

## Deliverables

The workflow outputs the following cleaned fields to Airtable:

- Company Name
- Founder Name(s)
- Industry
- Location (City)
- Startup Stage (e.g., Early Stage, Series A)
- Target Customer (B2B/B2C)
- Contact Email (Inferred or Extracted)
- Website URL
- Logo URL
- Outreach Drafts (Email & WhatsApp)
