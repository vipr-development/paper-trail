# Barrel Exports Quick Reference

## Current State Summary

### Bundle Sizes (Compiled JavaScript)

```
@vipr/common      624KB  ████████████████████
@vipr/core        440KB  ██████████████
@vipr/react       1.4MB  ████████████████████████████████████████████
@vipr/logging      16KB  ▌
@vipr/plugin-loader 224KB ███████
```

### Barrel Export Depth

```
@vipr/common
├── index.ts (73 lines)
│   └── types/index.ts
│       ├── core/index.ts (36 exports)
│       ├── plugin/index.ts (24 exports)
│       ├── analysis/index.ts (37 exports)
│       ├── output/index.ts (27 exports)
│       └── presentation/index.ts (10 exports)
└── Total: 5 levels, 180+ type exports

@vipr/core
├── index.ts (88 lines)
│   ├── Direct exports: 10
│   ├── Re-exports from @vipr/common: 13
│   ├── Utility exports: 30+
│   └── Presenter exports: 2
└── Total: 55+ exports

@vipr/react
├── index.ts (138 lines)
│   ├── Plugin: 1
│   ├── Analyses: 14
│   ├── Presenters: 10
│   ├── Types: 13
│   ├── Utils: 40+
│   └── Constants: 11
└── Total: 89+ exports
```

## Import Patterns: Current vs. Optimal

### CLI Client

#### Current Imports

```typescript
// Pulls in entire packages
import { AnalysisEngine } from '@vipr/core';              // 440KB
import { ReactAnalyzerPlugin } from '@vipr/react';        // 1.4MB
import type { AggregatedResult } from '@vipr/common';     // 624KB types
import { createReactPresenters } from '@vipr/react';      // Already loaded

Total: ~2.5MB for 6 imports
```

#### Optimal Imports (After Refactoring)

```typescript
// Granular, tree-shakeable
import { AnalysisEngine } from '@vipr/core/engine';       // 150KB
import { ReactAnalyzerPlugin } from '@vipr/react';        // 200KB
import type { AggregatedResult } from '@vipr/types/plugin'; // 0KB runtime
import { createReactPresenters } from '@vipr/presenters-react'; // 150KB

Total: ~500KB for 4 imports (80% reduction)
```

### VSCode Extension

#### Current Bundle

```typescript
import { AnalysisEngine } from '@vipr/core';
import { ReactAnalyzerPlugin } from '@vipr/react';

Bundle size: ~3MB
Activation time: 300-500ms
```

#### Optimal Bundle (With Lazy Loading)

```typescript
// Initial load
import { AnalysisEngine } from '@vipr/core/engine';

// Lazy load plugin
const plugin = await import('@vipr/react');

Initial bundle: ~400KB
Lazy load: ~1MB
Activation time: `&lt;100`ms (70-80% faster)
```

## Phase Implementation Impact

### Phase 1: ESM + Side Effects (1 week)

- No code changes required
- Bundle reduction: 30-40%
- Activation improvement: 20-30%

```
Before: 2.7MB CLI bundle
After:  1.8MB CLI bundle (-900KB)
```

### Phase 2: Subpath Exports (2 weeks)

- Optional migration to granular imports
- Bundle reduction: 50-60%
- Better tree-shaking

```
Before: 1.8MB CLI bundle
After:  1.3MB CLI bundle (-500KB)
```

### Phase 3: Extract Presenters (2 weeks)

- Breaking change (minor, with compat layer)
- Bundle reduction: 60-70%
- Enables lazy loading

```
Before: 1.3MB CLI bundle
After:  1.0MB CLI bundle (-300KB)

VSCode activation: 300ms → 100ms
```

### Phase 4: Split Packages (4 weeks)

- Breaking change (minor, with compat layer)
- Clean dependency graph
- Foundation for advanced optimizations

```
Improved developer experience
Better IDE performance
Clearer import sources
```

## Quick Decision Guide

### When to Use Barrel Exports

**Use barrels for:**

- Small packages (`&lt;10` exports)
- Packages where all exports are commonly used together
- Internal-only packages
- Type-only exports (zero runtime cost)

**Avoid barrels for:**

- Large packages (>20 exports)
- Mixed runtime + type exports
- Packages with optional features
- Client-facing packages

### Import Pattern Rules

