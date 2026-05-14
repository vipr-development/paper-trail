# Phase 1a — Fix `isAnalyzableFile` Drift (Bug Fix)

> **This is a correctness bug, not a performance optimization.** It happens to
> also save backfill time. Ship as a standalone commit, separate from any
> exclusion-list expansion work.

## Goal

Make backfill use the same file-exclusion logic that foreground (draft snapshot
service) and the AI remediation service use, so files in `mock/`, `__tests__/`,
`fixtures/`, `node_modules/`, etc. are not silently re-analyzed for every
historical commit.

## Why now

Three copies of `isAnalyzableFile` exist in the codebase. They have **drifted**:

| Location | Filter logic | Catches `mock/page-mock-handlers.ts`? |
|---|---|---|
| [`clients/desktop/src/main/analysis/draft-snapshot-service.ts:65`](../../clients/desktop/src/main/analysis/draft-snapshot-service.ts) | `hasSupportedExtension(path) && !isExcludedPath(path)` | ✅ Yes (foreground draft path) |
| [`clients/desktop/src/main/analysis/historical-snapshot-service.ts:135`](../../clients/desktop/src/main/analysis/historical-snapshot-service.ts) | `hasSupportedExtension(filePath) && !EXCLUDED_FILE_PATTERNS.some(p => p.test(filePath))` | ❌ **No** (backfill path) |
| [`clients/desktop/src/main/ai/ai-remediation-service.ts:143`](../../clients/desktop/src/main/ai/ai-remediation-service.ts) | (Read this file in implementation step 2 — confirm it matches one of the above) | ❓ Unknown — must audit |

The historical-snapshot path tests **only filename patterns** (`*.test.ts`,
`*.stories.tsx`, etc.). It skips the directory-level check that
`isExcludedPath` performs against `DEFAULT_IGNORE_PATTERNS` (which includes
`mock`, `mocks`, `__mocks__`, `__tests__`, `fixtures`, `docs`, `public`,
`generated`, `node_modules`, `.git`, `dist`, `build`, `.next`, `coverage`,
`.turbo`, `out`, `.vite`).

**Evidence** from baseline backfill data (see [`baselines.md`](./baselines.md)):
the slowest backfill files include `src/main/ipc/mock/page-mock-handlers.ts`
at 17.5s — a file that lives in a `mock/` directory and should be excluded.
Foreground correctly skips it; backfill does not.

## Prerequisites

- Working tree clean on branch `claude/unruffled-hawking-f89a04`
- `pnpm install` has been run
- `@vipr/common` and `@vipr/logging` are built (`pnpm --filter @vipr/common build && pnpm --filter @vipr/logging build`)

## Implementation

### Step 1 — Fix the historical service

File: [`clients/desktop/src/main/analysis/historical-snapshot-service.ts`](../../clients/desktop/src/main/analysis/historical-snapshot-service.ts)

Replace the local `isAnalyzableFile` function. Currently:

```ts
import { hasSupportedExtension } from '@/shared/utils/path-utils';
import { EXCLUDED_FILE_PATTERNS } from '@vipr/common/client-constants';

function isAnalyzableFile(filePath: string): boolean {
  return (
    hasSupportedExtension(filePath) &&
    !EXCLUDED_FILE_PATTERNS.some(pattern => pattern.test(filePath))
  );
}
```

Change to:

```ts
import { hasSupportedExtension } from '@/shared/utils/path-utils';
import { isExcludedPath } from '@vipr/common/client-constants';

function isAnalyzableFile(filePath: string): boolean {
  return hasSupportedExtension(filePath) && !isExcludedPath(filePath);
}
```

Notes:

- Drop the `EXCLUDED_FILE_PATTERNS` import; `isExcludedPath` already covers it
  internally (see [`packages/common/src/client-constants.ts:439`](../../packages/common/src/client-constants.ts)).
- `isExcludedPath` takes a relative path. Confirm callers pass relative paths,
  not absolute. `analyzeFileList` is called from `createFullSnapshot` and
  `createIncrementalSnapshot` — trace the call sites and verify what they pass.
  - Search call sites: `grep -n "isAnalyzableFile" clients/desktop/src/main/analysis/historical-snapshot-service.ts`
  - If absolute paths are flowing in, this fix needs a normalization pass at
    the call site. **This is the most likely subtle issue.** Test carefully.

### Step 2 — Audit the AI remediation service

File: [`clients/desktop/src/main/ai/ai-remediation-service.ts`](../../clients/desktop/src/main/ai/ai-remediation-service.ts)

Read line 143 and the surrounding function. There's a third `isAnalyzableFile`
defined here. Determine which behavior it matches:

- If it matches **historical-snapshot-service** (the buggy one): apply the same
  fix.
- If it matches **draft-snapshot-service** (the correct one): leave it alone
  but document in the commit message that you verified it.
