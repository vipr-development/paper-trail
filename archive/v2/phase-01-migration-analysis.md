# Phase 01: Migration Analysis

**Priority:** High - Core business value  
**Complexity:** High  
**Dependencies:** Phase 00 (Foundation) ✅  
**Status:** Ready for implementation

## Overview

This phase implements comprehensive React migration analysis capabilities, enabling teams to assess migration readiness, detect deprecated APIs, identify class components, and estimate migration effort. This is a high-value feature for enterprise React codebases upgrading between major versions.

All code examples use the `ts-morph` API for AST parsing and traversal.

## Implementation Notes

**Existing Foundation:** A basic React API catalog already exists at `src/constants/react-api-catalog.ts` with:

- `REACT_HOOKS_BY_VERSION` - Basic hooks catalog
- `DEPRECATED_APIS` - Simple deprecated API mappings
- Helper functions for version checking

**Strategy:** This phase will enhance the existing catalog with detailed metadata and implement the version detector and migration analyzer from scratch.

## Business Value

- Reduce React migration risk by identifying blockers early
- Estimate migration effort with data-driven metrics
- Automate detection of deprecated API usage
- Provide actionable codemod recommendations
- Support enterprise teams with large codebases

## Agent Assignments

| Agent                  | Role                                 | Capacity  |
| ---------------------- | ------------------------------------ | --------- |
| react-engineer         | Lead implementer, React expertise    | Primary   |
| typescript-engineer    | Type system design, API contracts    | Secondary |
| vscode-plugin-engineer | Preview extension integration points | Advisory  |

## Execution Strategy

### Milestone 1.1: React Version Detection

**Synchronous Tasks:**

1. Create React API catalog (react-engineer + typescript-engineer)
2. Implement version detector (react-engineer)

**Parallel Tasks:**

- Unit tests for version detection (Sonnet)
- Sample component fixtures (Haiku)

### Milestone 1.2: Deprecated API Detection

**Synchronous Tasks:**

1. Build deprecated API catalog (react-engineer)
2. Implement AST detection logic (react-engineer)
3. Type definitions review (typescript-engineer)

**Parallel Tasks:**

- Legacy lifecycle detection (Sonnet)
- String ref detection (Sonnet)
- PropTypes detection (Sonnet)

### Milestone 1.3: Migration Scoring & Effort

**Synchronous Tasks:**

1. Readiness score algorithm (react-engineer)
2. Effort estimation model (react-engineer)
3. Integration with main analyzer (typescript-engineer)

### Milestone 1.4: Integration & Testing

**Parallel Tasks:**

- Integration tests (Sonnet)
- Schema updates (Haiku)
- Documentation (Haiku)

## Detailed Tasks

### Task 1.1: Enhance React API Catalog

**File:** `src/constants/react-api-catalog.ts` (ENHANCE EXISTING)

**Action:** Replace the existing simple structures with detailed metadata-rich versions. The file already exists with basic mappings - we'll enhance it with comprehensive migration analysis data.

