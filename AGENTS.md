# AGENTS.md — budget-ingest-app

Model-neutral editing instructions for this repo. Applies to any coding
assistant (Claude, Codex, or otherwise). Read [`README.md`](README.md)
first for architecture/data-model/known-limitations; this file is the
rules for safely changing the code.

## Invariants — do not break these without understanding why they exist

All of this is implemented in `index.html`; search for the function name
to find it.

1. **Every write goes through `saveLocal()` + `FB.scheduleWrite()`.** Don't
   mutate `STATE` and skip either one - that desyncs the UI, local storage,
   and Firebase from each other, same rule every other app in this family
   follows.
2. **Never call `FB`'s internal write function (`_doWrite`) directly from a
   UI handler.** `scheduleWrite()` debounces (300ms) so rapid edits coalesce
   into one write.
3. **Firebase reads must go through `_fbArr()`** before being used as
   arrays - Firebase's Realtime Database silently turns JS arrays into
   numeric-keyed objects on write.
4. **The `savedAt` guard (`_lastSavedAt`) stops a device from echoing its
   own write back to itself.** Don't remove it when touching the sync
   listener.
5. **Firebase is additive, never required.** The app must still render and
   be usable (from `localStorage` alone) if the Firebase SDK fails to load
   or the network is down.
6. **Any change to the shape of `STATE.pull` or its `transactions`/
   `balances` entries** requires, together, in the same change:
   - Bump `SCHEMA_VERSION` by 1.
   - Update `_normalize()` in the FB module so older saved data gets the
     new field defaulted sanely instead of dropped or crashing.
   - Update the version-history note near the top of `index.html`'s script
     block.
   This is what makes `showStaleBlock()` work - a client behind the saved
   schema version gets blocked from editing instead of silently
   overwriting newer data. Skipping the bump is the one mistake that
   causes real, hard-to-notice data loss.
7. **The update-check banner (the `UPDATE CHECK` block near the end of the
   script) must stay present.** Ported from `ignite-trip-app`/
   `missionbudget01` - keeps a phone that's "Added to Home Screen" from
   missing a new push for days. If you ever remove or change it, do the
   same in those two repos (they're meant to stay in sync on this one
   feature) and update `missionbudget01/build/build_budget_tracker.js`'s
   build-time check if it also checks this repo somehow (it doesn't today
   - each repo's check only covers its own template).
8. **Categorization logic (`categorize()`, `normalizeDesc()`) must stay a
   faithful port of the Excel pipeline's own logic**
   (`normalize_lib.js` + `dedupAndCategorize()` in
   `_budget_ingest_analysis/handoff_v26_build_writeplan.js` and its
   predecessors - a separate repo, not this one). If that pipeline's
   categorization rules change, port the change here too, or this app and
   the Excel pipeline will categorize the same transaction differently.
9. **No cross-pull dedup logic belongs in this app** - see README's "What
   this app does and doesn't do." Adding it here would duplicate (and risk
   diverging from) the dedup already proven correct in the Excel pipeline.
   If duplicate-across-pulls ever becomes a real problem, that's a
   `_budget_ingest_analysis/AUTOMATION_ROADMAP.md` design discussion
   (a "known transactions" ledger, most likely), not a quick patch here.

## Workflow

- **Discuss → edit → test → push.** Same pattern as `ignite-trip-app`.
- No `gh` CLI on the machine this was built on - publishing a new repo or
  enabling Pages needs a human at the GitHub Desktop / github.com GUI (see
  README's "One-time setup still needed"). Routine pushes to an already-
  published repo work fine from a terminal (Git Credential Manager, same
  as every other app here).
- **Never commit `archive_reference_full.csv` or any real transaction/
  balance data into this repo.** This repo is (or will be) public. The app
  is designed around Chris uploading the reference file fresh each
  session specifically so it never needs to live here - don't "simplify"
  that by bundling a copy in, even temporarily.

## What's out of scope here

- The Excel reconciliation pipeline this app feeds - that's
  `_budget_ingest_analysis`, a separate project/repo structure entirely.
- The bill-tracker dashboard (`missionbudget01`) - a different app, same
  Firebase project, different path. Don't assume code here applies there
  or vice versa without checking.
- Building the actual "known transactions" dedup ledger, or wiring Claude's
  build scripts to read from this app's Firebase output - both logged as
  next steps in `AUTOMATION_ROADMAP.md`, not done yet.
