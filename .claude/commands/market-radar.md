# Market Radar - Daily Intelligence Report

Generate a daily HTML report of articles relevant to Ansible Automation Platform strategy.

## Sub-commands

Check if the user's message contains a sub-command:
- **"setup"** → Run the Setup flow (see below)
- **"refresh-outcomes"** → Run the Refresh Outcomes flow (see below)
- Otherwise → Run the full Daily Pipeline

---

## Daily Pipeline

### Step 1: Load Configuration

Read these files:
- `config/sources.yaml` — RSS feeds and search queries
- `config/topic-map.yaml` — Theme definitions, outcomes, keywords
- `.state/seen-articles.json` — Previously seen article URLs

### Step 2: Fetch RSS Feeds

For each feed in `sources.yaml`:
- Use **WebFetch** to retrieve the feed URL
- Parse the XML/RSS/Atom response to extract article entries (title, link, date, description)
- Filter to articles published within the last `freshness_days` days
- Collect all entries into a candidate list

Process feeds in batches of 5 parallel WebFetch calls.
If a feed fetch fails, log the error and continue — never abort the pipeline.

### Step 3: Web Search

For each search query in `sources.yaml`:
- Use **WebSearch** with the query string
- Extract article URLs, titles, and snippets from results
- Add to the candidate list

Process searches in batches of 5 parallel WebSearch calls.

### Step 4: Deduplicate

- Normalize all candidate URLs (strip tracking params like utm_*, remove trailing slashes)
- Skip any URL already present in `seen-articles.json`
- Skip duplicate URLs within the current batch
- Log how many candidates were filtered

### Step 5: Fetch & Summarize

For each new unique article (up to 50 max per run):
- Use **WebFetch** to retrieve the full article page
- Extract: title, author (if available), publication date, source domain
- Write a 2-3 sentence summary of the article content
- Process in batches of 5 parallel WebFetch calls
- If a fetch fails, skip the article and continue

### Step 6: Classify & Score

For each fetched article, using the topic-map.yaml themes:
1. Match the article against ALL themes using both keyword matching and semantic understanding
2. Assign each matching theme a relevance score: **HIGH**, **MEDIUM**, or **LOW**
   - **HIGH**: Article directly discusses the theme as a primary topic; highly actionable for the linked ANSTRAT Outcomes
   - **MEDIUM**: Article is relevant to the theme but not its primary focus; useful context
   - **LOW**: Article has tangential relevance only
3. An article can map to multiple themes with different relevance levels
4. Drop articles that only match LOW across all themes — they add noise
5. For kept articles, record: title, url, author, date, source, summary, and a list of `{theme_id, relevance, outcomes}` mappings

### Step 7: Generate HTML Report

Read the template from `templates/report.html` and populate it:

1. **{{DATE}}**: Today's date in "March 9, 2026" format
2. **{{TOTAL_ARTICLES}}**: Count of articles in the report
3. **{{HIGH_COUNT}}**: Count of articles with at least one HIGH relevance
4. **{{SOURCES_SCANNED}}**: Number of feeds + searches attempted
5. **{{NAV_LINKS}}**: Generate sidebar navigation links, one per theme that has articles:
   ```html
   <a href="#theme-id">Theme Label <span class="count">N</span></a>
   ```
6. **{{THEME_SECTIONS}}**: For each theme that has articles (sorted by theme order in topic-map.yaml):
   ```html
   <section class="theme-section" id="theme-id">
     <div class="theme-header">
       <h2>Theme Label</h2>
       <span class="outcomes">ANSTRAT-XXXX, ANSTRAT-YYYY</span>
     </div>
     <!-- Articles sorted: HIGH first, then MEDIUM -->
     <div class="article-card relevance-high">
       <div class="title"><a href="URL" target="_blank">Article Title</a></div>
       <div class="meta">
         <span class="relevance relevance-high">HIGH</span>
         · Source · Author · Date
       </div>
       <div class="summary">2-3 sentence summary.</div>
       <div class="outcomes-list">
         <span>ANSTRAT-XXXX</span>
         <span>ANSTRAT-YYYY</span>
       </div>
     </div>
   </section>
   ```

Write the completed HTML to `reports/market-radar-YYYY-MM-DD.html` (using today's date).

### Step 8: Update State

- Add all processed article URLs to `seen-articles.json` with metadata:
  ```json
  {
    "URL_HASH": {
      "url": "...",
      "title": "...",
      "date_seen": "2026-03-09",
      "themes": ["theme-id-1", "theme-id-2"]
    }
  }
  ```
  Use a simple hash: the URL string itself as the key.

- Print a summary to the user:
  ```
  Market Radar — March 9, 2026
  ─────────────────────────────
  Feeds scanned:    8
  Searches run:     20
  Candidates found: 45
  After dedup:      32
  Articles kept:    18 (14 dropped as low-relevance)
  HIGH relevance:   5
  MEDIUM relevance: 13
  Report: reports/market-radar-2026-03-09.html
  ```

### Step 9: Publish to GitHub Pages

1. Read `README.md` and add a new row to the top of the Reports table:
   ```markdown
   | March 11, 2026 | [market-radar-2026-03-11.html](reports/market-radar-2026-03-11.html) | N articles, M HIGH — top 2-3 headline summaries |
   ```
2. Read `reports/index.html` and add a new `<li>` entry at the top of the report list:
   ```html
   <li><a href="market-radar-YYYY-MM-DD.html">Month DD, YYYY<span class="date">N articles</span></a></li>
   ```
3. Stage the new report, updated README.md, updated reports/index.html, and .state/seen-articles.json
4. Commit with message: `Add Market Radar report for YYYY-MM-DD`
5. Push to origin main
6. Print the GitHub Pages URL: `https://cross-logic.github.io/market-radar/`

---

## Setup Flow (`/market-radar setup`)

1. Read `seeds/seed-urls.txt` — extract all non-comment, non-empty lines as URLs
2. For each seed URL (batch of 5):
   - **WebFetch** the page
   - Identify the source domain
   - Look for RSS/Atom feed links in the HTML (`<link rel="alternate" type="application/rss+xml">` etc.)
   - Note the article topic for query generation
3. Compile discovered feeds and update `config/sources.yaml`:
   - Add new feeds not already present
   - Suggest additional search queries based on article topics
4. Report what was discovered and what was added

---

## Refresh Outcomes Flow (`/market-radar refresh-outcomes`)

1. Read `config/topic-map.yaml` to get all ANSTRAT issue keys
2. For each unique ANSTRAT key, use **mcp__mcp-atlassian__jira_get_issue** to fetch the issue
3. Extract: key, summary, status, description snippet
4. Write results to `.state/outcomes-cache.json`:
   ```json
   {
     "ANSTRAT-1646": {
       "summary": "...",
       "status": "In Progress",
       "last_refreshed": "2026-03-09"
     }
   }
   ```
5. Compare outcomes against topic-map themes:
   - Flag any ANSTRAT issues that are Done/Closed (may need removal)
   - Suggest keyword additions based on outcome descriptions
6. Print summary of findings

---

## Error Handling Rules

- **Never abort the pipeline** for a single fetch/search failure
- Log failed URLs and continue
- If ALL feeds fail, still run web searches
- If ALL searches fail, still generate report from whatever was collected
- If zero articles survive classification, generate an empty report noting "No new articles found"
- Always write the seen-articles.json update, even on partial runs

## Processing Limits

- Max 50 articles fetched per run
- Max 30 articles in final report
- Batch size: 5 parallel tool calls
- Skip articles older than `freshness_days` from sources.yaml

$ARGUMENTS
