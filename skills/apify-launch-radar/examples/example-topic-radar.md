# Example — Discovery mode: "what launched on Product Hunt this week?"

A real run of the full chain (field names and the cross-platform match below
were verified against live Actor output; vote totals will differ run to run).

## Inputs collected (Step 1)

- **Subject:** weekly Product Hunt leaderboard (topic → Discovery mode)
- **Timeframe:** `weekly`
- **Platforms:** all three (Product Hunt, Hacker News, Reddit)
- **Volume:** PH 10, HN 10, Reddit 10 (small, to keep cost low)
- **Output:** Markdown digest

## Step 3 — Product Hunt (discovery)

```bash
apify actors call "nexgendata/product-hunt-scraper" \
  -i '{"timeframe":"weekly","maxProducts":10,"outputMode":"raw"}' \
  --json \
  --user-agent apify-awesome-skills/apify-launch-radar \
  2>/dev/null
```

Returns 10 products with `name`, `tagline`, `upvoteCount`, `commentCount`,
`makerName`, `topics`, `websiteUrl`, `launchDate`. Top of the board included
`Slashy` (a YC S25 AI tool) and `Cloudback for Linear`.

## Step 4 — Hacker News (reception)

```bash
apify actors call "harvestlab/hacker-news-scraper" \
  -i '{"mode":"search","searchQuery":"Slashy","maxItems":10,"minPoints":1}' \
  --json \
  --user-agent apify-awesome-skills/apify-launch-radar \
  2>/dev/null
```

Returns items with `title`, `points`, `commentCount`, `url`, `hackerNewsUrl`.
The join matched `Slashy` to **"Launch HN: Slashy (YC S25) – AI that connects to
apps"** at **70 points**.

## Step 5 — Reddit (discussion + sentiment)

```bash
apify actors call "trudax/reddit-scraper-lite" \
  -i '{"searches":["Slashy AI"],"maxItems":10,"maxPostCount":10,"sort":"relevance","skipComments":true}' \
  --json \
  --user-agent apify-awesome-skills/apify-launch-radar \
  2>/dev/null
```

Returns posts with `title`, `communityName`, `url`, `username`, `body`,
`createdAt` (lite → no numeric votes). A couple of launch-discussion threads
matched; their `body` text feeds the sentiment read.

## Step 6 — Merged digest (rendered)

> Traction score = `PH_upvotes + PH_comments + (HN_points × 2) + HN_comments + (Reddit_posts × 5)`
> (Reddit uses post count because the *lite* Actor returns no vote counts; swap
> in the full `trudax/reddit-scraper` for `upVotes`/`numberOfComments`.)

| Rank | Product | Tagline | PH up/com | HN pts/com | Reddit posts | Score | Sentiment | Evidence |
|------|---------|---------|----------:|-----------:|-------------:|------:|-----------|----------|
| 1 | Slashy | AI that connects to your apps | (live) | 70 / (live) | 2 | (computed) | positive | "Launch HN: Slashy (YC S25)" (HN, 70 pts) + 2 Reddit threads |
| 2 | Cloudback for Linear | Backup for Linear | (live) | — | 0 | (computed) | no-signal | PH only — no HN/Reddit hits |

Delivered as Markdown by default; offer CSV/JSON on request.
