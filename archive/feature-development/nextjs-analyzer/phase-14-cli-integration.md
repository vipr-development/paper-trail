# Phase 14: CLI Integration & Documentation

## Status

Not Started

## Goals

Integrate the Next.js analyzer plugin with the CLI plugin loader, create comprehensive user and developer documentation, and update workspace configuration to ensure the plugin is production-ready and discoverable.

## Files Created

### CLI Integration Files

- Update `clients/cli/src/plugins/loader.ts` - Add Next.js plugin discovery
- Update `clients/cli/src/formatters/json-formatter.ts` - Ensure Next.js reports handled
- Update `clients/cli/src/formatters/markdown-formatter.ts` - Format Next.js reports
- Update `clients/cli/src/formatters/terminal-formatter.ts` - Display Next.js reports
- `clients/cli/src/plugins/loader.test.ts` - Update tests for Next.js plugin

### User Documentation Files

- `documentation/docs/analyzers/nextjs-analyzer.md` - User guide
- `documentation/docs/analyzers/nextjs-reports.md` - Report types reference
- `documentation/docs/analyzers/nextjs-configuration.md` - Configuration guide
- `documentation/docs/analyzers/nextjs-examples.md` - Usage examples

### Developer Documentation Files

- Development guide (to be added in `analyzers/nextjs/README.md`)
- `analyzers/nextjs/ARCHITECTURE.md` - Architecture overview
- `analyzers/nextjs/CONTRIBUTING.md` - Contributing guide
- `analyzers/nextjs/docs/adding-analyses.md` - How to add new analyses
- `analyzers/nextjs/docs/testing.md` - Testing guide

### Workspace Configuration Updates

- Update `package.json` - Add nextjs analyzer scripts
- Update `turbo.json` - Add nextjs analyzer to pipeline
- Update `README.md` - Document Next.js analyzer
- Update `CHANGELOG.md` - Add Next.js analyzer release notes

## Implementation Details

### CLI Plugin Loader Integration

Update `clients/cli/src/plugins/loader.ts`:

```typescript
/**
 * Discover plugins bundled with the CLI
 */
private async discoverBundledPlugins(): Promise<ITechnologyPlugin[]> {
  const plugins: ITechnologyPlugin[] = [];

  // Import React analyzer
  try {
    const { ReactAnalyzerPlugin } = await import('@vipr/react');
    plugins.push(new ReactAnalyzerPlugin());
  } catch (error) {
    logger.warn(
      'React analyzer not available:',
      error instanceof Error ? error.message : String(error)
    );
  }

  // Import Core analyzer
  try {
    const { CoreAnalyzerPlugin } = await import('@vipr/core');
    plugins.push(new CoreAnalyzerPlugin());
  } catch (error) {
    logger.warn(
      'Core analyzer not available:',
      error instanceof Error ? error.message : String(error)
    );
  }

  // Import Next.js analyzer
  try {
    const { NextjsAnalyzerPlugin } = await import('@vipr/nextjs');
    plugins.push(new NextjsAnalyzerPlugin());
  } catch (error) {
    logger.warn(
      'Next.js analyzer not available:',
      error instanceof Error ? error.message : String(error)
    );
  }

  return plugins;
}
```

### User Documentation

#### Main User Guide (`documentation/docs/analyzers/nextjs-analyzer.md`)

````markdown
# Next.js Analyzer

The Next.js analyzer plugin provides comprehensive static analysis of Next.js applications, detecting issues across App Router, Pages Router, Server Components, Client Components, data fetching patterns, and migration readiness.

## Features

- **Router Analysis**: Detects App Router and Pages Router patterns, including coexistence
- **Server/Client Boundaries**: Identifies directive placement issues and serialization problems
- **Data Fetching**: Flags anti-patterns in getServerSideProps, getStaticProps, fetch usage
- **Migration Readiness**: Detects breaking changes across Next.js versions 12-15
- **Security**: Identifies Server Action vulnerabilities, environment variable exposure
- **Performance**: Detects bundle bloat, hydration issues, caching problems
- **Configuration**: Validates next.config.js for deprecated and removed options
- **Component Patterns**: Analyzes next/image, next/link, next/script usage
- **Accessibility**: Checks Next.js-specific a11y patterns
- **TypeScript**: Validates Next.js-specific type patterns and async types

