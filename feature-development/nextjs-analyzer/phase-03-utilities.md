# Phase 3: Core Utility Functions

## Status

Not Started

## Goals

Implement all detection utilities required by the analysis phases, including directive detection, router type detection, and Next.js component identification.

## Files Created

### Utility Modules

- `utils/directive-helpers.ts` - Directive detection and validation (7 functions)
- `utils/router-helpers.ts` - Router type detection (13 functions)
- `utils/nextjs-helpers.ts` - Next.js component detection (14 functions)
- `utils/index.ts` - Utility exports

### Test Files

- `utils/directive-helpers.test.ts` - 25 tests
- `utils/router-helpers.test.ts` - 43 tests
- `utils/nextjs-helpers.test.ts` - 40 tests

## Implementation Details

### Directive Helpers (11 functions)

**Directive Detection:**

- `hasUseClientDirective()` - Detects 'use client' directive
- `hasUseServerDirective()` - Detects 'use server' directive
- `getDirectivePlacement()` - Identifies directive placement issues
- `isDirectiveAtTop()` - Validates directive position

**Directive Validation:**

- `needsUseClientDirective()` - Checks if hooks/events require directive
- `hasUnnecessaryUseClientDirective()` - Detects directive without need
- `getUsedHooks()` - Lists all React hooks in file

### Router Helpers (15 functions)

**Directory Detection:**

- `isAppRouterFile()` - Detects app/ directory
- `isPagesRouterFile()` - Detects pages/ directory
- `isRouteHandler()` - Detects route.ts/js files
- `isServerAction()` - Detects Server Actions

**Router Type Detection:**

- `detectRouterType()` - Determines App vs Pages Router
- `hasPagesRouterDataFetching()` - Detects getServerSideProps, etc.
- `hasAppRouterDataFetching()` - Detects generateMetadata, etc.

**HTTP Method Detection:**

- `hasHTTPMethodExports()` - Detects GET, POST, etc.
- `getHTTPMethodExports()` - Lists HTTP methods
- `hasLowercaseHTTPMethods()` - Detects incorrect casing
- `getLowercaseHTTPMethods()` - Lists casing issues

**Router Import Validation:**

- `hasWrongRouterImport()` - Detects wrong router for directory
- `getCorrectRouterImport()` - Returns correct import

### Next.js Helpers (14 functions)

**Import Detection:**

- `getNextJsImports()` - Maps all Next.js imports
- `isNextImageUsage()` - Detects next/image
- `isNextLinkUsage()` - Detects next/link
- `isNextScriptUsage()` - Detects next/script

**Component Usage:**

- `getImageUsages()` - Finds all Image components
- `getLinkUsages()` - Finds all Link components
- `getScriptUsages()` - Finds all Script components

**Legacy Detection:**

- `usesLegacyImageImport()` - Detects next/legacy/image
- `usesDeprecatedFontImport()` - Detects @next/font

**Data Fetching Detection:**

- `hasFetchCalls()` - Detects fetch() usage
- `hasFetchWithCacheOptions()` - Detects cache options

**Component Patterns:**

- `isAsyncComponent()` - Detects async components
- `exportsMetadata()` - Detects metadata export
- `exportsDynamic()` - Detects dynamic export
- `exportsRevalidate()` - Detects revalidate export

## Test Coverage

### Summary

- **Total Tests**: 108 utility tests (+ 28 from previous phases = 136 total)
- **Coverage**: 100% of utility functions tested
- **All Tests Passing**: ✅

### Test Categories

**Directive Helpers (25 tests):**

- Directive detection (single/double quotes)
- Placement validation
- Hook detection
- Unnecessary directive detection
- Multiple hook detection

**Router Helpers (43 tests):**

- Directory detection (app/pages)
- Route handler detection
- Router type inference
- Data fetching function detection
- HTTP method validation
- Router import validation

**Next.js Helpers (40 tests):**

- Import mapping (including aliased imports)
- Component usage detection
- Legacy pattern detection
- Fetch call detection
- Async component detection
- Export detection

## Acceptance Criteria

All criteria met:

- [x] 100% test coverage on utility functions
- [x] Correctly identifies directive positions
- [x] Distinguishes App Router vs Pages Router files
- [x] Detects next/\* component usage
- [x] All 133 tests passing

## Technical Notes

### TypeScript Fixes Applied

Fixed strict null checks for:

- Array indexing in directive parsing
- Regex capture group access in HTTP method detection
- Added proper guards for undefined values

### Coverage Notes

- Hook detection includes modern client-only hooks (`useOptimistic`, `useActionState`, `useFormStatus`)
- Next.js component detection treats aliased imports as valid usage

### Extension Guidelines

- Keep hook detection current by adding new client-only hooks to all three places:
  - `needsUseClientDirective` (detection)
  - `hasUnnecessaryUseClientDirective` (false-positive prevention)
  - `getUsedHooks` (reporting/context)

### Pattern Detection Strategy

Utilities use multiple detection strategies for robustness:

1. **Directory patterns** - File path analysis
2. **Import analysis** - ts-morph AST traversal
3. **Text patterns** - Regex for directive/function detection
4. **AST analysis** - JSX element detection

This layered approach ensures high accuracy across different file structures.

## Next Steps

Proceed to Phase 4: Server/Client Component Analysis

With all utilities in place, we can now implement the six analysis modules that use these detection functions.
