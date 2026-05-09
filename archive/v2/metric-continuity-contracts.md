# Metric Continuity Contracts

## Purpose

Metric continuity contracts define what Desktop should preserve, recompute, and render for priority surfaces so traces can prove where a user-visible value first diverged.

The initial rollout focuses on Dashboard and Overview, then extends the same contract model to Velocity, Churn, and File Detail when those pages are visited during a trace.

## Contract Stages

Every phase 1 surface can emit a bounded, source-free contract at:

1. `db-query`
2. `ipc-main`
3. `ipc-preload`
4. `renderer-fetch`
5. `renderer-derive`
6. `ui-render`

Each contract includes:

- `surfaceId`
- `requestId`
- `stageId`
- route and hook/component context
- a bounded semantic payload
- a deterministic digest of that payload

## Comparison Modes

### Pass-through exact

Used when the value should survive the stage boundary unchanged.

Examples:

- DB result carried through main IPC
- preload validation preserving the payload shape
- renderer fetch state mirroring the transport payload

### Derived exact

Used when the next stage intentionally reshapes data but the result is deterministic and should match exactly.

Examples:

- severity bucket counts derived from an issue list
- stable ID lists or label lists

### Derived tolerant

Used only when a stage intentionally rounds or recomputes aggregate display values.

Current default tolerances:

- one-decimal averages: `±0.1`
- integer-rounded aggregates: `±0.5`
- ratios and percentages: `±0.5 percentage points`

No tolerance is allowed across main/preload IPC pass-through stages.

## Always-On Surfaces

## `dashboard.summary`

### Source of truth

- main DB handler: `db:getDashboardSummary`
- renderer hook: `useDashboardSummary`
- UI surface: dashboard/overview summary cards

### Expected payload shape

- `healthScore`
- `totalFiles`
- `criticalIssues`
- `avgComplexity`
- `trend`
- bounded optional summaries:
  - distribution
  - outlier files
  - pareto summary
  - plugin breakdown

### Rules

- counts and score values must remain exact across DB, IPC, preload, and renderer fetch
- renderer derive may only normalize optional fields, not change the core metrics
- UI render must reflect the derived dashboard summary, not a stale prior value

## `overview.topIssues`

### Source of truth

- main DB handler: `db:getTopIssues`
- renderer hook: `useOverviewData`
- UI surface: Overview top issues list/panel

### Expected payload shape

- bounded list count
- stable IDs where present
- severity/category distribution
- short preview metadata only

### Rules

- list count and severity/category distribution must remain exact across DB, IPC, preload, and renderer fetch
- renderer derive should not drop or reorder entries unexpectedly in the phase 1 hook contract
- UI render must match the derived top-issues contract

## `overview.issueDistribution`

### Source of truth

- main DB handler: `db:getIssueList`
- renderer hook: `useOverviewData`
- UI surface: Overview issue distribution card

### Expected payload shape

- source stage stores bounded issue list summary, not the full raw list
- derived stages store:
  - `critical`
  - `warning`
  - `info`
  - total count

### Rules

- severity regrouping is a `derived-exact` transformation
- the sum of derived buckets must equal the input issue count
- UI render must reflect the derived bucket counts exactly

## `overview.complexityDistribution`

### Source of truth

- main DB handler: `db:getComplexityDistribution`
- renderer hook: `useOverviewData`
- UI surface: Overview complexity distribution card

### Expected payload shape

- labels
- values
- bounded optional arrays:
  - `avgScores`
  - `avgComplexity`
  - `issueCounts`

### Rules

- labels and values are exact through DB, IPC, preload, and renderer fetch
- renderer derive may normalize optional arrays but must not mutate the bucket values
- UI render must reflect the derived distribution exactly

## Visited-Page Surfaces

These surfaces only appear in the normalized report and generated prompts when the user actually visits the page while tracing is active.

### Velocity

- `velocity.trend`
- `velocity.metricBreakdown`
- `velocity.leaderboard`
- `velocity.inflectionPoints`
- `velocity.buckets`
- `velocity.projection`

Rules:

- API-returned trend, leaderboard, breakdown, and inflection data are pass-through exact through DB, IPC, preload, and renderer fetch
- bucket and projection summaries are renderer-derived and compare exactly except for explicitly rounded display values

### Churn

- `churn.quadrant`
- `churn.toxicFiles`

Rules:

- quadrant totals, thresholds, and file counts are exact through DB, IPC, preload, and renderer fetch
- toxic-files table derivation is exact for selected file identity, filtered counts, and visible summary values

### File Detail

- `file.detail.live`
- `file.detail.history`
- `file.detail.trend`

Rules:

- live file detail now uses an aggregated internal continuity query so DB, IPC, preload, renderer, and UI stages can be compared end to end
- history and trend surfaces remain exact for file identity, score values, issue counts, and point counts
- panel and page context are included so the report can attribute mismatches to the active live/history/trend view

## Blame Assignment

The first adjacent divergence with complete evidence determines the likely fault domain:

- `db-query` -> `ipc-main`: `persistence-divergence`
- `ipc-main` -> `ipc-preload`: `ipc-divergence`
- `ipc-preload` -> `renderer-fetch` or `renderer-derive`: `renderer-divergence`
- `renderer-derive` -> `ui-render`: `ui-divergence`

If a required stage is missing, the result is `trace-gap` until more evidence exists.

Confidence levels:

- `high`: adjacent complete stages with explicit field mismatches
- `medium`: partial or digest-only adjacent evidence
- `low`: likely blame inferred from missing downstream evidence

## Bounded Payload Rules

Contracts must remain source-free and bounded.

Allowed:

- counts
- IDs
- labels
- distributions
- bounded previews
- deterministic digests

Disallowed:

- source text
- snippets
- raw AST data
- unbounded issue lists
- unbounded query result payloads

## Phase Expansion

Next planned surfaces:

- hotspots list
- adaptive hotspots
- directory velocities and heatmaps
- multi-metric trend overlays
