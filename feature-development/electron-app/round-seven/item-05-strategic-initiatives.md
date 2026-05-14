# Item 5 — Replace Per-Phase Docs with Strategic Initiatives

> The original per-phase docs (`01-bug-fix...`, `02-exclusion...`,
> `03-backfill-concurrency...`, etc.) describe SHIPPED work and have served
> their purpose. This item retires them in favour of two larger
> independent initiatives: **Option A** (file-lifecycle reuse) and
> **Option C** (process-level parallelism). Each becomes its own
> independent multi-session project rather than a "next phase" of the
> rolling sequence.

## Goal

1. Archive or remove the original per-phase plan docs that have shipped
   (preserving the historical context as a `done/` subfolder so the work
   trail is intact)
2. Create two new initiative-level plan docs — one each for **Option A**
   and **Option C** — that lay out the architecture, risks, and
   implementation milestones in enough detail that another agent can pick
   them up
3. Update `00-README.md` to reference the new initiative docs as separate
   tracks (not "the next phase")
4. **DO NOT IMPLEMENT either initiative in this item** — that's their own
   future sessions

## Why now

Two reasons to separate these initiatives from the rolling sequence:

1. **Both are weeks of work, not session-sized.** Treating them as the
   "next phase" implies they fit the same cadence as items 2-4 (each
   1-3 days). They don't.

2. **Both are architectural inflection points.** Each requires careful
   design, dedicated focus, and the ability to bail out early if the
   approach proves wrong. They deserve independent doc trails — not
   compressed into one phase doc.

The rolling sequence (items 1-4, 6) covers everything that fits the
"finish in a few days" cadence. Option A and Option C are the strategic
work that needs its own home.

## Prerequisites

- Items 1-4 shipped (or at least item 1)
- Familiarity with the existing per-phase docs that will be archived:
  - `01-bug-fix-historical-filter.md`
  - `02-exclusion-list-expansion.md`
  - `03-backfill-concurrency.md`
  - `05-config-productization.md` (paired with `05-config-worktree-review.md`)
  - `06-per-file-mean-reduction.md`

## Implementation outline

### Step 1 — Archive shipped docs

Move shipped per-phase docs into `.claude/plans/done/`:

```bash
mkdir -p .claude/plans/done/
git mv .claude/plans/01-bug-fix-historical-filter.md .claude/plans/done/
git mv .claude/plans/02-exclusion-list-expansion.md .claude/plans/done/
git mv .claude/plans/03-backfill-concurrency.md .claude/plans/done/
git mv .claude/plans/05-config-productization.md .claude/plans/done/
git mv .claude/plans/05-config-worktree-review.md .claude/plans/done/
git mv .claude/plans/06-per-file-mean-reduction.md .claude/plans/done/
```

(These are gitignored — the move is a filesystem move, not a git move.
Actual git commands depend on whether `.claude/` is tracked in this branch.)

KEEP at root level:
- `00-README.md` (rolling sequence index)
- `baselines.md` (raw baseline data)
- `04-rebaseline-protocol.md` (still actively used)
- All `item-NN-*.md` docs (rolling sequence)

### Step 2 — Create `initiative-option-a-file-lifecycle.md`

Full plan doc for Option A. Outline:

**Goal**: Stop disposing-and-recreating ts-morph SourceFiles per-file analysis,
so the TypeScript type checker can amortize import resolution across the
entire batch.

**Why**: Per-file mean cost is approximately conserved across analyses
(see decision log in 00-README.md). The dominant CPU floor is the type
checker warming up for each file's import graph. Currently
`engine.analyzeContent(filePath, content)` calls
`project.createSourceFile(...)` then `project.removeSourceFile(...)` per
file, throwing away the type checker's resolved state. If we kept the
files in the project, subsequent analyses on related files would skip
re-resolution entirely.

