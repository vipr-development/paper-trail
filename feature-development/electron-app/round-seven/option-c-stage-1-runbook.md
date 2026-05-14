# Option C — Stage 1 Validation Runbook (one-backfill mode)

> Run one backfill, walk away, run one script, report back. The
> validation gate uses **internal-consistency checks** + **wall-clock
> comparison to the published item-2 baseline** instead of a separate
> single-process baseline run.
>
> The rigorous double-build differential test is preserved as an
> appendix at the bottom for paranoid mode.

---

## Time investment

| What you do | Wall-clock |
|---|---|
| Setup + launch | ~3 min attended |
| Backfill (walk away) | ~70-95 min unattended |
| Quit app + run postcheck script | ~1 min attended |
| **Total attended time** | **~4 minutes** |

---

## What the validation actually proves

| Risk | How we catch it without a separate baseline |
|---|---|
| Pool dropped work (missing files / commits) | Postcheck detects zero-file snapshots, missing commit_metrics, and failed_commits in the indexing job |
| Pool corrupted analysis output | Postcheck detects out-of-range scores, orphan content_sha references, high null-score ratio |
| Pool routed wrong file to wrong process | Would surface as score / file_count anomalies in the postcheck distribution stats |
| Stage 1 didn't actually parallelize | Wall-clock comparison to the published 97-min baseline catches it |
| Subtle race in main-process DB writes | Postcheck detects orphan rows and missing FK targets |

What the postcheck does NOT catch (acceptable risk for Stage 1):

- Subtle non-determinism that produces different but equally plausible
  scores. The unit tests + content-addressable analysis design make
  this very unlikely. If you want byte-equal proof, run the appendix.

---

## The one-backfill procedure

### Step 1 — Confirm Stage 1 code + build (one-time, ~2 min)

```bash
cd /Users/jamesleebaker/Codespace/vipr

# Confirm you're on the Stage 1 commit
git log --oneline -1
# Expected: 1e38b1fe feat(desktop): introduce UtilityProcessPool for backfill (Option C Stage 1)

# Confirm pool size = 2
grep STAGE_1_BACKFILL_POOL_SIZE clients/desktop/src/main/index.ts
# Expected: const STAGE_1_BACKFILL_POOL_SIZE = 2;

# Build
pnpm --filter @vipr/desktop build
```

### Step 2 — Wipe DB and launch (~30 sec attended)

```bash
# macOS:
rm -rf "$HOME/Library/Application Support/Vipr Desktop"

pnpm --filter @vipr/desktop dev
```

In the running app:
1. Open the test repo (the same 1000-file `clients/desktop` you used
   for the item-2 baseline)
2. Wait for the initial scan to complete
3. **Walk away.** Backfill auto-starts and runs ~70-95 min.

While it's running, in a separate terminal you can passively monitor
RSS if you want:

```bash
# Optional, only if you care about peak RSS for the report:
while sleep 30; do
  ps -axo pid,rss,command | grep -E 'vipr-(analysis|backfill)-worker' | grep -v grep
done > /tmp/backfill-rss.log
```

(Stop with Ctrl-C when backfill completes.)

### Step 3 — When backfill completes (~1 min attended)

When the dev console shows backfill is done, **quit the app gracefully
(Cmd+Q)** so SQLite checkpoints. Then in any terminal:

```bash
cd /Users/jamesleebaker/Codespace/vipr
node clients/desktop/scripts/stage-1-postcheck.mjs
```

That's the whole step. No flags, no JSON files to copy-paste. The
script:

- Auto-discovers the newest workspace DB in
  `~/Library/Application Support/Vipr Desktop/`
- Runs 10 internal-consistency checks
- Reads the wall-clock automatically from `indexing_jobs`
  (`completed_at - started_at` of the most recent completed job)
- Compares wall-clock to the published 97-min item-2 baseline
- Emits a report ending in `✅ STAGE 1 POST-CHECK: PASS` or
  `❌ STAGE 1 POST-CHECK: FAIL`

### Step 4 — Report back

Paste the postcheck output to me. If you ran the optional RSS monitor,
include the peak observed RSS. That's everything I need to greenlight
Stage 2 or recommend a back-off step.

---

## What the postcheck output looks like

A passing run:

```
workspace DB: /Users/.../Library/Application Support/Vipr Desktop/vipr-xxx.db

─── Internal consistency checks ──────────────────────
  ✓ snapshots present
      snapshots row count = 392
  ✓ snapshot_files distribution
      total=149280, per-snapshot min=8, max=1003, avg=380.8
  ✓ no zero-file non-draft snapshots
      zero-file non-draft snapshots = 0
  ✓ all overall_score in [0, 100]
      out-of-range scores = 0
  ✓ null-score ratio reasonable
      null overall_score = 12/149280 (0.0%)
  ✓ no orphan content_sha references
      orphan content_sha refs = 0
  ✓ commit_metrics covers every non-draft snapshot
      snapshots without commit_metrics = 0
  ✓ most recent indexing job: no failures
      job ... status=completed processed=392/392 failed=0
  ✓ wall-clock recorded
      last backfill wall-clock = 4234567 ms (70.58 min)
  ✓ snapshot_file_deltas populated (incremental path active)
      snapshot_file_deltas row count = 60531

Consistency overall: PASS

─── Wall-clock comparison ────────────────────────────
  baseline (item 2 / bc47853b): 5840000 ms (97.33 min)
  current (this run):           4234567 ms (70.58 min)
  reduction:                    27.49%

  ✓ ≥25% reduction — Stage 1 wall-clock target MET.

Wall-clock overall: PASS

✅ STAGE 1 POST-CHECK: PASS
```