## Installation

The Next.js analyzer is bundled with the CLI:

```bash
npm install -g @vipr/cli
```
````

## Usage

### Basic Analysis

Analyze a Next.js application:

```bash
vipr analyze ./src --plugins nextjs
```

### Specific Reports

Generate specific report types:

```bash
# Overview report
vipr analyze ./app --plugins nextjs --report nextjs-overview

# Server/Client component issues
vipr analyze ./app --plugins nextjs --report server-client

# Migration readiness
vipr analyze ./app --plugins nextjs --report migration
```

### Output Formats

```bash
# Terminal output (default)
vipr analyze ./app --plugins nextjs

# JSON output
vipr analyze ./app --plugins nextjs --format json > report.json

# Markdown output
vipr analyze ./app --plugins nextjs --format markdown > REPORT.md
```

## Report Types

The Next.js analyzer provides 9 report types:

| Report Type       | Description                              | Icon |
| ----------------- | ---------------------------------------- | ---- |
| `nextjs-overview` | Composite score and dimensional analysis | ▲    |
| `server-client`   | Server/Client component boundary issues  | 🔄   |
| `data-fetching`   | Data fetching anti-patterns              | 📊   |
| `migration`       | Migration readiness and breaking changes | ⚡   |
| `security`        | Security vulnerabilities                 | 🔒   |
| `config`          | Configuration issues                     | ⚙️   |
| `component`       | Component anti-patterns                  | 🧩   |
| `performance`     | Performance issues                       | ⚡   |
| `a11y-typescript` | Accessibility and TypeScript issues      | ♿   |

**Note:** Report types reference documentation will be added in a future phase.

## Configuration

Create a `.viprrc.json` configuration file:

```json
{
  "plugins": {
    "nextjs": {
      "enabled": true,
      "targetVersion": "15",
      "analyses": {
        "nextjs-server-client": { "enabled": true },
        "nextjs-data-fetching": { "enabled": true },
        "nextjs-migration": {
          "enabled": true,
          "config": {
            "targetVersion": "15",
            "detectBreakingChanges": true
          }
        },
        "nextjs-security": { "enabled": true },
        "nextjs-performance": { "enabled": true }
      }
    }
  }
}
```

**Note:** Configuration guide documentation will be added in a future phase.

## Common Use Cases

### Pre-Migration Analysis

Before upgrading Next.js versions:

```bash
vipr analyze ./app --plugins nextjs --report migration --format markdown
```

Review blockers and warnings before proceeding with upgrade.

### CI/CD Integration

Add to your CI pipeline:

```yaml
- name: Analyze Next.js Code
  run: |
    npm install -g @vipr/cli
    vipr analyze ./app --plugins nextjs --format json > nextjs-report.json

- name: Check Score Threshold
  run: |
    SCORE=$(jq '.score' nextjs-report.json)
    if [ "$SCORE" -lt 70 ]; then
      echo "Next.js score too low: $SCORE"
      exit 1
    fi
```

### Security Audit

Focus on security issues:

```bash
vipr analyze ./app --plugins nextjs --report security --severity critical,high
```

### Performance Review

Identify performance bottlenecks:

```bash
vipr analyze ./app --plugins nextjs --report performance
```

## Examples

**Note:** Usage examples documentation will be added in a future phase.

## Troubleshooting

### Plugin Not Loading

Ensure Next.js plugin is available:

```bash
vipr analyze --list-plugins
```

Should show `nextjs` in the list.

### No Issues Detected

Verify files match Next.js patterns:

```bash
vipr analyze ./app --plugins nextjs --verbose
```

### Performance Issues

For large codebases, analyze incrementally:

```bash
vipr analyze ./app/components --plugins nextjs
vipr analyze ./app/api --plugins nextjs
```

## Further Reading

**Note:** Additional documentation for Next.js analyzer reports, configuration, and examples will be added in future phases.

````

#### Report Types Reference (`documentation/docs/analyzers/nextjs-reports.md`)

