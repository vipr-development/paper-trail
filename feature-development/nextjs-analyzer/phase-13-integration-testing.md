# Phase 13: Integration Testing & Fixtures

## Status

Not Started

## Goals

Create comprehensive test fixtures covering all Next.js patterns and integration tests that verify the entire plugin works end-to-end with the AnalysisEngine, including multi-file analysis, router coexistence scenarios, and presenter integration.

## Files Created

### Test Fixture Files

#### App Router Fixtures

- `testing/fixtures/app-router/page.tsx` - Basic app router page
- `testing/fixtures/app-router/layout.tsx` - Root layout with metadata
- `testing/fixtures/app-router/server-component.tsx` - Server Component patterns
- `testing/fixtures/app-router/client-component.tsx` - Client Component with hooks
- `testing/fixtures/app-router/route.ts` - Route handler (GET, POST)
- `testing/fixtures/app-router/loading.tsx` - Loading UI
- `testing/fixtures/app-router/error.tsx` - Error boundary
- `testing/fixtures/app-router/not-found.tsx` - Not found page
- `testing/fixtures/app-router/template.tsx` - Template component
- `testing/fixtures/app-router/dynamic-route/[id]/page.tsx` - Dynamic route with params
- `testing/fixtures/app-router/parallel/page.tsx` - Parallel routes
- `testing/fixtures/app-router/intercepting/(..)modal/page.tsx` - Intercepting routes

#### Pages Router Fixtures

- `testing/fixtures/pages-router/index.tsx` - Index page
- `testing/fixtures/pages-router/about.tsx` - Static page
- `testing/fixtures/pages-router/[slug].tsx` - Dynamic page with getStaticPaths
- `testing/fixtures/pages-router/ssr-page.tsx` - Page with getServerSideProps
- `testing/fixtures/pages-router/ssg-page.tsx` - Page with getStaticProps
- `testing/fixtures/pages-router/isr-page.tsx` - ISR with revalidate
- `testing/fixtures/pages-router/_app.tsx` - Custom App
- `testing/fixtures/pages-router/_document.tsx` - Custom Document
- `testing/fixtures/pages-router/api/users.ts` - API route
- `testing/fixtures/pages-router/api/[id].ts` - Dynamic API route

#### Server Actions Fixtures

- `testing/fixtures/server-actions/form-action.ts` - Form Server Action
- `testing/fixtures/server-actions/mutation-action.ts` - Data mutation with revalidate
- `testing/fixtures/server-actions/untyped-action.ts` - Untyped Server Action (error)
- `testing/fixtures/server-actions/no-auth-action.ts` - Missing authentication (error)
- `testing/fixtures/server-actions/non-async-action.ts` - Non-async Server Action (error)
- `testing/fixtures/server-actions/use-action-state.tsx` - Client using useActionState

#### Next.js Config Fixtures

- `testing/fixtures/next-configs/next.config.js` - Valid modern config
- `testing/fixtures/next-configs/deprecated-config.js` - Deprecated options
- `testing/fixtures/next-configs/removed-config.js` - Removed options (Next.js 15+)
- `testing/fixtures/next-configs/invalid-matcher.js` - Invalid middleware matcher
- `testing/fixtures/next-configs/images-domains.js` - Deprecated images.domains
- `testing/fixtures/next-configs/experimental.js` - Experimental features

#### Component Fixtures

- `testing/fixtures/components/image-issues.tsx` - next/image problems
- `testing/fixtures/components/link-issues.tsx` - next/link problems
- `testing/fixtures/components/script-issues.tsx` - next/script problems
- `testing/fixtures/components/font-optimization.tsx` - next/font usage
- `testing/fixtures/components/metadata-api.tsx` - Metadata API usage

#### Mixed Router Fixtures

- `testing/fixtures/mixed/app-page.tsx` - App Router page
- `testing/fixtures/mixed/pages-page.tsx` - Pages Router page
- `testing/fixtures/mixed/shared-component.tsx` - Component used by both
- `testing/fixtures/mixed/middleware.ts` - Middleware configuration

#### Migration Fixtures

