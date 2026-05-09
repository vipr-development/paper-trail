# Severity Taxonomy

## Overview

Vipr uses multiple severity classification systems across different analyzers. While this might seem inconsistent at first, each vocabulary is deliberately chosen to align with industry standards for its specific domain.

This document explains why multiple severity systems exist, how they map to each other, and how the system ensures consistent behavior across all client applications.

## Why Multiple Severity Vocabularies?

Different analysis domains have established industry standards with specific severity terminology:

1. **Accessibility (WCAG-aligned)** - Uses WCAG 2.x severity levels
2. **Security (CVSS-aligned)** - Uses Common Vulnerability Scoring System terminology
3. **General Analysis** - Uses simplified, developer-friendly terms
4. **Migration/Configuration** - Uses context-specific severity indicators

Using domain-specific vocabularies ensures that:

- Reports align with industry standards and existing tools
- Developers familiar with each domain recognize the terminology
- Severity levels carry the right semantic weight for their context

## Severity Vocabularies

### Core Analysis Severity

Used by the core analyzer for general code quality issues.

| Level    | Usage                        | Score Impact |
| -------- | ---------------------------- | ------------ |
| critical | Breaking issues, must fix    | High         |
| warning  | Should fix, but not blocking | Medium       |
| info     | Suggestions, best practices  | Low          |

**Example Use Cases:**

- Cyclomatic complexity violations
- Maintainability index warnings
- Code structure recommendations

### Security Severity (CVSS-Aligned)

Used for security vulnerability classification, aligned with CVSS standards.

| Level    | CVSS Range | Description                                  | Score Impact |
| -------- | ---------- | -------------------------------------------- | ------------ |
| critical | 9.0-10.0   | Requires immediate remediation               | 25 points    |
| high     | 7.0-8.9    | Important to remediate quickly               | 15 points    |
| medium   | 4.0-6.9    | Should remediate in normal development cycle | 8 points     |
| low      | 0.1-3.9    | Low priority, remediate when convenient      | 3 points     |

**Example Use Cases:**

- XSS vulnerabilities
- Authentication bypass issues
- Sensitive data exposure
- Input validation gaps

**Scoring Constants:**

```typescript
// From @vipr/common/utils/analysis-helpers.ts
export const DEFAULT_SECURITY_DEDUCTIONS = {
  critical: 25,
  high: 15,
  medium: 8,
  low: 3,
};
```

### Accessibility Severity (WCAG-Aligned)

Used for accessibility violation classification, aligned with WCAG 2.x impact levels.

| Level    | WCAG Impact | Description                                               | Score Impact |
| -------- | ----------- | --------------------------------------------------------- | ------------ |
| critical | Blocker     | Prevents access for users with disabilities               | 25 points    |
| serious  | Serious     | Causes significant difficulty for users with disabilities | 15 points    |
| moderate | Moderate    | Causes some difficulty, but workarounds exist             | 10 points    |
| minor    | Minor       | Minor inconvenience, accessibility still functional       | 5 points     |

**Example Use Cases:**

- Missing alt text on images
- Insufficient color contrast
- Missing ARIA labels
- Keyboard navigation issues

**Scoring Constants:**

```typescript
// From @vipr/common/utils/analysis-helpers.ts
export const A11Y_SEVERITY_WEIGHTS = {
  critical: 25,
  serious: 15,
  moderate: 10,
  minor: 5,
};
```

### Migration/Issue Severity

Used for Next.js migration analysis and configuration issues.

| Level    | Description                           | Score Impact |
| -------- | ------------------------------------- | ------------ |
| critical | Breaking changes that prevent upgrade | 25 points    |
| warning  | Deprecations that should be addressed | 10 points    |
| info     | Informational migration suggestions   | 3 points     |

**Example Use Cases:**

- Breaking API changes between Next.js versions
- Deprecated configuration options
- Router migration requirements

## Severity Mapping in VSCode Extension

The VSCode extension normalizes all severity vocabularies into a consistent tier system for UI display:

### Tier Mapping

