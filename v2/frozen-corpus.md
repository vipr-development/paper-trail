# Frozen Corpus

## What It Is

A frozen corpus item is a realistic, pinned, reviewable mini-workspace that becomes a long-lived regression guard.

Each item lives under:

```text
packages/fixtures/src/desktop-corpus/<corpus-id>-v<version>/
  workspace/...
  fixture.json
  README.md   # optional
```

## Why It Exists

Synthetic fixtures are good for exact analyzer checks, but they are too small to represent the real Desktop pipeline. Live self-analysis is realistic, but unstable. The frozen corpus is the stable middle lane.

## What Makes a Good Mini-Workspace

Use the smallest file set that preserves the behavior you care about:

- Single file when one file is enough
- Mini-workspace when routing, imports, hooks, or framework detection matter
- Preserve relative paths when analyzers depend on location
- Exclude unrelated repo noise

## `fixture.json`

Every corpus item must include:

- identity and provenance
  - `id`
  - `version`
  - `sourceCommit`
  - `sourcePaths`
  - `createdFrom`
- workspace definition
  - `entryFiles`
  - `requiredFiles`
  - `frameworkHints`
- QA expectations
  - `expectedPluginIds`
  - `expectedAnalysisIds`
  - `expectedIssueClasses`
  - `forbiddenIssueClasses`
  - `metricAssertions`
  - `scoreAssertions`
  - `traceAssertions`
  - `desktopAssertions`
- mock-project metadata
  - `mockProjectId`
  - `displayName`
  - `description`
  - `defaultRoute`
  - `uiScenarios`

## Versioning Rules

- Never rewrite an existing corpus item in place.
- New behavior gets a new corpus version.
- Old versions stay committed until intentionally retired.

## Strictness

Frozen corpus expectations should be strict about:

- plugin and analysis invariants
- issue classes
- metric ranges
- score ranges
- Desktop-visible surfaces

They should avoid brittle exact wording when the signal is the class of issue rather than the exact phrasing.