```markdown
# Next.js Report Types Reference

Complete reference for all Next.js analyzer report types.

## Overview Report (`nextjs-overview`)

**Purpose**: High-level summary of Next.js application health.

**Sections**:
- Overall Next.js Health Score
- Router Configuration (App/Pages/Mixed)
- Analysis Dimensions (scores for each category)
- Top 10 Issues (sorted by severity)

**When to use**: First analysis, dashboards, quick health check.

**Example Output**:
````

Overall Next.js Health: 72/100

Router Configuration:

- Router Type: app
- Coexistence: No

Analysis Dimensions:

- Server/Client Boundaries: 85/100 (3 issues)
- Data Fetching: 68/100 (5 issues)
- Migration Readiness: 90/100 (1 blocker)
- Security: 75/100 (2 vulnerabilities)
- Performance: 80/100 (4 issues)

```

## Server/Client Components (`server-client`)

**Purpose**: Identifies Server/Client component boundary issues.

**Detects**:
- "use client" directive placement errors
- "use server" directive placement errors
- Unnecessary client components
- Client hooks in Server Components
- Non-serializable props crossing boundaries
- Missing directives for hook usage

**Severity Levels**:
- **Critical**: Both directives in same file
- **Error**: Client hooks without "use client"
- **Warning**: Unnecessary "use client"
- **Info**: Optimization opportunities

**Example Issue**:
```

[ERROR] Client hooks used without "use client" directive
Line 15: app/components/Counter.tsx
Description: Component uses useState but missing "use client" directive
Recommendation: Add "use client" at the top of the file

```

## Data Fetching (`data-fetching`)

**Purpose**: Detects data fetching anti-patterns.

**App Router Checks**:
- fetch() without cache options
- Fetching own API routes from Server Components
- Server Actions without revalidation
- Missing error boundaries for async components

**Pages Router Checks**:
- getInitialProps usage (blocks static optimization)
- getStaticProps without getStaticPaths on dynamic routes
- Client-side fetch when SSR available

**Example Issue**:
```

[WARNING] fetch() call without cache option
Line 23: app/page.tsx
Description: Fetch call missing cache directive (behavior changed in Next.js 15)
Recommendation: Add cache: 'force-cache' or cache: 'no-store'

```

## Migration Readiness (`migration`)

**Purpose**: Assesses readiness for Next.js version upgrades.

**Detects**:
- Breaking changes (Next.js 12→13, 13→14, 14→15)
- Deprecated patterns
- Removed APIs
- Required codemods

**Breaking Changes by Version**:

**Next.js 15**:
- Async params/searchParams
- Async cookies/headers
- fetch() cache behavior
- useFormState → useActionState

**Next.js 13**:
- Image component API changes
- Link component no longer requires <a>
- Font optimization package changes

**Example Output**:
```

Migration Readiness: 65/100

Blockers (2):

- [CRITICAL] Synchronous params type in app/[id]/page.tsx
- [CRITICAL] Synchronous cookies() access in app/actions.ts

Warnings (5):

- [WARNING] Using next/router in app directory
- [WARNING] Deprecated images.domains in next.config.js

```

## Security (`security`)

**Purpose**: Identifies security vulnerabilities.

**Detects**:
- Server Actions without authentication
- Server Actions without input validation
- Environment variable exposure (NEXT_PUBLIC_)
- CVE-2025-29927 middleware bypass vulnerability
- XSS vectors in dynamic content

**Example Issue**:
```

[CRITICAL] Server Action missing authentication
Line 45: app/actions/deleteUser.ts
Description: Server Action performs privileged operation without auth check
Recommendation: Add authentication check at function start
CWE: CWE-306 (Missing Authentication)

```

## Configuration (`config`)

**Purpose**: Validates next.config.js.

**Detects**:
- Removed options (swcMinify, target)
- Deprecated options (images.domains, experimental.appDir)
- Invalid matcher patterns
- Incorrect HTTP method exports
- Wrong generateStaticParams return type

**Example Issue**:
```

[ERROR] Removed configuration option
File: next.config.js
Description: swcMinify option removed in Next.js 15
Recommendation: Remove this option (SWC is now always used)

```

## Component Patterns (`component`)

**Purpose**: Analyzes Next.js component usage.

**Detects**:
- next/image issues (missing dimensions, deprecated layout prop)
- next/link issues (unnecessary <a>, legacyBehavior)
- next/script issues (beforeInteractive placement, missing id)
- Plain <a> tags for internal navigation
- Missing priority on hero images