```typescript
// From clients/vscode-extension/src/utils/severity-mapper.ts

Tier 4 (Critical):  critical
Tier 3 (Error):     error, high, serious
Tier 2 (Warning):   warning, medium, moderate
Tier 1 (Info):      info, low, minor
```

### Visual Indicators

| Tier | VSCode Diagnostic | Icon         | Color  |
| ---- | ----------------- | ------------ | ------ |
| 4    | Error             | $(error)     | Red    |
| 3    | Warning           | $(warning)   | Yellow |
| 2    | Information       | $(info)      | Blue   |
| 1    | Hint              | $(lightbulb) | Gray   |

## Cross-Vocabulary Consistency

### Sorting and Priority

All issues are sorted by numeric weight regardless of vocabulary:

```typescript
function getSeverityWeight(severity: Severity): number {
  switch (severity) {
    case 'critical':
      return 4;
    case 'error':
    case 'high':
    case 'serious':
      return 3;
    case 'warning':
    case 'medium':
    case 'moderate':
      return 2;
    case 'info':
    case 'low':
    case 'minor':
      return 1;
    default:
      return 0;
  }
}
```

This ensures that:

- A "serious" accessibility issue (weight 3) sorts alongside a "high" security issue (weight 3)
- A "critical" security issue (weight 4) always appears before a "serious" a11y issue (weight 3)
- Issues are consistently prioritized across all analysis domains

### Score Impact Alignment

While vocabularies differ, the score impact is aligned by tier:

| Tier | Example Severities        | Typical Score Impact |
| ---- | ------------------------- | -------------------- |
| 4    | critical                  | 20-25 points         |
| 3    | error, high, serious      | 10-15 points         |
| 2    | warning, medium, moderate | 5-10 points          |
| 1    | info, low, minor          | 2-5 points           |

## Implementation Guidelines

### For Analyzer Developers

When creating a new analyzer:

1. **Choose the right vocabulary** for your domain:
   - Use **Security Severity** for security-related issues
   - Use **A11y Severity** for accessibility issues
   - Use **Core Severity** for general code quality
   - Create domain-specific vocabularies only when necessary

2. **Document your severity mapping** in the analyzer's constants:

   ```typescript
   export const SEVERITY_WEIGHTS: Record<MySeverity, number> = {
     critical: 25,
     // ...
   };
   ```

3. **Use consistent score deductions** aligned with the tier system

### For Client Developers

When building client applications:

1. **Import the severity mapper** from `@vipr/common` or the VSCode extension utils
2. **Use `getSeverityWeight()`** for sorting issues
3. **Use tier-based visual indicators** for consistent UX
4. **Never hardcode severity mappings** - always query from the centralized constants

## Related Documentation

- [Scoring Methodology](./scoring-methodology.md) - How scores are calculated
- [Plugin Architecture](./plugin-architecture.md) - How analyzers integrate with the system
- [Configuration Reference](./configuration-reference.md) - Configuring severity thresholds

## Technical References

### Source Files

- `packages/common/src/utils/severity-mapper.ts` - VSCode severity normalization
- `packages/common/src/utils/analysis-helpers.ts` - Security and A11y scoring
- `packages/common/src/types/analysis/index.ts` - Type definitions for all severity vocabularies
- `analyzers/*/src/constants/weights.ts` - Analyzer-specific severity weights

### Constants

All severity-related constants are centralized in `@vipr/common`:

```typescript
// Security
import { DEFAULT_SECURITY_DEDUCTIONS } from '@vipr/common';

// Accessibility
import { A11Y_SEVERITY_WEIGHTS } from '@vipr/common';

// Severity mapping (VSCode extension)
import { getSeverityWeight, mapSeverityToDiagnostic } from '@vipr/common';
```

## Summary

Vipr's multi-vocabulary severity system provides:

1. **Domain alignment** - Uses industry-standard terminology where it matters
2. **Consistent behavior** - Unified tier mapping ensures predictable sorting and display
3. **Centralized configuration** - All mappings defined in `@vipr/common`
4. **Extensibility** - New vocabularies can be added while maintaining cross-vocabulary consistency

The system prioritizes developer experience by using familiar terminology in each domain while ensuring that all severities behave consistently regardless of their source vocabulary.