```typescript
/**
 * React API Catalog - Enhanced Version
 *
 * Comprehensive catalog of React APIs with detailed metadata for migration analysis.
 * This REPLACES the existing simple structures with migration-ready data.
 *
 * @module constants/react-api-catalog
 */

// ============================================================================
// Type Definitions
// ============================================================================

/**
 * Detailed API definition for migration analysis
 */
export interface APIDefinition {
  /** API name */
  name: string;
  /** Version introduced */
  introducedIn: string;
  /** Version deprecated (if applicable) */
  deprecatedIn?: string;
  /** Version removed (if applicable) */
  removedIn?: string;
  /** API category */
  category: 'hook' | 'lifecycle' | 'component' | 'utility' | 'dom';
  /** Detection method */
  detection: 'import' | 'usage' | 'jsx' | 'class-method';
}

/**
 * Deprecated API with full migration metadata
 */
export interface DeprecatedAPI {
  name: string;
  deprecatedIn: string;
  removedIn: string | null;
  replacement: string;
  automatable: boolean;
  codemods: string[];
  reason: string;
  migrationGuide: string;
  severity: 'critical' | 'high' | 'medium' | 'low';
}

/**
 * Breaking change between React versions
 */
export interface BreakingChange {
  change: string;
  version: string;
  impact: string;
  action: string;
  automatable: boolean;
  codemods?: string[];
  documentation: string;
}

/**
 * React version type
 */
export type ReactVersion = '16.8.0' | '17.0.0' | '18.0.0' | '19.0.0';

// ============================================================================
// React API Catalog by Version
// ============================================================================

/**
 * React 16.8 APIs (Hooks introduction)
 */
export const REACT_16_8_APIS: APIDefinition[] = [
  { name: 'useState', introducedIn: '16.8.0', category: 'hook', detection: 'usage' },
  { name: 'useEffect', introducedIn: '16.8.0', category: 'hook', detection: 'usage' },
  { name: 'useContext', introducedIn: '16.8.0', category: 'hook', detection: 'usage' },
  { name: 'useReducer', introducedIn: '16.8.0', category: 'hook', detection: 'usage' },
  { name: 'useCallback', introducedIn: '16.8.0', category: 'hook', detection: 'usage' },
  { name: 'useMemo', introducedIn: '16.8.0', category: 'hook', detection: 'usage' },
  { name: 'useRef', introducedIn: '16.8.0', category: 'hook', detection: 'usage' },
  { name: 'useImperativeHandle', introducedIn: '16.8.0', category: 'hook', detection: 'usage' },
  { name: 'useLayoutEffect', introducedIn: '16.8.0', category: 'hook', detection: 'usage' },
  { name: 'useDebugValue', introducedIn: '16.8.0', category: 'hook', detection: 'usage' },
];

/**
 * React 18 APIs (Concurrent features)
 */
export const REACT_18_APIS: APIDefinition[] = [
  { name: 'useId', introducedIn: '18.0.0', category: 'hook', detection: 'usage' },
  { name: 'useDeferredValue', introducedIn: '18.0.0', category: 'hook', detection: 'usage' },
  { name: 'useTransition', introducedIn: '18.0.0', category: 'hook', detection: 'usage' },
  { name: 'useSyncExternalStore', introducedIn: '18.0.0', category: 'hook', detection: 'usage' },
  { name: 'useInsertionEffect', introducedIn: '18.0.0', category: 'hook', detection: 'usage' },
];

/**
 * React 19 APIs (Actions and async features)
 */
export const REACT_19_APIS: APIDefinition[] = [
  { name: 'use', introducedIn: '19.0.0', category: 'hook', detection: 'usage' },
  { name: 'useOptimistic', introducedIn: '19.0.0', category: 'hook', detection: 'usage' },
  { name: 'useActionState', introducedIn: '19.0.0', category: 'hook', detection: 'usage' },
  { name: 'useFormStatus', introducedIn: '19.0.0', category: 'hook', detection: 'usage' },
  { name: 'useFormState', introducedIn: '19.0.0', category: 'hook', detection: 'usage' },
];

/**
 * All React APIs combined (for easy lookup)
 */
export const ALL_REACT_APIS: APIDefinition[] = [
  ...REACT_16_8_APIS,
  ...REACT_18_APIS,
  ...REACT_19_APIS,
];

// ============================================================================
// Deprecated APIs with Full Metadata
// ============================================================================

/**
 * Comprehensive deprecated API catalog with migration guidance
 *
 * REPLACES the existing simple DEPRECATED_APIS object
 */
export const DEPRECATED_APIS: DeprecatedAPI[] = [
  // Legacy Lifecycle Methods (Critical)
  {
    name: 'componentWillMount',
    deprecatedIn: '16.3.0',
    removedIn: '17.0.0',
    replacement: 'constructor or componentDidMount',
    automatable: true,
    codemods: ['rename-unsafe-lifecycles'],
    reason: 'Causes issues with async rendering and Suspense',
    migrationGuide: 'https://react.dev/blog/2018/03/27/update-on-async-rendering',
    severity: 'critical',
  },
  {
    name: 'componentWillReceiveProps',
    deprecatedIn: '16.3.0',
    removedIn: '17.0.0',
    replacement: 'getDerivedStateFromProps or componentDidUpdate',
    automatable: true,
    codemods: ['rename-unsafe-lifecycles'],
    reason: 'Causes issues with async rendering and Suspense',
    migrationGuide: 'https://react.dev/blog/2018/03/27/update-on-async-rendering',
    severity: 'critical',
  },
  {
    name: 'componentWillUpdate',
    deprecatedIn: '16.3.0',
    removedIn: '17.0.0',
    replacement: 'getSnapshotBeforeUpdate or componentDidUpdate',
    automatable: true,
    codemods: ['rename-unsafe-lifecycles'],
    reason: 'Causes issues with async rendering and Suspense',
    migrationGuide: 'https://react.dev/blog/2018/03/27/update-on-async-rendering',
    severity: 'critical',
  },

  // UNSAFE Lifecycle Aliases (High)
  {
    name: 'UNSAFE_componentWillMount',
    deprecatedIn: '16.3.0',
    removedIn: null,
    replacement: 'constructor or componentDidMount',
    automatable: false,
    codemods: [],
    reason: 'Temporary alias for deprecated lifecycle',
    migrationGuide: 'https://react.dev/reference/react/Component#unsafe_componentwillmount',
    severity: 'high',
  },
  {
    name: 'UNSAFE_componentWillReceiveProps',
    deprecatedIn: '16.3.0',
    removedIn: null,
    replacement: 'getDerivedStateFromProps or componentDidUpdate',
    automatable: false,
    codemods: [],
    reason: 'Temporary alias for deprecated lifecycle',
    migrationGuide: 'https://react.dev/reference/react/Component#unsafe_componentwillreceiveprops',
    severity: 'high',
  },
  {
    name: 'UNSAFE_componentWillUpdate',
    deprecatedIn: '16.3.0',
    removedIn: null,
    replacement: 'getSnapshotBeforeUpdate or componentDidUpdate',
    automatable: false,
    codemods: [],
    reason: 'Temporary alias for deprecated lifecycle',
    migrationGuide: 'https://react.dev/reference/react/Component#unsafe_componentwillupdate',
    severity: 'high',
  },

  // DOM and Refs (High)
  {
    name: 'findDOMNode',
    deprecatedIn: '16.3.0',
    removedIn: null,
    replacement: 'refs with useRef or createRef',
    automatable: false,
    codemods: [],
    reason: 'Breaks encapsulation and prevents React optimizations',
    migrationGuide: 'https://react.dev/reference/react-dom/findDOMNode',
    severity: 'high',
  },
  {
    name: 'String Refs',
    deprecatedIn: '16.3.0',
    removedIn: null,
    replacement: 'Callback refs or createRef/useRef',
    automatable: true,
    codemods: ['string-ref-to-callback-ref'],
    reason: 'String refs have performance and flexibility issues',
    migrationGuide: 'https://react.dev/reference/react/Component#refs',
    severity: 'medium',
  },

  // Legacy Context (High)
  {
    name: 'Legacy Context',
    deprecatedIn: '16.3.0',
    removedIn: null,
    replacement: 'React.createContext API',
    automatable: false,
    codemods: [],
    reason: 'New Context API is more efficient and predictable',
    migrationGuide: 'https://react.dev/reference/react/createContext',
    severity: 'high',
  },
  {
    name: 'contextTypes',
    deprecatedIn: '16.3.0',
    removedIn: '18.0.0',
    replacement: 'React.createContext and useContext',
    automatable: false,
    codemods: [],
    reason: 'Legacy context API is inefficient and error-prone',
    migrationGuide: 'https://react.dev/reference/react/createContext',
    severity: 'high',
  },
  {
    name: 'childContextTypes',
    deprecatedIn: '16.3.0',
    removedIn: '18.0.0',
    replacement: 'Context.Provider',
    automatable: false,
    codemods: [],
    reason: 'Legacy context API is inefficient',
    migrationGuide: 'https://react.dev/reference/react/createContext',
    severity: 'high',
  },
  {
    name: 'getChildContext',
    deprecatedIn: '16.3.0',
    removedIn: '18.0.0',
    replacement: 'Context.Provider',
    automatable: false,
    codemods: [],
    reason: 'Legacy context API is inefficient',
    migrationGuide: 'https://react.dev/reference/react/createContext',
    severity: 'high',
  },

  // Props and Types (Medium)
  {
    name: 'defaultProps',
    deprecatedIn: '18.3.0',
    removedIn: null,
    replacement: 'ES6 default parameters or destructuring defaults',
    automatable: true,
    codemods: ['default-props-to-default-parameters'],
    reason: 'Moving to standard ES6 defaults',
    migrationGuide: 'https://react.dev/blog/2024/04/25/react-19-upgrade-guide',
    severity: 'medium',
  },
  {
    name: 'propTypes',
    deprecatedIn: '15.5.0',
    removedIn: null,
    replacement: 'TypeScript or Flow for type checking',
    automatable: false,
    codemods: [],
    reason: 'Type systems provide better static analysis',
    migrationGuide: 'https://react.dev/reference/react/Component#static-proptypes',
    severity: 'low',
  },

  // ReactDOM APIs (High)
  {
    name: 'ReactDOM.render',
    deprecatedIn: '18.0.0',
    removedIn: '19.0.0',
    replacement: 'ReactDOM.createRoot().render()',
    automatable: true,
    codemods: ['react-18-root-api'],
    reason: 'Required for React 18+ concurrent features',
    migrationGuide:
      'https://react.dev/blog/2022/03/08/react-18-upgrade-guide#updates-to-client-rendering-apis',
    severity: 'high',
  },
  {
    name: 'ReactDOM.hydrate',
    deprecatedIn: '18.0.0',
    removedIn: '19.0.0',
    replacement: 'ReactDOM.hydrateRoot()',
    automatable: true,
    codemods: ['react-18-root-api'],
    reason: 'Required for React 18+ concurrent features',
    migrationGuide:
      'https://react.dev/blog/2022/03/08/react-18-upgrade-guide#updates-to-client-rendering-apis',
    severity: 'high',
  },
  {
    name: 'ReactDOM.unmountComponentAtNode',
    deprecatedIn: '18.0.0',
    removedIn: '19.0.0',
    replacement: 'root.unmount()',
    automatable: true,
    codemods: ['react-18-root-api'],
    reason: 'New root API is required for concurrent features',
    migrationGuide:
      'https://react.dev/blog/2022/03/08/react-18-upgrade-guide#updates-to-client-rendering-apis',
    severity: 'high',
  },

  // Factory and Test Utilities
  {
    name: 'React.createFactory',
    deprecatedIn: '16.8.0',
    removedIn: '19.0.0',
    replacement: 'JSX or React.createElement',
    automatable: true,
    codemods: ['create-factory-to-jsx'],
    reason: 'JSX is now the standard way to create elements',
    migrationGuide: 'https://react.dev/warnings/legacy-factories',
    severity: 'medium',
  },
  {
    name: 'react-test-renderer/shallow',
    deprecatedIn: '18.0.0',
    removedIn: '19.0.0',
    replacement: '@testing-library/react',
    automatable: false,
    codemods: [],
    reason: 'Shallow rendering is discouraged in favor of full rendering',
    migrationGuide: 'https://testing-library.com/docs/react-testing-library/migrate-from-enzyme',
    severity: 'medium',
  },
];

// ============================================================================
// React 19 Breaking Changes
// ============================================================================

/**
 * React 19 breaking changes with detailed impact analysis
 */
export const REACT_19_BREAKING_CHANGES: BreakingChange[] = [
  {
    change: 'Errors in render are not re-thrown',
    version: '19.0.0',
    impact: 'Error boundaries behave differently, errors are reported but not re-thrown',
    action: 'Review error boundary implementations and error handling patterns',
    automatable: false,
    documentation:
      'https://react.dev/blog/2024/04/25/react-19-upgrade-guide#errors-in-render-are-not-rethrown',
  },
  {
    change: 'Removed deprecated React APIs',
    version: '19.0.0',
    impact: 'propTypes and defaultProps removed for function components',
    action: 'Migrate to TypeScript or default parameters',
    automatable: true,
    codemods: ['default-props-to-default-parameters'],
    documentation:
      'https://react.dev/blog/2024/04/25/react-19-upgrade-guide#removed-deprecated-apis',
  },
  {
    change: 'Ref cleanup functions',
    version: '19.0.0',
    impact: 'Ref callbacks can now return cleanup functions',
    action: 'Review ref callback patterns for compatibility',
    automatable: false,
    documentation:
      'https://react.dev/blog/2024/04/25/react-19-upgrade-guide#cleanup-functions-for-refs',
  },
  {
    change: 'useContext reads context eagerly',
    version: '19.0.0',
    impact: 'May expose bugs in conditional context usage',
    action: 'Review conditional useContext calls',
    automatable: false,
    documentation:
      'https://react.dev/blog/2024/04/25/react-19-upgrade-guide#usecontext-reads-context-value-eagerly',
  },
  {
    change: 'Stricter hydration warnings',
    version: '19.0.0',
    impact: 'Hydration mismatches are now errors instead of warnings',
    action: 'Fix all hydration warnings before upgrading',
    automatable: false,
    documentation:
      'https://react.dev/blog/2024/04/25/react-19-upgrade-guide#stricter-hydration-errors',
  },
  {
    change: 'String refs removed',
    version: '19.0.0',
    impact: 'String refs (ref="myRef") no longer work',
    action: 'Convert all string refs to callback refs or createRef/useRef',
    automatable: true,
    codemods: ['string-ref-to-callback-ref'],
    documentation: 'https://react.dev/blog/2024/04/25/react-19-upgrade-guide#removed-string-refs',
  },
  {
    change: 'Legacy Context removed',
    version: '19.0.0',
    impact: 'contextTypes, childContextTypes, getChildContext no longer work',
    action: 'Migrate to React.createContext API',
    automatable: false,
    documentation:
      'https://react.dev/blog/2024/04/25/react-19-upgrade-guide#removed-legacy-context',
  },
  {
    change: 'Module pattern factories removed',
    version: '19.0.0',
    impact: 'Functions returning components no longer work',
    action: 'Convert to standard class or function components',
    automatable: false,
    documentation:
      'https://react.dev/blog/2024/04/25/react-19-upgrade-guide#removed-module-pattern',
  },
];

// ============================================================================
// Helper Functions
// ============================================================================

/**
 * Get minimum React version required for an API
 */
export function getMinReactVersion(apiName: string): string | null {
  const api = ALL_REACT_APIS.find(a => a.name === apiName);
  return api?.introducedIn ?? null;
}

/**
 * Check if an API is deprecated
 */
export function isDeprecatedAPI(apiName: string): boolean {
  return DEPRECATED_APIS.some(api => api.name === apiName);
}

/**
 * Get deprecation details for an API
 */
export function getDeprecationInfo(apiName: string): DeprecatedAPI | null {
  return DEPRECATED_APIS.find(api => api.name === apiName) ?? null;
}

/**
 * Get all APIs available in a specific React version
 */
export function getAPIsForVersion(version: ReactVersion): APIDefinition[] {
  return ALL_REACT_APIS.filter(api => {
    const introduced = api.introducedIn;
    return introduced `<=` version;
  });
}

/**
 * Check if a version requires migration for deprecated APIs
 */
export function needsMigration(currentVersion: string, targetVersion: string): boolean {
  const deprecatedInTarget = DEPRECATED_APIS.filter(
    api => api.removedIn && api.removedIn `<=` targetVersion
  );
  return deprecatedInTarget.length > 0;
}
```

