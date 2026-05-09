# Security Analyzer Metrics Specification

This document provides a detailed specification of all metrics analyzed by the Security Analyzer (`v2/src/analyzers/security-analyzer.ts`) for implementation in other systems.

## Overview

The Security Analyzer detects security vulnerabilities in React applications by analyzing code patterns and calculating comprehensive security metrics. It extends the anti-pattern detection system with security-specific classifications and scoring.

## Core Metrics

### 1. Overall Security Score (`score`)

**Type:** `number` (0-100)  
**Description:** Normalized security score where 100 = no vulnerabilities, 0 = severe security issues  
**Calculation Algorithm:**

- Base score: 100
- Deductions per vulnerability based on severity:
  - Critical: -25 points each
  - High: -15 points each
  - Medium/Warning: -8 points each
  - Low/Info: -3 points each
- Final score: `Math.max(0, 100 - totalDeductions)`

**Example:**

- 1 critical + 2 high = 100 - 25 - 30 = 45
- 0 vulnerabilities = 100

### 2. Risk Level (`riskLevel`)

**Type:** `SecuritySeverity` (`'critical' | 'high' | 'medium' | 'low'`)  
**Description:** Overall risk assessment based on the highest severity vulnerability found  
**Calculation Algorithm:**

1. If any vulnerability has severity `'critical'` → return `'critical'`
2. Else if any vulnerability has severity `'high'` → return `'high'`
3. Else if any vulnerability has severity `'medium'` or `'warning'` → return `'medium'`
4. Else → return `'low'`

**Priority Order:** critical > high > medium > low

### 3. OWASP Compliance Score (`complianceScore`)

**Type:** `number` (0-100)  
**Description:** OWASP Top 10 compliance score where 100 = fully compliant, 0 = severe non-compliance  
**Calculation Algorithm:**

- Base score: 100
- Penalty weights per vulnerability:
  - Critical: -30 points each
  - High: -20 points each
  - Medium/Warning: -10 points each
  - Low/Info: -5 points each
- Final score: `Math.max(0, 100 - totalPenalties)`

**Note:** More stringent than security score, emphasizing critical issues

### 4. Vulnerability Counts by Type (`byType`)

**Type:** `Record<SecurityVulnerabilityType, number>`  
**Description:** Count of vulnerabilities grouped by vulnerability type  
**Vulnerability Types:**

- `'xss'` - Cross-Site Scripting attacks
- `'injection'` - Code injection vulnerabilities
- `'sensitive-data'` - Sensitive data exposure
- `'authentication'` - Authentication/session management issues
- `'access-control'` - Authorization and access control flaws
- `'cryptography'` - Weak cryptographic implementations
- `'input-validation'` - Missing or insufficient input validation

**Initialization:** All types start at 0, incremented for each matching vulnerability

### 5. Vulnerability Counts by Severity (`bySeverity`)

**Type:** `Record<SecuritySeverity, number>`  
**Description:** Count of vulnerabilities grouped by severity level  
**Severity Levels:**

- `'critical'` - Immediate exploitation risk, requires immediate fix
- `'high'` - Significant risk, should be fixed urgently
- `'medium'` - Moderate risk, should be scheduled for fix
- `'low'` - Minor risk, can be addressed during regular maintenance

**Mapping from AntiPattern Severity:**

- `'critical'` → `'critical'`
- `'high'` → `'high'`
- `'medium'` or `'warning'` → `'medium'`
- `'low'` or `'info'` → `'low'`

**Initialization:** All severities start at 0, incremented for each matching vulnerability

### 6. Vulnerabilities Array (`vulnerabilities`)

**Type:** `SecurityVulnerability[]`  
**Description:** Array of all detected security vulnerabilities with full details

**SecurityVulnerability Structure:**

```typescript
interface SecurityVulnerability {
  // From AntiPattern base
  id: string; // Unique rule ID (kebab-case)
  name: string; // Human-readable name
  category: AntiPatternCategory; // 'security' or related category
  severity: AntiPatternSeverity; // 'critical' | 'high' | 'medium' | 'warning' | 'low' | 'info'
  location: CodeLocation; // Precise code location (file, line, column)
  description: string; // What the problem is
  impact: string; // How this impacts security
  fix: string; // How to fix the issue
  autoFixable: boolean; // Whether auto-fix is available
  autoFix?: CodeFix; // Auto-fix details if available
  codeSnippet?: string; // Problematic code snippet
  references: string[]; // Documentation URLs
  context?: Record<string, unknown>; // Additional metadata

  // Security-specific extensions
  vulnerabilityType: SecurityVulnerabilityType; // Type classification
  cwe?: string; // CWE (Common Weakness Enumeration) ID
  owasp?: OwaspCategory | string; // OWASP Top 10 category
}
```