- `testing/fixtures/migration/next-12-patterns.tsx` - Next.js 12 patterns
- `testing/fixtures/migration/next-13-patterns.tsx` - Next.js 13 patterns
- `testing/fixtures/migration/next-14-patterns.tsx` - Next.js 14 patterns
- `testing/fixtures/migration/next-15-breaking.tsx` - Next.js 15 breaking changes
- `testing/fixtures/migration/legacy-image.tsx` - Legacy next/image usage
- `testing/fixtures/migration/old-router.tsx` - next/router in app directory

### Integration Test Files

- `plugin.integration.test.ts` - End-to-end plugin tests
- `multi-file-analysis.integration.test.ts` - Multi-file analysis scenarios
- `router-coexistence.integration.test.ts` - Mixed router scenarios
- `presenter-integration.test.ts` - Presenter output validation
- `cli-loader.integration.test.ts` - CLI plugin loader integration
- `performance-benchmark.integration.test.ts` - Performance benchmarks

## Implementation Details

### Test Fixture Organization

Fixtures should cover all patterns from the overview document:

**App Router Patterns:**

```typescript
// testing/fixtures/app-router/server-component.tsx
// Tests: Server Component data fetching, async components, streaming

export default async function ServerComponent({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  const data = await fetch(`https://api.example.com/data/${id}`, {
    cache: 'force-cache',
  }).then(r => r.json());

  return <div>{data.title}</div>;
}
```

**Client Component with Issues:**

```typescript
// testing/fixtures/app-router/client-component.tsx
// Tests: Unnecessary 'use client', directive placement, hook usage

import { useState } from 'react';

'use client'; // BAD: Not at top of file

export function ClientComponent() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}
```

**Server Action with Issues:**

```typescript
// testing/fixtures/server-actions/untyped-action.ts
// Tests: Missing types, no authentication, no revalidation

'use server';

export async function createPost(formData) {
  // BAD: Untyped
  // BAD: No authentication check
  const title = formData.get('title');
  await db.post.create({ data: { title } });
  // BAD: No revalidatePath or revalidateTag
}
```

**Next.js 15 Breaking Changes:**

```typescript
// testing/fixtures/migration/next-15-breaking.tsx
// Tests: Async params, cookies, headers

import { cookies } from 'next/headers';

// BAD: Synchronous params type
export default function Page({ params }: { params: { id: string } }) {
  // BAD: Synchronous cookies access
  const cookieStore = cookies();
  const token = cookieStore.get('token');

  return <div>{params.id}</div>;
}
```

**Deprecated Config:**

```javascript
// testing/fixtures/next-configs/deprecated-config.js
// Tests: Deprecated configuration options

/** @type {import('next').NextConfig} */
module.exports = {
  swcMinify: true, // Removed in Next.js 15
  images: {
    domains: ['example.com'], // Deprecated - use remotePatterns
  },
  experimental: {
    appDir: true, // No longer needed
  },
};
```

### Integration Test Scenarios

#### End-to-End Plugin Test

```typescript
// plugin.integration.test.ts
import { describe, it, expect, beforeAll } from 'vitest';
import { Project } from 'ts-morph';
import { NextjsAnalyzerPlugin } from '../plugin';
import { AnalysisEngine } from '@vipr/engine';
import { PresenterRegistry } from '@vipr/common';

