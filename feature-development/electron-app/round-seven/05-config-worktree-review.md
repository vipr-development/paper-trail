# Phase 3 Worktree Review — Verdict

> Read-only architecture review of the `productionize-vipr-config` work in
> `/Users/jamesleebaker/.codex/worktrees/1bd5/vipr`. Conducted per the bounded
> review protocol in [`05-config-productization.md`](./05-config-productization.md).
> No edits were made to the worktree.

## Verdict: **B — Fold in with specific tweaks**

The design substantially **exceeds** the principles laid out in Phase 5. It
delivers Phases 3a (file exclusion), 3b (per-plugin enablement), 3c
(per-analysis enablement), and adds a config-driven budgets system that wasn't
in our plan. **Throwing this away would be a serious waste**, and rebuilding
it from scratch would take 1–2 weeks of high-risk work.

The tweaks listed below are small. The largest issue is **operational, not
architectural** (the work is uncommitted in a detached HEAD).

## Worktree state — read this first

| Item | Value |
|---|---|
| Working dir | `/Users/jamesleebaker/.codex/worktrees/1bd5/vipr` |
| Reported branch | `productionize-vipr-config` |
| Actual git state | **Detached HEAD at `2627038d`** (no current branch) |
| Commits ahead of main | **0** |
| Modified files | 77 |
| Untracked files | 10 |
| Diff scope | **+2,005 / −615** across 87 files |

**This work is not committed anywhere.** It exists only as a working tree.
Before any incorporation can happen, someone (the owner of that worktree or
the incorporator) needs to `git checkout -b productionize-vipr-config && git
add . && git commit`. Otherwise a stray `git reset --hard` destroys it.
**Highest priority operational risk.**

## What was actually built

### New files (10)

| File | Purpose |
|---|---|
| `packages/common/src/config/schema.ts` (178 LoC) | Zod schema + `validateViprConfig` + `hashResolvedConfig` |
| `packages/common/src/config/paths.ts` (39 LoC) | Constants for config file names |
| `packages/common/src/config/targets.ts` (213 LoC) | `resolveAnalysisTargets()` — config-aware file expansion + exclusion |
| `packages/common/src/config/template.ts` (180 LoC) | Template strings for the `init` command |
| `packages/common/src/config/schema.test.ts` (64 LoC) | Schema validation tests |
| `packages/common/src/config/paths.test.ts` (37 LoC) | |
| `packages/common/src/config/targets.test.ts` (80 LoC) | |
| `packages/common/src/config/template.test.ts` (41 LoC) | |
| `clients/desktop/src/main/services/config-budget-sync.ts` (182 LoC) | Syncs `vipr.config.json` budgets into desktop DB |
| `clients/desktop/src/main/services/config-budget-sync.test.ts` | |

### Significantly modified

- `packages/common/src/config/loader.ts` (355 LoC, +substantial): `findConfigFile` walks up directory tree; `loadConfigWithExtends` handles preset chains with circular detection; `loadConfig` returns `{ config, configPath, loadedPaths, warnings }`
- `packages/common/src/config/merge.ts` (100 LoC): `deepMerge` + `mergeConfigs` with documented merge strategy (arrays REPLACE, plain objects merge, undefined skipped, null replaces)
- `packages/common/src/types/config/vipr-config.ts` (190 LoC): full `ViprConfig` interface — `extends`, `include`, `exclude`, `global`, `plugins`, `analyses`, `budgets`, `output`, `env`
- `packages/common/src/types/config/plugin-config.ts` (34 LoC): `PluginConfig` base with `enabled`, `thresholds`, `weights`, `analyses`
- `packages/common/src/config/defaults.ts` (44 LoC): `getSystemDefaults()` — pulls `DEFAULT_IGNORE_PATTERNS` from `client-constants` (the floor)
- `packages/common/schemas/vipr-config.schema.json` (526 LoC): JSON Schema for IDE autocomplete
- `packages/engine/src/analysis-engine.ts` (+215 LoC): accepts `projectConfig: ViprConfig` + `configHash`; `isAnalysisEnabled()` checks per-analysis config; `getProjectPluginConfig()` looks up plugin config (handles `react` and `@vipr/react` keys); `getConfigAwareContentHash()` invalidates cache on config change
- `clients/desktop/src/main/analysis/utility-process-manager.ts`: new `configureProject(repoPath)` method that calls `loadConfig` and sends `configureProject` IPC message to worker
- `clients/desktop/src/utility/worker.ts` (+68 LoC): handles `configureProject` IPC, rebuilds engine with project config
- `clients/desktop/src/shared/ipc/schemas.ts`: `configureProject` IPC message + payload
- `clients/vscode-extension/src/extension.ts` + `core/workspace-config-loader.ts`: VS Code reads same `vipr.config.json`
- `clients/cli/src/commands/{analyze,init}-command.ts`: CLI consumers updated
- `analyzers/{core,react}/src/{config/schema,plugin}.ts`: per-plugin config integration

