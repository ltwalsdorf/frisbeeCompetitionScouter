# Research Notes: How ulti-verse.com Works

Background research done while scoping our own scraper approach (see
`MISSION.md`). [ulti-verse.com](https://ulti-verse.com) is an existing
site that does something close to what we want (tournament results/scouting
for USAU teams), so we looked at how it's built to learn from its approach.
Everything below is inferred from **public, client-visible sources only**:
the site's shipped JS bundle and a public GitHub org — nothing private or
authenticated was accessed.

## Site architecture

- **Frontend:** a React single-page app, internally named `ultidb-ui`
  (visible in webpack chunk names in the bundle). All rendering happens
  client-side; a plain `curl` of any page just returns an empty `<div
  id="root">` shell.
- **Backend:** per the site's own credits text (found in the bundle):
  *"Built with React, Golang, and Postgres. Deployed on
  [Railway](https://railway.app/)."* So: Go API + Postgres database,
  hosted on Railway.
- **API:** the frontend calls its own first-party REST API at
  `https://ulti-verse.com/api`, versioned (`/v1/...`, `/v2/...`). The
  browser never talks to USAU directly — all USAU scraping happens
  server-side and is invisible over the network tab.

### Observed API endpoints (from bundle strings)
- `/v1/tournaments/{id}`
- `/v1/teams`, `/v2/teams`, `/v2/teams/search`, `/v2/teams/players-by-name`
- `/v1/teams/rosters/{id}`
- `/v1/profiles/{id}`
- `/v1/videos`, `/v1/videos/sources`
- `/v1/pageviews`
- `/v1/roles`, `/v1/roles/health`
- `/v1/ingest`
- `/v1/collisions`
- `/v1/scraper-health`

### Admin UI sections (also from bundle strings)
Users, Collisions, Bug Reports, Recently scraped, Video sources, Videos,
Stats — implies an internal moderation/admin panel for reviewing scraper
output and fixing bad data (e.g. "Collisions" sounds like it's for
resolving ambiguous/duplicate team matches).

## The scraper: `ultidb/source-data`

The bundle links to `github.com/ultidb/source-data` (branch `live`) as a
data credit. That's a **public** GitHub repo, actively pushed to, matching
the site's described stack closely enough that it's almost certainly the
real scraper feeding ulti-verse's Postgres DB (though this is inferred,
not confirmed by the ulti-verse team directly).

- **Language/stack:** Python, BeautifulSoup for parsing, Selenium +
  headless Chrome for fetching (USAU's pages aren't reliably scrapable
  with plain HTTP requests), Tor (`stem`/`pysocks`/`toripchanger`) for
  proxied/rotating requests, Flask + APScheduler for a server with
  background scraping jobs, Click for a CLI.
- **Config-driven:** `config.yaml` maps every USAU division name variant
  to a URL path segment, and maps year → USAU's internal `SeasonId`.
  Also configures scheduler cadence: calendar re-scrape every 8h, ongoing
  tournaments every 10min, upcoming every 12h, recently-ended every 4h,
  videos every 24h.
- **Storage:** raw HTML cached to `html/{year}/{tournament}/`, parsed
  output written to `csv/{year}/{tournament}.csv`. Years 2014–2026 are
  all present in the repo as real scraped data.
- **CSV data shape:** a hand-rolled multi-section format per tournament —
  header rows (name/division/date/location/source URL), then one block
  per team (name, seed, info, roster), separated by literal `break` rows,
  then a `stages` section with pool and bracket games (`teamA, teamB,
  scoreA, scoreB, datetime, round, status`). Note: due to a bug in their
  CSV writer (`writer.writerow("break")` instead of `writer.writerow(["break"])`),
  these marker rows literally appear as `b,r,e,a,k` and `s,t,a,g,e,s` in
  the file — one character per CSV column. Cosmetic, but any parser needs
  to detect it as such rather than expecting a clean `break`/`stages` cell.

### Branches
- **`main`** — updated only occasionally via batched "Live → Main" merge
  PRs. Can lag reality significantly.
- **`live`** — updated continuously by the scheduler, roughly every 10–30
  minutes. This is the branch with actually-current data.
- **`feat/score-report`** — a separate, abandoned side experiment
  (April–August 2024, no activity since). Not a deprecated version of a
  "score report" feature — it's a scraper for a *different* third-party
  site, `scorereport.net`, aimed at backfilling historical USAU results
  from 2004–2013 (before the main scraper's 2014 starting point). Includes
  an experiment feeding tournament bracket **images** to GPT-4 Turbo's
  vision API to extract games/scores as JSON, for brackets that only
  existed as images rather than structured HTML on that site.

### Data freshness pitfall (learned the hard way)
We pulled a tournament CSV from `main` for a tournament that had already
finished, and it showed several games as `0-0` / `Scheduled` — looking
stale. It wasn't scraper error: `main`'s last sync for that file predated
the tournament's second day entirely. The `live` branch had the correct,
complete results, updated within hours of the tournament ending. Lesson
for our own project: if we ever snapshot/batch our own scraped data
similarly, make sure whatever we treat as "current" is the continuously-
updated source, not a periodically-merged one.

### Security note (not our issue, but worth knowing)
The `feat/score-report` branch has a **plaintext OpenAI API key hardcoded**
in `openAiImageProcess.py`, committed to this public repo. It's old
(2024) and may already be revoked, but it's a good reminder for our own
project to keep secrets in `.env`/environment config only, never in
committed source — which, to their credit, their `main` branch does
correctly elsewhere (`.env`-based config via `pydantic-settings`).