**Example Issue**:
```

[WARNING] Image missing priority attribute
Line 12: app/components/Hero.tsx
Description: Above-the-fold image without priority delays LCP
Recommendation: Add priority prop to Image component

```

## Performance (`performance`)

**Purpose**: Identifies performance issues.

**Detects**:
- Bundle bloat (barrel imports, full lodash)
- Hydration mismatch causes
- Incorrect caching strategies
- force-dynamic when not needed
- Database queries without caching

**Example Issue**:
```

[WARNING] Barrel file import detected
Line 5: app/page.tsx
Description: Importing from @/components may include entire barrel
Recommendation: Import directly from specific component file
Impact: Potential +50KB bundle increase

```

## Accessibility & TypeScript (`a11y-typescript`)

**Purpose**: Checks accessibility and TypeScript patterns.

**Accessibility Checks**:
- Missing lang attribute in _document
- Missing alt on next/image
- Redundant alt text ("image of...")
- Decorative images without empty alt

**TypeScript Checks**:
- Untyped getServerSideProps/getStaticProps
- Synchronous params type (Next.js 15 breaking change)
- Missing NextRequest/NextResponse types
- Untyped Server Actions

**Example Issues**:
```

[ERROR] Missing alt attribute on Image
Line 34: app/page.tsx
WCAG: 1.1.1 (Non-text Content)
Recommendation: Add alt prop with descriptive text

[ERROR] Synchronous params type (Next.js 15 breaking)
Line 8: app/[id]/page.tsx
Description: params must be Promise in Next.js 15+
Recommendation: Change type to: params: Promise&lt;&#123; id: string &#125;&gt;

```

## Report Selection Guide

Choose reports based on your goal:

| Goal | Reports |
|------|---------|
| Initial assessment | nextjs-overview |
| Pre-upgrade check | migration |
| Security audit | security |
| Performance review | performance, data-fetching |
| Code review | server-client, component |
| Accessibility audit | a11y-typescript |
| Config validation | config |

## Severity Interpretation

All reports use consistent severity levels:

- **Critical**: Runtime errors, security vulnerabilities, breaking changes
- **High/Error**: Likely to cause issues, violates best practices
- **Medium/Warning**: May cause issues, should be addressed
- **Low/Info**: Optimization opportunities, suggestions
```

### Developer Documentation

#### Development Guide

**Note:** This guide will be added to `analyzers/nextjs/README.md` in a future phase.

```markdown
# Next.js Analyzer - Developer Guide

Development guide for contributors to the Next.js analyzer plugin.

## Architecture

The Next.js analyzer follows the plugin architecture established by the React analyzer:
```

analyzers/nextjs/
├── src/
│ ├── analyses/ # Analysis implementations
│ ├── presenters/ # Report presenters
│ ├── utils/ # Shared utilities
│ ├── constants/ # Thresholds and weights
│ ├── types/ # TypeScript types
│ ├── testing/ # Test fixtures
│ └── plugin.ts # Plugin entry point
├── package.json
└── tsconfig.json

````

## Development Setup

```bash
# Install dependencies
pnpm install

# Build the analyzer
pnpm build

# Run tests
pnpm test

# Run tests in watch mode
pnpm test:watch

# Type check
pnpm typecheck

# Lint
pnpm lint
````

## Architecture Principles

### Plugin System

**CRITICAL**: Never import presenters directly. Use PresenterRegistry:

```typescript
// ❌ WRONG
import { ServerClientPresenter } from '@vipr/nextjs/presenters';

// ✅ CORRECT
const presenter = registry.get('nextjs', 'server-client');
```

### Presenter Metadata

Every presenter must implement `getMetadata()`:

```typescript
getMetadata() {
  return this.createMetadata({
    reportType: 'server-client',
    pluginId: 'nextjs',
    label: 'Server/Client Components',
    hint: 'Component boundary issues',
    icon: '🔄',
    order: 20,
    categories: ['correctness', 'performance'],
  });
}
```

### Analysis Interface

All analyses implement `IAnalysis`:

```typescript
export class ServerClientAnalysis implements IAnalysis<unknown, ServerClientComplexity> {
  readonly id = 'nextjs-server-client';
  readonly name = 'Next.js Server/Client Analysis';
  readonly category = 'correctness' as const;
  readonly version = '1.0.0';
  readonly enabledByDefault = true;
  readonly executionCost = 3 as const;

  validateConfig(): true | string;
  getDefaultConfig(): unknown;
  execute(sourceFile: SourceFile): AnalysisResult<ServerClientComplexity>;
}
```

