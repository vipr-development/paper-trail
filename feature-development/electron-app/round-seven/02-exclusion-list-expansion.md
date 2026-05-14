# Phase 1b/c — Expand Default Exclusion Lists

> **Conditional plan.** Whether this ships as hardcoded changes (Phase 1b/c) or
> as defaults inside the new config schema (folded into Phase 3) depends on
> the verdict in [`05-config-productization.md`](./05-config-productization.md).
> Read that doc's "Decision tree" section first.

## Goal

Add directories and filename patterns that should be excluded from analysis by
default but currently leak through, producing noise in reports and wasted
analysis time.

## Why now

Baseline foreground data (see [`baselines.md`](./baselines.md)) shows two
Storybook config files in the top-10 slowest:

- `clients/desktop/.storybook/preview.tsx` — 5,356ms
- `clients/desktop/.storybook/main.ts` — 5,353ms

These slipped through because `EXCLUDED_FILE_PATTERNS` is filename-based
(`*.stories.*`, `*.config.*`, etc.) and `.storybook/main.ts` doesn't match any
filename pattern, while `.storybook` isn't in `DEFAULT_IGNORE_PATTERNS`.

This is a small wall-clock win (~10–30s) but matters more for **report
quality** — anti-pattern dashboards stop surfacing noise from generated code,
build configs, and tooling.

## Prerequisites

- **Phase 1a must be shipped first.** Otherwise the directory-level rules in
  this phase have no effect on the backfill path.
- Read [`05-config-productization.md`](./05-config-productization.md) decision tree
  to know whether this work goes here or there.

## Implementation

### Step 1 — Expand `DEFAULT_IGNORE_PATTERNS`

File: [`packages/common/src/client-constants.ts`](../../packages/common/src/client-constants.ts) (line 46)

Current:

```ts
export const DEFAULT_IGNORE_PATTERNS = [
  'node_modules',
  '.git',
  'dist',
  'build',
  '.next',
  'coverage',
  '.turbo',
  'out',
  '.vite',
  '__mocks__',
  'mock',
  'mocks',
  '__tests__',
  'fixtures',
  'docs',
  'public',
  'generated',
] as const;
```

Add (Tier 1 — high confidence):

```
'.storybook',          // Storybook config + setup (runtime config, not stories)
'.vercel',             // Vercel deployment artifacts
'.changeset',          // Changesets versioning files
'.husky',              // Git hooks
'storybook-static',    // Storybook build output
'.svelte-kit',         // SvelteKit build (future-proof)
'.output',             // Nuxt / SvelteKit output (future-proof)
'.claude',             // AI agent config — present in Vipr repo
'.codex',              // AI agent config — present in Vipr repo
'.cursor',             // AI agent config — present in some Vipr setups
```

**Do NOT add (deferred to user judgment / Phase 3):**

