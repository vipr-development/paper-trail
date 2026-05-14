# Vipr Paper Trail

This repository is the historical paper trail behind the Vipr ecosystem.

It began as an `archive/` folder inside the Vipr documentation tree. The archive
has now been collapsed into the repository root so the documents can be read as a
standalone record instead of a buried appendix. The thin archive landing page was
removed; this README is the main index.

The value here is the path, not just the destination. These files show how Vipr
grew from early complexity-analysis research into a broader product ecosystem:
analyzers, CLI workflows, desktop surfaces, VS Code integration, MCP tooling,
release infrastructure, QA loops, and operational guardrails.

There are 384 historical Markdown documents plus this README.

## How To Read This

For the cleanest narrative, start with the early foundations, then move through
the versioned planning tracks:

1. [`foundations/`](foundations/) - early scoring theory, testing priorities,
   accessibility and security analysis, Turborepo research, and barrel-export
   strategy.
2. [`v1/`](v1/) - React analyzer theory, schema work, quick wins, and the first
   implementation roadmap.
3. [`v2/`](v2/) - the shift from a focused analyzer into a platform with CLI,
   desktop, VS Code, QA, configuration, security, and release concerns.
4. [`0.7.0/`](0.7.0/) - a core audit era focused on algorithm correctness,
   maintainability metrics, concurrency, caching, edge cases, and property-based
   testing.
5. [`v3/`](v3/) - plugin architecture refactor planning and the split into
   cleaner analysis units.
6. [`v4/`](v4/) - formatter migration planning.
7. [`feature-development/`](feature-development/) - the large feature tracks
   that expanded Vipr into the desktop, editor, analyzer, and MCP ecosystem.

The documents are intentionally historical. Some package names, paths, product
choices, and assumptions reflect the moment when they were written.

## Narrative Map

| Chapter | Documents | What it captures |
| --- | ---: | --- |
| [`foundations/`](foundations/) | 14 | Early complexity, testing, security, accessibility, package-boundary, and Turborepo research |
| [`v1/`](v1/) | 9 | Initial React analyzer theory, JSON schema, roadmap, research, and enhancement strategy |
| [`v2/`](v2/) | 111 | Platform expansion across CLI, desktop, VS Code, QA, security, configuration, and release operations |
| [`0.7.0/`](0.7.0/) | 17 | Core analyzer audit, correctness validation, maintainability index work, and interactive CLI notes |
| [`v3/`](v3/) | 9 | Plugin system refactor and implementation phases |
| [`v4/`](v4/) | 2 | Formatter migration planning |
| [`feature-development/`](feature-development/) | 228 | Deep feature tracks for analyzers, desktop, VS Code, MCP, and multi-file analysis |
| [`architecture/`](architecture/) | 2 | Historical architecture and plugin architecture references |
| [`guides/`](guides/) | 3 | Migration and plugin-development guides |
| [`vscode-extension/`](vscode-extension/) | 9 | Earlier standalone VS Code extension phase specs |

## Root Tree

```text
.
├── README.md
├── 0.7.0/
│   ├── clacky/
│   └── core-audit/
├── architecture/
├── feature-development/
│   ├── electron-app/
│   ├── in-depth-nextjs-analysis/
│   ├── in-depth-react-analysis/
│   ├── mcp-server/
│   ├── multi-file-analysis/
│   ├── nextjs-analyzer/
│   └── vscode-extension/
├── foundations/
├── guides/
├── v1/
├── v2/
├── v3/
├── v4/
└── vscode-extension/
```

## Foundations

Start here when you want the earliest signals: what Vipr chose to measure, how
testing priorities were framed, and how package boundaries began to matter.

- [`foundations/complexity-analysis-index.md`](foundations/complexity-analysis-index.md)
  is the original index for the complexity-analysis set.
- [`foundations/testing-summary.md`](foundations/testing-summary.md),
  [`foundations/testing-roadmap.md`](foundations/testing-roadmap.md), and
  [`foundations/testing-priority-checklist.md`](foundations/testing-priority-checklist.md)
  describe the early test strategy.
- [`foundations/complexity-matrix.md`](foundations/complexity-matrix.md) and
  [`foundations/complexity-analysis-and-testing-priorities.md`](foundations/complexity-analysis-and-testing-priorities.md)
  explain risk and prioritization.
- [`foundations/security-analyzer-metrics.md`](foundations/security-analyzer-metrics.md)
  and [`foundations/accessibility-analysis-enhancements.md`](foundations/accessibility-analysis-enhancements.md)
  show early analyzer expansion beyond complexity.
- [`foundations/monorepo-barrel-exports-analysis.md`](foundations/monorepo-barrel-exports-analysis.md),
  [`foundations/barrel-export-strategy.md`](foundations/barrel-export-strategy.md),
  and [`foundations/barrel-exports-quick-reference.md`](foundations/barrel-exports-quick-reference.md)
  trace package API and module-boundary cleanup.
- [`foundations/turborepo-clean-research.md`](foundations/turborepo-clean-research.md)
  captures early build-system cleanup research.

## Versioned Evolution