A failing run (any of multiple categories) prints exactly which checks
failed and the diagnostic detail. Common failure shapes:

- **Zero-file snapshots > 0** → pool dropped work. STOP, revert.
- **Out-of-range scores > 0** → analysis pipeline produced corrupt
  output. STOP, revert.
- **Orphan content_sha refs > 0** → race between content fetch and
  analysis. STOP, revert.
- **failed_commits > 0** → some commits errored during pooled
  backfill. Capture the dev log for those commits and report.
- **Wall-clock reduction 10-25%** → STOP and analyze, but Stage 1 may
  still ship if RSS is fine and we can argue Stage 2's N=4 will close
  the gap.
- **Wall-clock reduction < 10%** → pool overhead exceeded its benefit.
  STOP. Likely commit-level serialization is the dominant cost; revisit
  the structural lever ("Option D" pipelining).

---

## Decision matrix

| Postcheck outcome | Decision |
|---|---|
| Consistency PASS + Wall-clock ≥25% reduction | **Ship Stage 1.** Proceed to Stage 2 (expand to N=4 + foreground pool) |
| Consistency PASS + Wall-clock 10-25% reduction | **Pause.** Try N=4 in Stage 2 first — at N=4 we may hit the 25% gate |
| Consistency PASS + Wall-clock < 10% reduction | **Stop.** Pool overhead exceeded benefit; commit-level serial cost is the floor. Revisit Option D / Option A |
| Consistency FAIL (any check) | **Revert immediately** (`git revert 1e38b1fe`); paste failing checks |
| Consistency PASS but failed_commits > 0 | **Pause.** Capture the dev log, identify which commits failed and why, decide based on root cause |

---

## Helper script reference

```bash
# DEFAULT (recommended) — consistency + wall-clock from indexing_jobs
node clients/desktop/scripts/stage-1-postcheck.mjs

# Explicit DB path
node clients/desktop/scripts/stage-1-postcheck.mjs --db <path>

# Override wall-clock with a saved Performance summary JSON
# (only useful if you captured one for a different timestamp basis)
node clients/desktop/scripts/stage-1-postcheck.mjs --perf-summary path/to/summary.json

# Capture an aggregate signature (paranoid double-build mode)
node clients/desktop/scripts/stage-1-postcheck.mjs --capture-signature > sig-N1.json

# Compare current DB against a saved signature (paranoid double-build mode)
node clients/desktop/scripts/stage-1-postcheck.mjs --compare-signature sig-N1.json

# Help
node clients/desktop/scripts/stage-1-postcheck.mjs --help
```

Auto-discovery picks the newest workspace DB in `~/Library/Application
Support/Vipr Desktop/`. Override with `--db <path>` if needed.

---

## Appendix — Rigorous double-build differential test (optional, paranoid mode)

> Use this only if Stage 1 ships and you want a stronger integrity gate
> for Stage 2 or if the consistency checks above flag something
> ambiguous. Total time: ~3 hours of mostly-unattended backfill +
> ~5 min attended.

### A1 — Baseline (single-process)

```bash
# Edit clients/desktop/src/main/index.ts:
#   const STAGE_1_BACKFILL_POOL_SIZE = 1;  // temporarily
pnpm --filter @vipr/desktop build
rm -rf "$HOME/Library/Application Support/Vipr Desktop"
pnpm --filter @vipr/desktop dev
# Open repo, walk away ~97 min, quit
node clients/desktop/scripts/stage-1-postcheck.mjs --capture-signature > sig-N1.json
```

### A2 — Pooled

```bash
# Restore: const STAGE_1_BACKFILL_POOL_SIZE = 2;
pnpm --filter @vipr/desktop build
rm -rf "$HOME/Library/Application Support/Vipr Desktop"
pnpm --filter @vipr/desktop dev
# Open repo, walk away ~70 min, quit
node clients/desktop/scripts/stage-1-postcheck.mjs --compare-signature sig-N1.json
```

The comparison reports byte-equal totals (snapshot count, file count,
sum of all scores) across the two runs. Any diff is a stop signal.

This gives you proof of byte-equal aggregate output. It does NOT prove
per-row byte equality — for that, dump the tables manually
(`option-c-stage-1-differential-test.md` for the SQL).
