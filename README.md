

**An SEO tool that doesn't stop at "here's what's wrong."**

Most SEO audit tools detect issues and hand you a list. This one closes
the loop: **crawl → detect → AI-generate a fix → a human approves it →
deploy straight to WordPress → verify it landed.** Nothing publishes
without a person clicking approve first, and every deploy is reversible.

---

## What makes this different

- **The loop actually closes.** Detected issues become AI-drafted fixes
  (title tags, meta descriptions, H1s, image alt text) that a human
  reviews and approves before anything touches the live site — then the
  approved fix is pushed to WordPress via a companion plugin and can be
  rolled back with one click.
- **Providers degrade, they don't break.** Keyword and SERP data comes
  from two independent providers (Semrush, DataForSEO) with automatic
  fallback — a dead account on one side degrades data depth instead of
  taking the feature down.
- **A real job/schedule system, not a cron hack.** Crawls, rank checks,
  backlink pulls, and keyword refreshes all run through one generic
  `Job`/`Schedule` model with independent worker lanes, so a slow crawl
  can't starve fast API-backed jobs queued behind it.

---

## What's shipped

| Feature | What it does |
|---|---|
| **Site Audit** | Crawls a site (sitemap + BFS discovery), checks title/meta/H1/H2/alt/schema/canonical/OpenGraph/Twitter/lang/thin-content, scores every page 0–100. |
| **AI Suggestions** | Claude-generated fixes for detected issues, with a full accept/reject/edit state machine before anything is deployed. |
| **WordPress Auto-Deploy + Rollback** | Approved fixes push live via a companion WordPress plugin (Yoast meta, post titles, media alt text), with token-encrypted connections and one-click rollback. |
| **Keyword Research** | Workspaces, tracked/saved keywords, dual-provider lookups (Semrush + DataForSEO) with honest no-data/error states — never a fake blank row. |
| **Rank Tracking** | Scheduled SERP position checks against your own domain, feeding trend data and an "Easy Wins" view — gated on real position data actually existing, not a placeholder flag. |
| **Backlink Analysis** | Authority score, referring domains, total backlinks, and new/lost link diffing on a schedule. |

Everything above is verified against the live codebase and database, not
inferred from a roadmap — see `prompts/audit-log-compiled.md` for the
dated, file:line evidence behind each item and `CHANGELOG.md` for
what's changed recently.

**Not built yet** (present in the UI as visibly disabled, not hidden):
Content Calendar, AI Visibility, Schema Generator, Link Analyzer,
Competitor Analysis, AI Writer, Reports, Settings. These are placeholders
for future work, not partially-working features.

---

## Screenshots

> 📸 **Add real screenshots/GIFs here before publishing.** None exist yet
> — placeholders below intentionally left unfilled rather than faked.

- [ ] Site Audit results view
- [ ] AI Suggestion review + approve flow
- [ ] WordPress deploy + rollback
- [ ] Keyword Research workspace
- [ ] Rank Tracking / Easy Wins card

---

## Tech stack

![FastAPI](https://img.shields.io/badge/FastAPI-009688?style=flat&logo=fastapi&logoColor=white)
![SQLAlchemy](https://img.shields.io/badge/SQLAlchemy-D71F00?style=flat&logo=python&logoColor=white)
![SQLite](https://img.shields.io/badge/SQLite-003B57?style=flat&logo=sqlite&logoColor=white)
![Jinja2](https://img.shields.io/badge/Jinja2-B41717?style=flat&logo=jinja&logoColor=white)
![Claude API](https://img.shields.io/badge/Claude_API-D97757?style=flat&logo=anthropic&logoColor=white)
![Semrush](https://img.shields.io/badge/Semrush-FF642D?style=flat)
![DataForSEO](https://img.shields.io/badge/DataForSEO-2E5BFF?style=flat)
![APScheduler](https://img.shields.io/badge/APScheduler-306998?style=flat)
![pytest](https://img.shields.io/badge/pytest-0A9EDC?style=flat&logo=pytest&logoColor=white)

---

## Quickstart

```bash
git clone <repo-url>
cd SEO-Automation

python -m venv .venv
.venv\Scripts\activate      # Windows
# source .venv/bin/activate # macOS/Linux

pip install -r requirements.txt
playwright install chromium   # required by crawl4ai

cp .env.example .env
# then fill in real values — see .env.example for what each key does
# and where to get it

python -m pytest             # 71 tests, should all pass with zero config
uvicorn app.main:app --reload
```

App runs at `http://127.0.0.1:8000`. SQLite database (`seo_automation.db`)
is created automatically on first run — no separate DB setup needed.

Requires Python 3.10–3.12 (`crawl4ai` pins `lxml~=5.3`, which has no
prebuilt wheel for 3.14+ yet).

---

## Architecture

Two design choices worth calling out, since they're not the obvious
"just add a cron job" / "just call the API" defaults:

### Job/Schedule system, not ad-hoc background tasks

Every recurring operation (crawl, rank check, backlink pull, keyword
refresh, audit) is a `job_type` string routed through one `JOB_HANDLERS`
dict. A generic `Schedule` table (unique on `project_id` + `job_type`)
holds each schedule's interval and `next_run_at`; a generic `Job` table
is the actual work queue. Two independent APScheduler ticks drive it:

- `dispatch_due_schedules` (every 60s) — finds due `Schedule` rows,
  creates a `Job` row for each, advances `next_run_at`.
- Two worker lanes, each on a 10s tick, each picking the oldest queued
  `Job` in its lane: a **crawl lane** (network/browser-bound, 900s
  timeout) and a **light lane** (API-backed jobs, 180s timeout) — so a
  slow crawl can't block a fast rank check queued behind it, and vice
  versa. Jobs run as killable subprocesses (`app/jobs/runner.py`) since
  crawl4ai/Playwright can hang the parent thread indefinitely on Windows.

Adding a new scheduled feature is one `JOB_HANDLERS` entry plus a handler
module — nothing else in the scheduler changes.

### Provider-adapter pattern for keyword/SERP data

`app/keyword_provider.py` is the only module the rest of the app talks
to for keyword and SERP data. Underneath it, `app/semrush.py` and
`app/dataforseo.py` are interchangeable adapters with the same per-call
contract: return a real row, `{"no_data": True}` (call succeeded, genuinely
nothing to show), or `{"error": ...}` (the call itself failed). Callers
branch on that three-way state explicitly — a broken provider is never
silently rendered as an empty-but-successful result. Each data path picks
its own primary/fallback order (Semrush-first for keyword overviews,
DataForSEO-first for SERP checks) based on which provider is stronger
for that specific call, and a `_source` tag on the result records which
one actually answered — so a degraded fallback is traceable, not hidden.

---

## License

Proprietary — all rights reserved. 

## Contributing

Internal project for now — not accepting external contributions or PRs
at this stage.