- [`v1/index.md`](v1/index.md) - early React analyzer planning.
- [`v1/theory.md`](v1/theory.md) - theoretical foundation for React complexity.
- [`v1/schema.md`](v1/schema.md) and
  [`v1/schema-implementation.md`](v1/schema-implementation.md) - JSON schema
  documentation and implementation summary.
- [`v1/implementation-roadmap.md`](v1/implementation-roadmap.md) - first roadmap.
- [`v2/index.md`](v2/index.md) - platform-era documentation home.
- [`v2/phase-index.md`](v2/phase-index.md) - the main v2 implementation phase
  map.
- [`v2/architecture.md`](v2/architecture.md) and
  [`v2/plugin-architecture.md`](v2/plugin-architecture.md) - architecture
  references.
- [`v3/index.md`](v3/index.md) and [`v3/summary.md`](v3/summary.md) - plugin
  refactor overview and architecture summary.
- [`v4/index.md`](v4/index.md) and
  [`v4/formatter-migration-plan.md`](v4/formatter-migration-plan.md) - formatter
  migration planning.

## v2 Subsystems

The v2 folder is the broadest platform snapshot. It is best read as a hub with
specialized branches:

- [`v2/cli/`](v2/cli/) - CLI commands, configuration, and usage.
- [`v2/desktop/`](v2/desktop/) - desktop installation, configuration, and feature
  docs.
- [`v2/vscode/`](v2/vscode/) - VS Code installation, user guide, testing, and
  advanced features.
- [`v2/testing/`](v2/testing/) - test coverage, presenter testing, and
  optimization.
- [`v2/qa/`](v2/qa/) - QA harness notes and LOC verification.
- [`v2/security/`](v2/security/) - CSP and desktop security cleanup.
- [`v2/audits/`](v2/audits/) - Electron best-practices audit and implementation
  summary.
- [`v2/research/`](v2/research/) - empirical repository-size research.
- [`v2/infrastructure/`](v2/infrastructure/) - update strategy.
- [`v2/fixes/`](v2/fixes/) - targeted regression and data-drift fixes.

## Feature Development Tracks

The feature-development tree is the largest part of the archive. It records the
major product-growth tracks after the early analyzer work had a shape.

- [`feature-development/electron-app/`](feature-development/electron-app/) - seven
  rounds of desktop product planning, from first user stories through historical
  analysis, velocity intelligence, widgets, backfill performance, release
  setup, and the consolidated perf + storage + analyzer-owned-documentation
  trifecta.
- [`feature-development/in-depth-react-analysis/`](feature-development/in-depth-react-analysis/)
  - research, audits, phased improvements, implementation summaries, and final
  reports for React analyzer depth.
- [`feature-development/nextjs-analyzer/`](feature-development/nextjs-analyzer/)
  and [`feature-development/in-depth-nextjs-analysis/`](feature-development/in-depth-nextjs-analysis/)
  - Next.js detection, server/client analysis, data fetching, migration,
  security, performance, accessibility, and CLI integration.
- [`feature-development/multi-file-analysis/`](feature-development/multi-file-analysis/)
  - batch analysis types, aggregation, presenters, formatters, and interactive
  mode.
- [`feature-development/vscode-extension/`](feature-development/vscode-extension/)
  - the later VS Code implementation track with diagrams, diagnostics, CodeLens,
  dashboards, storage, language model tooling, publishing, and performance.
- [`feature-development/mcp-server/`](feature-development/mcp-server/) - MCP
  server planning, tools, resources, prompts, pipeline integration, and testing.

## Product Surfaces

Use these branches when you are looking for product-facing behavior rather than
platform chronology:

- CLI: [`0.7.0/clacky/`](0.7.0/clacky/), [`v2/cli/`](v2/cli/), and
  [`feature-development/multi-file-analysis/06-interactive-mode.md`](feature-development/multi-file-analysis/06-interactive-mode.md).
- Desktop: [`v2/desktop/`](v2/desktop/) and
  [`feature-development/electron-app/`](feature-development/electron-app/).
- VS Code: [`vscode-extension/`](vscode-extension/),
  [`v2/vscode/`](v2/vscode/), and
  [`feature-development/vscode-extension/`](feature-development/vscode-extension/).
- MCP: [`feature-development/mcp-server/`](feature-development/mcp-server/).
- Analyzer architecture: [`architecture/`](architecture/),
  [`v2/analyzers/`](v2/analyzers/), [`v3/`](v3/), and
  [`feature-development/nextjs-analyzer/`](feature-development/nextjs-analyzer/).

## Operating Lessons

Across the archive, the same development rhythm shows up again and again:

- Turn uncertainty into phases, acceptance criteria, and explicit handoff
  points.
- Keep investigation, implementation, testing, and documentation as distinct
  kinds of work.
- Write guardrails while the system is forming, not after the system is already
  too large to explain.
- Let the CLI, analyzers, desktop app, editor integration, documentation, and
  MCP tooling teach one another.

That is the deeper story in these files. Vipr is a code-analysis ecosystem, but
this repository is also a record of a working practice: make ambition legible,
make change reviewable, and keep the trail clear enough that another mind can
pick it up later without losing the plot.

## Provenance

These Markdown files were extracted from `documentation/docs/archive.zip` in the
Vipr monorepo. macOS metadata files and non-Markdown archive contents were left
out so this repository can focus on the historical documentation itself.
