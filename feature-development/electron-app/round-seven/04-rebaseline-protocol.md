# Re-Baseline Protocol

> Reusable procedure for capturing before/after measurements. Use this between
> shipping phases to validate impact (or lack thereof). Save outputs to
> `baselines/<date>-<phase>.md` for diffing.

## Goal

Produce a `Performance summary` log line (foreground + backfill) under
**consistent conditions**, so each phase's impact is measurable rather than
inferred.

## Why this matters

Per-file mean is the dominant factor in foreground wall clock. If we want to
know whether a fix moved the needle, we need disciplined before/after
comparisons. Inconsistent conditions (cold vs warm cache, foreground vs
backfill overlap, different repos) destroy the signal.

## Standard test conditions

For all baselines on this branch, use:

- **Repository**: `/Users/jamesleebaker/Codespace/vipr/clients/desktop` (the desktop client itself, ~1013 source files)
- **Backfill window**: 90 days (procudes ~394 commits as of 2026-05)
- **State**: warm OS file cache, fresh Vipr DB
- **Order**: foreground first, then backfill (sequentially, not overlapping)
- **No other heavy work on the machine** during the run (no other dev servers, no large downloads, no AI model inference)

These are the same conditions used for [`baselines.md`](./baselines.md). Don't
deviate.

## Procedure

### Step 1 — Quit the desktop app

```bash
# macOS — Cmd+Q in the app, OR:
pkill -x "Vipr Desktop" 2>/dev/null
```

Wait 5 seconds. Confirm process is gone:

```bash
ps aux | grep -i "vipr desktop" | grep -v grep
```

This must be empty before proceeding. The in-process caches (ts-morph
SourceFiles, content-SHA dedup map) survive across analyses if the process
stays up — quitting kills them.

### Step 2 — Delete the analysis DB

```bash
rm "$HOME/Library/Application Support/Vipr Desktop/vipr-default.db"
rm "$HOME/Library/Application Support/Vipr Desktop/vipr-workspaces.json" 2>/dev/null
```

Don't delete `settings.db` (preserves user prefs). Don't delete the whole app
support dir (loses AI models).

### Step 3 — Confirm what you're testing

```bash
git status --short
git log --oneline -3
```

Capture this output in your baseline writeup. The baseline is meaningless
without knowing the commit it was taken on.

### Step 4 — Start the desktop app

```bash
cd /Users/jamesleebaker/Codespace/vipr
pnpm --filter @vipr/desktop dev
```

Wait for the Electron window to appear.

### Step 5 — Open the test repository

In the Vipr Desktop UI:

1. File → Open Repository (or wherever the open-repo affordance is in the current build)
2. Select `/Users/jamesleebaker/Codespace/vipr/clients/desktop`
3. The foreground analysis starts automatically

### Step 6 — Wait for foreground completion

Watch the dev server console for:

```
[performance-profile] Performance summary [foreground/<run_id>] {
  ...
}
```

This typically arrives 4–6 minutes after the repo opens (varies by hardware).
Copy the entire JSON object and save it.

### Step 7 — Wait for backfill completion

Backfill runs after foreground. It typically takes 90+ minutes for the 90-day
window on this repo. Watch for:

```
[performance-profile] Performance summary [backfill/backfill:<job_id>] {
  ...
}
```

Copy the JSON object and save it.

### Step 8 — Save and label

Create a file at:

```
.claude/plans/baselines/<YYYY-MM-DD>-<phase-shipped>.md
```

E.g. `.claude/plans/baselines/2026-05-15-phase-1a.md`. Template:

````md
# Baseline: <date> — after <phase>

## Conditions
- Branch: <branch>
- HEAD: <sha>
- Phases active: <list> (e.g. "Phase 0 + 0.5 + 1a")
- Notes: <anything unusual about the run>

## Foreground

```jsonc
<paste the foreground summary>
```

## Backfill

```jsonc
<paste the backfill summary>
```

## Headline diffs from previous baseline

| Metric | Previous | This run | Δ |
|---|---|---|---|
| Foreground wall clock | ... | ... | ... |
| Foreground per-file mean | ... | ... | ... |
| Backfill wall clock | ... | ... | ... |
| Backfill per-file analyzeMs mean | ... | ... | ... |
| Slowest file (foreground) | ... | ... | ... |
| Slowest commit (backfill) | ... | ... | ... |

## Interpretation

<2–4 sentences on whether the phase delivered what was predicted, and any surprises>
````

## What metrics to focus on per phase

| Phase shipped | Watch this metric most |
|---|---|
| 1a (bug fix) | Backfill `slowestBackfillFiles` should no longer contain `mock/` paths. Total backfill files (cache misses) should drop. |
| 1b/c (exclusion expansion) | Foreground `slowestFiles` should no longer contain `.storybook/*`. Total foreground files may drop slightly. |
| 2 (concurrency=1) | Backfill per-file `analyzeMs` mean should drop from ~2038ms to ~500ms. Wall clock should be roughly unchanged (±10%). |
| 4 (per-file mean reduction) | Foreground per-file `totalMs` mean should drop. Wall clock should drop roughly proportionally. |

## Common gotchas

- **Don't compare cold-start to warm-start runs.** The OS file cache changes per-file timing by 10–20%. Always run after the first warm-up.
- **Don't run other CPU-heavy work in parallel.** A `pnpm install` or background build during analysis will skew everything.
- **Don't trust a single measurement for ±5% claims.** Variance is real. For close calls, run twice and take the median.
- **Don't conflate `analyzeMs` (worker-reported) with `totalMs` (coordinator-measured).** They mean different things — `analyzeMs` is the time inside the worker for that one file, `totalMs` includes IPC + DB + event emit.
- **Slowest files should be roughly stable** across runs of the same repo. If they shift dramatically, something else changed (different repo, different exclusion rules, different concurrency).

## When to skip re-baselining

You can skip the full re-baseline (and just typecheck/test) when:

- The change is purely additive instrumentation
- The change touches only test files
- The change is a documentation-only edit

Always re-baseline when:

- Any production code in `clients/desktop/src/main/analysis/` changes
- Any constant in `packages/common/src/client-constants.ts` changes
- Any worker or IPC schema changes
- A phase plan claims a measurable improvement (you must verify the claim)

## Storage convention

```
.claude/plans/
├── 00-README.md
├── baselines.md                  ← original baseline (referenced from README)
├── baselines/
│   ├── 2026-05-15-phase-1a.md
│   ├── 2026-05-22-phase-2.md
│   └── ...
└── <other phase docs>
```

This keeps the original baseline canonical (in `baselines.md`) and treats each
post-phase baseline as a new dated artifact. Diff with `diff` or visually.