```typescript
// GOOD: Type imports (no runtime cost)
import type { Grade, Severity } from '@vipr/types/core';

// GOOD: Specific utility imports
import { scoreToGrade } from '@vipr/utils/scoring';

// GOOD: Main entry point for plugins
import { ReactAnalyzerPlugin } from '@vipr/react';

// BAD: Mixing types with runtime code
import { Grade, scoreToGrade } from '@vipr/common';

// BAD: Importing entire utility set
import { * as utils } from '@vipr/core/utils';

// BAD: Importing internals from barrel
import { StructuralAnalysis } from '@vipr/react';
```

## Package.json Configuration Template

### Optimal Package Setup

```json
{
  "name": "@vipr/example",
  "version": "1.0.0",
  "type": "module",
  "main": "./dist/index.cjs",
  "module": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js",
      "require": "./dist/index.cjs",
      "default": "./dist/index.js"
    },
    "./feature-a": {
      "types": "./dist/feature-a.d.ts",
      "import": "./dist/feature-a.js",
      "require": "./dist/feature-a.cjs"
    },
    "./feature-b": {
      "types": "./dist/feature-b.d.ts",
      "import": "./dist/feature-b.js",
      "require": "./dist/feature-b.cjs"
    }
  },
  "sideEffects": false,
  "files": ["dist"]
}
```

## Common Pitfalls

### 1. Re-exporting Types

```typescript
// BAD: Creates dual import paths
// @vipr/core/index.ts
export type { ITechnologyPlugin } from '@vipr/common';

// GOOD: Let consumers import directly
// Consumers:
import type { ITechnologyPlugin } from '@vipr/common/types/plugin';
```

### 2. Mixing Concerns in Barrels

```typescript
// BAD: Types + runtime + side effects
export * from './types';
export * from './utils';
export * from './constants';
export { init } from './init'; // Has side effects!

// GOOD: Separate barrels
// types/index.ts - types only
// utils/index.ts - utilities only
// constants/index.ts - constants only
// index.ts - main entry (minimal)
```

### 3. Deep Barrel Chains

```typescript
// BAD: 5-level chain
// a/index.ts
export * from './b';
// b/index.ts
export * from './c';
// c/index.ts
export * from './d';
// d/index.ts
export * from './e';
// e/index.ts
export { Thing } from './thing';

// GOOD: Flat structure
// index.ts
export { Thing } from './internal/thing';
```

## Verification Commands

### Check Bundle Size

```bash
# Build and analyze
cd clients/cli
pnpm build
pnpm exec rollup-plugin-visualizer dist/index.js

# Check compressed size
gzip -c dist/index.js | wc -c
brotli -c dist/index.js | wc -c
```

### Check Tree Shaking

```bash
# Build with tree-shaking enabled
rollup -c --treeshake

# Verify unused exports are removed
grep "unused_function" dist/index.js
# Should return nothing if tree-shaken
```

### Check Import Paths

```bash
# Find all @vipr imports
grep -r "from '@vipr/" clients/cli/src

# Find barrel imports
grep -r "from '@vipr/common'" clients/cli/src

# Find optimal imports
grep -r "from '@vipr/.*/.*'" clients/cli/src
```

## Success Metrics

### Target Outcomes

| Metric            | Current   | Target              | Status      |
| ----------------- | --------- | ------------------- | ----------- |
| CLI bundle size   | 2.7MB     | `&lt;1`MB           | Not started |
| VSCode bundle     | 3.0MB     | `&lt;500`KB initial | Not started |
| VSCode activation | 300-500ms | `&lt;100`ms         | Not started |
| Type resolution   | Slow      | Fast                | Not started |
| Build time        | 15s       | `&lt;8`s            | Not started |

### KPIs to Track

1. **Bundle Size**: Total JS bundle size (gzipped)
2. **Activation Time**: Time from import to ready
3. **Build Time**: Turborepo build duration
4. **Type Check Time**: tsc --noEmit duration
5. **Import Count**: Number of imports per file

## Next Steps

1. **Run bundle analysis**: `cd clients/cli && pnpm analyze`
2. **Review full document**: `docs/monorepo-barrel-exports-analysis.md`
3. **Start Phase 1**: Implement ESM builds
4. **Set up monitoring**: Add bundle size CI checks
5. **Plan migration**: Schedule Phase 2-3 work

## Resources

- [Full Analysis Document](./monorepo-barrel-exports-analysis.md)
- [Module Resolution Diagram](./monorepo-barrel-exports-analysis.md#module-resolution-chain)
- [Migration Guide](./monorepo-barrel-exports-analysis.md#migration-guide-for-consumers)
- [Implementation Timeline](./monorepo-barrel-exports-analysis.md#rollout-plan)
