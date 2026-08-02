# Pagsanjan Land Tracker

A self-updating dashboard tracking asking prices for agricultural and raw land in
Pagsanjan, Laguna, Philippines and nearby towns within roughly 10 km.

**Live dashboard → https://buffalobill208.github.io/pagsanjan-land-tracker/**

---

## Purpose

The dashboard exists to answer one practical question: *what are comparable
agricultural parcels in this area actually asking, right now?*

It is used to price and position family-owned agricultural parcels realistically for
a fair and reasonably quick sale. Over time, the week-over-week tracking becomes the
more valuable half — listings that **disappear** indicate what price actually clears,
while listings that **persist for months** indicate what is overpriced.

The figures are asking prices scraped from public property listings. They are not
appraisals, valuations, or offers.

---

## How it works

A GitHub Actions workflow runs once a week. It invokes Claude Code headlessly, which
searches three public property sites, verifies each listing, and writes a JSON data
file. GitHub Pages serves a static dashboard that reads that JSON.

```
.github/workflows/weekly-snapshot.yml   the weekly job
CLAUDE.md                               instructions the agent follows
docs/index.html                         static dashboard template (committed once)
docs/.nojekyll                          tells Pages to serve files as-is
docs/data.json                          latest week's data — rewritten each run
docs/archive/data-YYYY-MM-DD.json       dated snapshots, one per run
reports/read-YYYY-MM-DD.md              written market read, one per run
```

The template is deliberately separate from the data. The agent writes only JSON;
it never regenerates the HTML. This keeps the design stable week to week and keeps
running costs low, since generating output tokens is the expensive part.

### Legacy files

The first run (2026-07-27) predates the template/data split and left two artifacts
that are no longer produced or updated:

- `snapshots/2026-07-27.json` — superseded by `docs/archive/data-*.json`, which is
  now both the archive and the file the next run diffs against.
- `docs/archive/dashboard-2026-07-27.html` — a self-contained HTML dashboard from
  the old architecture. Still opens on its own, but is not served by the template.

Both are kept for history and can be deleted safely. The 2026-07-27 entry in
`docs/archive/data-*.json` was converted by hand from that first snapshot rather than
written by a run, so that week-over-week comparison could begin from the second run
instead of restarting at a baseline.

### Confidence tiers

Not all sources permit automated access, so every row is labeled:

| Tier | Meaning |
|---|---|
| **VERIFIED** | The individual listing post was opened and read. The link opens that exact post, and price ÷ area equals the stated ₱/sqm. |
| **INDICATIVE** | Figures were read from search-result summaries only. The math checks out, but the individual post could not be opened, so the link goes to a search page. Directional market context, not a confirmed listing. |

Rows that fail verification are dropped rather than guessed at, and the reason is
recorded in the dashboard's honesty notes. These sites do not reliably publish
listing dates, so no "days on market" figure is shown.

---

## Operating it

### Schedule

Runs every **Sunday at 23:00 UTC** (Mondays 7:00 AM Philippine time), defined by the
cron expression in `weekly-snapshot.yml`. GitHub's scheduler is best-effort — runs
are sometimes delayed by 30 minutes or more during peak load, and occasionally
skipped entirely. This is normal and harmless for a weekly snapshot.

> **Branch caveat:** scheduled workflows only run from the repository's **default
> branch**. As long as everything stays on `main`, the schedule works. Moving the
> workflow to another branch silently stops it from running on schedule.

> **Inactivity caveat:** GitHub disables scheduled workflows after 60 days without
> repository activity. The weekly commit normally prevents this, but if runs stop for
> two months, re-enable the workflow in the Actions tab.

### Running it manually

Actions tab → **Weekly Pagsanjan Farmland Snapshot** → **Run workflow** → **Run
workflow**. Takes roughly 5–15 minutes.

### Pausing or stopping it

To pause or stop it: Actions tab → select the workflow → the **⋯** menu on the right
→ **Disable workflow**. Re-enable the same way. You can still trigger manual runs via
**Run workflow** while the schedule is disabled.

### Checking cost per run

Every run prints a cost summary directly into the Actions log — no downloads needed.

1. Actions tab → click the run you want
2. Click the **snapshot** job
3. Expand the **Show run summary** step

It reports:

```
subtype     : success          ← "error_max_turns" means it was cut off mid-task
is_error    : False
num_turns   : 42
cost_usd    : 1.23             ← what that run cost
  claude-sonnet-4-6: $1.0100   ← per-model breakdown
  claude-haiku-4-5:  $0.2200
```

The full machine-readable log is also attached to each run as the `run-log-N`
artifact, at the bottom of the run summary page.

If costs climb unexpectedly, the two usual causes are the model not being pinned (see
`ANTHROPIC_MODEL` in the workflow) or the agent regenerating the HTML template
instead of writing only JSON. The workflow fingerprints `docs/index.html` before and
after each run and warns if it was modified.

### Viewing an old week

The dashboard URL always shows the most recent run. To view an archived snapshot,
append `?date=` and the run date:

```
https://buffalobill208.github.io/pagsanjan-land-tracker/?date=2026-07-27
```

A banner confirms you are viewing an archive, with a link back to the latest.

Valid dates are the `data-YYYY-MM-DD.json` filenames in
[`docs/archive/`](docs/archive) — one per run. Ignore any `dashboard-*.html` files
there; those are legacy artifacts (see above) and are not reachable via `?date=`.
The matching written analysis for each week lives in [`reports/`](reports).

---

## Configuration

| What | Where |
|---|---|
| Search scope, sources, verification rules, output schema | `CLAUDE.md` |
| Schedule, model, turn limit, timeout | `.github/workflows/weekly-snapshot.yml` |
| Dashboard layout and styling | `docs/index.html` |
| API credentials | Repository secret `ANTHROPIC_API_KEY` (Settings → Secrets and variables → Actions) |

Adjusting the geographic ring, parcel size band, or source list means editing
`CLAUDE.md` — the agent reads it at the start of every run.

---

## Notes

This repository is public, so its contents are readable by anyone. The
`ANTHROPIC_API_KEY` is stored as an encrypted repository secret, is never written to
any file, and does not appear in logs. Visiting the dashboard costs nothing and
triggers no API calls — the published page is plain static HTML, CSS, and JavaScript.
API usage is incurred only during the weekly run.

Do not commit private documents, ownership details, or credentials to this repository.
