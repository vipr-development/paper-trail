# Option E Phase 0 — Consumer Audit of Static Insight Fields

> **SUPERSEDED 2026-05-12** — this audit was completed and informed the
> consolidated plan at [`greedy-juggling-hamster.md`](./greedy-juggling-hamster.md).
> The ~103-file consumer inventory it produced is still accurate and
> remains the reference for Phase 6 wiring work (engine persistence
> boundary, desktop in-app docs, vscode hover/codelens, marketing
> Astro pages, MCP server).
>
> Historical content preserved below.
>
> ---
>
> Read-only catalog of every consumer of the static template fields
> we plan to hoist out of per-row JSON storage. Builds on the audit
> numbers in `initiative-option-e-storage-normalization.md` (130 MB
> DB; 85% of insight bytes are static). Goal: confirm Phase 1 manifest
> schema covers the actual reads, identify migration blast radius,
> flag risks before Phase 2.

## Search methodology

Per-field grep across `clients/`, `packages/`, `analyzers/`. Filtered
out `.test.` files (catalogued separately), `dist/`, `node_modules/`.
For high-noise fields (`message`, `category`, `severity` — used by
non-insight code paths too) I filtered to insight-shaped contexts
(`insight.message`, `insights[`, `insight.category`, etc.).

## Production consumers by field

### `message` (interpolated template)

| File:line | Surface | Read pattern | Notes |
|---|---|---|---|
| `packages/ui/src/components/common/InsightCard.tsx:459` | Shared UI primitive | `insight.message` direct | Renders as the heading text |
| `packages/ui/src/components/common/InsightCard.tsx:447` | Same | `aria-label={insight.message}` | a11y label |
| `packages/ui/src/components/common/InsightsList.tsx:22` | Shared UI list | `insight.message \|\| insight.title \|\| insight.description \|\| 'No description'` | **Three-way fallback chain — flag for Phase 2** |
| `packages/ui/src/utils/export-insights.ts:39` | Markdown export utility | `## ${emoji} Issue ${index}: ${insight.message}` | MCP/CLI bridge for insight export |
| `clients/desktop/src/renderer/components/file-detail/FileIssuesTable.tsx:87` | Issues table cell | `insight.message` direct | |
| `clients/desktop/src/renderer/components/source-drawer/FindingDetailPanel.tsx:43` | Source drawer | `['## What This Finding Is', '', insight.message]` | Composed into markdown sections |
| `clients/desktop/src/renderer/pages/Issues.tsx` | Issues page | search through grep results — uses `insight.message` for filtering / display | |
| `clients/desktop/src/renderer/pages/Overview/components/OverviewTopIssuesTableWidget.tsx` | Overview widget | direct read | |
| `clients/desktop/src/renderer/pages/FileDetail.tsx` | File detail page | direct read | |
| `clients/desktop/src/renderer/pages/Overview/components/IssuesTable.tsx` | Overview issues table | direct read | |
| `clients/desktop/src/main/services/pdf-export-service.ts` | PDF export | reads to build PDF body | Background-process consumer |
| `clients/desktop/src/main/services/function-service.ts` | Function service | analytics / filter | |
| `clients/desktop/src/main/ipc/handlers/database-utils.ts` | DB helpers | reads to construct DB writes | |
| `clients/desktop/src/main/ipc/handlers/analysis-handlers.ts` | Analysis IPC | reads in IPC response shaping | |
| `clients/desktop/src/renderer/lib/ai-prompts/generator.ts` | AI prompt builder | injects message into prompts | |
| `clients/desktop/src/renderer/lib/ai-prompts/prompt-builders.ts` | Same | | |
| `clients/desktop/src/renderer/lib/ai-prompts/templates.ts` | Same | | |
| `clients/desktop/src/renderer/lib/ai-chat/context.ts` | AI chat context | | |
| `clients/desktop/src/renderer/lib/source-drawer/normalize.ts` | Drawer normalization | reads + transforms | |
| `clients/desktop/src/renderer/lib/source-drawer/prompt.ts` | Drawer prompt builder | | |
| `clients/desktop/src/renderer/lib/issues/exclusion-payload.ts` | Issue exclusion | reads for payload construction | |
| `clients/desktop/src/renderer/lib/metric-feedback-subjects.ts` | Metric feedback | builds subject lines | |
| `clients/desktop/src/renderer/hooks/useInsightActions.ts` | Insight action hook | | |
| `clients/desktop/src/renderer/hooks/useAiChatLauncher.ts` | AI chat launcher | | |
| `clients/cli/src/formatters/cli-formatter.ts` | CLI output | renders insight to terminal | |
| `clients/vscode-extension/src/providers/diagnostic-provider.ts` | VSCode diagnostics | shows as VSCode problem | |
| `clients/vscode-extension/src/providers/decoration-provider.ts` | Editor decorations | hover text | |
| `clients/vscode-extension/src/providers/codelens-provider.ts` | Code lens | inline label | |
| `clients/vscode-extension/src/providers/codeaction-provider.ts` | Quick fix | action title | |
| `clients/vscode-extension/src/ai/template-engine.ts` | AI template | injects into prompts | |
| `clients/vscode-extension/src/commands/show-insight-actions.ts` | Action menu | menu label | |
| `clients/vscode-extension/src/commands/fix-with-ai.ts` | AI fix | prompt input | |
| `clients/vscode-extension/src/views/dashboard-provider.ts` | Webview dashboard | renders | |
| `clients/vscode-extension/src/views/dashboard/editor-indicator-manager.ts` | Editor indicator | | |
| `packages/mcp-server/src/db/analysis-queries.ts` | MCP tool surface | reads when returning insights to MCP clients | |
| `packages/common/src/utils/base-analysis.ts` | Common base utils | reads in normalization paths | |
| `packages/engine/src/analysis-engine.ts` | Engine | reads in aggregation | Engine itself produces these — flag for special handling |
| `packages/engine/src/trace-summary.ts` | Trace summary | renders for trace artifact | |
| `analyzers/core/src/plugin.ts` + 3 others | Analyzer plugin entrypoints | reads in `getReportPresenters` flow | |
| `analyzers/react/src/presenters/anti-pattern-presenter-legacy.ts`, `overview-presenter.ts` | React presenters | | |

