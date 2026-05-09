# Testing Coverage Analysis

## Summary

This document identifies files in the vipr codebase that need unit, integration, and visual regression testing. The analysis focuses on source files with business logic, excluding configuration files, type-only files, and files with comprehensive existing test coverage.

## Test Type Definitions

- **Unit**: Tests for individual functions, classes, and components in isolation
- **Integration**: Tests for interactions between multiple modules or system components
- **Visual Regression (CLI)**: Tests for CLI output formatting and display (using snapshot testing)

## Files Requiring Test Coverage

| File Path                                                                                       | Test Types              | Rationale                                                                                                                                      |
| ----------------------------------------------------------------------------------------------- | ----------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| `/Users/jamesleebaker/Codespace/vipr/analyzers/react/src/presenters/accessibility-presenter.ts` | Unit                    | Transforms accessibility data to presentation format; needs tests for edge cases, empty data, sorting logic                                    |
| `/Users/jamesleebaker/Codespace/vipr/analyzers/react/src/presenters/anti-pattern-presenter.ts`  | Unit                    | Presentation logic for anti-patterns; needs validation of grouping and severity mapping                                                        |
| `/Users/jamesleebaker/Codespace/vipr/analyzers/react/src/presenters/base-presenter.ts`          | Unit                    | Base class with shared utilities; critical for all presenters to test helpers like mapSeverity, createScore                                    |
| `/Users/jamesleebaker/Codespace/vipr/analyzers/react/src/presenters/dataflow-presenter.ts`      | Unit                    | Dataflow presentation formatting; needs tests for complex data structures                                                                      |
| `/Users/jamesleebaker/Codespace/vipr/analyzers/react/src/presenters/migration-presenter.ts`     | Unit                    | Migration report formatting; needs validation of version-specific logic                                                                        |
| `/Users/jamesleebaker/Codespace/vipr/analyzers/react/src/presenters/overview-presenter.ts`      | Unit                    | Overview report generation; critical entry point needing comprehensive testing                                                                 |
| `/Users/jamesleebaker/Codespace/vipr/analyzers/react/src/presenters/reliability-presenter.ts`   | Unit                    | Reliability metrics presentation; needs testing of crash risk calculations                                                                     |
| `/Users/jamesleebaker/Codespace/vipr/analyzers/react/src/presenters/security-presenter.ts`      | Unit                    | Security report formatting; needs validation of CWE/OWASP link generation                                                                      |
| `/Users/jamesleebaker/Codespace/vipr/analyzers/core/src/engine/analysis-cache.ts`               | Unit, Integration       | Cache management with complex invalidation logic; needs tests for TTL, hash collisions, memory management                                      |
| `/Users/jamesleebaker/Codespace/vipr/analyzers/core/src/engine/analysis-engine.ts`              | Unit, Integration       | Core orchestration engine; needs extensive testing for parallel execution, error handling, plugin coordination                                 |
| `/Users/jamesleebaker/Codespace/vipr/analyzers/core/src/utils/base-plugin-formatter.ts`         | Unit                    | Plugin result formatting with normalization; needs validation of score calculations and weight application                                     |
| `/Users/jamesleebaker/Codespace/vipr/clients/cli/src/formatters/cli-formatter.ts`               | Unit, Visual Regression | Main CLI output formatter; needs snapshot tests for different report types and layouts                                                         |
| `/Users/jamesleebaker/Codespace/vipr/clients/cli/src/formatters/renderer/cli-renderer.ts`       | Unit, Visual Regression | Renders presentation models to CLI strings; critical for all visual output testing                                                             |
| `/Users/jamesleebaker/Codespace/vipr/clients/cli/src/formatters/compact-json-formatter.ts`      | Unit                    | Compact JSON output; needs validation of summarization logic and statistics calculations                                                       |
| `/Users/jamesleebaker/Codespace/vipr/clients/cli/src/formatters/full-json-formatter.ts`         | Unit                    | Full JSON output; needs validation of complete data transformation and breakdown serialization                                                 |
| `/Users/jamesleebaker/Codespace/vipr/clients/cli/src/formatters/base/base-formatter.ts`         | Unit, Visual Regression | Base formatter with styling utilities; needs tests for color application, box drawing, score bars                                              |
| `/Users/jamesleebaker/Codespace/vipr/clients/cli/src/plugins/loader.ts`                         | Unit, Integration       | Plugin discovery and registration; needs tests for bundled plugin loading and filtering                                                        |
| `/Users/jamesleebaker/Codespace/vipr/clients/cli/src/utils/colors.ts`                           | Unit                    | Color utility functions; needs validation of color code generation                                                                             |
| `/Users/jamesleebaker/Codespace/vipr/packages/plugin-loader/src/discovery/workspace-scanner.ts` | Unit, Integration       | Workspace scanning with pnpm-workspace.yaml parsing; needs tests for glob resolution and validation                                            |
| `/Users/jamesleebaker/Codespace/vipr/packages/plugin-loader/src/discovery/package-analyzer.ts`  | Unit                    | Package.json analysis; needs tests for metadata extraction and marker detection                                                                |
| `/Users/jamesleebaker/Codespace/vipr/packages/plugin-loader/src/discovery/plugin-validator.ts`  | Unit                    | Plugin validation logic; needs tests for interface validation and version checking                                                             |
| `/Users/jamesleebaker/Codespace/vipr/packages/plugin-loader/src/loader/dynamic-loader.ts`       | Unit, Integration       | Dynamic module loading with ESM; needs tests for import errors, validation, and instantiation                                                  |
| `/Users/jamesleebaker/Codespace/vipr/packages/plugin-loader/src/loader/error-handler.ts`        | Unit                    | Error handling and wrapping; needs tests for error categorization and context preservation                                                     |
| `/Users/jamesleebaker/Codespace/vipr/packages/plugin-loader/src/loader/plugin-instantiator.ts`  | Unit                    | Plugin instance creation; needs tests for factory functions and configuration                                                                  |
| `/Users/jamesleebaker/Codespace/vipr/packages/plugin-loader/src/registry/plugin-registry.ts`    | Unit, Integration       | Central plugin registry; needs tests for registration, ordering, enabling/disabling                                                            |
| `/Users/jamesleebaker/Codespace/vipr/packages/common/src/utils/scoring.ts`                      | Unit                    | Core scoring utilities already tested but missing edge cases; needs additional tests for normalization warnings and composite score edge cases |
| `/Users/jamesleebaker/Codespace/vipr/packages/analyzer-template/src/create-analyzer.ts`         | Unit, Integration       | Interactive wizard for analyzer creation; needs tests for validation and file generation orchestration                                         |
| `/Users/jamesleebaker/Codespace/vipr/packages/analyzer-template/src/file-generator.ts`          | Unit                    | Template file generation; needs tests for template rendering and directory creation                                                            |
| `/Users/jamesleebaker/Codespace/vipr/packages/analyzer-template/src/template-engine.ts`         | Unit                    | Template variable replacement; needs tests for nested templates and edge cases                                                                 |
| `/Users/jamesleebaker/Codespace/vipr/packages/analyzer-template/src/validator.ts`               | Unit                    | Configuration validation; needs tests for all validation rules and error messages                                                              |