- `migrations` — *could* contain real signal a user cares about (e.g.,
  Vipr's own DB migration file is huge). Better surfaced as a default users
  opt out of in Phase 3 than baked into the floor.
- `e2e`, `tests` — same reasoning. Some teams' integration tests live here
  and ARE in scope.
- `scripts`, `tools` — some scripts are production code (release pipelines,
  build tooling). Don't blanket-exclude.

### Step 2 — Mirror into `EXTENDED_IGNORE_PATTERNS`

Same file, line 70. The list already mirrors `DEFAULT_IGNORE_PATTERNS` with
glob syntax. Add the same entries with `**/` prefix and `/**` suffix:

```ts
'**/.storybook/**',
'**/.vercel/**',
'**/.changeset/**',
'**/.husky/**',
'**/storybook-static/**',
'**/.svelte-kit/**',
'**/.output/**',
'**/.claude/**',
'**/.codex/**',
'**/.cursor/**',
```

This is consumed by file watchers (e.g., `clients/desktop/src/main/fs/watcher.ts`).

### Step 3 — Expand `EXCLUDED_FILE_PATTERNS`

Same file, line 264. Add filename patterns for generated code:

```ts
/\.gen\.(ts|tsx|js|jsx|mjs|cjs)$/,           // *.gen.ts — common codegen suffix
/\.generated\.(ts|tsx|js|jsx|mjs|cjs)$/,     // *.generated.ts
/\.codegen\.(ts|tsx|js|jsx|mjs|cjs)$/,       // *.codegen.ts
```

Skip `*.snap` for now — Jest snapshot files don't typically have `.ts`/`.tsx`
extensions, so they wouldn't pass `hasSupportedExtension` anyway. Noop add.

### Step 4 — Update tests

File: [`packages/common/src/client-constants.test.ts`](../../packages/common/src/client-constants.test.ts) (verify it exists; if not, the file likely exports tested-elsewhere constants)

If a test file exists for `isExcludedPath` / `DEFAULT_IGNORE_PATTERNS` / `EXCLUDED_FILE_PATTERNS`, add cases:

```ts
describe('DEFAULT_IGNORE_PATTERNS', () => {
  it('excludes .storybook directory', () => {
    expect(isExcludedPath('clients/desktop/.storybook/preview.tsx')).toBe(true);
    expect(isExcludedPath('clients/desktop/.storybook/main.ts')).toBe(true);
  });
  it('excludes other tooling directories', () => {
    expect(isExcludedPath('packages/foo/.vercel/output.json')).toBe(true);
    expect(isExcludedPath('.changeset/some-changeset.md')).toBe(true);
    expect(isExcludedPath('packages/foo/storybook-static/index.html')).toBe(true);
  });
  it('does NOT exclude legitimate paths that contain similar substrings', () => {
    // Make sure substring matches don't false-positive
    expect(isExcludedPath('src/components/CursorTracker.tsx')).toBe(false);
    expect(isExcludedPath('src/utils/output-formatter.ts')).toBe(false);
  });
});

describe('EXCLUDED_FILE_PATTERNS', () => {
  it('excludes generated code suffixes', () => {
    expect(isExcludedPath('src/api/types.gen.ts')).toBe(true);
    expect(isExcludedPath('src/api/types.generated.ts')).toBe(true);
    expect(isExcludedPath('src/api/client.codegen.tsx')).toBe(true);
  });
});
```

If no test file exists for these constants, add `client-constants.test.ts` next
to the source file.

## Acceptance criteria

- `DEFAULT_IGNORE_PATTERNS` and `EXTENDED_IGNORE_PATTERNS` contain the new directory entries
- `EXCLUDED_FILE_PATTERNS` contains the new filename patterns
- Tests pass demonstrating exclusion works for the new patterns
- Tests pass demonstrating no false-positive substring matches
- `pnpm --filter @vipr/common test` clean
- `pnpm --filter @vipr/common typecheck` clean
- `pnpm --filter @vipr/common build` clean
- Downstream consumers still build:
  - `pnpm --filter @vipr/desktop typecheck`
  - `pnpm --filter @vipr/desktop test -- run src/main/analysis/`

## Test plan

```bash
pnpm --filter @vipr/common test
pnpm --filter @vipr/common build
pnpm --filter @vipr/desktop typecheck
pnpm --filter @vipr/desktop test -- run src/main/analysis/
```

## Risk + rollback

- **Risk**: low. Strictly subtractive change — files newly excluded were noise.
- **One concern**: if a downstream test fixture uses `__mocks__/` or similar
  paths and expected those files to be analyzed, that test will now skip
  them. Check by running the full desktop suite. The two known pre-existing
  flakes (`budget-queries.test.ts:399`, `Documentation.test.tsx:80`) are
  unrelated.
- **Rollback**: revert the commit. No data implications.

## Out of scope

- **Do NOT** change the underlying filter logic — only data added to existing arrays
- **Do NOT** add `migrations`, `e2e`, `tests`, or `scripts` — those are user judgment calls
- **Do NOT** introduce config-loading code — that's Phase 3
- **Do NOT** ship this without first reading the Phase 3 decision tree

## Estimated effort

30–60 minutes. Single commit.

## Commit message template

```
chore(common): expand default exclusion lists with tooling directories

Adds .storybook, .vercel, .changeset, .husky, storybook-static, .svelte-kit,
.output, .claude, .codex, .cursor to DEFAULT_IGNORE_PATTERNS (and the
EXTENDED_IGNORE_PATTERNS glob mirror).

Adds *.gen.*, *.generated.*, *.codegen.* to EXCLUDED_FILE_PATTERNS.

Evidence: foreground baseline showed .storybook/preview.tsx (5.4s) and
.storybook/main.ts (5.4s) in the top-10 slowest files. These are tooling
config, not product code.

Deferred for Phase 3 (user-configurable defaults): migrations, e2e, tests,
scripts — these may contain real signal depending on the project.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
```