**Subtotal: ~40 production read sites for `.message`.**

### `explanation` (pure template)

| File:line | Surface | Pattern |
|---|---|---|
| `packages/ui/src/components/common/InsightCard.tsx:498` | Shared UI | `{insight.explanation && <ExplanationSection ... />}` — null-safe |
| `packages/ui/src/utils/export-insights.ts:57-60` | Markdown export | conditional inject |
| `clients/desktop/src/renderer/components/source-drawer/FindingDetailPanel.tsx:44` | Source drawer | `if (insight.explanation) sections.push(...)` |
| `clients/desktop/src/renderer/components/ai/AiChatModal.tsx` | AI chat | | 
| `clients/desktop/src/renderer/hooks/useInsightActions.ts` | Insight actions | |
| `clients/desktop/src/renderer/pages/ActionItems.tsx` | Action items page | |
| `clients/desktop/src/renderer/lib/ai-prompts/{generator,templates}.ts` | AI prompt builders | |
| `clients/desktop/src/renderer/lib/source-drawer/prompt.ts` | Drawer prompt | |
| `clients/desktop/src/main/services/report-service.ts` | Report service | |
| `clients/desktop/src/main/services/anti-pattern-query.ts` | Anti-pattern query | |
| `clients/desktop/src/main/ipc/handlers/database-utils.ts` | DB helpers | |
| `clients/desktop/src/main/ipc/mock/reports-mock-handlers.ts` | Mock handlers | |
| `clients/marketing/src/components/animations/DesktopOverviewDemo/MockInsightCard.tsx` | Marketing demo | Hard-coded mock — fine |
| `clients/vscode-extension/src/providers/decoration-provider.ts` | VSCode hover | |
| `clients/vscode-extension/src/views/dashboard-provider.ts`, `editor-indicator-manager.ts` | VSCode UI | |
| `packages/mcp-server/src/db/analysis-queries.ts` | MCP | |
| `packages/ai/src/schema.ts` | AI schema definitions | |
| `packages/common/src/utils/base-analysis.ts` | Common base | |
| `packages/engine/src/analysis-engine.ts` | Engine | |
| `packages/ui/src/utils/export-insights.ts` | Export util | |
| `analyzers/core/src/plugin.ts`, `nextjs/src/plugin.ts`, `react/src/plugin.ts`, `typescript/src/plugin.ts` | Analyzer plugins | Producers |
| `analyzers/core/src/utils/base-plugin-formatter.ts`, `analyzers/{nextjs,react}/src/presenters/base-presenter.ts` | Presenters | |