describe('NextjsAnalyzerPlugin Integration', () => {
  let plugin: NextjsAnalyzerPlugin;
  let engine: AnalysisEngine;
  let project: Project;

  beforeAll(() => {
    plugin = new NextjsAnalyzerPlugin();
    engine = new AnalysisEngine();
    engine.registerPlugin(plugin);
    project = new Project();
  });

  describe('End-to-end analysis', () => {
    it('analyzes app router page with all features', async () => {
      const sourceFile = project.createSourceFile(
        'app/page.tsx',
        await readFixture('app-router/page.tsx')
      );

      const result = await plugin.analyze(sourceFile);

      expect(result.pluginId).toBe('nextjs');
      expect(result.score).toBeGreaterThan(0);
      expect(result.score).toBeLessThanOrEqual(100);
      expect(result.analysisBreakdown).toBeDefined();
      expect(result.metrics).toBeDefined();
    });

    it('detects server/client boundary issues', async () => {
      const sourceFile = project.createSourceFile(
        'app/client-component.tsx',
        await readFixture('app-router/client-component.tsx')
      );

      const result = await plugin.analyze(sourceFile);
      const serverClientData = result.analysisBreakdown?.get('nextjs-server-client')?.data;

      expect(serverClientData).toBeDefined();
      expect(serverClientData.totalIssues).toBeGreaterThan(0);
      expect(serverClientData.issues.some(i => i.type === 'directive-placement')).toBe(true);
    });

    it('detects Next.js 15 breaking changes', async () => {
      const sourceFile = project.createSourceFile(
        'app/breaking.tsx',
        await readFixture('migration/next-15-breaking.tsx')
      );

      const result = await plugin.analyze(sourceFile);
      const migrationData = result.analysisBreakdown?.get('nextjs-migration')?.data;

      expect(migrationData).toBeDefined();
      expect(migrationData.blockers.length).toBeGreaterThan(0);
      expect(migrationData.blockers.some(b => b.breaking === true)).toBe(true);
    });

    it('analyzes server actions', async () => {
      const sourceFile = project.createSourceFile(
        'app/actions.ts',
        await readFixture('server-actions/untyped-action.ts')
      );

      const result = await plugin.analyze(sourceFile);
      const securityData = result.analysisBreakdown?.get('nextjs-security')?.data;

      expect(securityData).toBeDefined();
      expect(securityData.vulnerabilities.length).toBeGreaterThan(0);
    });

    it('all analyses produce valid results', async () => {
      const sourceFile = project.createSourceFile(
        'app/complex.tsx',
        await readFixture('app-router/server-component.tsx')
      );

      const result = await plugin.analyze(sourceFile);

      // Verify all analyses ran
      expect(result.analysisBreakdown?.size).toBeGreaterThan(0);

      // Verify each analysis has valid structure
      for (const [analysisId, analysisResult] of result.analysisBreakdown!) {
        expect(analysisResult.analysisId).toBe(analysisId);
        expect(analysisResult.data).toBeDefined();
        expect(typeof analysisResult.data.score).toBe('number');
        expect(analysisResult.executionTimeMs).toBeGreaterThanOrEqual(0);
      }
    });
  });

  describe('Presenter integration', () => {
    it('all presenters register successfully', () => {
      const registry = new PresenterRegistry();
      registry.registerFromPlugin(plugin);

      const availableReports = registry.getAvailableReports('nextjs');
      expect(availableReports.length).toBeGreaterThan(0);
    });

    it('presenters produce valid output', async () => {
      const sourceFile = project.createSourceFile(
        'app/test.tsx',
        await readFixture('app-router/page.tsx')
      );

      const result = await plugin.analyze(sourceFile);
      const presenters = plugin.getReportPresenters();

      for (const presenter of presenters) {
        if (presenter.canPresent(result)) {
          const presentation = presenter.present(result);

          expect(presentation.reportType).toBeDefined();
          expect(presentation.pluginId).toBe('nextjs');
          expect(presentation.title).toBeTruthy();
          expect(presentation.description).toBeTruthy();
          expect(Array.isArray(presentation.sections)).toBe(true);
        }
      }
    });

    it('metadata complete for all presenters', () => {
      const presenters = plugin.getReportPresenters();

      for (const presenter of presenters) {
        const metadata = presenter.getMetadata();

        expect(metadata.reportType).toBeTruthy();
        expect(metadata.pluginId).toBe('nextjs');
        expect(metadata.label).toBeTruthy();
        expect(metadata.hint).toBeTruthy();
        expect(metadata.icon).toBeTruthy();
        expect(typeof metadata.order).toBe('number');
      }
    });
  });

  describe('canHandle validation', () => {
    it('handles .tsx files with Next.js patterns', () => {
      const sourceFile = project.createSourceFile(
        'app/page.tsx',
        `export default function Page() { return <div>Hello</div>; }`
      );

      expect(plugin.canHandle(sourceFile)).toBe(true);
    });

    it('handles .ts files with server actions', () => {
      const sourceFile = project.createSourceFile(
        'app/actions.ts',
        `'use server'; export async function myAction() {}`
      );

      expect(plugin.canHandle(sourceFile)).toBe(true);
    });

    it('handles next.config files', () => {
      const sourceFile = project.createSourceFile(
        'next.config.js',
        `module.exports = { reactStrictMode: true }`
      );

      expect(plugin.canHandle(sourceFile)).toBe(true);
    });

    it('rejects non-Next.js files', () => {
      const sourceFile = project.createSourceFile(
        'utils/helpers.ts',
        `export function helper() { return true; }`
      );

      expect(plugin.canHandle(sourceFile)).toBe(false);
    });
  });
});
```

#### Multi-File Analysis Test

```typescript
// multi-file-analysis.integration.test.ts
import { describe, it, expect, beforeAll } from 'vitest';
import { Project } from 'ts-morph';
import { NextjsAnalyzerPlugin } from '../plugin';