### Documentation updates

- `documentation/docs/reference/configuration.md`
- `documentation/docs/reference/cli-commands.md`
- `packages/documentation/content/guides/{cli,configuration,desktop-app,vscode-extension}.md`
- `packages/documentation/content/reference/cli-{api,commands}.md`

## Principle-by-principle assessment

| Principle | Status | Note |
|---|---|---|
| 1. Hierarchical merge | ✅ | System defaults → `extends` chain → user config → env override → CLI override |
| 2. Append vs replace explicit | ⚠️ | Arrays REPLACE (documented). gitignore-style negation NOT supported. Acceptable but worth noting. |
| 3. Schema extensible | ✅ | Top-level `.strict()` (catches typos); plugin/analysis configs use `.catchall(z.unknown())` (extensible) |
| 4. Single canonical loader | ✅ | `loadConfig` from `@vipr/common/config` is consumed by CLI, desktop utility-process, snapshot-service, VS Code extension. **No duplicated parsing.** |
| 5. Zod validation | ✅ | Comprehensive. Recursive validation for env overrides. |
| 6. Graceful failure | ✅ | `ConfigValidationError` with helpful messages. Warnings collected separately for non-fatal issues. |

## What's MORE than we asked for (good surprises)

1. **Config hash for cache invalidation.** `hashResolvedConfig()` produces a stable sha256. The engine uses it via `getConfigAwareContentHash()` so analysis cache invalidates when config changes. The desktop's `configureProject` IPC carries the hash. **This solves a problem we didn't even spec.**
2. **`extends` mechanism with circular detection.** Supports `'@vipr/preset-strict'` style preset reuse and chained config files.
3. **Recursive env overlays.** `env.ci`, `env.development` etc. are validated recursively — an env override is itself a `Partial<ViprConfig>`.
4. **Config-driven budgets** (full DB sync via `config-budget-sync.ts`). Project config can declare budgets that flow into the desktop DB. Out of scope for our perf plan but real user value.
5. **`init` command template** — the CLI's `vipr init` likely scaffolds a `vipr.config.json` with sensible defaults.
6. **JSON Schema for IDE autocomplete** (526 LoC `vipr-config.schema.json`) — referenced via `$schema` in user configs.
7. **Helpful unsupported-file detection** — warns about `.viprrc`, `vipr.config.{ts,mts,js,mjs}` and points users to `vipr.config.json`.

## Specific tweaks needed before/after incorporation

### Tweak 1 — Commit the work first (operational, not in scope of incorporation but blocking)

```bash
cd /Users/jamesleebaker/.codex/worktrees/1bd5/vipr
git checkout -b productionize-vipr-config
git add .
git commit -m "feat: productionize vipr.config.json across all surfaces"
```

Then it can be evaluated as a real commit, cherry-picked, or merged.

### Tweak 2 — Phase 1a (the bug fix) is still required

I confirmed: this worktree did **NOT** fix the `isAnalyzableFile` drift in
`historical-snapshot-service.ts`. The local function still uses only
`EXCLUDED_FILE_PATTERNS` and skips the directory-level `isExcludedPath` check.

```ts
// historical-snapshot-service.ts (in worktree, still buggy):
function isAnalyzableFile(filePath: string): boolean {
  return (
    hasSupportedExtension(filePath) &&
    !EXCLUDED_FILE_PATTERNS.some(pattern => pattern.test(filePath))
  );
}
```

**Phase 1a should still ship as a separate commit.** It's pure correctness
work that doesn't depend on the config system. Ship it whether or not we
incorporate this worktree.

### Tweak 3 — `ai-remediation-service.ts` has its own broken filter

```ts
// Found in worktree at ai-remediation-service.ts:
function isAnalyzableFile(filePath: string): boolean {
  const extension = path.extname(filePath).toLowerCase();
  return ['.ts', '.tsx', '.js', '.jsx'].includes(extension);
}
```

This bypasses **both** `EXCLUDED_FILE_PATTERNS` and `DEFAULT_IGNORE_PATTERNS`.
Folds into Phase 1a's audit. Worth fixing in the same commit.

### Tweak 4 — Phase 1b (default exclusion expansion) still applies