**Risks**:
- Memory growth (every file's AST stays resident)
- Stale type info if files change between analyses
- Cross-file pollution (one file's local types appearing in another's
  analysis result)

**Milestones**:
1. Spike: keep all files in project for one full analysis run, measure
   memory + correctness
2. Bounded version: LRU eviction of project files
3. Full: integrate with cache invalidation, content hash tracking
4. Re-baseline at each milestone

**Estimated impact**: 30-50% wall-clock reduction on foreground (and
backfill) — the BIG lever.

**Estimated effort**: 1-3 weeks of focused work.

### Step 3 — Create `initiative-option-c-process-parallelism.md`

Full plan doc for Option C. Outline:

**Goal**: Spawn N foreground utility processes (currently 1) so the desktop
actually uses multiple CPU cores. Each process handles 1 in-flight file.
With 4 processes on a 4-core machine, near-4× wall-clock improvement.

**Why**: Single V8 isolate cannot parallelize CPU-bound JS work. Today's
foreground analysis runs on one core regardless of how many cores the
machine has. Spawning multiple utility processes is the standard way to
break out of this constraint without rewriting the engine.

**Pre-condition**: Item 7 (worker-thread pool experiment) has been
shipped and either (a) validated worker_threads as the cap of what we
can do without process-level parallelism, OR (b) failed Stage 0/1 due
to architectural limits that worker_threads can't overcome. Do NOT
start Option C before item 7 ships its assessment — Option C is the
larger, riskier change that should only be pursued when the cheaper
worker_thread path is proven insufficient.

**Risks**:
- DB contention (multiple processes writing to the same SQLite file)
- IPC complexity (routing requests to the right process, handling
  restarts, graceful shutdown across all of them)
- Per-process startup cost (ts-morph initialization, plugin loading)
- Memory: N processes × ts-morph state can be expensive

**Milestones**:
1. Architecture spike: load-balance design (round-robin? hash by file
   path? least-busy?)
2. Two-process implementation, validate IPC routing + DB safety
3. Configurable process count (default = `cpus().length - 1`)
4. Graceful crash recovery (restart one process without disrupting
   others)
5. Re-baseline at each milestone

**Estimated impact**: ~75% wall-clock reduction on foreground if we hit
4 processes. Modest backfill impact (backfill already runs in its own
process; would only help if we also parallelize commit processing).

**Estimated effort**: 1-2 weeks for a 2-process MVP, 3-5 weeks for
configurable + crash-recovery production version.

### Step 4 — Update 00-README.md

Add a "Strategic Initiatives" section that points to the two new
initiative docs as separate tracks. Make it clear they're NOT items 7
or 8 in the rolling sequence — they're parallel tracks that the
rolling sequence may or may not interleave with.

Sketch:

```markdown
## Strategic initiatives (parallel tracks, multi-session)

These are not part of the rolling sequence above. They're independent
multi-week efforts, each with their own plan doc:

- **Option A — File-Lifecycle Reuse** (see `initiative-option-a-file-lifecycle.md`)
- **Option C — Process-Level Parallelism** (see `initiative-option-c-process-parallelism.md`)
```

## Acceptance criteria

- Original per-phase docs moved into `done/` subfolder (history preserved)
- `initiative-option-a-file-lifecycle.md` exists with the full plan
  outlined above
- `initiative-option-c-process-parallelism.md` exists with the full plan
  outlined above
- `00-README.md` updated with the new structure
- No code changes — this is purely a docs/planning item
- A fresh agent reading `00-README.md` can find the rolling sequence vs
  the strategic initiatives without confusion

## Test plan

No code tests for this item. Verification is qualitative:

- Read `00-README.md` cold and confirm it's clear what's what
- Read each initiative doc and confirm it answers: goal, why now, risks,
  milestones, estimated effort

## Risk + rollback

- **Risk**: minimal — docs only.
- **Rollback**: revert the commits.

## Out of scope

- **Implementing Option A or Option C** — those are their own future
  initiatives. This item only documents them.
- **Removing the original per-phase docs entirely.** Move to `done/`
  but keep them. Future agents may want the historical context.
- **Adding more strategic initiatives.** Two is plenty. If a third one
  emerges, file it in a future item.

## Estimated effort

Half a day for the writing + restructuring. Single commit.