**Subtotal: ~28 sites. Most are null-safe (`if (explanation)` or `?.`).**

### `detailsMarkdown` (pure template — largest single field)

| File:line | Surface | Pattern |
|---|---|---|
| `clients/desktop/src/renderer/components/source-drawer/FindingDetailPanel.tsx:125` | Source drawer | `insight.detailsMarkdown ?? fallbackMarkdown` — **null fallback** |
| `clients/desktop/src/renderer/components/anti-patterns/AntiPatternInstanceDetailModal.tsx` | Anti-pattern modal | |
| `clients/desktop/src/renderer/lib/source-drawer/prompt.ts` | Drawer prompt | |
| `clients/desktop/src/main/services/anti-pattern-query.ts` | Anti-pattern query | |
| `clients/desktop/src/main/ipc/handlers/database-utils.ts` | DB helpers | |
| `clients/cli/src/formatters/markdown-formatter.ts` | CLI markdown | |
| `clients/vscode-extension/src/providers/decoration-provider.ts`, `webview/components/issues-list.ts`, `webview/dashboard-app.ts`, `views/dashboard-provider.ts`, `views/dashboard/editor-indicator-manager.ts` | VSCode UI | |
| `packages/ui/src/utils/export-insights.ts:64-67` | Export | conditional |
| `packages/common/src/utils/base-analysis.ts` | Base | |
| `packages/engine/src/analysis-engine.ts` | Engine | |
| `analyzers/core/src/plugin.ts`, `analyzers/{nextjs,react}/src/presenters/base-presenter.ts`, `analyzers/typescript/src/plugin.ts`, `analyzers/core/src/utils/base-plugin-formatter.ts` | Producers | |

**Subtotal: ~21 sites. Universally null-safe.**

### `suggestion` (pure template)

Pattern: similar shape to `explanation`. ~34 sites, predominantly in
CLI formatters (3 files), VSCode providers (multiple), source drawer
panel, and analyzer presenters as producers.

### `prompt` / `prompt.template` (AI prompt template)

Producer side: every analyzer analysis file builds an `AIPromptTemplate`
object with `template` + `variables`. ~20 analyzer files.

Consumer side (reads `.prompt.template` or full prompt):
- `clients/desktop/src/renderer/lib/ai-chat/context.ts` — feeds prompt into AI service
- `clients/desktop/src/renderer/lib/ai-prompts/*` — multiple builders
- `clients/desktop/src/renderer/hooks/useAiChatLauncher.ts` — launches with prompt context
- `clients/desktop/src/main/services/deep-analysis-service.ts` — feeds into deep-analysis pipeline
- `clients/vscode-extension/src/ai/template-engine.ts` — interpolates variables and dispatches

### `category`, `severity` (mostly static per type)

`severity` consumed extensively — `<SeverityBadge severity={insight.severity} />` is the canonical pattern in shared UI. Read in nearly every list/table/filter UI. ~30 insight-context sites.

`category` consumed in `insight.category` form in ~25 insight-context sites; widely used for grouping and filtering.

### `autoFixable`

Narrower, ~14 sites. Drives VSCode "code action" availability (`codeaction-provider.ts`), the analyzer presenters' grouping (`anti-pattern-presenter.ts`), and the CLI's full-JSON output schema.

### `recommendation` (nested structured object)

23 sites. Consumed by:
- `packages/ui/src/components/common/InsightCard.tsx` (Level 4 progressive disclosure)
- `clients/desktop/src/renderer/components/anti-patterns/AntiPatternInstanceDetailModal.tsx`
- `clients/desktop/src/renderer/components/source-drawer/FindingDetailPanel.tsx`
- AI prompt builders (multiple — recommendations seed prompts)
- `packages/ui/src/utils/export-insights.ts:86-97` — exports priority, effort, summary, steps
- Marketing demo (hard-coded mock)
- Plus the analyzer producers

## Test consumers (assertions on message text)

~58 test files contain assertions touching `.message`. Most are Error
messages, not insight-message — `grep -l 'insights:.*message:'`
returned a tighter set:

Estimated **20-30 test files** assert on insight `.message` content.
Concentrated in:
- `clients/desktop/src/renderer/lib/ai-prompts/prompt-builders.test.ts` (prompt strings include message text)
- `clients/desktop/src/renderer/lib/ai-chat/context.test.ts`
- `clients/desktop/src/renderer/pages/file-detail-utils.test.ts`
- `clients/desktop/src/shared/ipc/serialization.test.ts`
- `clients/desktop/src/main/analysis/snapshot-service.test.ts`
- A subset of analyzer test files

**Phase 2 implication:** these tests need to either (a) be updated to
assert by `templateId + vars` rather than rendered text, or (b) opt into
a hydration helper that resolves templateId at test time. The latter is
the smaller blast radius.

## Cross-client coverage

| Client | Reads | Notes |
|---|---|---|
| Desktop renderer | message, explanation, detailsMarkdown, suggestion, severity, category, recommendation, prompt | Heaviest consumer; near every field |
| Desktop main process | message, explanation, detailsMarkdown, severity, category | Background services + IPC handlers |
| `@vipr/ui` shared | message, severity, explanation, recommendation | InsightCard / InsightsList are the canonical primitives |
| `@vipr/ui` export-insights | message, explanation, detailsMarkdown, severity, category, recommendation, all subfields | Used by MCP + future bridges to produce markdown |
| VSCode extension | message, explanation, detailsMarkdown, suggestion, severity, category, autoFixable, prompt | Diagnostics, decorations, codelens, codeactions, AI |
| CLI | message, suggestion, detailsMarkdown, autoFixable | Formatters only; no rich UI |
| MCP server | message, explanation | Narrow surface (`packages/mcp-server/src/db/analysis-queries.ts`) |
| Marketing site | None (only renders prose docs from `@vipr/documentation`, plus a hard-coded `MockInsightCard.tsx` demo) | **Marketing does NOT consume runtime insights.** Confirmed. |
| Analyzer presenters | All fields (producers) | These define the templates; they'll be the source for Phase 1's manifest generator |

## Surprises / risks

1. **`InsightsList.tsx:22` has a three-way fallback chain** —
   `insight.message || insight.title || insight.description || 'No description'`. The `title` and `description` fields aren't on the current `PluginInsight` type. Either they're legacy (some older insight shape), or runtime objects exist that don't strictly match the type. **Investigate before Phase 2** — if there's a legacy shape, the migration needs to handle it.

2. **`anti-pattern-query.ts:1484-1523` uses dynamic property lookup**
   on insight objects: `insight['category']`, `insight['fix']`,
   `insight['recommendation']`, with `||` fallbacks. These are
   invisible to typed grep but real. The code is also tolerating
   multiple shapes (`insight['description'] || insight['message'] || insight['name']`), which suggests the runtime data has historically had inconsistent shape. **Phase 2 will need to either preserve this tolerance or normalize the input shape upstream.**

3. **`finding-narratives.ts` is a producer, not just a consumer.**
   At `packages/common/src/utils/finding-narratives.ts` it GENERATES
   `explanation` and `detailsMarkdown` strings at runtime from
   structured inputs. If we hoist templates, this utility's role
   needs to change — either it produces `vars` to be merged with a
   templateId, or it stays producing rendered text and the templates
   it uses get migrated to the manifest.

4. **Engine itself reads back insight fields.** `analysis-engine.ts`
   reads `.message`, `.explanation`, `.detailsMarkdown` for
   aggregation / pickPreferredFileType / scoring purposes. The engine
   sits on the producer side of the data flow but consumes the
   fields. Phase 2 will need to either keep producing the full
   in-memory shape AND THEN strip-for-persistence (the dual-write
   pattern is set up for this), or have the engine accept
   {templateId, vars} as its result format.

5. **VSCode and CLI consume insight fields but don't consume
   `@vipr/documentation`.** Means insight UI surfaces in those
   clients have no link to long-form docs today. Option F (analyzers
   own the docs) would be the right cross-client fix here.

6. **Marketing's `MockInsightCard.tsx` is hard-coded.** Fine for a
   demo, but worth noting that anything we change in the insight
   schema needs a manual update in that mock OR a regeneration
   pipeline.

7. **`InsightCard.tsx:498-504` renders a `<ExplanationSection ... onToggleRecommendation={insight.recommendation ? toggleLevel4 : undefined} hasRecommendation={!!insight.recommendation}>` pattern.** The `recommendation` field gates a UI affordance (Level 4 progressive disclosure). If our manifest doesn't carry recommendation per-template, that UI affordance breaks. **Recommendation is per-insight-type, not per-instance — confirmed by reading
   the producer side.** So the manifest CAN carry it; just don't
   forget.