- If it diverges from both: that's a separate bug. Document and decide whether
  to fold the fix into this commit or split it out.

### Step 3 — Consider extraction (optional, only if it falls out cleanly)

If you're confident, you may extract a single shared `isAnalyzableFile` to
`clients/desktop/src/shared/utils/path-utils.ts` and import it from all three
sites. This eliminates the drift class going forward. **Do not pursue this if
it requires touching more than 4 files** — keep this commit small.

## Acceptance criteria

- `historical-snapshot-service.ts`'s `isAnalyzableFile` calls `isExcludedPath`
- `ai-remediation-service.ts`'s `isAnalyzableFile` is either correct or also fixed (commit message states which)
- Existing tests for `historical-snapshot-service.test.ts` (41 tests) still pass
- Existing tests for `ai-remediation-service.test.ts` (if it exists) still pass
- New test: `historical-snapshot-service.test.ts` covers a file in `mock/` to prove it gets excluded
- `pnpm --filter @vipr/desktop typecheck` clean
- Prettier clean on touched files

## Test plan

```bash
# Build deps
pnpm --filter @vipr/common build
pnpm --filter @vipr/logging build

# Typecheck
pnpm --filter @vipr/desktop typecheck

# Targeted tests
pnpm --filter @vipr/desktop test -- run src/main/analysis/historical-snapshot-service.test.ts
pnpm --filter @vipr/desktop test -- run src/main/analysis/ # full analysis dir

# Optional broader run (will surface unrelated date-window flakes — see "Known flakes" below)
pnpm --filter @vipr/desktop test
```

### New test to add

Add to `clients/desktop/src/main/analysis/historical-snapshot-service.test.ts`
near the existing file-filtering tests:

```ts
it('excludes files in mock/ directory from analysis', () => {
  expect(isAnalyzableFile('src/main/ipc/mock/page-mock-handlers.ts')).toBe(false);
  expect(isAnalyzableFile('src/main/__mocks__/foo.ts')).toBe(false);
  expect(isAnalyzableFile('src/main/__tests__/bar.ts')).toBe(false);
  expect(isAnalyzableFile('src/main/fixtures/data.ts')).toBe(false);
});

it('still includes regular source files', () => {
  expect(isAnalyzableFile('src/renderer/pages/FileDetail.tsx')).toBe(true);
  expect(isAnalyzableFile('src/main/orchestration/workspace-orchestrator.ts')).toBe(true);
});
```

If `isAnalyzableFile` is not exported from the module, either export it or
test through a public function that consumes it.

### Known flakes (pre-existing, NOT caused by this change)

- `src/main/db/budget-queries.test.ts:399` — date-window test using April 2026 fixtures, fails on the boundary day
- `src/renderer/pages/Documentation.test.tsx:80` — anchor-id rendering test, also date-related

Both fail identically on clean main with no changes. Don't be alarmed if they
appear in a full suite run. Don't try to fix them in this commit.

## Risk + rollback

- **Risk**: very low. `isExcludedPath` is a strict superset of
  `EXCLUDED_FILE_PATTERNS.some(...)` — every file currently included will still
  be included; only previously-incorrectly-included files (mocks, tests,
  fixtures, etc.) will be newly excluded.
- **Subtle risk**: if call sites pass absolute paths to `isAnalyzableFile`, the
  directory check inside `isExcludedPath` will not match (it splits on `/` and
  checks segments). Verify path shape at call sites in step 1.
- **Rollback**: trivial — revert the commit. No data migration, no schema
  change.

## Out of scope

- **Do NOT** add new directories or filename patterns to `DEFAULT_IGNORE_PATTERNS`
  or `EXCLUDED_FILE_PATTERNS` in this commit. That work is [Phase 1b/c](./02-exclusion-list-expansion.md)
  and is conditional on the Phase 3 worktree review.
- **Do NOT** change `historicalBatchConcurrency`. That's [Phase 2](./03-backfill-concurrency.md).
- **Do NOT** refactor the analysis pipeline. That's [Phase 4](./06-per-file-mean-reduction.md).

## Estimated effort

30–60 minutes including the new test. Single commit.

## Commit message template

```
fix(desktop): use isExcludedPath in historical-snapshot-service for backfill filtering

The historical snapshot service used a local isAnalyzableFile that only
checked EXCLUDED_FILE_PATTERNS (filename patterns), bypassing the directory-
level check in isExcludedPath against DEFAULT_IGNORE_PATTERNS. As a result,
backfill silently analyzed files in mock/, __tests__/, fixtures/, generated/,
etc. that foreground correctly skipped.

Evidence: backfill perf instrumentation surfaced
src/main/ipc/mock/page-mock-handlers.ts in the slowest-files list at 17.5s.

Switch to the shared isExcludedPath helper. Also audited
ai-remediation-service.ts (verdict: <state result here>).

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
```