## Priority Files

The following files are highest priority due to their critical role in the system:

### Critical Path

1. `analyzers/core/src/engine/analysis-engine.ts` - Core engine orchestrating all analysis
2. `clients/cli/src/commands/analyze-command.ts` - Already has tests but needs visual regression for output
3. `clients/cli/src/formatters/cli-formatter.ts` - Primary user-facing output

### Presentation Layer

4. `analyzers/react/src/presenters/base-presenter.ts` - Base for all React presenters
5. `clients/cli/src/formatters/renderer/cli-renderer.ts` - Core rendering logic
6. `clients/cli/src/formatters/base/base-formatter.ts` - Shared styling and formatting

### Plugin Infrastructure

7. `packages/plugin-loader/src/loader/dynamic-loader.ts` - Plugin loading mechanism
8. `packages/plugin-loader/src/registry/plugin-registry.ts` - Plugin lifecycle management
9. `clients/cli/src/plugins/loader.ts` - CLI plugin integration

## Test Coverage Recommendations

### Unit Tests Priority

1. **Presenters**: All presenter files need comprehensive unit tests for data transformation logic
2. **Formatters**: JSON formatters need validation of complete data structures
3. **Utilities**: Scoring, validation, and helper utilities need edge case testing
4. **Plugin System**: All plugin loader components need isolation testing

### Integration Tests Priority

1. **Analysis Engine**: End-to-end analysis workflow with multiple plugins
2. **Plugin Loading**: Complete plugin discovery and registration flow
3. **CLI Command**: Full command execution with real file analysis
4. **Cache System**: Multi-level cache interaction and invalidation

### Visual Regression Tests Priority

1. **CLI Formatter**: All report types (overview, security, accessibility, etc.)
2. **Base Formatter**: Score bars, boxes, headers, severity icons
3. **CLI Renderer**: Section rendering, metrics, items, subsections
4. **Report Variations**: Different score ranges, empty data, error states

## Testing Strategy

### Unit Tests

- Use Vitest for all unit tests
- Co-locate test files with source files (e.g., `file.ts` -> `file.test.ts`)
- Follow AAA pattern (Arrange, Act, Assert)
- Mock external dependencies (file system, imports)
- Test edge cases: empty data, null/undefined, boundary values
- Test error conditions and exception handling

### Integration Tests

- Test interactions between modules
- Use real implementations where possible
- Mock only external services (file system for safety)
- Test complete workflows end-to-end
- Validate data flow between components

### Visual Regression Tests

- Use Vitest snapshots for CLI output
- Test all report types individually
- Test with various data scenarios (high scores, low scores, empty, errors)
- Test with/without colors (--no-color flag)
- Test different terminal widths if applicable
- Include examples with Unicode characters, special formatting

## Notes

### Already Well-Tested

The following areas have comprehensive test coverage and are excluded:

- All analyzer analysis files in `analyzers/react/src/analyses/` have `.test.ts` files
- All analyzer analysis files in `analyzers/core/src/analyses/` have `.test.ts` files
- Utility helpers in `analyzers/react/src/utils/` have tests
- Core utilities in `analyzers/core/src/utils/` have tests
- JSON formatter has existing tests

### Type-Only Files (Excluded)

Files in `/types/` directories contain only TypeScript interfaces and types, requiring no runtime testing.

### Configuration Files (Excluded)

- `tsconfig.json`, `vitest.config.ts`, `package.json` files
- Constant definition files (thresholds, weights, dependencies)

### Fixtures (Excluded)

Test fixture components in `analyzers/react/src/fixtures/` are test data, not production code.
