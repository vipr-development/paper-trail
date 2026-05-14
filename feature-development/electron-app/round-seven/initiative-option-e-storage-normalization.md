# Option E — Storage Normalization (Static Template Hoisting)

> **SUPERSEDED 2026-05-12** — merged into the consolidated initiative at
> [`greedy-juggling-hamster.md`](./greedy-juggling-hamster.md). That plan
> collapses Option E (storage normalization) and Option F (documentation
> as a client) into one architecture: analyzer-owned documentation
> compiled into a build-time bundle, consumed by every Vipr surface,
> with slim insight persistence joining against the bundle at read time.
>
> Implementation: Phases 1-3 + 4 + 5 shipped on PR
> [vipr#72](https://github.com/glorioustephan/eephus-vipr/pull/72) on the
> `claude/analyzer-owned-docs` branch.
>
> The historical content of this file is preserved below for reference
> only.
>
> ---
>
> Strategic initiative to remove redundant static/template data from
> analysis storage. Targets the `snapshot_files.plugin_results` and
> `analyses.{result,insights,metrics}` JSON columns, which together hold
> ~130 MB on a single 1000-file project after one foreground analysis.
> Estimated 70-85% of that content is constant across instances and can
> be hoisted into client-bundled definition tables.

## Status

📋 Queued. Independent of Option C (process parallelism). Can begin in
any session; user has explicitly requested it as a follow-on after
Stage 2.

## Audit findings (2026-05-10) — concrete numbers

Measured against the user's workspace DB
(`vipr-a09b89f5dfa9.db`) after a fresh foreground analysis of
`@vipr/desktop` (1010 files, no backfill yet — pure foreground):

### Database size

| Total DB | 130 MB |
|---|---|

### Largest tables (top 5)

| Table | Size | Notes |
|---|---|---|
| `analyses` | **84.8 MB** | One row per (file_id, plugin_id); replaced on each analysis run |
| `snapshot_files` | **42.5 MB** | One row per (snapshot_id, file_path); preserves analysis at snapshot time |
| `deep_analysis_findings` | 0.66 MB | (out of scope for now) |
| `todo_items` | 0.30 MB | (out of scope) |
| `files` | 0.21 MB | (out of scope) |

**Together `analyses` + `snapshot_files` = 97.7% of the DB.** Both
tables hold the same fundamental analysis output, just indexed
differently (current state vs snapshot in time). They're the entire
target.

### Bloat breakdown in `analyses`

| Column | Total | Avg per row | Max | Rows |
|---|---|---|---|---|
| `result` (JSON) | 41.55 MB | 12,489 B | 288,396 B | 3,488 |
| `insights` (JSON) | 30.63 MB | 9,207 B | 409,708 B | 3,488 |
| `metrics` (JSON) | 10.48 MB | ~3,000 B | (varies) | 3,488 |

Per-plugin breakdown of `analyses.result`:

| Plugin | Rows | Total | Avg row | Max row |
|---|---|---|---|---|
| `core` | 1003 | 17.80 MB | 18.6 KB | 130 KB |
| `react` | 474 | 17.17 MB | 38.0 KB | 282 KB |
| `typescript` | 1001 | 5.92 MB | 6.2 KB | 22 KB |
| `linting` | 1010 | 0.66 MB | 0.7 KB | 400 KB |

### Bloat breakdown in `snapshot_files`

| Column | Total | Avg per row | Max | Rows |
|---|---|---|---|---|
| `plugin_results` (JSON) | 41.58 MB | 43,170 B | 545,921 B | 1,010 |

`snapshot_files.plugin_results` is the **same shape** as what would be
reconstructed from joining all `analyses` rows for a file at snapshot
time. There is structural duplication between the two tables.

### A real sample of the bloat (single React insight, 2,600 bytes)

```json
{
  "id": "react-structural-0",                                   // STATIC (type ID)
  "severity": "warning",                                        // STATIC (per insight type)
  "category": "structural",                                     // STATIC (per insight type)
  "message": "AIPromptModal has high structural complexity (67)", // DYNAMIC (interpolated)
  "location": { "line": 58, "column": 0, "endLine": 410, ... }, // DYNAMIC
  "suggestion": "Break down complex conditional rendering into smaller components", // STATIC (template)
  "source": "react-structural",                                 // STATIC (already on parent)
  "autoFixable": null,                                          // STATIC (per type)
  "autoFix": null,                                              // STATIC (per type)
  "prompt": {
    "template": "The React component {{componentName}} in {{file}} has high structural complexity (score: {{complexity}}), with {{branches}} branches, {{jsxConditionals}} JSX conditionals, {{earlyReturns}} early returns, {{loops}} loops, and {{logicalOperators}} logical operators. This makes the component difficult to understand, test, and maintain. Refactor by: 1) Extracting complex conditional rendering logic into separate sub-components with clear, descriptive names. 2) Simplifying nested ternaries and && operators by using early returns or separate render functions. 3) Extracting repeated patterns into reusable helper components. 4) Moving complex filtering/mapping logic into useMemo hooks with clear variable names. 5) Breaking down multi-responsibility components following the Single Responsibility Principle. Provide a detailed refactoring plan with before/after code examples showing how to decompose this component into a clearer component hierarchy with improved testability and maintainability.", // STATIC (~1,400 bytes)
    "variables": { "file": true, "context": {                  // DYNAMIC values
      "componentName": "AIPromptModal", "complexity": "67",
      "branches": "55", "jsxConditionals": "5", "earlyReturns": "7",
      "loops": "0", "logicalOperators": "17"
    }}
  },
  "metadata": null,                                             // DYNAMIC (when present)
  "explanation": "AIPromptModal has a structural complexity score of 67, driven by branches, loops, and JSX conditionals. High structural complexity makes a component hard to read, test thoroughly, and reason about.", // ~280 bytes STATIC + tiny DYNAMIC prefix
  "detailsMarkdown": "**Impact:** Each additional branch (if/ternary/&&) roughly doubles the number of test cases needed for full coverage. ... (many bytes) ..." // ~500 bytes STATIC
}
```

**Of 2,600 bytes in this insight, ~2,200 are static template content
(85%).** Multiplied across thousands of insights, this is the bulk of
the 130 MB DB.

### Projected reduction

Conservative estimate (80% of JSON columns are static):

| Component | Current | Projected | Saved |
|---|---|---|---|
| `analyses.result` | 41.5 MB | ~8.3 MB | 33 MB |
| `analyses.insights` | 30.6 MB | ~6.1 MB | 24 MB |
| `analyses.metrics` | 10.5 MB | ~2.1 MB | 8 MB |
| `snapshot_files.plugin_results` | 41.6 MB | ~8.3 MB | 33 MB |
| Total | 124 MB | **~25 MB** | **~99 MB** |

**Expected DB reduction: 130 MB → ~35 MB (73% smaller).**

Combined with the user's observation that less data to write = faster
writes, we expect a small but real perf improvement on both initial
analysis (saveAnalysisResult: 1010 calls) and backfill (per-commit
persistence).

## Goal

Hoist static/template content out of per-row JSON columns into one of:

(a) **Normalized DB tables**, joined at read time
(b) **Client-bundled static files** (generated at build time, shipped
    in `dist/` or `bin/`)
(c) **Hybrid** — versioned per-client manifest with a bootstrap on
    schema mismatch

Apply with zero UI regression. Validate via schema (Zod) at the read
boundary so a missing or mis-versioned definition fails loudly rather
than silently showing blank tooltips.

## Design alternatives

### Option A — Normalized DB tables (in-DB hoisting)

Introduce:
- `plugin_definitions` (pluginId, label, hint, icon, ...)
- `insight_templates` (templateId, pluginId, insightTypeId, severity, category, message_template, explanation, detailsMarkdown, suggestion, prompt_template, autoFixable, ...)
- `analysis_definitions` (analysisId, pluginId, category, ...)

Then `analyses.result` and `snapshot_files.plugin_results` slim to:

```json
{
  "pluginId": "react",
  "score": 35,
  "executionTimeMs": 4123,
  "insights": [
    { "templateId": 142, "location": {...}, "vars": {...} },
    { "templateId": 87, "location": {...}, "vars": {...} }
  ],
  "metrics": { /* numeric only */ }
}
```

**Pros**:
- Self-contained: definitions ship with the DB, no separate runtime asset
- Backfill / historical snapshots survive across plugin upgrades (template_id resolves to its row at the time it was stored, regardless of current code)
- Queryable: can SQL-join insights to templates for reporting

**Cons**:
- Schema migration is more invasive
- Read joins required for every UI surface that displays insights
- Plugin-version drift: when a plugin updates its template wording, do we update existing rows or version them?

### Option B — Client-bundled static manifests (out-of-DB hoisting)

Generate `dist/insight-definitions.json` (or `.ts`) per-client at build
time:

```ts
// generated at @vipr/desktop build time
export const INSIGHT_TEMPLATES: Record<string, InsightTemplate> = {
  'react.structural.high-complexity': {
    severity: 'warning',
    category: 'structural',
    messageTemplate: '{{componentName}} has high structural complexity ({{complexity}})',
    suggestion: 'Break down complex conditional rendering into smaller components',
    explanation: '{{componentName}} has a structural complexity score of ...',
    detailsMarkdown: '**Impact:** Each additional branch ...',
    promptTemplate: '...',
    autoFixable: false,
  },
  // ...
};
```

DB stores: `{ templateId: 'react.structural.high-complexity', vars: {...}, location: {...} }`.

UI rehydrates by lookup at read time.

**Pros**:
- Zero per-row storage cost for templates
- Versioned with the binary — no DB schema migration when wording changes
- Build-time generation makes definitions trivially auditable / reviewable
- Cross-client consistency: same generator feeds desktop, VSCode, CLI

**Cons**:
- Two client versions can disagree on template content for the same DB (acceptable: UI always reflects the current binary's wording, which is the desired behavior)
- A template ID that's been REMOVED in a newer binary breaks rehydration on old DB rows (mitigation: keep deprecated IDs in a `__legacy` map until N versions later)
- Build pipeline gains a "definitions generator" step

### Option C — Hybrid

Store the canonical definitions as **build-time static manifests
(Option B)** but also persist a snapshot of the manifest in the DB at
schema initialization (`manifest_versions` table). UI reads from
in-binary first; falls back to the DB-persisted version for legacy IDs.

**Pros**: forward-compatible across binary upgrades; old DBs still render
**Cons**: most complex; needs versioning discipline

### Recommendation

**Option B (client-bundled static manifests)** is the right call for Vipr:

1. **Vipr's plugin architecture already ties definitions to the binary.** Plugins are bundled into each client (desktop, VSCode, CLI). The current "definitions live in plugin source code" pattern means a binary upgrade already brings new definitions; we're just making that explicit and removing the DB-side duplication.

2. **The "old DB row references removed template" case is rare.** Insight IDs change when a plugin gets a major refactor (which is also when the user's DB would naturally need migrating). A `__legacy` shim in the generator handles the small set of ID renames that happen between releases.

3. **Schema-validation at the read boundary** is straightforward with Zod: the rehydration function validates the loaded manifest matches the expected shape. If a template is missing or malformed, the UI shows a clear fallback ("(template missing — please update Vipr to v0.X)") instead of breaking.

4. **Backfill speed wins.** Each insight stored shrinks from ~2,600 to ~350 bytes (87% reduction). 1010 files × N insights × 87% saved = real disk bandwidth saved per commit.

Falling back to Option A or C is acceptable if Option B surfaces a blocking issue during Phase 1 (e.g., the cross-client manifest generation pipeline turns out to be costly).

## Implementation phases

### Phase 0 — Consumer audit (✅ COMPLETE — 2026-05-11)

Catalog at [`followup-option-e-phase0-consumer-audit.md`](./followup-option-e-phase0-consumer-audit.md).

**Key findings:**
- ~73 production files + ~30 test files = ~103 files to update in Phase 2
- ~40 read sites for `.message`, ~28 for `.explanation`, ~21 for `.detailsMarkdown`, etc.
- Marketing does NOT consume runtime insights (only renders prose docs from `@vipr/documentation`) — Option E scope unaffected
- VSCode + CLI consume insights but don't consume `@vipr/documentation` — flagged for Option F
- Two specific risks: `InsightsList.tsx:22` three-way fallback chain (`message || title || description`), and `anti-pattern-query.ts:1484-1523` dynamic property lookup — both need careful handling in Phase 2
- Manifest schema can carry all consumed fields cleanly; extensibility for Option F (`metricDefinitions`, `analysisDefinitions`, `pluginDefinitions`) confirmed safe

**Original Phase 0 plan, preserved for reference:**

Map every consumer of the static fields:

```bash
grep -rn '\.message\|\.explanation\|\.detailsMarkdown\|\.suggestion\|\.prompt\b' \
  clients/desktop/src/renderer/components clients/desktop/src/renderer/pages \
  packages/ui/src --include="*.ts" --include="*.tsx"
```

Catalog:
- Which UI surfaces read each field
- Whether any consumer destructures with `??` fallbacks
- Whether MCP server / VSCode extension / marketing site read the same fields
- Test files asserting specific message content (these need to be updated to assert by template ID, not message text)

**Deliverable**: a single doc listing every read site with the field
it consumes. No code changes.

### Phase 1 — Build manifest generator (~2-3 days)

Create `packages/insight-manifest/` (or similar — placement TBD during
Phase 0). Responsibilities:

- Scan every analyzer package's plugin definitions
- Extract `InsightTemplate` entries (severity, category, message
  template, explanation, detailsMarkdown, suggestion, prompt template,
  autoFixable)
- Emit a typed `INSIGHT_TEMPLATES` constant + a Zod schema
- Emit a deprecation map (`__legacy`) for IDs that have been renamed
  in this version

Wire into each client's build:
- `@vipr/desktop` — generate before `electron-forge package`
- `@vipr/vscode-extension` — generate before bundle
- `@vipr/cli` — generate before tsup build

Acceptance:
- Generator runs in CI and emits identical output for all three clients
- Output is type-safe and Zod-validated
- Generation is idempotent (no churn on rerun)
- Unit test catches if a plugin defines an insight without a stable ID

### Phase 2 — Dual-write writer + lookup reader (~3-5 days)

Modify the engine's analysis serialization so that:

1. **At write time**, the engine still produces a full `PluginResult`
   for in-memory use, but the persistence layer (`snapshot-file-state.ts`
   `serializePluginResults` + `analyses` table writer) STRIPS
   static fields before persisting, leaving only `{ templateId, vars,
   location }` per insight + numeric metrics.

2. **At read time** (UI / MCP / presenters), a new hydration utility
   reads the slim DB row, looks up each `templateId` in the bundled
   manifest, interpolates `vars`, returns a fully-formed
   `PluginInsight` object identical to the pre-migration shape.

Crucially: **dual-write phase first.** Persist BOTH the slim form and the
old fat form for a period, so we can switch reads back to fat if the
slim path has a bug. Drop fat-write after Phase 3 validation.

Acceptance:
- All current UI surfaces render identically (snapshot tests or visual
  regression)
- MCP server / VSCode / CLI continue to return rich insight payloads
- Zod validation passes on every hydration

### Phase 3 — Schema migration + measurement (~1-2 days)

- Add a migration that DROPs the fat-write columns (or null-coalesces
  them) once Phase 2 read-path is validated
- Capture before/after DB sizes on the same project
- Capture before/after backfill wall-clock to quantify the persistence
  speedup
- Run the postcheck script — every invariant must still pass

Acceptance gate:
- DB size ≥ 50% smaller on the baseline (130 MB → ≤ 65 MB)
- No new failed-write events
- Backfill wall-clock unchanged or faster (not slower)
- Postcheck PASS, all 4084+ desktop tests still green

### Phase 4 — Cleanup (~half day)

- Remove dual-write code
- Update CLAUDE.md docs to reflect the new persistence model
- Add a "definitions registry" entry to plan docs

## Risk + rollback

- **Risk: a template ID drifts between client versions during dev.**
  Mitigation: enforce stable IDs in the generator; CI fails if a
  generated diff shows ID rename without a `__legacy` entry.
- **Risk: a UI surface destructures a field that no longer exists in
  the DB row.** Mitigation: Phase 0 audit catches every site; Phase 2's
  hydration utility is the ONLY allowed read path for these fields.
  TypeScript types enforce the boundary.
- **Risk: backfill writes get SLOWER because of new tables/joins.**
  Mitigation: Phase 3 baseline gate explicitly disallows wall-clock
  regression.
- **Rollback**: drop the new manifests, revert the persistence-layer
  change. The fat-write data is still in the DB during Phase 2 (dual
  write), so no data loss.

## Out of scope

- **Migrating EXISTING fat-write data to slim form retroactively.**
  Existing users get the benefit on their next analysis run; old data
  stays as-is. (Adding a one-shot retro migration is a separate item.)
- **Hoisting `deep_analysis_findings` or other small JSON tables.**
  They're <1 MB combined; not worth the effort yet.
- **Compressing the binary content_sha blobs in `file_versions`.** A
  separate orthogonal optimization (zstd compression at the BLOB level
  could shrink that table further).
- **Renaming or refactoring plugins themselves.** This initiative is
  pure storage normalization; no analyzer logic changes.

## Acceptance criteria (for closing the initiative)

- DB size on the baseline 1000-file project: ≤ 50 MB (was 130 MB)
- Per-snapshot `plugin_results` column size: ≤ 10 KB avg (was 43 KB)
- All UI surfaces render identically (visual regression or snapshot tests)
- MCP server returns identical-shape responses pre/post migration
- Zod validation at the hydration boundary fails loudly for malformed input
- Postcheck script + all desktop tests pass
- No backfill or foreground wall-clock regression vs Stage 2 baselines

## Estimated effort

| Phase | Effort | Notes |
|---|---|---|
| Phase 0 (consumer audit) | half day | Pure read; no code change |
| Phase 1 (manifest generator) | 2-3 days | Includes build wiring for 3 clients |
| Phase 2 (dual-write + reader) | 3-5 days | Most of the implementation effort |
| Phase 3 (schema migration + measure) | 1-2 days | One backfill re-run for measurement |
| Phase 4 (cleanup) | half day | Remove dual-write, doc updates |
| **Total** | **~2 weeks** of focused work, multi-session |

## When to do this

User-requested follow-on after Option C Stage 2. Higher ROI than Option
C Stage 3 (crash recovery) because:

- Storage normalization ships user-visible benefit (smaller disk
  footprint, slightly faster writes)
- Stage 3's value (crash recovery hardening) is speculative — we
  haven't observed a crash loop in the wild yet
- Both are independent; doing Option E first does NOT block Stage 3

Recommended sequence: **Option E first**, then Stage 3 if needed.