describe('Multi-file Analysis', () => {
  let plugin: NextjsAnalyzerPlugin;
  let project: Project;

  beforeAll(() => {
    plugin = new NextjsAnalyzerPlugin();
    project = new Project();
  });

  it('analyzes app directory structure', async () => {
    // Create multiple files simulating app directory
    const layout = project.createSourceFile(
      'app/layout.tsx',
      await readFixture('app-router/layout.tsx')
    );
    const page = project.createSourceFile('app/page.tsx', await readFixture('app-router/page.tsx'));
    const route = project.createSourceFile(
      'app/api/users/route.ts',
      await readFixture('app-router/route.ts')
    );

    const results = await Promise.all([
      plugin.analyze(layout),
      plugin.analyze(page),
      plugin.analyze(route),
    ]);

    // Verify all analyzed successfully
    expect(results).toHaveLength(3);
    results.forEach(result => {
      expect(result.pluginId).toBe('nextjs');
      expect(result.score).toBeGreaterThanOrEqual(0);
    });

    // Aggregate statistics
    const totalIssues = results.reduce((sum, r) => sum + (r.insights?.length || 0), 0);
    expect(totalIssues).toBeGreaterThanOrEqual(0);
  });

  it('handles incremental analysis of changing files', async () => {
    const sourceFile = project.createSourceFile(
      'app/dynamic.tsx',
      `'use client'; export function Component() { return <div>Initial</div>; }`
    );

    const result1 = await plugin.analyze(sourceFile);
    const initialScore = result1.score;

    // Modify file to fix issues
    sourceFile.replaceWithText(
      `'use client';\nimport { useState } from 'react';\nexport function Component() { const [count, setCount] = useState(0); return <button onClick={() => setCount(count + 1)}>{count}</button>; }`
    );

    const result2 = await plugin.analyze(sourceFile);
    const newScore = result2.score;

    // Score should change based on modifications
    expect(newScore).not.toBe(initialScore);
  });
});
```

#### Router Coexistence Test

```typescript
// router-coexistence.integration.test.ts
import { describe, it, expect, beforeAll } from 'vitest';
import { Project } from 'ts-morph';
import { NextjsAnalyzerPlugin } from '../plugin';

describe('Router Coexistence', () => {
  let plugin: NextjsAnalyzerPlugin;
  let project: Project;

  beforeAll(() => {
    plugin = new NextjsAnalyzerPlugin();
    project = new Project();
  });

  it('detects app router and pages router coexistence', async () => {
    const appPage = project.createSourceFile(
      'app/about/page.tsx',
      await readFixture('mixed/app-page.tsx')
    );
    const pagesPage = project.createSourceFile(
      'pages/contact.tsx',
      await readFixture('mixed/pages-page.tsx')
    );

    const appResult = await plugin.analyze(appPage);
    const pagesResult = await plugin.analyze(pagesPage);

    // Both should analyze successfully
    expect(appResult.pluginId).toBe('nextjs');
    expect(pagesResult.pluginId).toBe('nextjs');

    // Check router detection
    const appRouterType = appResult.metrics.routerInfo?.type;
    const pagesRouterType = pagesResult.metrics.routerInfo?.type;

    expect(appRouterType).toBe('app');
    expect(pagesRouterType).toBe('pages');
  });

  it('warns about router-specific imports in wrong context', async () => {
    const sourceFile = project.createSourceFile(
      'app/wrong-router.tsx',
      await readFixture('migration/old-router.tsx')
    );

    const result = await plugin.analyze(sourceFile);
    const migrationData = result.analysisBreakdown?.get('nextjs-migration')?.data;

    expect(migrationData?.warnings.some(w => w.type === 'wrong-router-import')).toBe(true);
  });
});
```

#### Performance Benchmark Test

```typescript
// performance-benchmark.integration.test.ts
import { describe, it, expect, beforeAll } from 'vitest';
import { Project } from 'ts-morph';
import { NextjsAnalyzerPlugin } from '../plugin';

