# Phase 2: Core Types and Constants

## Status

Not Started

## Goals

Define the type system and constants for the Next.js analyzer, including all analysis-specific types, weights, thresholds, and version-specific patterns.

## Files Created

### Type Definitions

- `types/server-client-types.ts` - Server/Client component analysis types
- `types/data-fetching-types.ts` - Data fetching analysis types
- `types/migration-types.ts` - Migration analysis types
- `types/security-types.ts` - Security analysis types
- `types/config-types.ts` - Configuration analysis types
- `types/index.ts` - Type exports

### Constants

- `constants/weights.ts` - Analysis weights and scoring multipliers
- `constants/thresholds.ts` - Warning and critical thresholds
- `constants/nextjs-versions.ts` - Version-specific patterns and deprecations
- `constants/index.ts` - Constants exports
- `constants/constants.test.ts` - Constants validation tests

## Type System Design

### Server/Client Types

Defines types for analyzing Server and Client Components:

- `DirectivePlacement` - Directive placement issue types
- `ServerClientIssueType` - Issue classification
- `ServerClientIssue` - Individual issues with context
- `ServerClientComplexity` - Analysis results with statistics

### Data Fetching Types

Covers both Pages Router and App Router patterns:

- `RouterType` - Router classification (app/pages/unknown)
- `DataFetchingIssueType` - Issue classification
- `DataFetchingIssue` - Individual issues
- `DataFetchingStats` - Usage statistics by router type
- `DataFetchingComplexity` - Analysis results

### Migration Types

Tracks version-specific changes:

- `NextJsVersion` - Version identifiers (12-15)
- `MigrationIssueType` - Breaking changes and deprecations
- `MigrationIssue` - Issues with version information
- `MigrationReadiness` - Readiness assessment
- `MigrationComplexity` - Analysis results

### Security Types

Security vulnerability analysis:

- `SecurityIssueType` - Security issue classification
- `ServerActionIssue` - Server Action security details
- `EnvVarIssue` - Environment variable issues
- `MiddlewareIssue` - Middleware security issues
- `SecurityComplexity` - Analysis results with risk level

### Config Types

Configuration analysis:

- `ConfigIssueType` - Config issue classification
- `DeprecatedOption` - Deprecated config details
- `ConfigIssue` - Individual config issues
- `ConfigComplexity` - Analysis results

## Constants Design

### Analysis Weights

Composite score weights (sum to 1.0):

- `serverClient: 0.25` - Highest priority with security
- `security: 0.25` - Critical for production apps
- `dataFetching: 0.20` - Core Next.js functionality
- `migration: 0.15` - Version compatibility
- `config: 0.10` - Configuration issues
- `components: 0.05` - Lowest priority (usage issues)

### Severity Deductions

Each analysis has severity-based deductions:

- Server/Client: critial(20), warning(10), info(2)
- Data Fetching: critial(15), warning(8), info(2)
- Migration: critial(25), warning(10), info(3)
- Security: critical(40), high(20), medium(10), low(4)
- Config: critial(15), warning(8), info(2)
- Components: critial(10), warning(5), info(1)

### Issue Type Weights

Additional deductions per specific issue type, calibrated based on impact.

### Version-Specific Patterns

- `DEPRECATED_IMAGE_PROPS` - Image props by version
- `DEPRECATED_CONFIG_OPTIONS` - Config options by version
- `BREAKING_CHANGES` - Breaking changes per version with patterns
- `ROUTER_API_DIFFERENCES` - Pages vs App Router APIs
- `SENSITIVE_ENV_PATTERNS` - Patterns for sensitive data
- `NODE_APIS_NOT_IN_EDGE` - APIs unavailable in Edge Runtime
- `AUTH_LIBRARY_PATTERNS` - Auth library detection
- `VALIDATION_LIBRARY_PATTERNS` - Validation library detection

## Acceptance Criteria

All criteria met:

- [x] All types compile without errors
- [x] Types are exported through `types/index.ts`
- [x] Weights sum to 1.0 (validated in test)
- [x] Constants exported through `constants/index.ts`
- [x] Test suite passes (28/28 tests)

## Test Coverage

Created validation tests for constants:

- Weights sum to 1.0 with floating-point precision
- All weights are positive
- Expected analysis dimensions present
- Priority weighting verified (server/client and security highest)

## Next Steps

Proceed to Phase 3: Core Utility Functions