### Task 1.2: Version Detector Implementation

**File:** `src/utils/react-version.ts`

```typescript
import { Project, SourceFile, Node, SyntaxKind } from 'ts-morph';
import { REACT_19_APIS, REACT_18_APIS, APIDefinition } from '../constants/react-api-catalog';

export interface ReactVersionInfo {
  /** Detected React version */
  detectedVersion: string | null;
  /** Version range (e.g., ">=18.0.0") */
  versionRange: string | null;
  /** Confidence in detection */
  confidenceLevel: 'high' | 'medium' | 'low';
  /** How version was detected */
  detectionMethod: 'packageJson' | 'imports' | 'apiUsage' | 'mixed';
  /** Evidence supporting detection */
  evidence: VersionEvidence[];
}

export interface VersionEvidence {
  /** What was detected */
  indicator: string;
  /** Minimum version required */
  minVersion: string;
  /** Confidence score (0-1) */
  confidence: number;
  /** Code location if applicable */
  location?: { line: number; column: number };
}

/**
 * Detects React version from source code analysis using ts-morph
 */
export class ReactVersionDetector {
  private apiCatalog: Map<string, APIDefinition>;

  constructor() {
    this.apiCatalog = this.buildApiCatalog();
  }

  /**
   * Detect React version from SourceFile
   */
  detectFromSourceFile(sourceFile: SourceFile): ReactVersionInfo {
    const evidence: VersionEvidence[] = [];

    // Check imports
    sourceFile.getImportDeclarations().forEach(importDecl => {
      const moduleSpecifier = importDecl.getModuleSpecifierValue();

      if (moduleSpecifier === 'react') {
        importDecl.getNamedImports().forEach(namedImport => {
          const apiName = namedImport.getName();
          const apiDef = this.apiCatalog.get(apiName);

          if (apiDef) {
            evidence.push({
              indicator: `import { ${apiName} }`,
              minVersion: apiDef.introducedIn,
              confidence: 0.95,
              location: {
                line: namedImport.getStartLineNumber(),
                column:
                  namedImport.getStart() -
                  sourceFile.getLineStarts()[namedImport.getStartLineNumber() - 1],
              },
            });
          }
        });
      }
    });

    // Check hook usage
    sourceFile.forEachDescendant(node => {
      if (Node.isCallExpression(node)) {
        const expr = node.getExpression();

        if (Node.isIdentifier(expr)) {
          const hookName = expr.getText();

          if (hookName.startsWith('use')) {
            const apiDef = this.apiCatalog.get(hookName);

            if (apiDef) {
              evidence.push({
                indicator: `${hookName}()`,
                minVersion: apiDef.introducedIn,
                confidence: 0.9,
                location: {
                  line: node.getStartLineNumber(),
                  column: 0,
                },
              });
            }
          }
        }
      }

      // Check for "use client" directive (React 19 / RSC)
      if (Node.isExpressionStatement(node)) {
        const expr = node.getExpression();
        if (Node.isStringLiteral(expr) && expr.getLiteralValue() === 'use client') {
          evidence.push({
            indicator: '"use client" directive',
            minVersion: '18.0.0',
            confidence: 0.85,
            location: {
              line: node.getStartLineNumber(),
              column: 0,
            },
          });
        }
      }
    });

    return this.analyzeEvidence(evidence);
  }

  /**
   * Detect from package.json
   */
  detectFromPackageJson(packageJson: {
    dependencies?: Record<string, string>;
    devDependencies?: Record<string, string>;
  }): ReactVersionInfo {
    const reactVersion = packageJson.dependencies?.react ?? packageJson.devDependencies?.react;

    if (!reactVersion) {
      return this.createUnknownVersion();
    }

    const cleanVersion = this.parseVersionString(reactVersion);

    return {
      detectedVersion: cleanVersion,
      versionRange: reactVersion,
      confidenceLevel: 'high',
      detectionMethod: 'packageJson',
      evidence: [
        {
          indicator: 'package.json dependencies.react',
          minVersion: cleanVersion,
          confidence: 1.0,
        },
      ],
    };
  }

  /**
   * Combined detection from multiple sources
   */
  detect(
    sourceFile: SourceFile,
    packageJson?: {
      dependencies?: Record<string, string>;
      devDependencies?: Record<string, string>;
    }
  ): ReactVersionInfo {
    const astResult = this.detectFromSourceFile(sourceFile);

    if (packageJson) {
      const pkgResult = this.detectFromPackageJson(packageJson);

      if (pkgResult.detectedVersion) {
        // Package.json is authoritative, but enrich with API evidence
        return {
          ...pkgResult,
          detectionMethod: 'mixed',
          evidence: [...pkgResult.evidence, ...astResult.evidence],
        };
      }
    }

    return astResult;
  }

  private buildApiCatalog(): Map<string, APIDefinition> {
    const catalog = new Map<string, APIDefinition>();

    [...REACT_19_APIS, ...REACT_18_APIS].forEach(api => {
      catalog.set(api.name, api);
    });

    return catalog;
  }

  private analyzeEvidence(evidence: VersionEvidence[]): ReactVersionInfo {
    if (evidence.length === 0) {
      return this.createUnknownVersion();
    }

    // Find highest required version
    const versions = evidence.map(e => e.minVersion);
    const highestVersion = this.getHighestVersion(versions);

    // Calculate overall confidence
    const avgConfidence = evidence.reduce((sum, e) => sum + e.confidence, 0) / evidence.length;

    const confidenceLevel: 'high' | 'medium' | 'low' =
      avgConfidence >= 0.9 ? 'high' : avgConfidence >= 0.7 ? 'medium' : 'low';

    return {
      detectedVersion: highestVersion,
      versionRange: `>=${highestVersion}`,
      confidenceLevel,
      detectionMethod: 'apiUsage',
      evidence,
    };
  }

  private getHighestVersion(versions: string[]): string {
    return (
      versions.sort((a, b) => {
        const aParts = a.split('.').map(Number);
        const bParts = b.split('.').map(Number);

        for (let i = 0; i < 3; i++) {
          if ((aParts[i] ?? 0) !== (bParts[i] ?? 0)) {
            return (bParts[i] ?? 0) - (aParts[i] ?? 0);
          }
        }
        return 0;
      })[0] ?? '16.8.0'
    );
  }

  private parseVersionString(version: string): string {
    // Remove semver prefixes: ^, ~, >=, >, <, `<=`, =
    return version.replace(/^[\^~>=<]+/, '').split('-')[0];
  }

  private createUnknownVersion(): ReactVersionInfo {
    return {
      detectedVersion: null,
      versionRange: null,
      confidenceLevel: 'low',
      detectionMethod: 'apiUsage',
      evidence: [],
    };
  }
}
```