**CodeLocation Structure:**

```typescript
interface CodeLocation {
  file: string; // File path
  line: number; // Line number (1-indexed)
  column: number; // Column number (1-indexed)
  endLine?: number; // End line if spanning multiple lines
  endColumn?: number; // End column if spanning multiple lines
}
```

## Detected Vulnerability Rules

The analyzer maps specific rule IDs to vulnerability types. Currently implemented mappings:

### XSS Vulnerabilities

- **`'dangerous-html'`** → `'xss'`
  - CWE: `'CWE-79'`
  - OWASP: `'A03:2021'` (Injection)
- **`'dynamic-url-injection'`** → `'xss'`
  - CWE: `'CWE-79'`
  - OWASP: `'A03:2021'` (Injection)

### Injection Vulnerabilities

- **`'eval-usage'`** → `'injection'`
  - CWE: `'CWE-95'`
  - OWASP: `'A03:2021'` (Injection)
- **`'unsafe-regex'`** → `'injection'`
  - CWE: `'CWE-1333'`
  - OWASP: `'A03:2021'` (Injection)

### Sensitive Data Vulnerabilities

- **`'sensitive-data-exposure'`** → `'sensitive-data'`
  - CWE: `'CWE-798'`
  - OWASP: `'A02:2021'` (Cryptographic Failures)
- **`'sensitive-data-logging'`** → `'sensitive-data'`
  - CWE: `'CWE-532'`
  - OWASP: `'A09:2021'` (Security Logging and Monitoring Failures)
- **`'sensitive-data-storage'`** → `'sensitive-data'`
  - CWE: `'CWE-922'`
  - OWASP: `'A02:2021'` (Cryptographic Failures)

### Cryptography Vulnerabilities

- **`'insecure-random'`** → `'cryptography'`
  - CWE: `'CWE-330'`
  - OWASP: `'A02:2021'` (Cryptographic Failures)

**Default Mapping:** If a rule ID is not in the mapping but has category `'security'`, it defaults to `'xss'` type.

## OWASP Top 10 Categories

The analyzer references OWASP Top 10 2021 categories:

- `'A01:2021'` - Broken Access Control
- `'A02:2021'` - Cryptographic Failures
- `'A03:2021'` - Injection
- `'A04:2021'` - Insecure Design
- `'A05:2021'` - Security Misconfiguration
- `'A06:2021'` - Vulnerable and Outdated Components
- `'A07:2021'` - Identification and Authentication Failures
- `'A08:2021'` - Software and Data Integrity Failures
- `'A09:2021'` - Security Logging and Monitoring Failures
- `'A10:2021'` - Server-Side Request Forgery

## Metadata

All results include metadata from `BaseAnalyzerResult`:

```typescript
interface AnalyzerResultMetadata {
  analyzer: string; // Analyzer identifier (e.g., "SecurityAnalyzer")
  timestamp: string; // ISO 8601 timestamp of analysis
  durationMs: number; // Analysis duration in milliseconds
  filePath?: string; // Source file path if available
}
```

## Aggregation Metrics (Multi-File Analysis)

When analyzing multiple files, additional aggregated metrics are calculated:

### 1. Total Files (`totalFiles`)

**Type:** `number`  
**Description:** Count of files analyzed

### 2. Total Vulnerabilities (`totalVulnerabilities`)

**Type:** `number`  
**Description:** Sum of all vulnerabilities across all files

### 3. Average Score (`averageScore`)

**Type:** `number` (0-100)  
**Description:** Average security score across all files  
**Calculation:** `Math.round(sumOfAllScores / totalFiles)`

### 4. Overall Risk Level (`overallRiskLevel`)

**Type:** `SecuritySeverity`  
**Description:** Highest risk level found across all files  
**Calculation:** Same algorithm as single-file risk level, applied to aggregated severity counts

