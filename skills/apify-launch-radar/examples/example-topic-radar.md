# Example — Discovery mode: "AI dev tools, this week"

A worked run of the full chain. Values are illustrative; real numbers come from
the live Actor runs.

## Inputs collected (Step 1)

- **Subject:** `AI dev tools` (topic → Discovery mode)
- **Timeframe:** `weekly`
- **Platforms:** all three (Product Hunt, Hacker News, Reddit)
- **Volume:** PH 50, HN 30, Reddit 40 (defaults)
- **Output:** Markdown digest

## Step 3 — Product Hunt (discovery)

```bash
apify actors call "nexgendata/product-hunt-scraper" \
  -i '{"timeframe":"weekly","maxProducts":50,"outputMode":"raw"}' \
  --json \
  --user-agent apify-awesome-skills/apify-launch-radar \
  2>/dev/null
```

Returns ~50 products. After keeping the dev-tooling ones, two lead the board:
`Acme Copilot` (842 votes) and `RepoForge` (511 votes).

## Step 4 — Hacker News (reception)

```bash
apify actors call "harvestlab/hacker-news-scraper" \
  -i '{"mode":"search","searchQuery":"Acme Copilot","maxItems":30,"minPoints":1}' \
  --json \
  --user-agent apify-awesome-skills/apify-launch-radar \
  2>/dev/null
```

`Acme Copilot` → a Show HN at 240 points / 96 comments. `RepoForge` → no HN hits.

## Step 5 — Reddit (discussion + sentiment)

```bash
apify actors call "trudax/reddit-scraper-lite" \
  -i '{"searches":["Acme Copilot"],"maxItems":40,"maxPostCount":40,"sort":"relevance"}' \
  --json \
  --user-agent apify-awesome-skills/apify-launch-radar \
  2>/dev/null
```

`Acme Copilot` → 3 r/programming threads, 1.2k combined upvotes, mostly positive.
`RepoForge` → one skeptical r/devops thread.

## Step 6 — Merged digest (rendered)

> Traction score = `PH_votes + (HN_points × 2) + HN_comments + Reddit_upvotes + (Reddit_comments × 2)`

| Rank | Product | Tagline | PH votes | HN pts/com | Reddit up/com | Score | Sentiment | Evidence |
|------|---------|---------|---------:|-----------:|--------------:|------:|-----------|----------|
| 1 | Acme Copilot | AI pair-programmer for tests | 842 | 240 / 96 | 1200 / 310 | 3278 | positive | "finally a copilot that writes runnable tests" (HN); PH/HN/Reddit links |
| 2 | RepoForge | Auto-fix CI failures | 511 | — | 80 / 45 | 681 | mixed | "neat idea, but how is this different from X?" (r/devops); PH/Reddit links |

Delivered as Markdown by default; offer CSV/JSON on request.
