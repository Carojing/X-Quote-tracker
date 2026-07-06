---
name: x-quote-tracker
description: Track quote-post performance for a specific X/Twitter post. Use when the user provides an X status URL and asks for 24h/48h quote tracking, quote views, quote publish links, likes, Engage(likes)/View, paid KOL quote monitoring, or a spreadsheet/TSV report of quote posts.
---

# X Quote Tracker

## Overview

Use this skill to collect quote posts for one X status, classify each quote into 24h and 48h windows relative to the original post time, and deliver traceable JSON, TSV, and XLSX outputs with views, likes, and Engage(likes)/View.

## Workflow

1. Normalize the input status URL.
   - Strip query strings such as `?s=20`.
   - Preserve the original status id.
   - Use `https://x.com/<handle>/status/<id>` as the canonical original URL.

2. Read the original post in Chrome.
   - Use Chrome browser automation when available because X often hides quote data or views without logged-in state.
   - Capture the original post `time[datetime]`; this is the baseline for 24h/48h windows.
   - Capture the original post visible stats for context if useful.

3. Open the quotes page.
   - Navigate to `<canonical-original-url>/quotes`.
   - Scroll until no new quote status URLs appear for a full pass or the page clearly stops loading older quotes.
   - Extract only quote posts by other accounts; exclude the embedded original post inside each quote card.
   - For extraction details and browser code patterns, read `references/browser_extraction.md`.

4. Normalize quote rows.
   - Required fields per quote: `name`, `handle`, `quote_url`, `quote_published_at_utc`, `views`, `likes`.
   - Recommended fields: `replies`, `reposts`, `stat_label`, `text_preview`.
   - Compute:
     - `hours_after_original`
     - `within_24h`
     - `within_48h`
     - `Engage(likes)/View = likes / views`
     - `Engage(likes)/View %`

5. Build deliverables.
   - Save the raw extracted JSON for traceability.
   - Run `scripts/build_quote_tracking.py` to create:
     - `<basename>_quote_tracking.json`
     - `<basename>_quote_tracking.tsv`
     - `<basename>_quote_tracking.xlsx` when `openpyxl` is available
   - If the original post is younger than 24h, state that the 24h and 48h buckets are not final. If it is between 24h and 48h, state that 24h is final and 48h is still updating.

## Script Usage

Prepare an input JSON with this shape:

```json
{
  "original_url": "https://x.com/sparkohai/status/2073799620392808840",
  "original_published_at_utc": "2026-07-05T16:02:13.000Z",
  "quotes": [
    {
      "name": "Diana Osire",
      "handle": "@Diana_Osire",
      "quote_url": "https://x.com/Diana_Osire/status/2073972302794903866",
      "quote_published_at_utc": "2026-07-06T03:28:24.000Z",
      "views": 4026,
      "likes": 83,
      "replies": 6,
      "reposts": 7,
      "stat_label": "6 replies, 7 reposts, 83 likes, 6 bookmarks, 4026 views",
      "text_preview": "The fact that it generates editable CAD programs..."
    }
  ]
}
```

Then run:

```bash
python3 scripts/build_quote_tracking.py \
  --input /path/to/raw_quotes.json \
  --output-dir /path/to/output-dir \
  --basename sparkohai_2073799620392808840
```

## Final Response

Return:

- Count of quotes captured, count within 24h, count within 48h.
- The original post timestamp and extraction timestamp with concrete dates and timezone.
- Links to XLSX, TSV, and raw JSON files.
- Top 3-5 quote posts by views with handle, views, likes, and Engage(likes)/View.
- A caveat when a time window is not complete yet.

Keep browser/internal implementation details out of the final answer unless the user asks.
