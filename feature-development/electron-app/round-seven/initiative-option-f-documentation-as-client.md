# Option F — Documentation as a Client

> **SUPERSEDED 2026-05-12** — merged into the consolidated initiative at
> [`greedy-juggling-hamster.md`](./greedy-juggling-hamster.md). Option F
> and Option E (storage normalization) collapsed into one architecture
> per discussion with the founder.
>
> Implementation: Phases 1-3 + 4 + 5 shipped on PR
> [vipr#72](https://github.com/glorioustephan/eephus-vipr/pull/72) on the
> `claude/analyzer-owned-docs` branch.
>
> Historical content preserved below.
>
> ---
>
> Architectural reshape: move `@vipr/documentation` into the
> `clients/` tier as a deployable/buildable artifact, and have
> analyzers OWN per-analysis documentation that flows into the docs
> client at build time. Other clients (marketing, desktop, eventually
> VSCode + CLI) consume the docs client's built artifact rather than
> importing its TypeScript source directly.
>
> Pre-launch foundation work. Sequenced AFTER Option E's storage
> normalization ships.

## Status

📋 Queued — starts after Option E Phase 4 (cleanup). Phase 0 (Option E
consumer audit, 2026-05-11) confirmed the drift surface is physical:
30+ hand-maintained markdown files in `packages/documentation/content/analysis/`
that mirror analyzer code, plus analyzer-side template text with its
own description fields.

## Goal

Three outcomes, ordered by priority:

1. **Single source of truth for per-analyzer documentation.** What an
   analyzer says about itself (description, methodology, examples,
   prompt templates, metric definitions, thresholds) lives in the
   analyzer's source. The docs surface DERIVES from analyzer code,
   not from hand-maintained prose.
2. **Cross-client docs coverage.** VSCode and CLI today have no in-IDE
   /in-terminal docs surface. Option F's manifest pipeline gives them
   one for free.
3. **Independent docs deployment.** The docs client compiles to a
   built artifact (rendered MDX + manifest JSON + nav data). Marketing
   and desktop consume the artifact; the docs site itself can deploy
   independently to `docs.vipr.dev` for a proper launch presence.

## Why now (pre-launch)

The founder's explicit framing: "having a proper documentation site
that is consumable across marketing and desktop would be a great
component of a successful launch." This is not a hedge against future
drift — it's foundation work for v1.

Three forcing reasons that converge:
- The current hand-maintained `content/analysis/*.md` is already
  drifting from analyzer code (concrete cases enumerated in the Option
  E Phase 0 audit)
- VSCode and CLI need a docs surface before users discover an insight
  there and have to context-switch out
- Marketing and desktop currently consume the same package, but their
  needs are diverging (marketing wants long-form SEO content; desktop
  wants contextual in-app help)

## Background — the current shape

`packages/documentation` today:

- 48 markdown files in `content/` across 6 sections (advanced,
  analysis, integration, getting-started, guides, reference)
- A `scripts/build-content.ts` script that compiles MDX → typed
  TypeScript registry (`src/generated/docs-data.ts`)
- Two exports: `.` (browser-safe types + helpers) and `./node`
  (Node-only file ops)
- Consumed by:
  - **Marketing (Astro)**: 9+ files — nav, breadcrumbs, search, layouts,
    per-category pages
  - **Desktop (React renderer)**: 4 files — search modal, analysis
    landing, markdown renderer, navigation lib
  - **VSCode + CLI**: not consumed at all

Analyzers today:

- Each analysis (`analyzers/*/src/analyses/*.ts`) has its own
  `description`, `category`, `version` fields
- Each emits insights with template strings (`message`, `explanation`,
  `detailsMarkdown`, `suggestion`, `prompt.template`)
- 22 analyses across 4 plugins (core, react, typescript, nextjs)
- Linting also has its own plugin surface

The hand-maintained mirror: `content/analysis/react-anti-patterns.md`
(prose) corresponds 1:1 with `analyzers/react/src/analyses/anti-pattern-analysis.ts`
(code). Both describe the same concept. Already drifting.

## Architectural model

```
                                  build time
                                  ──────────
    analyzers/* ────────emit─────► analyzer manifest (JSON/TS)
                                       │
                                       │ consumed at build by
                                       ▼
clients/documentation (Astro/             merge with prose
custom builder)                     ◄────────── content/prose/
    │
    │ produces dist/ with:
    │   - rendered HTML pages
    │   - search index
    │   - typed manifest data
    │
    ├──── consumed at build time by:
    │
    ├───► marketing (links to docs.vipr.dev, OR includes dist/ as a sub-route)
    ├───► desktop (bundles dist/ snapshot for in-app docs)
    ├───► vscode (bundles a subset for hover + sidebar docs)
    └───► cli (bundles compact text-only subset for `vipr help <analysis>`)
```

Key inversions vs today:

- **Source of truth for analysis-specific docs flips** from
  `packages/documentation/content/analysis/*.md` to the analyzer code
  itself
- **`packages/documentation` becomes `clients/documentation`** — a
  buildable artifact, not a TypeScript library that clients import
- **VSCode and CLI gain a docs surface** as a side effect

## What stays in the prose package (not analyzer-derived)

Roughly half the current content:
- `getting-started/` — onboarding, install, first analysis
- `guides/` — how-tos, walkthroughs, IDE integration setup
- `integration/` — Git provider integration, CI/CD setup
- `advanced/` — architecture deep-dives, customization
- `reference/cli-commands.md`, `reference/config-schema.md` — these
  may BECOME analyzer/CLI-derived in time, but staying prose for v1

The `content/analysis/*.md` directory becomes auto-generated; the
hand-written counterpart is the analyzer source code.

## Phases

### Phase F0 — Refined scope audit (~half day)

After Option E Phase 1 ships (the manifest generator). With that
generator in place:

1. Enumerate every analysis (`getAnalyses()` from each plugin)
2. For each, identify what's currently in `content/analysis/<slug>.md`
   vs what's in the analyzer code (`description`, `version`,
   `methodology` if present)
