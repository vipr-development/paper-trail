---
sidebar_label: 'Round Seven — Perf + Storage + Docs Trifecta'
---

# Round Seven — Desktop Analysis Performance, Storage Normalization, and Analyzer-Owned Documentation

> Historical paper trail for the rolling desktop-analysis performance
> work and the two strategic initiatives that followed it (Option C
> process parallelism, and the consolidated Option E + Option F
> analyzer-owned documentation effort).
>
> Captured from the live planning folder
> (`~/Codespace/vipr/.claude/plans/`) on 2026-05-13 as the work
> finished shipping. The remaining open items moved into a fresh
> agent-pickable structure in the same folder.

## What this round delivered

Three overlapping streams, run roughly chronologically over April-May 2026:

1. **Backfill + foreground performance** — the rolling sequence of small
   wins (items 1-2 plus the per-phase docs 01-06). Closed by
   measurement: the incremental write path was already cheap; remaining
   wall-clock reduction required structural change.
2. **Option C — Process-level parallelism.** Two stages shipped at the
   CPU-parallelism ceiling. Backfill went 97:22 → 50:08 (48.5%
   reduction at N=4). Foreground went 4:42 → 2:35 (45%, peak in-flight
   4). Four QA remediations (`814f7d1b`, `78edf73e`, `b1ff3169`,
   `4b18e212`) plus orphan-window defense-in-depth (`e3500fd1`).
3. **Analyzer-owned documentation** (originally Option E +
   Option F, collapsed into a single initiative). Architectural
   reshape that retired the legacy hand-maintained per-analysis
   markdown in favor of analyzer-co-located `.docs.ts` files compiled
   into a build-time bundle consumed by every Vipr surface. Persistence
   normalised to a slim insight row that joins against the bundle at
   read time. Shipped across 11 PRs (`#72` → `#88`); workspace DB went
   15.2 MB → 3.0 MB (80% reduction) with 100% migrated rows on the
   real workspace.

## How to read this round

Start with [`00-overview.md`](./00-overview.md) — the rolling-sequence
README captured at end of round. It's the authoritative summary of
which items shipped, which commits, and what the decision log looked
like.

Then follow whichever thread interests you:

- **Per-phase shipped work:** `01-bug-fix-historical-filter.md` through
  `06-per-file-mean-reduction.md`. These were the original session-sized
  items that landed before the strategic initiatives took over.
- **Measurement + protocol:** [`baselines.md`](./baselines.md) and
  [`04-rebaseline-protocol.md`](./04-rebaseline-protocol.md).
- **Item docs:** the `item-NN-*.md` files describe individual chunks of
  work (each scoped for one commit). Items 2, 5, and 7 are here.
  Items 3, 4, 6 remained open at end of round and moved with the
  live plan folder; check the current vipr repo for their status.
- **Option C strategic initiative:** [
  `initiative-option-c-process-parallelism.md`](./initiative-option-c-process-parallelism.md),
  the stage 1 runbook ([`option-c-stage-1-runbook.md`](./option-c-stage-1-runbook.md))
  and differential-test plan ([`option-c-stage-1-differential-test.md`](./option-c-stage-1-differential-test.md)),
  and the stage 2 QA deferred items ([`followup-option-c-stage2-deferred.md`](./followup-option-c-stage2-deferred.md)).
- **Analyzer-owned docs initiative (was Option E + Option F):**
  the two original initiative docs ([
  `initiative-option-e-storage-normalization.md`](./initiative-option-e-storage-normalization.md)
  and [`initiative-option-f-documentation-as-client.md`](./initiative-option-f-documentation-as-client.md))
  are both marked SUPERSEDED at their top — the consolidated execution
  lives in the global plan-dump at
  `~/.claude/plans/greedy-juggling-hamster.md` and the per-phase docs
  alongside it. The Phase 0 consumer audit
  ([`followup-option-e-phase0-consumer-audit.md`](./followup-option-e-phase0-consumer-audit.md))
  fed the consolidated implementation directly.

## Items that moved forward (NOT in this archive)

The following items were still open or queued when this round closed
and remain in the live plan folder at `~/Codespace/vipr/.claude/plans/`
under a fresh agent-pickable structure (see that folder's `README.md`):

- Item 3 — Telemetry/tracing: move `PerformanceProfile` into the
  `@vipr/tracing` system
- Item 4 — `vipr.config.json` audit + user docs
- Item 6 — Analysis-level filtering (per-analysis preconditions)
- Followups: foreground ETA stability gate, backfill ETA + bottom-bar
  layout, FileDetail tail latency, PerformanceProfile ingestion
  refactor
- Three Option C Stage 2 deferred items (D2 pool restart coupling,
  D3 unbounded queue, D4 pool-size log assertion test)

Plus the still-unscoped strategic initiative **Option A — file-lifecycle
reuse**, which the FileDetail tail-latency investigation will inform.