## Adding New Analyses

**Note:** Documentation for adding new analyses to the Next.js analyzer will be added in a future phase.

## Testing

**Note:** Testing documentation for the Next.js analyzer will be added in a future phase.

## Code Style

- 2-space indentation
- Single quotes
- Semicolons required
- 100-character line width
- Prefer TypeScript for all new code

## Commit Guidelines

- Use imperative mood ("Add feature" not "Added feature")
- Keep commits focused and atomic
- Reference issues when applicable

## Pull Request Process

1. Create feature branch from `main`
2. Implement changes with tests
3. Ensure all tests pass (`pnpm test`)
4. Update documentation if needed
5. Submit PR with clear description

## Release Process

Releases are managed by the core team:

1. Version bump in package.json
2. Update CHANGELOG.md
3. Create git tag
4. Publish to npm

## Questions?

Open an issue or discussion on GitHub.

````

#### Architecture Document (`analyzers/nextjs/ARCHITECTURE.md`)

```markdown
# Next.js Analyzer Architecture

## Overview

The Next.js analyzer is a plugin for the Vipr analysis engine that provides comprehensive static analysis of Next.js applications.

## Architecture Diagram

```mermaid
graph TD
    A[NextjsAnalyzerPlugin] --> B[Analyses]
    A --> C[Presenters]
    A --> D[Utils]

    B --> B1[ServerClientAnalysis]
    B --> B2[DataFetchingAnalysis]
    B --> B3[MigrationAnalysis]
    B --> B4[SecurityAnalysis]
    B --> B5[PerformanceAnalysis]
    B --> B6[ConfigAnalysis]
    B --> B7[ComponentAnalysis]
    B --> B8[A11yTypeScriptAnalysis]

    C --> C1[OverviewPresenter]
    C --> C2[ServerClientPresenter]
    C --> C3[DataFetchingPresenter]
    C --> C4[MigrationPresenter]
    C --> C5[SecurityPresenter]
    C --> C6[ConfigPresenter]
    C --> C7[ComponentPresenter]
    C --> C8[PerformancePresenter]
    C --> C9[A11yTypeScriptPresenter]

    D --> D1[Router Helpers]
    D --> D2[Directive Helpers]
    D --> D3[Component Helpers]
    D --> D4[Version Detection]

    E[AnalysisEngine] --> A
    F[PresenterRegistry] --> C
    G[CLI] --> E
    G --> F
````

## Component Responsibilities

### Plugin (`plugin.ts`)

- Implements `ITechnologyPlugin` interface
- Registers all analyses
- Creates and provides presenters
- Implements `canHandle()` for file eligibility
- Builds metrics from analysis results
- Calculates composite score

### Analyses

Each analysis module:

- Detects specific pattern category
- Returns `AnalysisResult<T>` with typed data
- Provides scoring and insights
- Runs independently (parallelizable)

### Presenters

Each presenter:

- Transforms `AnalysisResult` to `ReportPresentation`
- Defines metadata (icon, label, categories)
- Creates sections, metrics, items
- Maps severity levels
- Generates documentation links

### Utilities

Shared helpers:

- Router type detection (App/Pages/Mixed)
- Directive parsing ("use client", "use server")
- Component pattern matching
- Version feature detection

## Data Flow

```mermaid
sequenceDiagram
    participant CLI
    participant Engine
    participant Plugin
    participant Analysis
    participant Presenter

    CLI->>Engine: analyze(sourceFile)
    Engine->>Plugin: canHandle(sourceFile)
    Plugin-->>Engine: true
    Engine->>Plugin: getAnalyses()
    Plugin-->>Engine: [analyses...]

    par Run Analyses
        Engine->>Analysis: execute(sourceFile)
        Analysis-->>Engine: AnalysisResult
    end

    Engine->>Plugin: buildCompositeScore(results)
    Plugin-->>Engine: score
    Engine->>Plugin: buildMetrics(results)
    Plugin-->>Engine: metrics

    CLI->>Plugin: getReportPresenters()
    Plugin-->>CLI: [presenters...]
    CLI->>Presenter: present(pluginResult)
    Presenter-->>CLI: ReportPresentation
```

