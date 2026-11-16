# budget-ingest-app

A single-page tool for ingesting a bank CSV export, auto-categorizing
transactions, and letting Chris review/categorize anything the automatic
matching couldn't resolve - the mobile front end for Phase 2 of the budget
automation roadmap (see **Where things live** below).

If you are an AI coding assistant picking this repo up with no memory of
prior sessions, this file plus [`AGENTS.md`](AGENTS.md) are meant to be
everything you need to work on the app itself. The data contract this app
produces for Claude Code to consume, and the Excel-side reconciliation
pipeline it feeds into, live in a separate project (see below) - this repo
is the app only.

## Architecture

- **One file, no build step.** `index.html` is the entire app: HTML + CSS +
  vanilla JS inline. Same pattern as `ignite-trip-app` and `missionbudget01`
  - see `APP TEMPLATES/family-web-app/TEMPLATE_PROCESS.md` in the vault for
  the reusable process this is built on.
- **One pull at a time.** `STATE.pull` holds the in-progress ingestion
  session (a single bank CSV upload, its parsed transactions, and its
  balance snapshot). There's no history of past pulls kept in this app -
  once a pull is marked ready and a Claude Code session has built the next
  handoff from it, "Start over" clears it for the next one.
- **Firebase Realtime Database for sync** - reuses the existing
  `mcdonald-budget-01` project (same one `missionbudget01`'s dashboard
  uses), under a new path, `budget/ingest/current` - no new Firebase
  project, no new security rules (`budget/` is already open read/write).
- **Hosting is GitHub Pages**, once this repo is published (see
  **One-time setup still needed**, below).

## What this app does and doesn't do

**Does:**
- Parses a bank export CSV (transactions + the leading account-balance
  header lines every export starts with).
- Auto-categorizes each transaction using the same logic the Excel
  reconciliation pipeline already uses: the Amazon/Kroger-fuel/Kroger-
  McDonald's-ChickfilA-Walmart special cases, then a lookup against
  `archive_reference_full.csv` (Chris selects this file too, each session -
  see **Known limitations**).
- Lets Chris pick a category for anything that didn't match (a dropdown of
  every known sub, or type a new one).
- Lets Chris flag anything - on purpose, or automatically when he types a
  brand-new sub name - as "needs Claude," since a new sub usually means a
  new catalog row decision this app can't make on its own.
- Saves the whole staged pull to Firebase so a Claude Code session can read
  it when asked to build the next handoff.

**Does not do (by design, v1):**
- **Cross-pull dedup.** If the same transaction appears in two overlapping
  bank exports, this app does not know that - it has no memory of what's
  already been ingested in a *previous* pull. That safety net stays with
  the existing Claude-side reconciliation script
  (`dedupAndCategorize()` in `_budget_ingest_analysis/handoff_v26_build_writeplan.js`
  and its predecessors), which already does this correctly against the
  live T1 table. Practical effect: if Chris uploads a CSV that overlaps
  with the last pull, the overlapping rows will show up again as "needs
  review" here - redundant tapping, not a data error, since Claude's own
  build step still dedups correctly either way.
- **The actual reconciliation.** Shape 1/2/3 matching against T2, week-
  rollover, balance-snapshot writing into the real T1 table, building the
  handoff workbook, rebuilding the dashboard, and pushing - all of that
  stays in a Claude Code session, reading this app's staged output instead
  of a raw CSV.

## Data contract (what this app saves to Firebase)

Path: `budget/ingest/current` in `mcdonald-budget-01`.

```json
{
  "kind": "budget-ingest",
  "version": 1,
  "savedAt": "...",
  "pull": {
    "pullDate": "2026-11-20",
    "status": "staging | ready",
    "transactions": [
      {"id":"...", "dateStr":"11/20/2026", "serial":0, "description":"...",
       "amount":12.34, "isDebit":true, "account":"...",
       "sub":"...", "main":"...", "logistics":"...", "priority":"...", "schedule":"...",
       "confidence":"high|medium|low|needs-review",
       "resolved":true, "flaggedForClaude":false, "claudeNote":""}
    ],
    "balances": [{"account":"...", "balance":0}]
  }
}
```

A Claude Code session reads this (plain REST GET against the open rules,
no SDK needed) when asked to build the next handoff, instead of taking a
raw CSV path. See `_budget_ingest_analysis/AUTOMATION_ROADMAP.md` for the
full roadmap this fits into.

## Known limitations (v1, 2026-11-16)

- **The reference file is uploaded each session, not fetched automatically.**
  `archive_reference_full.csv` isn't committed into this repo (it's
  categorization metadata from the Excel pipeline, kept there, not here) -
  Chris selects it as a second file alongside the bank CSV each time.
  Skipping it is fine; everything just lands in "needs your review" instead
  of being auto-categorized.
- **No cross-pull dedup** - see above.
- **No historical "known transactions" bootstrap.** The three bank pulls
  already ingested before this app existed (11/09, 11/11, 11/16) aren't
  reflected here in any way - this only matters if a future pull overlaps
  with one of those specific dates, which is unlikely but possible.
- **Week boundaries are hardcoded to November 2026** (`WK_BOUNDS` in
  `index.html`), same limitation as the rest of this pipeline. Update when
  this app is used past November.
- **No icons/manifest yet** - "Add to Home Screen" works but without a
  custom icon. Low priority, not blocking.

## Where things live

- This repo: the app's own code and these two docs.
- `_budget_ingest_analysis/AUTOMATION_ROADMAP.md` (separate project, not in
  this repo) - the roadmap this app is Phase 2 of, including the Firebase-
  openness privacy flag that was explicitly accepted before building this.
- `_budget_ingest_analysis/handoff_v26_build_writeplan.js` and its
  predecessors - the Excel reconciliation pipeline this app's output feeds
  into.
- `missionbudget01` - the separate bill-tracker dashboard app (same family-
  web-app pattern, same Firebase project, different path:
  `budget/bills/...`). Not the same app as this one, despite living in the
  same Firebase project.

## Published, live

Public repo: `github.com/servicemac74/budget-ingest-app`. Live at
`https://servicemac74.github.io/budget-ingest-app/`. Published 2026-11-16
via the GitHub CLI (`gh repo create --public --source=. --push`, then
`gh api .../pages` to enable Pages) - `gh` is installed and authenticated
on this machine, so a future app in this family no longer needs the
GitHub Desktop GUI steps `TEMPLATE_PROCESS.md` describes; `gh repo create`
covers repo creation + first push in one step, `gh api` covers Pages.

Verified live in a real browser (not just a local file): page loads with
0 console errors, Firebase connects and the sync badge shows "synced."
The actual upload-a-CSV flow hasn't been verified through browser
automation (file pickers can't be driven that way) - the categorization
logic itself was validated separately, directly against real data, before
this was ever committed.
