# Gotchas — apify-launch-radar

Cost guardrails, error recovery, and common pitfalls. Read this on demand when
building inputs or when a run fails.

## Cost guardrails

Apify Actors use one of three pricing models. Before running, check the model
via `apify actors info "ACTOR_ID" --json 2>/dev/null` (look at `pricingInfo`).

| Model | What to watch for |
|-------|-------------------|
| `FREE` | No cost — safe to run. |
| `PAY_PER_EVENT` / pay-per-result | Cost scales with results. The launch-radar Actors are mostly this. Estimate before running. |
| `FLAT_PRICE_PER_MONTH` | Subscription — runs are unlimited once paid. |

Because this skill **chains three Actors**, total cost is the sum of all three
runs. Estimate per platform, then sum:

- Product Hunt: `maxProducts` × per-item price.
- Hacker News: `maxItems` × ~$0.001/item, **plus ~$0.05 if `includeAiAnalysis` is on**.
- Reddit: `maxItems` / `maxPostCount` × per-result price.

### Confirmation thresholds (suggested)

- Estimated total cost **>$5** → warn the user.
- Estimated total cost **>$20** → require explicit confirmation before running.
- Always present cost as a **rough estimate** ("around $X"), not a guarantee.
- Default volumes (PH 50 / HN 30 / Reddit 40) are deliberately modest. Only raise
  them when the user asks for depth.

## Common errors

| Error | Cause | Fix |
|-------|-------|-----|
| Product Hunt returns 0 items | Topic has no leaderboard hits for the timeframe | Use `query` search instead of leaderboard mode; widen `weekly` → `monthly`. |
| HN search flooded with unrelated hits | Short/ambiguous product name (e.g. "Arc", "Bolt") | Add a qualifier: `"<product> launch"`, `"<product> app"`; raise `minPoints`. |
| Reddit run slow or over budget | `maxItems`/`maxPostCount` too high | Lower both; scope with `searchCommunityName`. |
| Empty cross-platform join | Product names differ across platforms (casing, suffixes) | Match case-insensitively; fall back to fuzzy contains; keep unmatched as `no-signal`. |
| `searches` ignored on Reddit | `startUrls` also provided | Use one or the other — never both in the same input. |

## Actor-specific notes

### `nexgendata/product-hunt-scraper`
- Leaderboard mode (`timeframe`) and search mode (`query`) are mutually exclusive — leave `query` empty for leaderboard.
- `maxProducts` caps at 1000; for a digest you rarely need more than ~50.

### `harvestlab/hacker-news-scraper`
- `mode: "show"` is the cheapest way to catch fresh launches (Show HN).
- `includeAiAnalysis` is optional and adds cost — only enable when the user wants an LLM theme/TLDR digest.

### `trudax/reddit-scraper-lite`
- Pay-per-result: every post/comment counts, so cap volume tightly.
- For sentiment, fetch a few comments (`maxComments`) — titles alone under-read tone.