## Scoring System

### Analysis Scores

Each analysis produces 0-100 score:

- 0-40: Critical issues
- 40-60: Moderate issues
- 60-80: Minor issues
- 80-100: Healthy

### Composite Score

Weighted average of all analyses:

```typescript
const COMPLEXITY_WEIGHTS = {
  serverClient: 0.15,
  dataFetching: 0.15,
  migration: 0.1,
  security: 0.2,
  performance: 0.15,
  config: 0.05,
  component: 0.1,
  a11yTypeScript: 0.1,
};
```

Security weighted highest due to criticality.

## Type System

### Analysis Result Types

```typescript
interface AnalysisResult<T = unknown> {
  analysisId: string;
  data: T;
  insights: ComplexityInsight[];
  executionTimeMs: number;
}
```

### Complexity Types

Each analysis defines complexity type:

```typescript
interface ServerClientComplexity {
  score: number;
  totalIssues: number;
  issues: ServerClientIssue[];
  stats: ServerClientStats;
  bySeverity: Record<string, number>;
  byType: Record<string, number>;
}
```

## Extension Points

### Adding New Analysis

1. Create analysis class implementing `IAnalysis`
2. Define complexity type
3. Implement detection logic
4. Register in plugin constructor
5. Create presenter
6. Add to metrics builder

### Adding New Report

1. Create presenter extending `NextjsBasePresenter`
2. Implement `getMetadata()` and `present()`
3. Add to `createNextjsPresenters()` factory
4. Add tests

## Performance Considerations

### AST Traversal

- Single-pass traversal when possible
- Cache expensive lookups
- Use specific node type checks

### Memory Management

- No global state
- Clean up references
- Use streaming for large files

### Parallelization

- Analyses run in parallel
- Share AST across analyses
- No inter-analysis dependencies

## Testing Strategy

### Unit Tests

- Each analysis module
- Each presenter
- Each utility function

### Integration Tests

- End-to-end plugin
- Multi-file scenarios
- Router coexistence
- Performance benchmarks

### Fixtures

- Real-world patterns
- Edge cases
- Version-specific features

````

### Workspace Configuration

#### Update Root package.json

```json
{
  "scripts": {
    "analyze:nextjs": "pnpm --filter @vipr/nextjs analyze",
    "build:nextjs": "pnpm --filter @vipr/nextjs build",
    "test:nextjs": "pnpm --filter @vipr/nextjs test",
    "dev:nextjs": "pnpm --filter @vipr/nextjs dev"
  }
}
````

#### Update turbo.json

```json
{
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**"],
      "inputs": ["src/**", "package.json", "tsconfig.json"]
    },
    "test": {
      "dependsOn": ["build"],
      "outputs": [],
      "inputs": ["src/**", "*.test.ts"]
    }
  }
}
```

## Test Coverage

### CLI Integration Tests

**Plugin Discovery (3 tests):**

- Discovers Next.js plugin
- Loads plugin successfully
- Registers with engine

**Presenter Registry (4 tests):**

- All presenters registered
- Presenters queryable by ID
- Metadata complete
- Reports available

**Formatter Integration (6 tests):**

- JSON formatter handles Next.js reports
- Markdown formatter handles Next.js reports
- Terminal formatter handles Next.js reports
- Report types displayed correctly
- Icons rendered
- Links formatted

**End-to-End (5 tests):**

- CLI analyze command works
- Reports generated
- Output formatted
- Exit codes correct
- Performance acceptable

Total: 18 CLI integration tests

## Acceptance Criteria

All criteria must be met:

- [ ] `pnpm build` succeeds without errors across workspace
- [ ] All 18 CLI integration tests pass
- [ ] Next.js plugin discovered by loader
- [ ] All presenters register with PresenterRegistry
- [ ] Formatters handle all Next.js report types
- [ ] User documentation complete and accurate
- [ ] Developer documentation complete
- [ ] Architecture diagrams current
- [ ] README.md updated with Next.js analyzer
- [ ] CHANGELOG.md includes release notes
- [ ] Package.json scripts added
- [ ] Turbo pipeline configured
- [ ] CLI commands work end-to-end
- [ ] Examples in documentation tested
- [ ] Links in documentation valid

## Technical Notes

### Plugin Discovery Pattern

```typescript
// Import Next.js analyzer
try {
  const { NextjsAnalyzerPlugin } = await import('@vipr/nextjs');
  plugins.push(new NextjsAnalyzerPlugin());
} catch (error) {
  logger.warn(
    'Next.js analyzer not available:',
    error instanceof Error ? error.message : String(error)
  );
}
```

### Presenter Registration Pattern

```typescript
// In CLI after loading plugins
const registry = new PresenterRegistry();
for (const plugin of loadedPlugins) {
  registry.registerFromPlugin(plugin);
}

