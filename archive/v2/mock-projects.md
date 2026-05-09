# Mock Projects

## Purpose

Generated mock projects let Desktop load a realistic, already-analyzed project without hand-maintained fake data.

The source of truth is:

1. `workspace/`
2. `fixture.json`

The generated derivative is:

- compiled corpus project artifact
- mock project registry entry

## Build-Time Generation

`@vipr/fixtures` now emits:

- `@vipr/fixtures/corpus`
- `@vipr/fixtures/mock-projects`

Those generated modules are built from frozen corpus items and are not hand-edited.

## Current Phase-1 Coverage

Corpus-backed mock projects drive:

- repository metadata
- dashboard summary
- file list
- issue list / top issues
- file detail
- velocity
- churn

Unsupported advanced surfaces still fall back to the legacy hand-authored mock fixtures.

## How Desktop Uses Them

In Developer Tools:

1. Turn on `Use Mock Data`
2. Choose a `Mock Project`

Desktop then loads the selected corpus-generated project through the mock repository abstraction instead of the old one-size-fits-all fixture set.

## Local Verification Workflow

```bash
pnpm --filter @vipr/qa-harness build
pnpm --filter @vipr/fixtures build
pnpm vipr:qa mock-projects
```

Then start Desktop in mock mode and inspect the selected project’s UI.
