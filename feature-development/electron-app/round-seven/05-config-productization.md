# Phase 3 — Productionize `vipr.config.json` Across All Surfaces

> Two-step phase: (1) bounded read-only review of an existing in-flight worktree
> by another agent; (2) incorporation, modification, or replacement decision
> based on that review. Step 1 is **gated** before any code changes.

## Goal

Bring user-facing project configuration to all three Vipr surfaces (CLI,
desktop app, VS Code extension), starting with file/directory exclusion and
extensible to per-plugin / per-analysis enablement, with a single canonical
config schema (`vipr.config.json`) shared across surfaces.

## Why now

Two converging needs:

1. **The current hardcoded exclusion lists are inflexible.** Different projects
   have legitimately different needs — e.g., one team's `migrations/` should
   be analyzed; another's shouldn't. Hardcoding can't serve both.
2. **Per-plugin / per-analysis enablement is a real perf lever.** Each TS file
   currently runs ~9 TS analyses + ~7 React analyses. If a user can disable
   the ones they don't care about, mean per-file time drops proportionally,
   compounding across all files in the repo.

The CLI already has `vipr.config.json`. The work is to make that the source of
truth across surfaces and to expose a settings UI for it in desktop and VS
Code.

## Architectural principles (do not violate)

These were established in earlier discussion. Any incoming design (including
the existing worktree's) is evaluated against them:

1. **Hierarchical merge**: Built-in defaults (the floor) → user-global config
   → project config → (optionally) inline overrides. Each layer extends or
   overrides the previous.
2. **Append vs replace semantics must be explicit.** Decide whether project
   config *adds to* defaults or *replaces* them. Gitignore-style negation
   (`!path/to/include`) is worth supporting.
3. **Schema must be extensible without breaking changes.** Today's schema
   covers file/dir exclusion. Tomorrow's must add per-plugin / per-analysis
   enablement, per-anti-pattern toggles, and per-metric visibility. Closed
   types (e.g., `excludePatterns: string[]`) that can't grow are a red flag.
4. **Single canonical loader / merger** lives in `@vipr/common/config` (which
   already exists per CLAUDE.md). All three surfaces use it. No duplicated
   parsing logic.
5. **Schema validation with Zod**, consistent with the rest of the codebase.
6. **Graceful failure**: invalid config never crashes the analyzer. Warn and
   fall back to defaults.

## Step 1 — Bounded review of the existing worktree

**This step is gated. No code changes happen until the review is complete and
the verdict is in.**

### Scope of the review

- **Read-only.** Zero edits, zero commits, zero "while I'm here" cleanups.
- **Time-boxed.** 30–45 minutes. If incomplete, write up what's known and
  stop.
- **Output**: a verdict written to `.claude/plans/05-config-worktree-review.md`
  (sibling of this doc). One of three categories:
  - **A. Fold in as-is** — design matches our principles, ship it
  - **B. Fold in with N specific tweaks** — design is sound but needs deltas
    (list them concretely with file paths and changes)
  - **C. Throw away** — fundamental design problem (state which principle is
    violated and why incompatible)

### Specific questions to answer

1. **Does the schema leave room for per-plugin and per-analysis enablement?**
   I.e., is the exclusion list the *only* knob, or part of a larger config
   surface that can grow? Closed flat array = red flag.
2. **What are the merge semantics?** Does project config extend or replace
   built-in defaults? Is gitignore-style negation supported?
3. **Is the loader pluggable across surfaces?** Can the desktop and VS Code
   extension both consume it without code duplication? Or is it CLI-coupled?
4. **What's its relationship to the existing `DEFAULT_IGNORE_PATTERNS` /
   `EXCLUDED_FILE_PATTERNS`?** Replace? Layer on top? Run in parallel?
5. **Is there a settings UI surface in their design?** Or is that Phase 3.5
   work?
6. **Is Zod validation present?** Schema documented?
7. **Test coverage?** Loader tested? Merger tested? Edge cases (missing file,
   invalid file, partial overrides)?

### Files to read (in this order)

1. The new schema file (likely `packages/common/src/config/schema.ts` or similar)
2. The loader/merger file
3. The CLI integration (where the loader is called from)
4. Any new tests
5. Spot-check the `vipr.config.json` example file if one exists

The user has the worktree path; ask them for it.

### What "done" looks like for Step 1

A new file at `.claude/plans/05-config-worktree-review.md` with:

- Verdict (A / B / C)
- For each principle above: ✅ / ⚠️ / ❌ with one-sentence justification
- For verdict B: explicit list of tweaks needed (file paths, what to change, why)
- For verdict C: reason and recommendation (start over with what specific design)
- Inventory of what's already done (so we don't re-build it)

## Step 2 — Incorporation (verdict-dependent)

Branches based on Step 1 verdict.

### If verdict A (fold in as-is)

