# Barrel Export Strategy

This document defines the barrel export architecture used across the Vipr monorepo.

## Philosophy

Barrel exports are used strategically to:

1. Provide clear public APIs for packages
2. Separate public from internal exports
3. Organize related exports by domain
4. Enable tree-shaking through proper module boundaries

## Anti-Patterns to Avoid

- **Circular Dependencies**: Never create import cycles between barrel files
- **Deep Nesting**: Avoid barrels that import from other barrels more than 2 levels deep
- **Wildcard Re-exports in Public APIs**: Use selective exports in public APIs, wildcards only for internal exports
- **Direct Subpath Imports**: Consumers should import from package roots or documented subpaths only

## Package Structure Patterns

### Pattern 1: Shared Library (packages/common)

**Structure:**

```
packages/common/
├── src/
│   ├── index.ts              # Main public API (selective exports)
│   ├── types/
│   │   ├── core/index.ts     # Domain-specific type exports
│   │   ├── plugin/index.ts   # Domain-specific type exports
│   │   └── ...
│   ├── utils/
│   │   ├── scoring.ts        # Individual utility modules
│   │   ├── ast-helpers.ts
│   │   └── ...
│   └── constants/
│       └── index.ts          # Constant exports
└── package.json
```

**Export Strategy:**

- `index.ts`: Main barrel that directly imports from source files (not from intermediate barrels)
- Intermediate barrels exist only for organizational purposes
- `package.json` exports only the main entry point

**Example package.json:**

```json
{
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "default": "./dist/index.js"
    },
    "./constants": {
      "types": "./dist/constants/index.d.ts",
      "default": "./dist/constants/index.js"
    }
  }
}
```

### Pattern 2: Analyzer Plugin (analyzers/react, analyzers/core)

**Structure:**

```
analyzers/react/
├── src/
│   ├── index.ts              # Public API (minimal, stable exports)
│   ├── internal.ts           # Internal exports (wildcard re-exports)
│   ├── plugin.ts             # Plugin implementation
│   ├── analyses/
│   │   ├── index.ts          # Analysis barrel
│   │   ├── security-analysis.ts
│   │   └── ...
│   ├── presenters/
│   │   ├── index.ts          # Presenter barrel
│   │   └── ...
│   ├── types/
│   │   ├── index.ts          # Type barrel
│   │   └── ...
│   └── utils/
│       ├── index.ts          # Utility barrel
│       └── ...
└── package.json
```

**Export Strategy:**

- `index.ts`: Public API - only the plugin class and key types
- `internal.ts`: Wildcard re-exports from all domain barrels
- Domain barrels (`analyses/index.ts`, etc.): Organize related exports
- `package.json` exports both main and internal entry points

**Example index.ts (Public API):**

```typescript
// ============================================================================
// Public API (Stable)
// ============================================================================

export { ReactAnalyzerPlugin } from './plugin';

export type { ReactComplexityResult, ComponentAnalysis } from './types';
```

**Example internal.ts (Internal Exports):**

```typescript
/**
 * Internal Exports
 *
 * NOT part of the stable public API.
 * Used by:
 * - Presenter packages
 * - Testing utilities
 * - Advanced integrations
 */

// Re-export all type definitions
export type * from './types';

// Re-export all analyses
export * from './analyses';

// Re-export utilities
export * from './utils';

// Re-export constants
export * from './constants';
```

**Example package.json:**

```json
{
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "default": "./dist/index.js"
    },
    "./internal": {
      "types": "./dist/internal.d.ts",
      "default": "./dist/internal.js"
    }
  }
}
```

## Import Guidelines

### For Package Consumers

**Good:**

```typescript
// Import from main package entry point
import { ReactAnalyzerPlugin } from '@vipr/react';
import { AnalysisEngine } from '@vipr/engine';
import { scoreToRating, QualityScore } from '@vipr/common';
```

**Bad:**

```typescript
// Don't import from internal paths
import { SecurityAnalysis } from '@vipr/react/analyses';
import { isHookCall } from '@vipr/react/utils';
```

### For Internal Package Development

**Good:**

```typescript
// Use internal exports when needed
import { SecurityAnalysis, AccessibilityAnalysis, isHookCall } from '@vipr/react/internal';
```

**Good:**

```typescript
// Import directly from source files within the same package
import { SecurityAnalysis } from './analyses/security-analysis';
import { isHookCall } from './utils/react-helpers';
```

## Verification

### Check for Circular Dependencies

```bash
npx madge --circular packages/common/src/index.ts
npx madge --circular analyzers/react/src/index.ts
npx madge --circular analyzers/core/src/index.ts
```

All should report: `✔ No circular dependency found!`

### Verify Tree-Shaking

1. Ensure `sideEffects: false` in package.json
2. Use selective exports in public APIs (not wildcard `export *`)
3. Keep intermediate barrels shallow (max 2 levels deep)

## Migration Checklist

When creating a new package or refactoring an existing one:

- [ ] Define public API in main `index.ts` (selective exports)
- [ ] Create `internal.ts` if the package has implementation details used by other packages
- [ ] Organize related exports into domain barrels (types/, utils/, etc.)
- [ ] Configure `package.json` exports field
- [ ] Set `sideEffects: false` in package.json
- [ ] Verify no circular dependencies
- [ ] Document public API surface
- [ ] Add tests for public API exports

## Benefits

1. **Clear Boundaries**: Public vs. internal APIs are explicitly defined
2. **Tree-Shaking**: Selective exports enable better dead code elimination
3. **Maintainability**: Easy to understand what's public vs. internal
4. **Versioning**: Breaking changes are clear when public API changes
5. **Performance**: Reduces import resolution time
6. **Documentation**: Self-documenting API surface

## Related Documentation

- [Monorepo Barrel Exports Analysis](../archive/monorepo-barrel-exports-analysis.md)
- [Plugin Architecture](../plugin-architecture.md)
- [CLAUDE.md](https://github.com/jamesleebaker/vipr/blob/main/CLAUDE.md)
