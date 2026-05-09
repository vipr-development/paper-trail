# Desktop Tracing

## Overview

Vipr Desktop tracing records a source-free execution timeline across:

- analyzer and engine execution in the utility process
- database persistence in the main process
- IPC and preload boundaries
- renderer fetch and derivation work
- UI route and render contracts for priority metric surfaces

Tracing is Desktop-only as a user feature. Shared runtime and analysis instrumentation lives in:

- `packages/tracing`
- `packages/engine`
- `packages/trace-contracts`
- `packages/qa-harness`
- `packages/trace-report`

## User Flow

Tracing is a global **System** setting. It can be armed before a workspace is opened, which allows initial analysis and backfill to be captured from their first traced step.

### System Settings

Open tracing from:

- the welcome screen System Settings modal
- workspace `Settings -> System -> Tracing`
- the Developer Tools quick action for developer-only arming

Two arming modes are available:

- `Record next analysis`: arms tracing and auto-starts on the next qualifying analysis flow
- `Enable manual recording`: arms tracing and waits for the titlebar or welcome-header control to start recording

### Chrome Controls

Shared tracing controls appear in:

- the workspace titlebar
- the welcome header
- the analysis progress modal header

State mapping:

- red `LiveOn` while armed
- red `LiveOff` while recording
- download icon when the newest archive is ready

Click behavior:

- while `next-analysis` is armed, clicking cancels the pending arm
- while `manual` is armed, clicking starts recording
- while recording, clicking stops recording
- while ready, clicking downloads the newest archive

Recording behavior:

- standard recordings stop automatically after 3 minutes
- traces started by `initial-analysis` or `recovery-analysis` can switch into `await-backfill`
- backfill-triggered traces start in `await-backfill`
- `await-backfill` recordings continue until backfill completes, fails, is cancelled, the user stops them, or the 30-minute hard cap is reached

## Storage and Retention

Trace data is stored under Electron app data, never in the repository:

```text
<userData>/traces/
  active/<sessionId>/
  archives/<archiveId>.zip
  archives/<archiveId>.json
  latest.zip
  latest.json
```

Retention policy:

- purge archives older than 72 hours
- then keep at most 5 archives
- then keep total retained archive size under 250 MB

Purge runs:

- on app start
- after archive finalization
- after explicit archive deletion

## Archive Format

Desktop trace ZIPs retain the raw trace data and include a generated investigation report:

```text
manifest.json
timeline.ndjson
main/*.ndjson
utility/*.ndjson
preload/*.ndjson
renderer/*.ndjson
investigation/
  index.html
  report.js
  report.css
  report-data.js
  vipr-icon.svg
  fonts/inter/*.woff2
  summary.json
  audit-findings.json
  normalized-trace.json
  flow-summaries.json
  investigation-summary.md
  agent-prompts.json
  agent-prompts.md
  source-verification.json      # optional
  baseline-diff.json            # optional
```

The report is file://-safe. Unzip the archive and open `investigation/index.html` directly in a browser.

## What Is Captured

### Upstream analysis truth

- file path and content hash
- selected plugins and executed analyses
- raw metric summaries
- composite score inputs and outputs
- cache hits, warnings, failures, and timing

### Desktop pipeline continuity

- database query summaries for priority surfaces
- main-process IPC contract summaries
- preload validation and contract summaries
- renderer fetch and derive contract summaries
- UI render contract summaries

### Route and flow context

- route and hook names
- renderer request IDs for aggregate metric queries
- analysis correlation IDs for file-analysis flows

## What Is Not Captured

The trace archive is intentionally source-free. It does not contain:

- raw source code
- code snippets
- raw AST payloads
- unbounded plugin result blobs
- unnecessary absolute machine paths

If source code is needed for diagnosis, pair it separately with the trace or with the generated investigation bundle.

## Continuity Contracts

Priority Desktop surfaces emit bounded contracts at these stages:

- `db-query`
- `ipc-main`
- `ipc-preload`
- `renderer-fetch`
- `renderer-derive`
- `ui-render`

Always-on priority surfaces:

- `dashboard.summary`
- `overview.topIssues`
- `overview.issueDistribution`
- `overview.complexityDistribution`

Visited-page surfaces:

- `velocity.trend`
- `velocity.metricBreakdown`
- `velocity.leaderboard`
- `velocity.inflectionPoints`
- `velocity.buckets`
- `velocity.projection`
- `churn.quadrant`
- `churn.toxicFiles`
- `file.detail.live`
- `file.detail.history`
- `file.detail.trend`

Root cause is assigned by first divergence between adjacent complete stages:

- DB differs first: `persistence-divergence`
- main/preload differs first: `ipc-divergence`
- renderer fetch/derive differs first: `renderer-divergence`
- UI render differs first: `ui-divergence`
- missing required stages: `trace-gap`

Counts, IDs, labels, and pass-through values compare exactly. Tolerances are used only for explicitly recomputed or rounded aggregates.

See [metric-continuity-contracts.md](./metric-continuity-contracts.md) for the phase 1 contract definitions.

## Investigation Report

Every Desktop archive and CLI investigation bundle includes a static report with:

- findings-first triage dashboard
- Vipr-themed offline styling with bundled fonts and local assets
- agent handoff panel with copyable targeted prompts
- continuity tab for surface-level blame assessment
- searchable findings and files tables
- flow and coverage views
- optional comparison output when source or baseline data is supplied

The preferred handoff artifact for humans and LLMs is:

- `investigation/investigation-summary.md`
- `investigation/agent-prompts.md`

That summary stays compact and source-free while pointing to the most suspicious surfaces, files, and stages.

## CLI Intake

Trace ZIPs can be analyzed directly without unpacking raw NDJSON by hand.

Commands:

- `pnpm vipr:trace inspect <zip-path>`
- `pnpm vipr:trace audit <zip-path>`
- `pnpm vipr:trace compare --trace <zip-path> --source <workspace-or-file>`
- `pnpm vipr:trace diff --trace <zip-path> --baseline <normalized-json>`
- `pnpm vipr:trace bundle --trace <zip-path> [--source <path>] [--baseline <path>]`

Recommended local directories:

- inbox: `./.vipr/trace-inbox/`
- generated reports: `./.vipr/trace-reports/`

`trace bundle` produces the same HTML report format used by Desktop and prints the preview path.

## Operational Notes

- armed state is session-only
- if the app exits during recording, Desktop finalizes a partial archive with stop reason `app-exit`
- stale active sessions are recovered into a completed archive on next launch
- Desktop flushes preload and renderer bridge buffers before final ZIP creation
- download uses Save As copy semantics from internal app data

## Privacy Boundary

The design intentionally separates:

- trace archive: execution path and derived data continuity
- source code: external input supplied only when needed

That keeps stored traces safer while still supporting differential testing and LLM-assisted diagnosis.

## Feeding Traces into the Frozen Corpus Workflow

Once a trace identifies behavior worth preserving as a regression guard:

```bash
pnpm vipr:trace bundle --trace /absolute/path/to/trace.zip --source /absolute/path/to/workspace
pnpm vipr:qa promote --trace /absolute/path/to/trace.zip --source /absolute/path/to/workspace
```

That promotion bundle is the review artifact you use to decide whether a new frozen corpus mini-workspace should be committed under `packages/fixtures/src/desktop-corpus/`.

After the corpus item is committed:

```bash
pnpm vipr:qa run --fixture <fixture-id>
pnpm vipr:qa compile --fixture <fixture-id>
```

The compiled artifact then powers both QA assertions and Desktop mock-project previews.
