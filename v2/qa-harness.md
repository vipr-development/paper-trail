# QA Harness

`@vipr/qa-harness` now has two responsibilities:

1. Trace intake and comparison via `vipr:trace`
2. Fixture and frozen-corpus execution via `vipr:qa`

## Lanes

| Lane            | Purpose                                                                                         | Strictness                                                 |
| --------------- | ----------------------------------------------------------------------------------------------- | ---------------------------------------------------------- |
| `synthetic`     | Existing analyzer fixtures in `packages/fixtures/src/{core,react,nextjs,javascript,typescript}` | Exact analyzer assertions remain in analyzer package tests |
| `frozen-corpus` | Curated mini-workspaces in `packages/fixtures/src/desktop-corpus`                               | Invariant and range based                                  |
| `live-canary`   | Reserved for self-host and trace drift checks                                                   | Tolerance only, no committed oracle                        |

## Commands

```bash
pnpm vipr:qa list
pnpm vipr:qa run --fixture table-workspace
pnpm vipr:qa run --lane frozen-corpus
pnpm vipr:qa run --fixture table-workspace --with-trace /absolute/path/to/trace.zip
pnpm vipr:qa compile --fixture table-workspace
pnpm vipr:qa mock-projects
pnpm vipr:qa explain --fixture table-workspace
pnpm vipr:qa promote --trace /absolute/path/to/trace.zip --source /absolute/path/to/workspace
```

## Output Locations

- Trace reports: `.vipr/trace-reports/`
- QA reports: `.vipr/qa-reports/`
- Promotion bundles: `.vipr/qa-reports/<timestamp>-promotion-*/`

## Reading Failures

`vipr:qa run` returns assertion records with:

- `scope`
  - `analysis`
  - `metric`
  - `score`
  - `desktop`
  - `trace`
- `message`
- optional `details`

Common examples:

- Missing plugin or analysis IDs: analyzer routing drift
- Metric range failure: algorithmic drift or changed normalization
- Missing derived surface: Desktop mock artifact or continuity gap
- Trace-scoped desktop assertion failure: UI/IPC continuity regression

## Typical Workflow

1. Capture a Desktop trace when you see suspicious behavior.
2. Run `pnpm vipr:trace bundle --trace <zip> --source <workspace>`.
3. Decide whether the issue should become a permanent regression guard.
4. If yes, create a promotion bundle:

```bash
pnpm vipr:qa promote --trace <zip> --source /absolute/path/to/workspace
```

5. Review the proposed mini-workspace and `fixture.json`.
6. Commit the raw workspace slice under `packages/fixtures/src/desktop-corpus/<id>-v1/`.
7. Run:

```bash
pnpm vipr:qa run --fixture <id>
pnpm vipr:qa compile --fixture <id>
```

8. Open Desktop in mock mode and select the generated mock project to inspect the post-analysis UI.