### 5. Critical Files (`criticalFiles`)

**Type:** `string[]`  
**Description:** Array of file paths that contain critical severity vulnerabilities

### 6. Aggregated bySeverity (`bySeverity`)

**Type:** `Record<SecuritySeverity, number>`  
**Description:** Sum of vulnerability counts by severity across all files

### 7. Aggregated byType (`byType`)

**Type:** `Record<SecurityVulnerabilityType, number>`  
**Description:** Sum of vulnerability counts by type across all files

## Configuration Options

The analyzer supports configuration that affects metric calculation:

### SecurityAnalyzerConfig

```typescript
interface SecurityAnalyzerConfig {
  registry?: AntiPatternRegistry; // Custom registry (default: security-focused)
  focusAreas?: SecurityVulnerabilityType[]; // Filter to specific vulnerability types
  minSeverity?: SecuritySeverity; // Minimum severity to report (default: 'low')
  includeRelatedRules?: boolean; // Include non-security rules with security implications
}
```

**Impact on Metrics:**

- `focusAreas`: Filters `vulnerabilities` array and affects all type-based metrics
- `minSeverity`: Filters out vulnerabilities below threshold before metric calculation
- `includeRelatedRules`: Affects which anti-pattern categories are analyzed

## Utility Functions for Metric Access

The analyzer provides helper functions to extract specific metric subsets:

- `hasCriticalVulnerabilities(result)` - Returns `boolean` if any critical vulnerabilities exist
- `hasHighPriorityVulnerabilities(result)` - Returns `boolean` if critical or high vulnerabilities exist
- `getHighPriorityVulnerabilities(result)` - Returns `SecurityVulnerability[]` with critical and high severity
- `getVulnerabilitiesByType(result, type)` - Returns `SecurityVulnerability[]` filtered by type
- `getXSSVulnerabilities(result)` - Returns XSS vulnerabilities
- `getSensitiveDataVulnerabilities(result)` - Returns sensitive data vulnerabilities
- `getInjectionVulnerabilities(result)` - Returns injection vulnerabilities
- `passesSecurityCheck(result, threshold)` - Returns `boolean` if score >= threshold (default: 80)

## Implementation Notes

1. **Score Calculation:** Both `score` and `complianceScore` use penalty-based systems but with different weights. The compliance score is more punitive for critical issues.

2. **Severity Mapping:** Anti-pattern severities (`'warning'`, `'info'`) are mapped to security severities (`'medium'`, `'low'`) during aggregation.

3. **Default Vulnerability Type:** If a security rule doesn't have a mapping, it defaults to `'xss'` type. This ensures all security-category anti-patterns are classified.

4. **Empty State:** When no vulnerabilities are found:
   - `score`: 100
   - `complianceScore`: 100
   - `riskLevel`: `'low'`
   - All counts: 0
   - `vulnerabilities`: `[]`

5. **Filtering:** Vulnerabilities are filtered by:
   - Security category (must be `'security'` category OR have a rule mapping)
   - Focus areas (if specified in config)
   - Minimum severity (if specified in config)

## Example Result Structure

```typescript
{
  // Core metrics
  score: 72,
  riskLevel: 'high',
  complianceScore: 65,

  // Counts
  byType: {
    xss: 2,
    injection: 1,
    'sensitive-data': 0,
    authentication: 0,
    'access-control': 0,
    cryptography: 0,
    'input-validation': 0
  },
  bySeverity: {
    critical: 0,
    high: 2,
    medium: 1,
    low: 0
  },

  // Detailed vulnerabilities
  vulnerabilities: [
    {
      id: 'dangerous-html',
      name: 'Dangerous HTML Usage',
      category: 'security',
      severity: 'high',
      vulnerabilityType: 'xss',
      cwe: 'CWE-79',
      owasp: 'A03:2021',
      location: { file: 'Component.tsx', line: 15, column: 8 },
      description: '...',
      impact: '...',
      fix: '...',
      // ... other fields
    }
    // ... more vulnerabilities
  ],

  // Metadata
  metadata: {
    analyzer: 'SecurityAnalyzer',
    timestamp: '2024-01-15T10:30:00.000Z',
    durationMs: 45,
    filePath: 'src/Component.tsx'
  }
}
```