// Query for Next.js reports
const nextjsReports = registry.getAvailableReports('nextjs');
```

### Formatter Report Handling

Formatters query registry, not hardcoded:

```typescript
// ✅ CORRECT
const reports = registry.getAvailableReports(pluginId);
for (const metadata of reports) {
  console.log(`${metadata.icon} ${metadata.label}`);
}

// ❌ WRONG
const hardcodedReports = ['server-client', 'migration'];
```

### Documentation Mermaid Diagrams

Use mermaid for architecture diagrams:

```markdown
## Architecture Diagram

\`\`\`mermaid
graph TD
A[Plugin] --> B[Analyses]
A --> C[Presenters]
\`\`\`
```

### CLI Usage Examples

All examples must be tested:

```bash
# Test each example in documentation
vipr analyze ./app --plugins nextjs
vipr analyze ./app --plugins nextjs --report migration
vipr analyze ./app --plugins nextjs --format json
```

### Version Documentation

Document supported Next.js versions:

```markdown
## Supported Next.js Versions

- Next.js 12.x
- Next.js 13.x
- Next.js 14.x
- Next.js 15.x (latest)

Breaking change detection covers 12→13, 13→14, 14→15 migrations.
```

### Performance Documentation

Document performance characteristics:

```markdown
## Performance

- Small files (< 50 lines): ~100ms
- Medium files (50-200 lines): ~250ms
- Large files (200-500 lines): ~500ms
- Extra large files (500+ lines): ~1000ms

Batch analysis: ~100ms per file (parallelized)
```

### Configuration Examples

Provide complete configuration examples:

```json
{
  "plugins": {
    "nextjs": {
      "enabled": true,
      "targetVersion": "15",
      "analyses": {
        "nextjs-server-client": { "enabled": true },
        "nextjs-migration": {
          "enabled": true,
          "config": { "targetVersion": "15" }
        }
      }
    }
  },
  "output": {
    "format": "json",
    "file": "nextjs-report.json"
  }
}
```

### Troubleshooting Section

Include common issues and solutions:

```markdown
## Troubleshooting

### Plugin Not Found

**Issue**: "Next.js plugin not available"

**Solution**:

1. Verify installation: `npm list @vipr/cli`
2. Check version: `vipr --version`
3. Reinstall if needed: `npm install -g @vipr/cli@latest`

### No Issues Detected

**Issue**: Analysis runs but reports no issues

**Solution**:

1. Verify files match patterns (app/, pages/, next.config.js)
2. Run with --verbose flag
3. Check file eligibility with --dry-run
```

## Next Steps

After completing Phase 14:

### Production Readiness Checklist

- [ ] All phases 1-14 completed
- [ ] All tests passing (unit, integration, CLI)
- [ ] Documentation complete and reviewed
- [ ] Performance benchmarks met
- [ ] Security audit passed
- [ ] Accessibility review completed
- [ ] Code review by maintainers
- [ ] Version number assigned
- [ ] CHANGELOG.md finalized
- [ ] Release notes prepared

### Pre-Release Tasks

- [ ] Create release branch
- [ ] Final integration testing
- [ ] User acceptance testing
- [ ] Performance profiling
- [ ] Memory leak testing
- [ ] Cross-platform testing (Windows, macOS, Linux)
- [ ] CI/CD pipeline green
- [ ] Documentation deployed

### Release Tasks

- [ ] Merge to main branch
- [ ] Create git tag
- [ ] Publish to npm
- [ ] Deploy documentation
- [ ] Announce release
- [ ] Monitor for issues

### Post-Release Tasks

- [ ] Monitor error reports
- [ ] Gather user feedback
- [ ] Track adoption metrics
- [ ] Plan next iteration
- [ ] Update roadmap