### Task 1.3: Migration Analyzer Implementation

**File:** `src/analyzers/migration-analyzer.ts`

```typescript
import { SourceFile, Node, SyntaxKind } from 'ts-morph';
import { BaseAnalyzer, BaseAnalyzerResult } from './base-analyzer-tsmorph';
import { ReactVersionDetector, ReactVersionInfo } from '../utils/react-version';
import {
  DEPRECATED_APIS,
  REACT_19_BREAKING_CHANGES,
  DeprecatedAPI,
  BreakingChange,
} from '../constants/react-api-catalog';
import type { CodeLocation } from '../types';

export interface MigrationBlocker {
  type: 'deprecated-api' | 'breaking-change' | 'class-component';
  description: string;
  severity: 'critical' | 'high' | 'medium' | 'low';
  location: CodeLocation;
  replacement?: string;
  codemod?: string;
  migrationGuide?: string;
}

export interface MigrationEffort {
  hours: number;
  complexity: 'trivial' | 'simple' | 'moderate' | 'complex' | 'extensive';
  breakdown: {
    classComponents: number;
    deprecatedLifecycles: number;
    stringRefs: number;
    legacyContext: number;
    propTypes: number;
    other: number;
  };
}

export interface MigrationResult extends BaseAnalyzerResult {
  versionInfo: ReactVersionInfo;
  readinessScore: number;
  targetVersion: string;
  blockers: MigrationBlocker[];
  warnings: MigrationBlocker[];
  estimatedEffort: MigrationEffort;
  codemods: string[];
  classComponentCount: number;
  functionalComponentCount: number;
}

/**
 * Migration Analyzer using ts-morph
 *
 * Analyzes React components for migration readiness and identifies
 * deprecated APIs, class components, and other migration blockers.
 */
export class MigrationAnalyzer extends BaseAnalyzer<MigrationResult> {
  private versionDetector: ReactVersionDetector;
  private targetVersion: string;

  constructor(sourceFile: SourceFile, targetVersion = '19.0.0') {
    super(sourceFile);
    this.versionDetector = new ReactVersionDetector();
    this.targetVersion = targetVersion;
  }

  protected analyzeAST(): Omit<MigrationResult, 'metadata'> {
    const versionInfo = this.versionDetector.detectFromSourceFile(this.sourceFile);
    const blockers: MigrationBlocker[] = [];
    const warnings: MigrationBlocker[] = [];
    const codemods = new Set<string>();

    let classComponentCount = 0;
    let functionalComponentCount = 0;

    // Detect class components and deprecated lifecycles
    this.sourceFile.getClasses().forEach(classDecl => {
      const heritage = classDecl.getHeritageClauses();
      const isReactComponent = heritage.some(
        clause =>
          clause.getText().includes('Component') || clause.getText().includes('PureComponent')
      );

      if (isReactComponent) {
        classComponentCount++;

        // Check for deprecated lifecycle methods
        classDecl.getMethods().forEach(method => {
          const methodName = method.getName();
          const deprecatedApi = DEPRECATED_APIS.find(api => api.name === methodName);

          if (deprecatedApi) {
            const blocker: MigrationBlocker = {
              type: 'deprecated-api',
              description: `${methodName} is deprecated since React ${deprecatedApi.deprecatedIn}`,
              severity: deprecatedApi.severity,
              location: this.getNodeLocation(method),
              replacement: deprecatedApi.replacement,
              codemod: deprecatedApi.codemods[0],
              migrationGuide: deprecatedApi.migrationGuide,
            };

            if (deprecatedApi.severity === 'critical' || deprecatedApi.severity === 'high') {
              blockers.push(blocker);
            } else {
              warnings.push(blocker);
            }

            deprecatedApi.codemods.forEach(cm => codemods.add(cm));
          }
        });
      }
    });

    // Count functional components
    this.sourceFile.getFunctions().forEach(fn => {
      if (this.isFunctionalComponent(fn)) {
        functionalComponentCount++;
      }
    });

    this.sourceFile.getVariableDeclarations().forEach(varDecl => {
      const init = varDecl.getInitializer();
      if (init && (Node.isArrowFunction(init) || Node.isFunctionExpression(init))) {
        if (this.isFunctionalComponent(init)) {
          functionalComponentCount++;
        }
      }
    });

    // Detect string refs
    this.sourceFile.forEachDescendant(node => {
      if (Node.isJsxAttribute(node)) {
        const name = node.getNameNode();
        if (Node.isIdentifier(name) && name.getText() === 'ref') {
          const initializer = node.getInitializer();
          if (initializer && Node.isStringLiteral(initializer)) {
            blockers.push({
              type: 'deprecated-api',
              description: 'String refs are deprecated',
              severity: 'medium',
              location: this.getNodeLocation(node),
              replacement: 'useRef or createRef',
              codemod: 'string-ref-to-callback-ref',
            });
            codemods.add('string-ref-to-callback-ref');
          }
        }
      }

      // Detect findDOMNode
      if (Node.isCallExpression(node)) {
        const expr = node.getExpression();
        if (Node.isIdentifier(expr) && expr.getText() === 'findDOMNode') {
          blockers.push({
            type: 'deprecated-api',
            description: 'findDOMNode is deprecated',
            severity: 'high',
            location: this.getNodeLocation(node),
            replacement: 'useRef or createRef',
          });
        }
      }
    });

    // Calculate readiness score
    const readinessScore = this.calculateReadinessScore(
      blockers,
      warnings,
      classComponentCount,
      functionalComponentCount
    );

    // Estimate effort
    const estimatedEffort = this.estimateEffort(blockers, classComponentCount);

    return {
      score: readinessScore,
      versionInfo,
      readinessScore,
      targetVersion: this.targetVersion,
      blockers,
      warnings,
      estimatedEffort,
      codemods: Array.from(codemods),
      classComponentCount,
      functionalComponentCount,
    };
  }

  private isFunctionalComponent(node: Node): boolean {
    let hasJsx = false;
    node.forEachDescendant(desc => {
      if (
        Node.isJsxElement(desc) ||
        Node.isJsxSelfClosingElement(desc) ||
        Node.isJsxFragment(desc)
      ) {
        hasJsx = true;
      }
    });
    return hasJsx;
  }

  private calculateReadinessScore(
    blockers: MigrationBlocker[],
    warnings: MigrationBlocker[],
    classCount: number,
    functionalCount: number
  ): number {
    let score = 100;

    // Deduct for blockers
    blockers.forEach(blocker => {
      switch (blocker.severity) {
        case 'critical':
          score -= 20;
          break;
        case 'high':
          score -= 10;
          break;
        case 'medium':
          score -= 5;
          break;
        case 'low':
          score -= 2;
          break;
      }
    });

    // Deduct for warnings (less severely)
    warnings.forEach(() => {
      score -= 1;
    });

    // Deduct for class components
    const totalComponents = classCount + functionalCount;
    if (totalComponents > 0) {
      const classRatio = classCount / totalComponents;
      score -= classRatio * 20;
    }

    return Math.max(0, Math.min(100, Math.round(score)));
  }

  private estimateEffort(blockers: MigrationBlocker[], classCount: number): MigrationEffort {
    const breakdown = {
      classComponents: classCount * 2, // 2 hours per class component
      deprecatedLifecycles: blockers.filter(b => b.description.includes('lifecycle')).length * 1,
      stringRefs: blockers.filter(b => b.description.includes('String ref')).length * 0.5,
      legacyContext: blockers.filter(b => b.description.includes('context')).length * 3,
      propTypes: blockers.filter(b => b.description.includes('propTypes')).length * 0.25,
      other: 0,
    };

    const totalHours = Object.values(breakdown).reduce((sum, h) => sum + h, 0);

    const complexity: MigrationEffort['complexity'] =
      totalHours < 1
        ? 'trivial'
        : totalHours < 4
          ? 'simple'
          : totalHours < 8
            ? 'moderate'
            : totalHours < 24
              ? 'complex'
              : 'extensive';

    return {
      hours: Math.round(totalHours * 10) / 10,
      complexity,
      breakdown,
    };
  }
}

/**
 * Analyze migration readiness for source code
 */
export function analyzeMigration(source: string, targetVersion = '19.0.0'): MigrationResult {
  const { Project } = require('ts-morph');
  const project = new Project({ useInMemoryFileSystem: true });
  const sourceFile = project.createSourceFile('temp.tsx', source);
  const analyzer = new MigrationAnalyzer(sourceFile, targetVersion);
  return analyzer.analyze();
}
```

