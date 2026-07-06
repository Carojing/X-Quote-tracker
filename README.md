# X Quote Tracker

A Codex skill for tracking quote-post performance on X/Twitter.

Use it when you have an X post URL and want to collect quote posts within 24h/48h windows, including:

- quote URL
- quote publish time
- views
- likes
- Engage(likes)/View
- replies and reposts
- TSV / JSON / XLSX outputs

## Install Locally

Copy this folder into your Codex skills directory:

```bash
cp -R x-quote-tracker ~/.codex/skills/
```

Restart Codex if the skill does not appear immediately.

## Example Prompt

```text
Use $x-quote-tracker to track 24h and 48h quote views, likes, and Engage/View for this X post:
https://x.com/sparkohai/status/2073799620392808840
```

## Output Files

The skill can produce:

- `<basename>_quote_tracking.json`
- `<basename>_quote_tracking.tsv`
- `<basename>_quote_tracking.xlsx`

## Notes

X often requires a logged-in browser session to show quote lists and view counts, so the skill is designed to use Chrome browser automation when available.