describe('Performance Benchmarks', () => {
  let plugin: NextjsAnalyzerPlugin;
  let project: Project;

  beforeAll(() => {
    plugin = new NextjsAnalyzerPlugin();
    project = new Project();
  });

  it('analyzes small file within time budget', async () => {
    const sourceFile = project.createSourceFile(
      'app/small.tsx',
      `export default function Page() { return <div>Hello</div>; }`
    );

    const startTime = performance.now();
    const result = await plugin.analyze(sourceFile);
    const endTime = performance.now();

    expect(endTime - startTime).toBeLessThan(100); // 100ms budget
    expect(result.executionTimeMs).toBeLessThan(100);
  });

  it('analyzes medium file within time budget', async () => {
    const sourceFile = project.createSourceFile(
      'app/medium.tsx',
      await readFixture('app-router/server-component.tsx')
    );

    const startTime = performance.now();
    const result = await plugin.analyze(sourceFile);
    const endTime = performance.now();

    expect(endTime - startTime).toBeLessThan(250); // 250ms budget
    expect(result.executionTimeMs).toBeLessThan(250);
  });

  it('analyzes large file within time budget', async () => {
    // Create large file with many patterns
    const largeContent = generateLargeNextjsFile(500); // 500 lines
    const sourceFile = project.createSourceFile('app/large.tsx', largeContent);

    const startTime = performance.now();
    const result = await plugin.analyze(sourceFile);
    const endTime = performance.now();

    expect(endTime - startTime).toBeLessThan(500); // 500ms budget
    expect(result.executionTimeMs).toBeLessThan(500);
  });

  it('handles batch analysis efficiently', async () => {
    const files = [
      'app/page1.tsx',
      'app/page2.tsx',
      'app/page3.tsx',
      'app/page4.tsx',
      'app/page5.tsx',
    ].map(path =>
      project.createSourceFile(
        path,
        `export default function Page() { return <div>${path}</div>; }`
      )
    );

    const startTime = performance.now();
    const results = await Promise.all(files.map(f => plugin.analyze(f)));
    const endTime = performance.now();

    expect(results).toHaveLength(5);
    expect(endTime - startTime).toBeLessThan(500); // 100ms per file budget
  });
});

function generateLargeNextjsFile(lines: number): string {
  const components = [];
  for (let i = 0; i < lines / 10; i++) {
    components.push(`
      function Component${i}() {
        const [state${i}, setState${i}] = useState(0);
        useEffect(() => {
          console.log('Effect ${i}');
        }, [state${i}]);
        return <div onClick={() => setState${i}(state${i} + 1)}>{state${i}}</div>;
      }
    `);
  }
  return `
    'use client';
    import { useState, useEffect } from 'react';
    ${components.join('\n')}
    export default function LargePage() {
      return <div>${components.map((_, i) => `<Component${i} />`).join('')}</div>;
    }
  `;
}
```

### Fixture Helper Functions

```typescript
// testing/fixtures/helpers.ts
import { readFile } from 'fs/promises';
import { join } from 'path';

/**
 * Read fixture file content
 */
export async function readFixture(relativePath: string): Promise<string> {
  const fixturePath = join(__dirname, relativePath);
  return await readFile(fixturePath, 'utf-8');
}

/**
 * Create test project with fixtures
 */