### Task 1.4: Sample Test Components

**Directory:** `src/sample-components/migration/` (CREATE NEW)

Create sample components for testing migration detection. These will be used in tests and as examples.

**File:** `src/sample-components/migration/ClassComponentLegacy.tsx`

```typescript
import React from 'react';

interface Props {
  name: string;
  count: number;
}

interface State {
  message: string;
}

/**
 * Legacy class component with deprecated lifecycle methods
 * This should trigger critical migration warnings
 */
export class ClassComponentLegacy extends React.Component<Props, State> {
  constructor(props: Props) {
    super(props);
    this.state = { message: '' };
  }

  // DEPRECATED: Removed in React 17
  componentWillMount() {
    console.log('Component will mount');
    this.setState({ message: 'Mounting...' });
  }

  // DEPRECATED: Removed in React 17
  componentWillReceiveProps(nextProps: Props) {
    if (nextProps.count !== this.props.count) {
      console.log('Props changing');
    }
  }

  // DEPRECATED: Removed in React 17
  componentWillUpdate(nextProps: Props, nextState: State) {
    console.log('Component will update');
  }

  componentDidMount() {
    this.setState({ message: 'Mounted!' });
  }

  render() {
    return (
      <div>
        <h1>{this.props.name}</h1>
        <p>Count: {this.props.count}</p>
        <p>{this.state.message}</p>
      </div>
    );
  }
}
```

**File:** `src/sample-components/migration/StringRefsExample.tsx`

```typescript
import React from 'react';

/**
 * Component using deprecated string refs
 * Should trigger medium severity warnings
 */
export class StringRefsExample extends React.Component {
  componentDidMount() {
    // String ref access - deprecated pattern
    const input = this.refs.myInput as HTMLInputElement;
    if (input) {
      input.focus();
    }
  }

  handleClick = () => {
    const input = this.refs.myInput as HTMLInputElement;
    alert(input.value);
  };

  render() {
    return (
      <div>
        {/* String ref - deprecated in React 16.3 */}
        <input ref="myInput" type="text" />
        <button onClick={this.handleClick}>Show Value</button>
      </div>
    );
  }
}

/**
 * Modern equivalent using useRef
 */
export function StringRefsModern() {
  const inputRef = React.useRef<HTMLInputElement>(null);

  React.useEffect(() => {
    inputRef.current?.focus();
  }, []);

  const handleClick = () => {
    if (inputRef.current) {
      alert(inputRef.current.value);
    }
  };

  return (
    <div>
      <input ref={inputRef} type="text" />
      <button onClick={handleClick}>Show Value</button>
    </div>
  );
}
```

**File:** `src/sample-components/migration/PropTypesExample.tsx`

