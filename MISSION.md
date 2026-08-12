# Mission Statement

## Overview

Build a website for our ultimate frisbee team that centralizes tournament
information sourced from [USA Ultimate (USAU)](https://www.usaultimate.org/).
All tournament, team, schedule, and result data is populated by a web
scraper that pulls from the USAU website — the site itself is a read-only
presentation layer over that scraped data.

The core use case: a team member should be able to open the site, see every
tournament the team has registered for, and drill into either the results
(if the tournament is over) or the schedule and scouting info (if it's
upcoming) — including a scouting report on any upcoming opponent.

## Data Source

- **Source of truth:** USAU website, accessed via a web scraper (not the
  USAU API, unless one becomes available/necessary).
- Scraped data should cover, at minimum: tournament metadata (name, date,
  location, format, USAU event URL), divisions/pools, team rosters/seeding,
  schedules, game scores, and final placements.
- The scraper is expected to run on some recurring or triggered basis to
  keep tournament data current as events progress (schedules posted,
  scores updated, placements finalized).

## Technical Approach: Scraping USAU

We're modeling our scraper on a public reference implementation,
[`ultidb/source-data`](https://github.com/ultidb/source-data) (Python,
BeautifulSoup + Selenium), which already solves most of the hard parts of
scraping USAU. See `ULTIVERSE-RESEARCH.md` for how it was found and how it
fits into a live production site (ulti-verse.com). Notes below are
technical takeaways relevant to building our own scraper — we don't need
to replicate its scale (it scrapes *all* USAU tournaments/divisions back
to 2014); we only need our team's tournaments and its opponents'.

### Pages to scrape
- **Season/year calendar:** `https://play.usaultimate.org/events/tournament/?ViewAll=true&IsLeagueType=false&IsClinic=false&FilterByCategory=AE&SeasonId=<id>`
  — lists all tournaments for a year. Each year has its own internal
  `SeasonId` (not the same as the year number) that has to be mapped by
  hand, e.g. 2024→19, 2025→20, 2026→21.
- **Current college/club schedule pages:** `usaultimate.org/college/schedule/`
  and `usaultimate.org/club/schedule/` — a simpler path for "what's
  happening right now" without needing the season ID.
- **Tournament page:** `play.usaultimate.org/events/<tournament-slug>/schedule/<division-path>/`
  — contains pools, brackets, games, scores, and team links. Division
  names on USAU are inconsistently labeled ("Club - Mixed", "Club Mixed",
  "Club - Mixed's", etc.) and each maps to a different URL path segment —
  we'll need a division-name → URL-path lookup table.
- **Team page:** linked from each team entry on a tournament page (`?TeamId=<id>`),
  contains roster and team info (location, coaches, website/social links).

### Rendering & access considerations
- USAU's tournament pages are not reliably scrapable with a plain HTTP
  GET — the reference scraper uses **headless Chrome via Selenium**, with
  explicit waits for content to load and page-identity validation (it
  checks the page's breadcrumb links match the tournament slug it
  requested, retrying if a stale/wrong page was returned).
- The reference scraper also proxies all requests through **Tor** with
  circuit rotation, which matters for scraping USAU's *entire* site
  without getting rate-limited/blocked. Given our scope is one team and
  its opponents (a small, bounded set of pages), we likely don't need
  Tor — plain headless Chrome should be fine — but should keep an eye out
  for rate limiting if scraping grows to cover many opponents' full
  histories.

### Data model (what to capture per tournament)
- **Tournament:** name, division, city/state, start/end date, source URL.
- **Team:** name, seed, USAU `TeamId`, roster (jersey # + player name),
  nickname/location/coaches/website/social links.
- **Game:** team A, team B, score A, score B, datetime, round/pool name,
  status (`Final` / `Scheduled` / `In Progress` / `Cancelled`).
- **Stages:** pool play (teams grouped into pools, round-robin games) and
  bracket play (single/double elimination games), both containing lists
  of `Game`s.

### Gotchas learned from studying real scraped data
- **Game rows are `teamA, teamB, scoreA, scoreB` — not "us" and
  "opponent."** Team order reflects whatever order USAU listed them in,
  not which team is "home." Any parser must match on team name/ID
  explicitly rather than assuming a column always belongs to a given
  side — misreading this produces exactly-backwards results (final score
  flipped, win reported as a loss).
- **Unplayed/future games appear as `0-0` with `status="Scheduled"`.**
  This is the natural signal for our "upcoming schedule" feature — a
  tournament isn't "past" or "future" as a whole, individual games within
  it are, so a tournament in progress can have a mix of `Final` and
  `Scheduled` games.
- **Team identity should be tracked by USAU's internal `TeamId`, not by
  display name.** Names can vary slightly between tournaments/years;
  the ID is the durable key for linking a team's history and for matching
  head-to-head history against our own team.
- **Don't let scraped data silently go stale.** In the reference repo's
  production setup, we watched a real example where a fully-completed
  tournament still showed unplayed/`Scheduled` games because the data
  store hadn't been refreshed after the games finished. Our scraper
  should keep re-scraping a tournament until every game reaches a
  terminal status (`Final`/`Cancelled`), not just scrape it once during
  the event.
- **Historical depth is inconsistent.** USAU's own site is reliably
  scrapable back to roughly 2014; deeper history requires other sources.
  Not a concern for our use case (recent opponents only), but worth
  knowing if the scouting report ever needs to reach further back.

## Core Features

### 1. Tournament List
- Show every tournament our team is/was registered for.
- Each entry should be visually distinguishable as **past** or **upcoming**.
- Clicking a tournament opens its detail page.

### 2. Tournament Detail Page — Past Tournaments
- Display final results: our team's placement/seed outcome.
- Show full results of games played at that tournament (opponents, scores).
- Link out to the tournament's USAU webpage.

### 3. Tournament Detail Page — Upcoming Tournaments
- Show our team's schedule for the tournament (game times, pools/bracket
  position if known).
- Show full tournament info as scraped from USAU (format, divisions,
  participating teams, location/dates).
- Link out to the tournament's USAU webpage.
- Each opponent on our schedule should be a clickable link to their
  **scouting report** (see below).

### 4. Opponent Scouting Report
For any given opponent team, show:
- **Tournament history:** every tournament that team has played in.
- **Within each of those tournaments:** the other teams they played and the
  scores of those games.
- **Seeding and placement:** the opponent's seed going into each tournament
  and their final placement.
- **Head-to-head history:** the most recent 5 matchups between our team and
  that opponent (if any exist), with scores and context (tournament/date).

### 5. USAU Cross-Linking
- Every tournament page (past or upcoming) includes a direct link to the
  corresponding USAU tournament webpage, so users can cross-reference the
  authoritative source.

## Stretch Goal: Game Projection

Project our team's likely next opponent(s) before results are final, based
on the tournament's bracket/pool format:

- Use the tournament format (pool play, bracket structure, seeding rules)
  to determine the decision tree of possible next games.
- Use seeding to estimate whether we are favored or projected to lose our
  currently scheduled game(s).
- Based on that projection, identify and surface the opponent we would
  face in the next round if the projected outcome holds.
- This is explicitly speculative/best-effort — it should be presented as a
  projection, not a guarantee, and should update as actual results come in.

## Out of Scope (for now)

- Live/real-time score updates during games (data freshness is tied to
  scraper run cadence, not live feeds).
- Data for teams/divisions we have no relationship to (scouting reports
  are opponent-driven, not a general USAU database browser).