3. Identify the gaps: what does each `.md` file have that the code
   doesn't expose? Promote those fields to the analyzer's source.
4. Identify the orphans: what does the code expose that the `.md`
   doesn't render? Surface those in the docs pages.
5. Decide doc-builder technology (extend the existing
   `packages/documentation/scripts/build-content.ts` vs Astro
   standalone vs Docusaurus). My guess: extend the existing
   compiler — it already handles MDX → registry shape.

Deliverable: a per-analysis migration plan (e.g., "react-anti-patterns:
move 12 paragraphs from .md into analyzer code, expose 4 new fields
in IReportMetadata, render via existing compiler").

### Phase F1 — Extend analyzer-side metadata (~3-5 days)

For each of the 22 analyses:

1. Add a `documentation` field to each analysis's class metadata, OR
   add a `getDocumentation()` method on `IAnalysis`. Shape:
   ```ts
   {
     longDescription: string;   // markdown — for docs site
     methodology: string;       // markdown — "how this analysis works"
     thresholds?: Record<string, { value: number; meaning: string }>;
     examples?: { good: string; bad: string }[];
     references?: { label: string; href: string }[];
   }
   ```
2. Port content from `content/analysis/*.md` into these fields.
3. Extend `Phase 1` (Option E)'s manifest generator to emit
   `analysisDefinitions` populated from these fields.

Constraint: keep the analyzer source readable. If `longDescription`
balloons to 200 lines of markdown, factor it into an adjacent
`.docs.md` file in the analyzer's directory, imported as a string
at build time. (TypeScript can `import doc from './analysis.docs.md?raw'`
with the right Vite plugin.)

### Phase F2 — Build docs client (~1 week)

1. Move `packages/documentation` → `clients/documentation`. Update all
   imports. CI checks workspace integrity.
2. The docs client consumes the analyzer manifest from Option E
   Phase 1, merges in the prose `content/` (now prose-only:
   getting-started, guides, integration, advanced, reference), and
   produces:
   - `dist/static/` — rendered HTML per page (Astro or similar)
   - `dist/manifest.json` — full nav + search index
   - `dist/data/` — typed TypeScript data for clients that prefer
     typed access over rendered HTML
3. Add a content-validation step: every analyzer in
   `analyzerManifest.analysisDefinitions` MUST have a corresponding
   rendered docs page in dist. CI fails if mismatch.

### Phase F3 — Migrate marketing consumption (~half week)

1. Marketing today imports from `@vipr/documentation`. Switch to
   importing from `clients/documentation`'s built artifact.
2. Two options: (a) marketing builds depend on the docs client's
   build (turbo `dependsOn: ["clients/documentation#build"]`), reads
   from `clients/documentation/dist/`; or (b) docs deploys
   independently to `docs.vipr.dev`, marketing links to it.
3. Recommend (a) for v1 (single deployable, simpler), with the option
   to switch to (b) later when traffic / SEO benefits justify
   separation.

### Phase F4 — Migrate desktop consumption (~half week)

1. Desktop today imports from `@vipr/documentation`. Switch to
   bundling a snapshot of `clients/documentation/dist/` at desktop's
   build time.
2. Resolve the offline concern: desktop ALWAYS has the docs because
   they're in its bundle. No network dependency.
3. The in-app docs renderer becomes a static-asset viewer that reads
   from the bundled snapshot.

### Phase F5 — Add VSCode + CLI docs surfaces (~1 week)

1. **VSCode**: hover + sidebar docs. When a user hovers over an
   insight, show a snippet from the analyzer's `longDescription`.
   When they open the sidebar's "Docs" tab, show the full rendered
   page. All from the bundled manifest.
2. **CLI**: `vipr help <analysis-id>` command. Looks up the analysis
   in the manifest, prints `longDescription` formatted for terminal
   (markdown → ANSI).

### Phase F6 — Cleanup (~half day)

- Remove `content/analysis/*.md` (now analyzer-derived)
- Add a CLAUDE.md in `clients/documentation` enforcing prose-only
  `content/` scope going forward (no metric-specific MD files
  allowed)
- Update root CLAUDE.md to reflect the new convention: clients can
  emit build artifacts consumed by other clients
- Final cross-client smoke test: every analysis renders identically
  in marketing, desktop, VSCode, and CLI

## Convention refinement

Root CLAUDE.md should be updated to add:

> **`packages/*`** = shared TypeScript libraries (compile-time
> consumed by clients via imports).
>
> **`clients/*`** = deployable / installable artifacts.
>
> Clients MAY emit build artifacts that other clients consume at
> their respective build time. Clients MUST NOT import TypeScript
> source from another client.

## Risks

| Risk | Mitigation |
|---|---|
| Analyzer source files balloon with embedded markdown | Use adjacent `.docs.md?raw` imports; keep analysis files focused on logic |
| Marketing build complexity increases (depends on docs client) | turbo handles the dep graph; CI validates |
| Desktop bundle size grows (docs snapshot embedded) | Measure before/after; if material, consider a lazy-loaded subset for the in-app docs page |
| VSCode marketplace size grows (embedded docs) | Same — measure; subset if necessary |
| CI complexity for "every analysis has a doc" validation | Single linter pass over the manifest; fast |
| Drift creeps back if discipline slips | Convention refinement in CLAUDE.md; CI check for new `content/analysis/*.md` files |
| Scope creep — turns into a 3-month effort | Phased delivery; phases F0-F6 are independently mergeable; can pause between any two |

## Estimated effort

| Phase | Effort | Cumulative |
|---|---|---|
| F0 (scope audit) | half day | 0.5 day |
| F1 (analyzer-side metadata) | 3-5 days | 4-6 days |
| F2 (docs client) | 1 week | ~2 weeks |
| F3 (marketing migration) | half week | ~2.5 weeks |
| F4 (desktop migration) | half week | ~3 weeks |
| F5 (VSCode + CLI docs surfaces) | 1 week | ~4 weeks |
| F6 (cleanup) | half day | ~4 weeks |

**Total: ~4 weeks of focused work, across many sessions.** Most of
the effort is in Phase F1 (porting content into analyzer code) and
Phase F5 (new docs surfaces for VSCode + CLI).

## Acceptance criteria (for closing the initiative)

- `content/analysis/*.md` removed; every analysis's docs render from
  the analyzer manifest
- Marketing consumes the docs client's build artifact (no
  `@vipr/documentation` import)
- Desktop bundles + renders the docs snapshot for in-app docs
- VSCode shows analyzer documentation in hover + sidebar
- CLI has `vipr help <analysis-id>` working
- CI fails if a new analysis lands without a `getDocumentation()`
  return value
- CI fails if `clients/documentation/content/` contains a file
  matching `*.md` under an analysis-specific name pattern
- Root CLAUDE.md updated with the refined convention

## Out of scope

- A separate `docs.vipr.dev` deployment (deferred until traffic /
  SEO benefits justify it; v1 ships docs as a sub-route under
  marketing)
- Internationalization of docs (English-only for v1)
- User-contributed docs / a community wiki
- Search across the docs site beyond the basic in-package index
  (could revisit with Algolia or pagefind later)
- Analytics / instrumentation specific to docs page views (use
  existing PostHog setup as-is)

## When to start

After Option E Phase 4 cleanup ships and the analyzer manifest is
proven in production. Option F **directly extends** the manifest
generator, so the foundation must be stable first. Earliest start:
2-3 weeks from Option E Phase 1 kickoff.