```typescript
import React from 'react';
import PropTypes from 'prop-types';

/**
 * Component using PropTypes (deprecated for function components)
 * Should trigger low severity warning
 */
export function PropTypesExample({ name, age, onUpdate }) {
  return (
    <div>
      <h2>{name}</h2>
      <p>Age: {age}</p>
      <button onClick={onUpdate}>Update</button>
    </div>
  );
}

// PropTypes on function component - deprecated in React 15.5
PropTypesExample.propTypes = {
  name: PropTypes.string.isRequired,
  age: PropTypes.number,
  onUpdate: PropTypes.func,
};

// defaultProps on function component - deprecated in React 18.3
PropTypesExample.defaultProps = {
  age: 0,
  onUpdate: () => {},
};

/**
 * Modern TypeScript equivalent
 */
interface ModernProps {
  name: string;
  age?: number;
  onUpdate?: () => void;
}

export function PropTypesModern({
  name,
  age = 0,
  onUpdate = () => {}
}: ModernProps) {
  return (
    <div>
      <h2>{name}</h2>
      <p>Age: {age}</p>
      <button onClick={onUpdate}>Update</button>
    </div>
  );
}
```

**File:** `src/sample-components/migration/React19Features.tsx`

```typescript
import { use, useOptimistic, useActionState } from 'react';

/**
 * Component using React 19 features
 * Should be detected as React 19+ codebase
 */

// React 19: use() hook for promises
export function UserProfile({ userPromise }: { userPromise: Promise<{ name: string }> }) {
  const user = use(userPromise);

  return <div>User: {user.name}</div>;
}

// React 19: useOptimistic for optimistic UI updates
export function TodoList({ todos }: { todos: string[] }) {
  const [optimisticTodos, addOptimisticTodo] = useOptimistic(
    todos,
    (state, newTodo: string) => [...state, newTodo]
  );

  async function handleAdd(formData: FormData) {
    const todo = formData.get('todo') as string;
    addOptimisticTodo(todo);
    await fetch('/api/todos', {
      method: 'POST',
      body: JSON.stringify({ todo })
    });
  }

  return (
    <div>
      <ul>
        {optimisticTodos.map((todo, i) => (
          <li key={i}>{todo}</li>
        ))}
      </ul>
      <form action={handleAdd}>
        <input name="todo" />
        <button type="submit">Add</button>
      </form>
    </div>
  );
}

// React 19: useActionState for form state
export function ContactForm() {
  async function submitAction(prevState: any, formData: FormData) {
    const name = formData.get('name');
    const response = await fetch('/api/contact', {
      method: 'POST',
      body: JSON.stringify({ name }),
    });
    return { success: response.ok };
  }

  const [state, formAction] = useActionState(submitAction, { success: false });

  return (
    <form action={formAction}>
      <input name="name" />
      <button type="submit">Submit</button>
      {state.success && <p>Success!</p>}
    </form>
  );
}
```

**File:** `src/sample-components/migration/ModernComponent.tsx`

```typescript
import { useState, useEffect, useCallback, useMemo, useRef } from 'react';

/**
 * Fully modern React component using best practices
 * Should score 100 on migration readiness
 */

interface ModernComponentProps {
  title: string;
  items: string[];
  onItemClick?: (item: string) => void;
}

export function ModernComponent({
  title,
  items,
  onItemClick
}: ModernComponentProps) {
  const [filter, setFilter] = useState('');
  const [selectedItem, setSelectedItem] = useState<string | null>(null);
  const inputRef = useRef<HTMLInputElement>(null);

  // Modern lifecycle with useEffect
  useEffect(() => {
    inputRef.current?.focus();
  }, []);

  // Memoized computation
  const filteredItems = useMemo(() => {
    return items.filter(item =>
      item.toLowerCase().includes(filter.toLowerCase())
    );
  }, [items, filter]);

  // Memoized callback
  const handleItemClick = useCallback((item: string) => {
    setSelectedItem(item);
    onItemClick?.(item);
  }, [onItemClick]);

  // Modern event handler
  const handleFilterChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    setFilter(e.target.value);
  };

  return (
    <div>
      <h1>{title}</h1>

      <input
        ref={inputRef}
        type="text"
        value={filter}
        onChange={handleFilterChange}
        placeholder="Filter items..."
      />

      <ul>
        {filteredItems.map((item, index) => (
          <li
            key={index}
            onClick={() => handleItemClick(item)}
            style={{
              fontWeight: selectedItem === item ? 'bold' : 'normal',
              cursor: 'pointer'
            }}
          >
            {item}
          </li>
        ))}
      </ul>

      {selectedItem && (
        <div>Selected: {selectedItem}</div>
      )}
    </div>
  );
}
```

**File:** `src/sample-components/migration/LegacyContextExample.tsx`

```typescript
import React from 'react';
import PropTypes from 'prop-types';

/**
 * Component using legacy context API
 * Should trigger high severity warning
 */
export class ThemeProvider extends React.Component {
  // Legacy context API - deprecated in React 16.3
  static childContextTypes = {
    theme: PropTypes.string,
  };

  getChildContext() {
    return {
      theme: 'dark',
    };
  }

  render() {
    return <div>{this.props.children}</div>;
  }
}

export class ThemedButton extends React.Component {
  // Legacy context API
  static contextTypes = {
    theme: PropTypes.string,
  };

  render() {
    return (
      <button style={{ background: this.context.theme === 'dark' ? '#333' : '#fff' }}>
        Click me
      </button>
    );
  }
}

/**
 * Modern context API equivalent
 */
const ThemeContext = React.createContext<string>('light');

export function ModernThemeProvider({ children }: { children: React.ReactNode }) {
  return (
    <ThemeContext.Provider value="dark">
      {children}
    </ThemeContext.Provider>
  );
}

export function ModernThemedButton() {
  const theme = React.useContext(ThemeContext);

  return (
    <button style={{ background: theme === 'dark' ? '#333' : '#fff' }}>
      Click me
    </button>
  );
}
```

## Acceptance Criteria

### Functional Requirements

- [ ] Accurately detect React version from package.json
- [ ] Detect React version from API usage when package.json unavailable
- [ ] Identify all deprecated lifecycle methods
- [ ] Detect string refs, findDOMNode usage
- [ ] Detect PropTypes and defaultProps on function components
- [ ] Identify class components vs functional components
- [ ] Calculate migration readiness score (0-100)
- [ ] Estimate migration effort in hours
- [ ] List applicable codemods
- [ ] Provide links to migration guides

### Score Accuracy

- [ ] Score of 100 for fully modern React 19 codebase
- [ ] Score of 0-20 for legacy class-based codebase
- [ ] Consistent scoring across multiple runs
- [ ] Score correlates with actual migration effort

### Performance

- [ ] Analysis completes in < 500ms for files < 500 LOC
- [ ] Analysis completes in < 2s for files < 2000 LOC
- [ ] Memory usage < 50MB per analysis

### Integration

- [ ] Integrates with main analyzer output
- [ ] JSON output matches extended schema
- [ ] CLI `--migration` flag works
- [ ] Works with `--threshold` flag

## Testing Instructions

### Manual Testing Steps

1. **Test Version Detection from package.json**

   ```bash
   # Create test package.json
   echo '{"dependencies": {"react": "^18.2.0"}}' > /tmp/package.json

   # Run analysis (implementation should detect version)
   npm run analyze -- src/sample-components/DataTable.tsx --verbose
   # Expected: Shows "Detected React version: 18.2.0"
   ```