export function createTestProject(fixtures: Record<string, string>): Project {
  const project = new Project({ useInMemoryFileSystem: true });

  for (const [path, content] of Object.entries(fixtures)) {
    project.createSourceFile(path, content);
  }

  return project;
}
```

## Test Coverage

### Integration Test Metrics

**End-to-end Plugin Tests (15 tests):**

- Analyzes all router types
- Detects all issue categories
- All analyses produce valid results
- Presenter registration works
- Presenter output validation
- Metadata completeness
- canHandle validation (4 tests)

**Multi-file Analysis Tests (10 tests):**

- App directory structure
- Pages directory structure
- Server actions in separate files
- Shared components
- Config file analysis
- Incremental analysis
- Aggregate statistics
- Cross-file references
- Import resolution
- Dynamic imports

**Router Coexistence Tests (8 tests):**

- Detects both routers
- Router-specific pattern detection
- Wrong router import warnings
- Middleware in mixed environment
- Shared utilities
- Migration path suggestions
- API route compatibility
- Component compatibility

**Performance Benchmarks (12 tests):**

- Small file analysis (under 100ms)
- Medium file analysis (under 250ms)
- Large file analysis (under 500ms)
- Batch analysis efficiency
- Memory usage tracking
- Analysis breakdown timing
- Presenter performance
- Registry lookup performance
- Cache effectiveness
- Parallel analysis throughput
- AST traversal optimization
- Score calculation performance

**CLI Loader Integration (8 tests):**

- Plugin discovery
- Plugin registration
- Presenter registration with registry
- Formatter can query registry
- Report generation
- Multiple plugin coordination
- Plugin lifecycle hooks
- Error handling

Total: 53 integration tests

## Acceptance Criteria

All criteria must be met:

- [ ] `pnpm build` succeeds without errors
- [ ] All 53 integration tests pass
- [ ] Fixtures cover all patterns from overview document
- [ ] App Router fixtures include all file types (page, layout, route, etc.)
- [ ] Pages Router fixtures include SSG, SSR, ISR patterns
- [ ] Server Action fixtures cover all security issues
- [ ] Config fixtures include deprecated and removed options
- [ ] Migration fixtures cover Next.js 12, 13, 14, 15
- [ ] Multi-file analysis works correctly
- [ ] Router coexistence detected and handled
- [ ] Presenters integrate with PresenterRegistry
- [ ] CLI plugin loader discovers and loads plugin
- [ ] Performance budgets met (100ms small, 250ms medium, 500ms large)
- [ ] Batch analysis efficient
- [ ] All analyses produce valid AnalysisResult
- [ ] Composite score calculated correctly
- [ ] Metrics object complete and accurate

## Technical Notes

### Fixture File Requirements

Each fixture should:

- Represent a real-world pattern
- Include comments explaining what it tests
- Cover both correct and incorrect usage
- Be minimal but realistic
- Include line numbers in expected test assertions

### Test Organization

Integration tests organized by scope:

- `plugin.integration.test.ts` - Plugin-level tests
- `multi-file-analysis.integration.test.ts` - Cross-file tests
- `router-coexistence.integration.test.ts` - Router mixing tests
- `presenter-integration.test.ts` - Presentation layer tests
- `cli-loader.integration.test.ts` - CLI integration tests
- `performance-benchmark.integration.test.ts` - Performance tests

### Performance Budgets

Target execution times:

- Small files (< 50 lines): 100ms
- Medium files (50-200 lines): 250ms
- Large files (200-500 lines): 500ms
- Extra large files (500+ lines): 1000ms
- Batch (5 files): 500ms total

### Memory Considerations

Monitor memory usage:

- Single file analysis: < 50MB heap increase
- Batch analysis: < 200MB heap increase
- No memory leaks between analyses
- Proper cleanup in afterEach hooks

### Router Detection Strategy

Test router detection logic:

```typescript
function detectRouterType(filePath: string): 'app' | 'pages' | 'unknown' {
  if (filePath.includes('/app/')) return 'app';
  if (filePath.includes('/pages/')) return 'pages';
  return 'unknown';
}
```

### Async Testing Patterns

Use proper async patterns:

```typescript
it('handles async analysis', async () => {
  const result = await plugin.analyze(sourceFile);
  expect(result).toBeDefined();
});
```

### Snapshot Testing

Consider snapshots for presenter output:

```typescript
it('generates consistent presentation', async () => {
  const result = await plugin.analyze(sourceFile);
  const presentation = presenter.present(result);
  expect(presentation).toMatchSnapshot();
});
```

### Edge Cases to Test

- Empty files
- Files with only comments
- Files with syntax errors
- Very large dependency arrays
- Deeply nested components
- Complex type annotations
- Multiple directive occurrences
- Malformed config files

## Next Steps

After completing Phase 13, proceed to:

- Phase 14: CLI Integration & Documentation - Integrate with CLI, create user and developer documentation, update workspace configuration
