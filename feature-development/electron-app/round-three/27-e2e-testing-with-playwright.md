# Prompt: Playwright E2E Testing Plan for Electron Desktop App

## Role

You are a senior test architect planning end-to-end testing for an Electron desktop application (`clients/desktop/`). Your output is a comprehensive implementation plan that a future engineer can follow without asking clarifying questions.

## Context

The desktop app analyzes source code files for anti-patterns across a spectrum of analyzers. A dedicated fixture set lives in `packages/fixtures/` — sample files representing known anti-patterns that tests should exercise.

## Research Phase

Before writing the plan, study the following references to understand Electron-specific Playwright APIs and best practices:

- https://playwright.dev/docs/api/class-electron
- https://playwright.dev/docs/api/class-electronapplication
- https://www.electronjs.org/docs/latest/tutorial/automated-testing#using-playwright

Synthesize the key patterns (launching `ElectronApplication`, accessing `BrowserWindow`, handling IPC, file-dialog stubbing) and use them as the foundation for the plan.

## Deliverable

Write the implementation plan to:

```
documentation/docs/feature-development/electron-app/round-three/27-e2e-testing-with-playwright.md
```

## Test Directory Structure

Establish the following layout under `clients/desktop/tests/`:

```
clients/desktop/tests/
├── unit/            # Vitest unit tests
├── integration/     # Vitest integration tests
└── e2e/             # Playwright E2E tests
```

## Plan Requirements

The document must cover each of the following areas with enough detail to implement directly:

### 1. Environment & Tooling Setup

- Dependencies to install (Playwright, Electron-specific packages, Vitest)
- `playwright.config.ts` configuration for Electron (no browser download needed)
- Scripts to add to `package.json` (run unit, integration, e2e independently and together)
- Any monorepo/Turborepo considerations for task orchestration

### 2. App Lifecycle Helpers

- Utility to launch the Electron app in test mode pointing at `packages/fixtures/`
- Helpers to stub or automate the native file-picker dialog so tests select fixtures deterministically
- Teardown logic to close the app and clean up temp state between tests

### 3. Branded Types & Test Abstractions

- Define branded/nominal types for domain concepts (e.g., `FixturePath`, `AnalyzerName`, `ReportId`) to catch misuse at compile time
- Page Object Models (or equivalent abstraction) for the key UI surfaces: file picker, analysis runner, sidebar navigation, report viewer
- A shared `TestFixtures` type and factory that enumerates available fixture files and their expected analysis outcomes
- Reusable assertion helpers (e.g., `expectAnalysisComplete()`, `expectReportVisible(reportName)`)

### 4. Core E2E Test Scenarios

Cover a full click-through of the application:

| Scenario                        | Steps                                                                                      |
| ------------------------------- | ------------------------------------------------------------------------------------------ |
| **App Launch**                  | App starts, main window renders, no crash                                                  |
| **Select Fixtures**             | Open file picker → choose `packages/fixtures/` directory → files listed                    |
| **Run Initial Analysis**        | Trigger analysis → progress indicator shown → analysis completes without error             |
| **Sidebar Navigation**          | Each report/analyzer entry in the sidebar is clickable and renders the correct report view |
| **Report Content Verification** | At least one report view asserts meaningful content against expected fixture outcomes      |
| **Error / Empty States**        | Selecting an empty directory or unsupported file shows the correct empty/error state       |

### 5. Unit & Integration Test Guidance

- Describe what belongs in `unit/` vs `integration/` vs `e2e/`
- Provide example test skeletons for at least one unit test (pure function) and one integration test (component + analyzer logic)

### 6. CI / Local Ergonomics

- Instructions to run E2E tests locally with zero internet access (all fixtures are local, no external browser downloads)
- Recommended environment variables or flags for headed vs headless mode
- Guidance on screenshot/video capture on failure for debugging

## Constraints

- **Offline-first**: No internet access required at test runtime. All data comes from `packages/fixtures/`.
- **Idiomatic TypeScript**: All test code should be strict TypeScript. Use branded types and abstractions that make tests self-documenting and hard to misuse.
- **Deterministic**: Tests must not depend on timing, network, or OS-specific behavior. Stub anything non-deterministic.
- **Pragmatic scope**: Focus on the vital structure and the highest-value scenarios first. Flag nice-to-have expansions as "Future Work" rather than bloating the initial plan.

## Success Criteria

The plan is successful if an engineer can follow it to:

1. Set up the test infrastructure from scratch in a single session
2. Run `pnpm test:e2e` (or equivalent) and see Playwright launch the Electron app, select fixtures, run analysis, and navigate reports — all passing
3. Confidently extend the suite by following the established patterns and abstractions without inventing new conventions