The worktree did NOT add `.storybook`, `.vercel`, `.changeset`, etc. to
`DEFAULT_IGNORE_PATTERNS`. Adding them is still valuable AND now flows through
the config system automatically (because `getSystemDefaults()` spreads from
`DEFAULT_IGNORE_PATTERNS`).

**Caveat from Tweak 2's merge semantics**: a user who sets
`global.ignorePatterns: ['custom']` in their config will REPLACE the defaults
(arrays don't merge). Document this in the config docs. Negation support is a
future enhancement.

### Tweak 5 — Settings UI (Phase 3e) is NOT included

The worktree integrates config consumption everywhere but does not expose a
settings UI for editing `vipr.config.json` from inside the desktop app or VS
Code. Users must edit the file directly. Phase 3e remains future work,
unblocked by this incorporation.

### Tweak 6 — Re-baseline after incorporation

Cache-key changes (the new `getConfigAwareContentHash`) mean every analysis
will see a "config changed, invalidate" on first run after incorporation.
That's expected — capture a new baseline so Phase 4 measurements are honest.

### Tweak 7 — Verify `analyses` enablement actually short-circuits

Spot-check that when `vipr.config.json` sets `analyses: { 'core-halstead':
{ enabled: false } }`, the engine actually skips that analysis (not just
filters its output). The `isAnalysisEnabled` filter at line 673 of
`analysis-engine.ts` looks correct, but write a test confirming the analysis
function is never called. **This is the perf lever we want for Phase 4.**

## Inventory: what's done vs what remains

| Phase 3 sub-phase | Status |
|---|---|
| 3a — File/directory exclusion in config | ✅ Done (via `global.ignorePatterns`, `include`, `exclude`) |
| 3b — Per-plugin enablement | ✅ Done (via `plugins[id].enabled`) |
| 3c — Per-analysis enablement | ✅ Done (top-level `analyses[id].enabled` and per-plugin `plugins[id].analyses[id].enabled`) |
| 3d — Per-anti-pattern enablement | ⚠️ Implicit via per-analysis. Whether you can disable individual anti-patterns within an analysis depends on plugin design — needs verification per-plugin. |
| 3e — Settings UI in desktop + VS Code | ❌ Not in this work. Future. |
| 3f — Per-metric visibility | ❌ Not in this work. Future. |
| Bonus: budgets in config | ✅ Done (not in original plan but valuable) |
| Bonus: config-aware cache invalidation | ✅ Done (not in original plan but valuable) |

## Plan revisions needed after this verdict

Once we proceed with incorporation, revise these plans:

- **`02-exclusion-list-expansion.md`** — drop "conditional on Phase 3" framing.
  Additions to `DEFAULT_IGNORE_PATTERNS` automatically flow through
  `getSystemDefaults()`. Note the merge-replace caveat.
- **`05-config-productization.md`** — almost entirely superseded by this
  verdict. Rewrite as a tight "incorporation execution plan" that points
  here.
- **`00-README.md`** — update phase index to reflect Phase 3 was largely
  delivered by another agent and is being incorporated, not built from
  scratch. Move Phase 3e (settings UI) and Phase 3f (per-metric visibility)
  to a new "deferred" section.
- **`06-per-file-mean-reduction.md`** — note that the per-analysis enablement
  knob (Phase 3c) is now an additional perf lever: users can disable expensive
  analyses they don't care about. This complements (doesn't replace) the
  per-file work.

## Recommended incorporation sequence

If you green-light Verdict B:

1. **Owner of `productionize-vipr-config` worktree commits and pushes the branch** (Tweak 1). Coordinate so the work survives.
2. **On `claude/unruffled-hawking-f89a04`**: ship Phase 1a (the bug fix) as a standalone commit. Pure correctness, not config-related, ships independently.
3. **Cherry-pick or merge `productionize-vipr-config`** onto our branch. Resolve any conflicts (likely few — different files for the most part).
4. **Add Tweak 4** (Phase 1b additions to `DEFAULT_IGNORE_PATTERNS`) as a follow-up commit.
5. **Audit Tweak 7** (verify per-analysis enablement actually short-circuits) and write a test if missing.
6. **Re-baseline** per [`04-rebaseline-protocol.md`](./04-rebaseline-protocol.md). Capture as `2026-05-XX-after-config-incorporation.md`.
7. **Revise the affected plan docs** as listed above.
8. **Then proceed with Phase 2 (concurrency) and Phase 4 (per-file mean reduction).**

## Time spent on this review

Approximately 35 minutes of focused reading. Within the 30–45 minute budget.
No edits were made.