## Estimated blast radius (file count per client, Phase 2 changes)

| Client | Production files to update | Test files to update | Total |
|---|---|---|---|
| Desktop renderer | ~25 | ~15 | ~40 |
| Desktop main | ~10 | ~3 | ~13 |
| `@vipr/ui` | 3 | ~3 | ~6 |
| VSCode extension | ~12 | ~3 | ~15 |
| CLI | 4 | ~2 | ~6 |
| MCP server | 2 | 1 | 3 |
| `@vipr/common`, `@vipr/engine` | ~5 | ~5 | ~10 |
| Analyzer plugins + presenters | ~12 (producers) | varies | ~12 |
| Marketing | 0 (only renders prose) | 0 | 0 |
| **Total** | **~73** | **~30** | **~103 files** |

This isn't trivial. But the changes are mechanical (swap field access
for a hydration utility call) and well-typed (TypeScript will catch
omissions). Plan to land Phase 2 across multiple sessions.

## Recommendations for Phase 1 manifest schema

Confirmed Phase 1 needs to carry, per insight template:
- `id` (stable identifier, e.g. `react.anti-pattern.unstable-dependency`)
- `severity` (default; may be overridden per-instance in `vars`)
- `category`
- `message` (interpolated template — handles `{{varName}}` placeholders)
- `explanation` (template; null-safe)
- `detailsMarkdown` (template; null-safe — fallback exists in source drawer)
- `suggestion` (template; null-safe)
- `prompt.template` + `prompt.variables` shape
- `autoFixable` (boolean default)
- `recommendation` (nested structured object with `summary`, `steps`, `effort`, `priority` — all template; null-safe)
- `source` (already redundant with parent `pluginId` — can drop or keep for compatibility)
- `references[]` (URLs — likely template too, found in `analyzers/react/src/analyses/anti-pattern-analysis.ts`)

Schema should be extensible (per the founder's "design for Option F"
guidance):
```ts
export interface AnalyzerManifest {
  insightTemplates: Record<string, InsightTemplate>;   // Phase 1
  metricDefinitions?: Record<string, MetricDefinition>; // Future (Option F)
  analysisDefinitions?: Record<string, AnalysisDefinition>; // Future
  pluginDefinitions?: Record<string, PluginDefinition>;     // Future
}
```

Validators must include a Zod schema mirroring this; clients
hydrate insights at read time via:
```ts
function hydrate(slim: SlimInsight, manifest: AnalyzerManifest): EnrichedInsight {
  const template = manifest.insightTemplates[slim.templateId];
  if (!template) throw new Error(`Missing template ${slim.templateId}`);
  return interpolate(template, slim.vars, slim.location);
}
```

## What Phase 0 has NOT verified

- Exact count of insight TEMPLATES (vs insight instances). Need to
  enumerate every analyzer's `analysisBreakdown` / insight emission
  site to count the manifest's expected key cardinality. Rough
  estimate from analyzer file count + sample inspection: ~150-300
  distinct templates across all analyzers.
- Whether `insight.id` is stable across runs / versions (we'll need
  to enforce this in the generator).
- Whether any analyzer generates insight templates DYNAMICALLY at
  runtime (e.g., concatenates strings to produce the `message`
  template itself). If so, those are NOT hoistable and will need a
  separate path.
- The compiler in `packages/documentation/src/compiler.ts` — same
  build-time-MDX-compiler pattern Option F may want to reuse.

These belong in Phase 1's implementation work, not Phase 0.

## Verdict for Option E Phase 1

**Proceed with C (narrow Phase 1, extensible manifest schema).**
The audit confirms:
- The fields we're hoisting ARE the bloat (75-85% of insight bytes)
- Consumers are well-typed and grep-discoverable (only a few dynamic-
  access sites; flagged for Phase 2 careful handling)
- Marketing's docs path is OUT of scope for Option E (it doesn't read
  runtime insights at all)
- Phase 2 blast radius is ~100 files, mostly mechanical

**Schema should be designed for Option F extension from day one.**
The plan doc already calls this out; the audit confirms there's no
hidden risk in doing so.