2. **Test Deprecated API Detection**

   ```bash
   # Create a file with deprecated lifecycle
   cat > /tmp/LegacyComponent.tsx << 'EOF'
   import React from 'react';

   class LegacyComponent extends React.Component {
     componentWillMount() {
       console.log('Will mount');
     }

     componentWillReceiveProps(nextProps) {
       console.log('Will receive props');
     }

     render() {
       return <div>Legacy</div>;
     }
   }

   export default LegacyComponent;
   EOF

   npm run analyze -- /tmp/LegacyComponent.tsx --verbose
   # Expected: Detects componentWillMount and componentWillReceiveProps
   # Expected: Shows migration blockers with severity
   ```

3. **Test React 19 Feature Detection**

   ```bash
   cat > /tmp/ModernComponent.tsx << 'EOF'
   import { use, useOptimistic } from 'react';

   export function ModernComponent({ dataPromise }) {
     const data = use(dataPromise);
     const [optimisticData, setOptimistic] = useOptimistic(data);

     return <div>{optimisticData}</div>;
   }
   EOF

   npm run analyze -- /tmp/ModernComponent.tsx --verbose
   # Expected: Detects React 19 usage
   # Expected: High readiness score
   ```

4. **Test Migration Score Calculation**

   ```bash
   npm run analyze -- src/sample-components/ --json | jq '.migration'
   # Expected: JSON with readinessScore, blockers, effort
   ```

5. **Test Codemod Recommendations**
   ```bash
   npm run analyze -- /tmp/LegacyComponent.tsx --json | jq '.migration.codemods'
   # Expected: ["rename-unsafe-lifecycles"]
   ```

### Automated Test Cases

**File:** `src/analyzers/migration-analyzer.test.ts`

```typescript
import { describe, it, expect } from 'vitest';
import { Project } from 'ts-morph';
import { MigrationAnalyzer } from './migration-analyzer';

const createSourceFile = (code: string) => {
  const project = new Project({ useInMemoryFileSystem: true });
  return project.createSourceFile('temp.tsx', code);
};

describe('MigrationAnalyzer', () => {
  describe('version detection', () => {
    it('should detect React 19 from use() hook', () => {
      const code = `
        import { use } from 'react';
        export function Component() {
          const data = use(promise);
          return <div>{data}</div>;
        }
      `;

      const sourceFile = createSourceFile(code);
      const analyzer = new MigrationAnalyzer(sourceFile);
      const result = analyzer.analyze();

      expect(result.versionInfo.detectedVersion).toBe('19.0.0');
      expect(result.versionInfo.confidenceLevel).toBe('high');
    });

    it('should detect React 18 from useId hook', () => {
      const code = `
        import { useId } from 'react';
        export function Component() {
          const id = useId();
          return <label id={id}>Label</label>;
        }
      `;

      const sourceFile = createSourceFile(code);
      const analyzer = new MigrationAnalyzer(sourceFile);
      const result = analyzer.analyze();

      expect(result.versionInfo.detectedVersion).toBe('18.0.0');
    });
  });

  describe('deprecated API detection', () => {
    it('should detect componentWillMount', () => {
      const code = `
        import React from 'react';
        class Component extends React.Component {
          componentWillMount() {}
          render() { return <div />; }
        }
      `;

      const sourceFile = createSourceFile(code);
      const analyzer = new MigrationAnalyzer(sourceFile);
      const result = analyzer.analyze();

      expect(result.blockers.length).toBeGreaterThan(0);
      expect(result.blockers[0].description).toContain('componentWillMount');
      expect(result.blockers[0].severity).toBe('critical');
    });

    it('should detect string refs', () => {
      const code = `
        import React from 'react';
        class Component extends React.Component {
          render() {
            return <input ref="myInput" />;
          }
        }
      `;

      const sourceFile = createSourceFile(code);
      const analyzer = new MigrationAnalyzer(sourceFile);
      const result = analyzer.analyze();

      expect(result.blockers).toContainEqual(
        expect.objectContaining({
          type: 'deprecated-api',
          description: expect.stringContaining('String ref'),
        })
      );
    });

    it('should detect findDOMNode usage', () => {
      const code = `
        import { findDOMNode } from 'react-dom';

        class Component extends React.Component {
          componentDidMount() {
            const node = findDOMNode(this);
          }
          render() { return <div />; }
        }
      `;

      const sourceFile = createSourceFile(code);
      const analyzer = new MigrationAnalyzer(sourceFile);
      const result = analyzer.analyze();

      expect(result.blockers).toContainEqual(
        expect.objectContaining({
          description: expect.stringContaining('findDOMNode'),
        })
      );
    });
  });

  describe('readiness scoring', () => {
    it('should score 100 for modern functional component', () => {
      const code = `
        import { useState } from 'react';

        export function ModernComponent({ name }) {
          const [count, setCount] = useState(0);
          return <div>{name}: {count}</div>;
        }
      `;

      const sourceFile = createSourceFile(code);
      const analyzer = new MigrationAnalyzer(sourceFile);
      const result = analyzer.analyze();

      expect(result.readinessScore).toBe(100);
      expect(result.blockers).toHaveLength(0);
    });

    it('should reduce score for each blocker', () => {
      const code = `
        import React from 'react';
        class Component extends React.Component {
          componentWillMount() {}
          componentWillReceiveProps() {}
          render() { return <div />; }
        }
      `;

      const sourceFile = createSourceFile(code);
      const analyzer = new MigrationAnalyzer(sourceFile);
      const result = analyzer.analyze();

      expect(result.readinessScore).toBeLessThan(80);
    });
  });

  describe('effort estimation', () => {
    it('should estimate low effort for modern code', () => {
      const code = `
        export const Component = () => <div>Hello</div>;
      `;

      const sourceFile = createSourceFile(code);
      const analyzer = new MigrationAnalyzer(sourceFile);
      const result = analyzer.analyze();

      expect(result.estimatedEffort.hours).toBeLessThan(1);
      expect(result.estimatedEffort.complexity).toBe('trivial');
    });

    it('should estimate higher effort for legacy code', () => {
      const code = `
        import React from 'react';
        class Component extends React.Component {
          componentWillMount() {}
          componentWillReceiveProps() {}
          componentWillUpdate() {}
          render() { return <div ref="myRef" />; }
        }
      `;

      const sourceFile = createSourceFile(code);
      const analyzer = new MigrationAnalyzer(sourceFile);
      const result = analyzer.analyze();

      expect(result.estimatedEffort.hours).toBeGreaterThan(2);
    });
  });

  describe('codemod recommendations', () => {
    it('should recommend lifecycle codemod for deprecated lifecycles', () => {
      const code = `
        import React from 'react';
        class Component extends React.Component {
          componentWillMount() {}
          render() { return <div />; }
        }
      `;

      const sourceFile = createSourceFile(code);
      const analyzer = new MigrationAnalyzer(sourceFile);
      const result = analyzer.analyze();

      expect(result.codemods).toContain('rename-unsafe-lifecycles');
    });
  });
});
```

## Schema Updates

**File:** `schema.json` (additions)

```json
{
  "definitions": {
    "MigrationMetrics": {
      "type": "object",
      "properties": {
        "versionInfo": { "$ref": "#/definitions/ReactVersionInfo" },
        "readinessScore": { "type": "number", "minimum": 0, "maximum": 100 },
        "targetVersion": { "type": "string" },
        "blockers": {
          "type": "array",
          "items": { "$ref": "#/definitions/MigrationBlocker" }
        },
        "warnings": {
          "type": "array",
          "items": { "$ref": "#/definitions/MigrationWarning" }
        },
        "opportunities": {
          "type": "array",
          "items": { "$ref": "#/definitions/MigrationOpportunity" }
        },
        "estimatedEffort": { "$ref": "#/definitions/MigrationEffort" },
        "codemods": {
          "type": "array",
          "items": { "type": "string" }
        }
      },
      "required": ["versionInfo", "readinessScore", "targetVersion", "blockers", "codemods"]
    }
  }
}
```

