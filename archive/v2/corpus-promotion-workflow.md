# Corpus Promotion Workflow

## Goal

Turn a real trace or self-host issue into a committed frozen corpus item and a generated mock project.

## Step 1: Investigate the Real Issue

Capture a trace and generate the investigation bundle:

```bash
pnpm vipr:trace bundle --trace /absolute/path/to/trace.zip --source /absolute/path/to/workspace
```

Use:

- `investigation/index.html`
- `investigation/investigation-summary.md`
- `investigation/agent-prompts.md`

## Step 2: Decide Whether It Should Become a Corpus Guard

Promote when the issue is:

- realistic
- likely to regress
- visible in Desktop UI
- useful for future analyzer or continuity work

## Step 3: Generate a Promotion Bundle

```bash
pnpm vipr:qa promote --trace /absolute/path/to/trace.zip --source /absolute/path/to/workspace
```

This writes a review bundle under `.vipr/qa-reports/` with:

- `workspace/` copied file slice
- `fixture.json` proposal
- `promotion-summary.md`

## Step 4: Review the Candidate

Check:

- Are the copied files the smallest useful workspace?
- Does the proposed `fixture.json` need tighter expectations?
- Should this become a new version of an existing corpus item instead of a new ID?

## Step 5: Commit the Raw Source of Truth

Move the reviewed bundle into:

```text
packages/fixtures/src/desktop-corpus/<id>-v1/
```

Commit only:

- `workspace/`
- `fixture.json`
- optional `README.md`

Do not commit generated artifacts from `.vipr/qa-reports/`.

## Step 6: Compile and Run It

```bash
pnpm vipr:qa run --fixture <id>
pnpm vipr:qa compile --fixture <id>
pnpm vipr:qa mock-projects
```

## Step 7: Validate the UI

Enable mock mode in Desktop Developer Tools, select the generated mock project, and inspect the post-analysis UI using the compiled corpus-backed data.
