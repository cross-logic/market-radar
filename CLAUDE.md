# Market Radar

Daily HTML intelligence report for Ansible Automation Platform strategy. Discovers articles from RSS feeds and web searches, classifies them by strategic theme, and maps them to ANSTRAT Jira Outcomes.

## Skills

- `/market-radar` — Run the full daily pipeline: fetch feeds, search web, classify articles, generate HTML report
- `/market-radar setup` — Discover RSS feeds from seed URLs in `seeds/seed-urls.txt`, update `config/sources.yaml`
- `/market-radar refresh-outcomes` — Query Jira for current ANSTRAT Outcomes, update cache, suggest topic-map changes

## Project Structure

```
config/sources.yaml      — RSS feeds + web search queries
config/topic-map.yaml    — 10 strategic themes → ANSTRAT Outcomes + keywords
templates/report.html    — Self-contained HTML template (inline CSS, no JS)
.state/seen-articles.json — Dedup store (URL → metadata)
.state/outcomes-cache.json — Cached ANSTRAT Outcome data from Jira
reports/                 — Daily HTML output (git-ignored)
seeds/seed-urls.txt      — User-provided article URLs for setup
```

## Configuration

- **sources.yaml**: Add/remove RSS feeds and search queries. Set `freshness_days` to control article age window.
- **topic-map.yaml**: Edit themes, ANSTRAT outcome mappings, and keywords. The skill uses both keyword matching and semantic understanding for classification.

## Key Design Decisions

- Pure Claude Code skill — no Python, no external dependencies
- Claude classifies articles (better nuanced relevance than keyword-only)
- YAML config for human-editable settings
- Flat JSON for dedup (~10K entries/year)
- Self-contained HTML reports (inline CSS, no JS, print-friendly)