## Rollback Plan

1. Migration analyzer is an additive feature
2. Can be disabled via config without affecting core analyzer
3. No breaking changes to existing API
4. Schema additions are backward compatible

## Implementation Checklist

Use this checklist to track implementation progress:

### Step 1: Enhance React API Catalog (Priority: HIGH)

- [ ] Replace existing catalog types with new interfaces (`APIDefinition`, `DeprecatedAPI`, `BreakingChange`)
- [ ] Create `REACT_16_8_APIS` array (10 base hooks)
- [ ] Create `REACT_18_APIS` array (5 concurrent hooks)
- [ ] Create `REACT_19_APIS` array (5 action hooks)
- [ ] Replace `DEPRECATED_APIS` with detailed array (19 entries)
- [ ] Create `REACT_19_BREAKING_CHANGES` array (8 breaking changes)
- [ ] Add helper functions (`getMinReactVersion`, `isDeprecatedAPI`, etc.)
- [ ] Remove old `REACT_HOOKS_BY_VERSION` object (no longer needed)
- [ ] Update exports in `src/constants/index.ts`

### Step 2: Create Version Detector (Priority: HIGH)

- [ ] Create `src/utils/react-version.ts` file
- [ ] Implement `ReactVersionInfo` and `VersionEvidence` interfaces
- [ ] Implement `ReactVersionDetector` class
- [ ] Implement `detectFromSourceFile()` method (import-based detection)
- [ ] Implement hook usage detection in `detectFromSourceFile()`
- [ ] Implement "use client" directive detection
- [ ] Implement `detectFromPackageJson()` method
- [ ] Implement `detect()` combined method
- [ ] Add private helper methods (buildApiCatalog, analyzeEvidence, etc.)
- [ ] Create unit tests in `src/utils/react-version.test.ts`

### Step 3: Create Migration Analyzer (Priority: HIGH)

- [ ] Create `src/analyzers/migration-analyzer.ts` file
- [ ] Implement `MigrationBlocker`, `MigrationEffort`, `MigrationResult` interfaces
- [ ] Implement `MigrationAnalyzer` class extending `BaseAnalyzer`
- [ ] Implement class component detection
- [ ] Implement deprecated lifecycle method detection
- [ ] Implement functional component counting
- [ ] Implement string ref detection
- [ ] Implement `findDOMNode` detection
- [ ] Implement readiness score calculation
- [ ] Implement effort estimation algorithm
- [ ] Export `analyzeMigration()` convenience function
- [ ] Create unit tests in `src/analyzers/migration-analyzer.test.ts`

### Step 4: Create Sample Components (Priority: MEDIUM)

- [ ] Create `src/sample-components/migration/` directory
- [ ] Create `ClassComponentLegacy.tsx` (deprecated lifecycles)
- [ ] Create `StringRefsExample.tsx` (string refs + modern equivalent)
- [ ] Create `PropTypesExample.tsx` (PropTypes + TypeScript equivalent)
- [ ] Create `React19Features.tsx` (use, useOptimistic, useActionState)
- [ ] Create `ModernComponent.tsx` (modern best practices)
- [ ] Create `LegacyContextExample.tsx` (legacy + modern context)

### Step 5: Integration & Testing (Priority: MEDIUM)

- [ ] Add migration types to `src/types/index.ts`
- [ ] Update `schema.json` with `MigrationMetrics` definition
- [ ] Add CLI flag `--migration` support in `src/cli.ts`
- [ ] Create integration tests with sample components
- [ ] Test version detection accuracy
- [ ] Test readiness score calculation
- [ ] Test effort estimation
- [ ] Document in README.md

## Dependencies

- Phase 00 must be complete (BaseAnalyzer, constants structure)
- `ts-morph` for AST analysis
- Existing `BaseAnalyzer` class from `base-analyzer-tsmorph.ts`

## Downstream Impact

- Phase 07-08 (VS Code) will use migration analysis for diagnostics
- Phase 04 (Technical Debt) may reference migration blockers

## Key Implementation Notes

### API Catalog Enhancement Strategy

1. **Complete replacement** - Don't try to maintain old structures, just replace them
2. **Comprehensive coverage** - Include all 19 deprecated APIs with full metadata
3. **Version-specific arrays** - Separate arrays for each React version for clear detection
4. **Rich metadata** - Every deprecated API includes codemods, severity, migration guides

### Version Detection Strategy

1. **Multi-source** - Combine package.json and source code analysis
2. **Evidence-based** - Collect evidence with confidence scores
3. **Hook-based** - Primary detection via hook usage patterns
4. **Directive support** - Detect "use client" for RSC compatibility

### Migration Analyzer Strategy

1. **Severity-based scoring** - Weight blockers by severity (critical=20, high=10, etc.)
2. **Component ratio** - Factor in class vs functional component ratio
3. **Realistic estimates** - Base effort on industry benchmarks (2hr/class, 1hr/lifecycle, etc.)
4. **Codemod suggestions** - Surface automated migration options when available

### Testing Strategy

1. **Sample components** - Create realistic migration scenarios
2. **Version coverage** - Test detection for React 16.8, 18, and 19 features
3. **Score accuracy** - Verify readiness scores match expectations
4. **Edge cases** - Test UNSAFE\_ prefixed lifecycles, legacy context patterns

## Estimated Effort

| Task                    | Estimated Time  | Notes                                     |
| ----------------------- | --------------- | ----------------------------------------- |
| 1.1 Enhance API Catalog | 2-3 hours       | Replace existing + add 19 deprecated APIs |
| 1.2 Version Detector    | 3-4 hours       | New file, multi-source detection          |
| 1.3 Migration Analyzer  | 4-5 hours       | New file, complex scoring logic           |
| 1.4 Sample Components   | 2-3 hours       | 6 comprehensive example files             |
| Unit Tests (Version)    | 2 hours         | Test detection accuracy                   |
| Unit Tests (Migration)  | 2 hours         | Test scoring and effort                   |
| Integration Tests       | 2 hours         | End-to-end migration analysis             |
| Schema Updates          | 30 minutes      | Add MigrationMetrics types                |
| CLI Integration         | 1 hour          | Add --migration flag                      |
| Documentation           | 1 hour          | Update README                             |
| **Total**               | **19-23 hours** | Comprehensive implementation              |

## Success Criteria

This phase will be considered complete when:

1. ✅ **Catalog Enhanced**
   - 19+ deprecated APIs with full metadata
   - Version-specific API arrays (16.8, 18, 19)
   - Breaking changes catalog for React 19

2. ✅ **Version Detection Working**
   - Accurate detection from package.json
   - Source code analysis via hook usage
   - Combined detection with confidence levels
   - Test coverage >90%

3. ✅ **Migration Analysis Working**
   - Class component detection
   - Deprecated API detection (all 19 types)
   - Readiness score (0-100) calculated correctly
   - Effort estimation in hours
   - Codemod recommendations provided

4. ✅ **Sample Components Complete**
   - 6 example components covering all scenarios
   - Can be used for testing and documentation
   - Cover React 16.3 → 19 migration paths

5. ✅ **Integration Complete**
   - CLI --migration flag works
   - JSON output matches schema
   - Documentation updated
   - All tests passing

---

**Document Version:** 3.0  
**Updated:** January 10, 2026  
**Status:** Ready for Implementation  
**Foundation:** Builds on existing `react-api-catalog.ts`