1. Cherry-pick or rebase their commits onto our branch
2. Resolve any conflicts with our Phase 0/0.5 instrumentation (unlikely — different files)
3. **Decide what happens to Phase 1b/c** (see decision tree below)
4. Run typecheck + tests across affected packages
5. Re-baseline per [`04-rebaseline-protocol.md`](./04-rebaseline-protocol.md)
6. Commit

### If verdict B (tweaks needed)

1. Cherry-pick their commits
2. Apply the listed tweaks as additional commits (one per tweak, easy review)
3. Same finalization as verdict A

### If verdict C (throw away)

1. Document the verdict
2. Move Phase 1b/c forward as planned (hardcoded changes to `DEFAULT_IGNORE_PATTERNS`)
3. Schedule Phase 3 from scratch as future work, with the design constraints we derived
4. Inform the other agent's owner that their work needs to be redone

## Decision tree: Phase 1b/c vs Phase 3

The relationship between this phase and [`02-exclusion-list-expansion.md`](./02-exclusion-list-expansion.md):

```
Has Phase 3 worktree review verdict come in?
├── Verdict A or B (fold in their work):
│   └── Did their work include the .storybook / .vercel / etc. defaults we want in Phase 1b?
│       ├── Yes: Phase 1b/c is OBSOLETE. Don't ship it. Mark plan as superseded.
│       └── No:  Add those defaults via the new config schema's defaults table,
│                NOT via hardcoded changes to DEFAULT_IGNORE_PATTERNS.
├── Verdict C (throw away): Ship Phase 1b/c hardcoded as planned. Phase 3 restarts.
└── Review not done yet:
    └── Ship Phase 1a (bug fix) but HOLD Phase 1b/c. Reassess after Step 1.
```

## Future Phase 3 sub-items (post-incorporation)

These are deliberately deferred until after the basic config surface lands.
Listed here so future planning has the full picture:

- **Phase 3b — Per-plugin enablement**: e.g., disable React analyzer on a
  non-React project. Schema: `{ plugins: { react: { enabled: boolean } } }`.
  Perf upside: skips an entire plugin per file.
- **Phase 3c — Per-analysis enablement within a plugin**: e.g., run
  `core-cyclomatic` but skip `core-halstead`. Schema: `{ plugins: { core: {
  analyses: { 'core-halstead': false } } } }`. Perf upside: skips a single
  AST traversal per file. Real lever once we know which analyses dominate
  (Phase 4 research will tell us).
- **Phase 3d — Per-anti-pattern enablement**: e.g., opt in to barrel
  detection, opt out of prop-drilling. Lower-level than per-analysis. May
  not be needed if 3c is granular enough.
- **Phase 3e — Settings UI in desktop + VS Code** that reads/writes the same
  config the CLI consumes.
- **Phase 3f — Per-metric visibility**: which metrics show in the UI.
  Distinct from analysis enablement (you might compute Halstead but hide it).

These should each get their own plan doc once Step 2 is done and the foundation
is laid.

## Acceptance criteria (Phase 3 main work)

- A `vipr.config.json` at a workspace root is read by the desktop app on open
- The same `vipr.config.json` is read by the CLI (existing behavior preserved)
- The same `vipr.config.json` is read by the VS Code extension
- File/directory exclusion in `vipr.config.json` works alongside built-in defaults (extends, not replaces, per principle #2)
- Schema is extensible (no breaking-change risk for adding per-plugin enablement later)
- Zod validation rejects invalid config with helpful errors
- Missing or invalid config falls back to defaults with a warning
- Tests cover: load, merge, validation, missing file, invalid file
- All existing exclusion behavior is preserved (no regressions)
- Re-baseline shows no perf regression

## Test plan

To be filled in once worktree is reviewed and incorporation path is chosen.

## Risk + rollback

- **Highest risk phase.** Changes touch a shared config surface used by 3
  surfaces.
- **Critical mitigation**: do not start Step 2 without a clear verdict from
  Step 1. The verdict is the gate.
- **Rollback**: if incorporation reveals problems post-merge, revert the
  cherry-picked commits. Phase 1a (bug fix) and Phase 2 (concurrency=1) are
  independent and stay in place.

## Out of scope

- **Per-plugin / per-analysis enablement implementation** (Phases 3b/3c —
  separate doc once foundation lands)
- **Settings UI for desktop / VS Code** (Phase 3e — separate doc)
- **Per-metric visibility** (Phase 3f — separate doc)
- **Migration tooling** for existing CLI config users — only needed if the
  new schema breaks the old. Determine during Step 1.

## Estimated effort

- Step 1: 30–45 minutes (read-only review)
- Step 2: depends on verdict
  - Verdict A: 1–2 hours (cherry-pick + verify)
  - Verdict B: 1–2 days (incorporate + tweaks + test)
  - Verdict C: 0 hours for this phase, but Phase 3 restarts as new work (1–2 weeks)

## Sequencing

This phase can run in parallel with Phase 4 research, since they touch
different layers (config surface vs. analysis engine internals). But Phase 3
should not block Phase 4 work.
