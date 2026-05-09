# Phase 05: Security Analysis

**Priority:** Medium - Risk mitigation  
**Complexity:** Low-Medium  
**Dependencies:** Phase 00 (Foundation) ✅, Phase 02 (Anti-Pattern System) ✅  
**Status:** Ready for implementation

**Phase Duration:** 1-2 days  
**Estimated Effort:** 6-8 hours

All code examples use the `ts-morph` API for AST parsing and traversal.

## Overview

This phase extends the existing security detection capabilities by adding more comprehensive security rules and creating a dedicated security analyzer. The codebase already has basic security detection through the `dangerous-html` rule in the JSX anti-pattern rules. This phase will add additional security rules and create a focused security analyzer that can be used independently or integrated with the anti-pattern system.

## Current State

### Already Implemented ✅

- **Anti-Pattern System**: Fully functional rule-based detection system using `ts-morph`
- **Base Analyzer**: `BaseAnalyzer<TResult>` abstract class for analyzers
- **Anti-Pattern Registry**: `AntiPatternRegistry` for rule management
- **Existing Security Rule**: `dangerous-html` rule in JSX rules (detects `dangerouslySetInnerHTML`)
- **Type System**: Core types including `AntiPattern`, `AntiPatternRule`, `CodeLocation`

### What This Phase Adds 🆕

- Additional security-focused anti-pattern rules
- Dedicated `SecurityAnalyzer` class for focused security analysis
- Security-specific types and metrics
- Security formatter for reporting
- Comprehensive security tests

## Business Value

- Catch security vulnerabilities early in development
- Reduce security review burden
- Ensure compliance with security best practices
- Prevent XSS, injection, and data exposure issues
- Support security-focused code reviews

## Technical Approach

### Architecture Decision

The implementation will follow a **hybrid approach**:

1. **Security Rules** will be added to the existing anti-pattern rule system (`src/rules/anti-patterns/security-rules.ts`)
2. **Security Analyzer** will extend `BaseAnalyzer` and use `AntiPatternRegistry` internally
3. Rules will be registered with the `AntiPatternRegistry` and can be used by both:
   - `AntiPatternAnalyzer` (for general analysis)
   - `SecurityAnalyzer` (for focused security analysis)

This approach provides:

- ✅ Consistency with existing architecture
- ✅ Reusability of rules
- ✅ Focused security analysis when needed
- ✅ Integration with existing formatters and CLI

## Detailed Tasks

### Task 5.1: Security Types

**Model:** Opus (security-critical types)  
**File:** `src/types/security.ts`  
**Estimated Time:** 45 minutes

Create type definitions that extend the existing anti-pattern types for security-specific use cases.

```typescript
/**
 * Security Type Definitions
 *
 * Extends the anti-pattern system with security-specific types and metrics.
 *
 * @module types/security
 */

import type { CodeLocation } from './core';
import type { AntiPattern, AntiPatternSeverity } from './anti-patterns';

/**
 * Security vulnerability types
 */
export type SecurityVulnerabilityType =
  | 'xss'
  | 'injection'
  | 'sensitive-data'
  | 'authentication'
  | 'access-control'
  | 'cryptography'
  | 'input-validation';

/**
 * Security severity levels (aligned with anti-pattern severity)
 */
export type SecuritySeverity = 'critical' | 'high' | 'medium' | 'low';

/**
 * Detected security vulnerability (extends AntiPattern)
 */
export interface SecurityVulnerability extends AntiPattern {
  /** Vulnerability type classification */
  vulnerabilityType: SecurityVulnerabilityType;
  /** CWE ID if applicable */
  cwe?: string;
  /** OWASP category if applicable */
  owasp?: string;
}

/**
 * Complete security metrics
 */
export interface SecurityMetrics {
  /** Overall security score (0-100, higher is more secure) */
  score: number;
  /** Risk level assessment */
  riskLevel: SecuritySeverity;
  /** Detected vulnerabilities */
  vulnerabilities: SecurityVulnerability[];
  /** OWASP compliance score (0-100) */
  complianceScore: number;
  /** Vulnerability counts by type */
  byType: Record<SecurityVulnerabilityType, number>;
  /** Vulnerability counts by severity */
  bySeverity: Record<SecuritySeverity, number>;
}

/**
 * Helper to convert AntiPattern to SecurityVulnerability
 */
export function toSecurityVulnerability(
  antiPattern: AntiPattern,
  vulnerabilityType: SecurityVulnerabilityType,
  cwe?: string,
  owasp?: string
): SecurityVulnerability {
  return {
    ...antiPattern,
    vulnerabilityType,
    cwe,
    owasp,
  };
}
```

### Task 5.2: Security Anti-Pattern Rules

**Model:** Sonnet (pattern detection)  
**File:** `src/rules/anti-patterns/security-rules.ts`  
**Estimated Time:** 2.5 hours

Create security-focused anti-pattern rules that integrate with the existing rule system. Note: `dangerous-html` already exists in `jsx-rules.ts`.

```typescript
/**
 * Security Anti-Pattern Rules
 *
 * Detects security vulnerabilities in React applications including XSS risks,
 * sensitive data exposure, and dangerous patterns.
 *
 * Rules:
 * - dynamic-url-injection: Dynamic URLs that could contain javascript: protocol
 * - sensitive-data-exposure: Hardcoded secrets, API keys, tokens
 * - sensitive-data-logging: Logging sensitive information to console
 * - sensitive-data-storage: Storing sensitive data in localStorage/sessionStorage
 * - eval-usage: Usage of eval() or Function() constructor
 *
 * Note: dangerous-html rule already exists in jsx-rules.ts
 *
 * @module rules/anti-patterns/security-rules
 */

import { SourceFile, Node, SyntaxKind } from 'ts-morph';
import type { AntiPatternRule, AntiPattern } from '../../types/anti-patterns';
import { createAntiPattern } from './utils';

// ============================================================================
// Rule: Dynamic URL Injection
// ============================================================================

/**
 * Dynamic URL Injection Detection
 *
 * Detects dynamic URLs in href, src, action attributes that could be exploited
 * via javascript: protocol or other injection vectors.
 */
const dynamicUrlRule: AntiPatternRule = {
  id: 'dynamic-url-injection',
  name: 'Dynamic URL Injection',
  category: 'security',
  defaultSeverity: 'high',
  description: 'Dynamic URLs can be used for XSS attacks via javascript: protocol',
  references: [
    'https://owasp.org/www-community/attacks/xss/',
    'https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html',
  ],
  fixable: false,

  check(sourceFile: SourceFile): AntiPattern[] {
    const violations: AntiPattern[] = [];

    sourceFile.forEachDescendant(node => {
      if (!Node.isJsxAttribute(node)) return;

      const nameNode = node.getNameNode();
      if (!Node.isIdentifier(nameNode)) return;

      const attrName = nameNode.getText();

      // Check href, src, action, formAction
      if (!['href', 'src', 'action', 'formAction'].includes(attrName)) return;

      const initializer = node.getInitializer();
      if (!Node.isJsxExpression(initializer)) return;

      const expr = initializer.getExpression();
      if (!expr) return;

      // Skip static strings
      if (Node.isStringLiteral(expr) || Node.isNoSubstitutionTemplateLiteral(expr)) {
        return;
      }

      let isHighRisk = false;
      let reason = '';
      let severity: 'high' | 'critical' = 'high';

      // Template literals with expressions
      if (Node.isTemplateExpression(expr)) {
        isHighRisk = true;
        reason = 'template literal with dynamic expressions';
        severity = 'high';
      }

      // Direct variable
      if (Node.isIdentifier(expr)) {
        const name = expr.getText();
        if (/url|link|href|src|path|redirect/i.test(name)) {
          isHighRisk = true;
          reason = `dynamic "${attrName}" from variable "${name}"`;
          severity = attrName === 'href' ? 'critical' : 'high';
        }
      }

      // Function call
      if (Node.isCallExpression(expr)) {
        isHighRisk = true;
        reason = `dynamic "${attrName}" from function return value`;
        severity = attrName === 'href' ? 'critical' : 'high';
      }

      if (isHighRisk) {
        violations.push(
          createAntiPattern({
            id: 'dynamic-url-injection',
            name: `Dynamic ${attrName}`,
            category: 'security',
            severity,
            node,
            description: `${reason} - potential javascript: protocol injection`,
            impact:
              'Attacker could inject javascript: URLs to execute arbitrary code. This is a common XSS vector.',
            fix: 'Validate URL protocol (only allow http:, https:, mailto:) before rendering. Use a URL validation library or whitelist approach.',
            context: { attrName, reason },
            references: [
              'https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html',
            ],
          })
        );
      }
    });

    return violations;
  },
};

// ============================================================================
// Rule: Sensitive Data Exposure
// ============================================================================

/**
 * Sensitive Data Exposure Detection
 *
 * Detects hardcoded sensitive data like API keys, secrets, passwords, tokens.
 */
const sensitiveDataExposureRule: AntiPatternRule = {
  id: 'sensitive-data-exposure',
  name: 'Sensitive Data Exposure',
  category: 'security',
  defaultSeverity: 'critical',
  description: 'Sensitive data like API keys should not be hardcoded in client code',
  references: [
    'https://owasp.org/Top10/A02_2021-Cryptographic_Failures/',
    'https://cwe.mitre.org/data/definitions/798.html',
  ],
  fixable: false,

  check(sourceFile: SourceFile): AntiPattern[] {
    const violations: AntiPattern[] = [];

    // Patterns for sensitive data
    const sensitivePatterns = [
      { pattern: /api[_-]?key/i, name: 'API key', cwe: 'CWE-798' },
      { pattern: /secret/i, name: 'secret', cwe: 'CWE-798' },
      { pattern: /password/i, name: 'password', cwe: 'CWE-259' },
      { pattern: /private[_-]?key/i, name: 'private key', cwe: 'CWE-798' },
      { pattern: /auth[_-]?token/i, name: 'auth token', cwe: 'CWE-798' },
      { pattern: /bearer/i, name: 'bearer token', cwe: 'CWE-798' },
      { pattern: /jwt/i, name: 'JWT token', cwe: 'CWE-798' },
      { pattern: /credentials/i, name: 'credentials', cwe: 'CWE-798' },
      { pattern: /access[_-]?token/i, name: 'access token', cwe: 'CWE-798' },
    ];

    sourceFile.forEachDescendant(node => {
      // Check variable declarations
      if (Node.isVariableDeclaration(node)) {
        const nameNode = node.getNameNode();
        if (!Node.isIdentifier(nameNode)) return;

        const varName = nameNode.getText();
        const init = node.getInitializer();

        // Check if variable name matches sensitive patterns
        for (const { pattern, name, cwe } of sensitivePatterns) {
          if (pattern.test(varName)) {
            // Check if it's a hardcoded string (likely a real secret)
            if (
              init &&
              (Node.isStringLiteral(init) || Node.isNoSubstitutionTemplateLiteral(init)) &&
              init.getText().length > 12 // Reasonable minimum length for real secrets
            ) {
              violations.push(
                createAntiPattern({
                  id: 'sensitive-data-exposure',
                  name: `Hardcoded ${name}`,
                  category: 'security',
                  severity: 'critical',
                  node,
                  description: `Hardcoded ${name} detected in variable "${varName}"`,
                  impact:
                    'Sensitive credentials exposed in client-side code can be extracted by attackers and used to compromise your systems.',
                  fix: 'Use environment variables (NEXT_PUBLIC_*, REACT_APP_*, VITE_*) for public keys, or handle sensitive keys server-side only.',
                  context: { varName, sensitiveType: name, cwe },
                  references: ['https://cwe.mitre.org/data/definitions/798.html'],
                })
              );
            }
          }
        }
      }
    });

    return violations;
  },
};

// ============================================================================
// Rule: Sensitive Data Logging
// ============================================================================

/**
 * Sensitive Data Logging Detection
 *
 * Detects console logging of potentially sensitive information.
 */
const sensitiveDataLoggingRule: AntiPatternRule = {
  id: 'sensitive-data-logging',
  name: 'Sensitive Data Logging',
  category: 'security',
  defaultSeverity: 'high',
  description: 'Sensitive data should not be logged to console',
  references: ['https://owasp.org/Top10/A09_2021-Security_Logging_and_Monitoring_Failures/'],
  fixable: true,

  check(sourceFile: SourceFile): AntiPattern[] {
    const violations: AntiPattern[] = [];

    const sensitivePatterns = [
      { pattern: /api[_-]?key/i, name: 'API key' },
      { pattern: /secret/i, name: 'secret' },
      { pattern: /password/i, name: 'password' },
      { pattern: /token/i, name: 'token' },
      { pattern: /credentials/i, name: 'credentials' },
      { pattern: /private[_-]?key/i, name: 'private key' },
      { pattern: /auth/i, name: 'authentication data' },
    ];

    sourceFile.forEachDescendant(node => {
      if (!Node.isCallExpression(node)) return;

      const expr = node.getExpression();
      if (!Node.isPropertyAccessExpression(expr)) return;

      const obj = expr.getExpression();
      const prop = expr.getName();

      if (
        !Node.isIdentifier(obj) ||
        obj.getText() !== 'console' ||
        !['log', 'warn', 'error', 'info', 'debug'].includes(prop)
      ) {
        return;
      }

      // Check arguments for sensitive variable names
      const args = node.getArguments();
      args.forEach(arg => {
        if (Node.isIdentifier(arg)) {
          const name = arg.getText();
          for (const { pattern, name: sensitiveType } of sensitivePatterns) {
            if (pattern.test(name)) {
              violations.push(
                createAntiPattern({
                  id: 'sensitive-data-logging',
                  name: 'Sensitive Data Logged',
                  category: 'security',
                  severity: 'high',
                  node,
                  description: `Potential ${sensitiveType} "${name}" logged to console`,
                  impact:
                    'Sensitive data may be exposed in browser console, log aggregation systems, or error tracking services.',
                  fix: `Remove console.${prop}() call or redact sensitive data before logging`,
                  autoFixable: true,
                  context: { varName: name, sensitiveType, consoleMethod: prop },
                  references: [],
                })
              );
            }
          }
        }
      });
    });

    return violations;
  },
};

// ============================================================================
// Rule: Sensitive Data Storage
// ============================================================================

/**
 * Sensitive Data Storage Detection
 *
 * Detects storage of sensitive data in localStorage or sessionStorage.
 */
const sensitiveDataStorageRule: AntiPatternRule = {
  id: 'sensitive-data-storage',
  name: 'Sensitive Data in Storage',
  category: 'security',
  defaultSeverity: 'high',
  description: 'Sensitive data should not be stored in localStorage/sessionStorage',
  references: [
    'https://cheatsheetseries.owasp.org/cheatsheets/HTML5_Security_Cheat_Sheet.html',
    'https://owasp.org/www-community/vulnerabilities/Insecure_Storage',
  ],
  fixable: false,

  check(sourceFile: SourceFile): AntiPattern[] {
    const violations: AntiPattern[] = [];

    const sensitivePatterns = [
      { pattern: /api[_-]?key/i, name: 'API key' },
      { pattern: /secret/i, name: 'secret' },
      { pattern: /password/i, name: 'password' },
      { pattern: /token/i, name: 'token' },
      { pattern: /credentials/i, name: 'credentials' },
      { pattern: /auth/i, name: 'authentication data' },
      { pattern: /jwt/i, name: 'JWT token' },
    ];

    sourceFile.forEachDescendant(node => {
      if (!Node.isCallExpression(node)) return;

      const expr = node.getExpression();
      if (!Node.isPropertyAccessExpression(expr)) return;

      const obj = expr.getExpression();
      const method = expr.getName();

      if (
        !Node.isIdentifier(obj) ||
        !['localStorage', 'sessionStorage'].includes(obj.getText()) ||
        method !== 'setItem'
      ) {
        return;
      }

      const args = node.getArguments();
      if (args.length < 1) return;

      const keyArg = args[0];
      if (Node.isStringLiteral(keyArg) || Node.isNoSubstitutionTemplateLiteral(keyArg)) {
        const key = keyArg.getLiteralText();
        const storageType = obj.getText();

        for (const { pattern, name } of sensitivePatterns) {
          if (pattern.test(key)) {
            violations.push(
              createAntiPattern({
                id: 'sensitive-data-storage',
                name: `Sensitive Data in ${storageType}`,
                category: 'security',
                severity: 'high',
                node,
                description: `Potential ${name} stored in ${storageType} with key "${key}"`,
                impact:
                  'Sensitive data in web storage is accessible to any JavaScript on the page, including third-party scripts. This increases XSS risk.',
                fix: 'Use secure HTTP-only cookies for sensitive data, or encrypt data before storing in web storage.',
                context: { key, sensitiveType: name, storageType },
                references: [
                  'https://cheatsheetseries.owasp.org/cheatsheets/HTML5_Security_Cheat_Sheet.html',
                ],
              })
            );
          }
        }
      }
    });

    return violations;
  },
};

// ============================================================================
// Rule: eval() Usage
// ============================================================================

/**
 * eval() Usage Detection
 *
 * Detects usage of eval() or Function() constructor which can execute arbitrary code.
 */
const evalUsageRule: AntiPatternRule = {
  id: 'eval-usage',
  name: 'eval() Usage',
  category: 'security',
  defaultSeverity: 'critical',
  description: 'eval() and Function() constructor can execute arbitrary code',
  references: [
    'https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/eval#never_use_eval!',
    'https://owasp.org/www-community/attacks/Code_Injection',
  ],
  fixable: false,

  check(sourceFile: SourceFile): AntiPattern[] {
    const violations: AntiPattern[] = [];

    sourceFile.forEachDescendant(node => {
      if (!Node.isCallExpression(node)) return;

      const expr = node.getExpression();

      // Check for eval()
      if (Node.isIdentifier(expr) && expr.getText() === 'eval') {
        const args = node.getArguments();
        const argText = args.length > 0 ? args[0]!.getText() : 'unknown';

        violations.push(
          createAntiPattern({
            id: 'eval-usage',
            name: 'eval() Call',
            category: 'security',
            severity: 'critical',
            node,
            description: `eval() called with: ${argText}`,
            impact:
              'eval() executes arbitrary JavaScript code, which can be exploited for code injection attacks if the input contains user data.',
            fix: 'Use safer alternatives like JSON.parse() for data, or refactor to avoid dynamic code execution.',
            context: { method: 'eval', argument: argText },
            references: [
              'https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/eval#never_use_eval!',
            ],
          })
        );
      }

      // Check for new Function()
      if (Node.isNewExpression(node)) {
        const newExpr = node.getExpression();
        if (Node.isIdentifier(newExpr) && newExpr.getText() === 'Function') {
          violations.push(
            createAntiPattern({
              id: 'eval-usage',
              name: 'Function() Constructor',
              category: 'security',
              severity: 'critical',
              node,
              description: 'Function() constructor creates functions from strings',
              impact:
                'Function constructor is similar to eval() and can execute arbitrary code, creating code injection risks.',
              fix: 'Use normal function declarations or arrow functions instead of dynamic function creation.',
              context: { method: 'Function' },
              references: [
                'https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function',
              ],
            })
          );
        }
      }
    });

    return violations;
  },
};

// ============================================================================
// Export All Security Rules
// ============================================================================

/**
 * All security anti-pattern rules
 *
 * Note: dangerous-html rule is in jsx-rules.ts and will be included
 * in security analysis via category filtering
 */
export const SECURITY_RULES: AntiPatternRule[] = [
  dynamicUrlRule,
  sensitiveDataExposureRule,
  sensitiveDataLoggingRule,
  sensitiveDataStorageRule,
  evalUsageRule,
];

// Named exports for individual rules
export {
  dynamicUrlRule,
  sensitiveDataExposureRule,
  sensitiveDataLoggingRule,
  sensitiveDataStorageRule,
  evalUsageRule,
};
```

### Task 5.3: Update Anti-Pattern Registry

**Model:** Haiku (simple integration)  
**File:** `src/rules/anti-pattern-registry.ts`  
**Estimated Time:** 10 minutes

Update the registry to include security rules:

```typescript
// In registerBuiltinRules() method, add:
import { SECURITY_RULES } from './anti-patterns/security-rules';

private registerBuiltinRules(): void {
  const allRules = [
    ...HOOKS_RULES,
    ...PERFORMANCE_RULES,
    ...STATE_RULES,
    ...JSX_RULES,
    ...SECURITY_RULES, // Add this line
  ];

  allRules.forEach(rule => this.register(rule));
}
```

### Task 5.4: Security Analyzer

**Model:** Opus (business logic)  
**File:** `src/analyzers/security-analyzer.ts`  
**Estimated Time:** 1.5 hours

Create a focused security analyzer that uses the anti-pattern registry.

```typescript
/**
 * Security Analyzer
 *
 * Dedicated analyzer for security vulnerability detection in React applications.
 * Uses the anti-pattern registry with security-focused configuration.
 *
 * @module analyzers/security-analyzer
 */

import { Project, SourceFile } from 'ts-morph';
import {
  BaseAnalyzer,
  type BaseAnalyzerResult,
  type AnalyzerConfig,
} from './base-analyzer-tsmorph';
import { AntiPatternRegistry, createAntiPatternRegistry } from '../rules/anti-pattern-registry';
import type { AntiPattern, AntiPatternCategory } from '../types/anti-patterns';
import type {
  SecurityMetrics,
  SecurityVulnerability,
  SecurityVulnerabilityType,
  SecuritySeverity,
} from '../types/security';
import { toSecurityVulnerability } from '../types/security';

// ============================================================================
// Configuration Types
// ============================================================================

/**
 * Security analyzer configuration
 */
export interface SecurityAnalyzerConfig extends AnalyzerConfig {
  /** Custom anti-pattern registry (if not provided, creates security-focused one) */
  registry?: AntiPatternRegistry;
  /** Focus on specific vulnerability types */
  focusAreas?: SecurityVulnerabilityType[];
}

/**
 * Security analyzer result
 */
export interface SecurityAnalyzerResult extends SecurityMetrics, BaseAnalyzerResult {}

// ============================================================================
// Security Analyzer Class
// ============================================================================

/**
 * Security Analyzer
 *
 * Focused analyzer for detecting security vulnerabilities in React code.
 * Uses the anti-pattern registry internally with security-specific configuration.
 */
export class SecurityAnalyzer extends BaseAnalyzer<SecurityAnalyzerResult> {
  private registry: AntiPatternRegistry;
  private config: SecurityAnalyzerConfig;

  constructor(ast: SourceFile, config: SecurityAnalyzerConfig = {}) {
    super(ast, config);
    this.config = config;

    // Create registry focused on security rules
    this.registry =
      config.registry ??
      createAntiPatternRegistry({
        // Focus on security category
        ignoreCategories: this.getIgnoredCategories(),
        minSeverity: 'low', // Catch all security issues
      });
  }

  /**
   * Get categories to ignore based on configuration
   */
  private getIgnoredCategories(): AntiPatternCategory[] {
    // Ignore non-security categories for focused analysis
    const allCategories: AntiPatternCategory[] = [
      'hooks',
      'performance',
      'state-management',
      'lifecycle',
      'jsx',
      'props',
      'testing',
      'accessibility',
    ];

    // Keep only 'security' category
    return allCategories;
  }

  protected analyzeAST(): Omit<SecurityAnalyzerResult, 'metadata'> {
    // Run security rules
    const antiPatterns = this.registry.analyze(this.ast);

    // Convert to security vulnerabilities
    const vulnerabilities = this.convertToVulnerabilities(antiPatterns);

    // Filter by focus areas if specified
    const filteredVulnerabilities = this.config.focusAreas
      ? vulnerabilities.filter(v => this.config.focusAreas!.includes(v.vulnerabilityType))
      : vulnerabilities;

    // Calculate metrics
    const byType = this.groupByType(filteredVulnerabilities);
    const bySeverity = this.groupBySeverity(filteredVulnerabilities);
    const riskLevel = this.calculateRiskLevel(filteredVulnerabilities);
    const complianceScore = this.calculateComplianceScore(filteredVulnerabilities);
    const score = Math.max(0, 100 - this.calculateDeductions(filteredVulnerabilities));

    return {
      score,
      riskLevel,
      vulnerabilities: filteredVulnerabilities,
      complianceScore,
      byType,
      bySeverity,
    };
  }

  /**
   * Convert anti-patterns to security vulnerabilities
   */
  private convertToVulnerabilities(antiPatterns: AntiPattern[]): SecurityVulnerability[] {
    return antiPatterns
      .filter(ap => ap.category === 'security')
      .map(ap => {
        const { vulnerabilityType, cwe, owasp } = this.mapToVulnerabilityInfo(ap.id);
        return toSecurityVulnerability(ap, vulnerabilityType, cwe, owasp);
      });
  }

  /**
   * Map rule ID to vulnerability metadata
   */
  private mapToVulnerabilityInfo(ruleId: string): {
    vulnerabilityType: SecurityVulnerabilityType;
    cwe?: string;
    owasp?: string;
  } {
    const mapping: Record<
      string,
      { type: SecurityVulnerabilityType; cwe?: string; owasp?: string }
    > = {
      'dangerous-html': { type: 'xss', cwe: 'CWE-79', owasp: 'A03:2021' },
      'dynamic-url-injection': { type: 'xss', cwe: 'CWE-79', owasp: 'A03:2021' },
      'sensitive-data-exposure': { type: 'sensitive-data', cwe: 'CWE-798', owasp: 'A02:2021' },
      'sensitive-data-logging': { type: 'sensitive-data', cwe: 'CWE-532', owasp: 'A09:2021' },
      'sensitive-data-storage': { type: 'sensitive-data', cwe: 'CWE-922', owasp: 'A02:2021' },
      'eval-usage': { type: 'injection', cwe: 'CWE-95', owasp: 'A03:2021' },
    };

    const info = mapping[ruleId];
    return {
      vulnerabilityType: info?.type ?? 'xss',
      cwe: info?.cwe,
      owasp: info?.owasp,
    };
  }

  /**
   * Group vulnerabilities by type
   */
  private groupByType(
    vulnerabilities: SecurityVulnerability[]
  ): Record<SecurityVulnerabilityType, number> {
    const result: Partial<Record<SecurityVulnerabilityType, number>> = {};

    vulnerabilities.forEach(v => {
      result[v.vulnerabilityType] = (result[v.vulnerabilityType] ?? 0) + 1;
    });

    // Ensure all types are present
    const allTypes: SecurityVulnerabilityType[] = [
      'xss',
      'injection',
      'sensitive-data',
      'authentication',
      'access-control',
      'cryptography',
      'input-validation',
    ];

    allTypes.forEach(type => {
      if (!(type in result)) {
        result[type] = 0;
      }
    });

    return result as Record<SecurityVulnerabilityType, number>;
  }

  /**
   * Group vulnerabilities by severity
   */
  private groupBySeverity(
    vulnerabilities: SecurityVulnerability[]
  ): Record<SecuritySeverity, number> {
    return {
      critical: vulnerabilities.filter(v => v.severity === 'critical').length,
      high: vulnerabilities.filter(v => v.severity === 'high').length,
      medium: vulnerabilities.filter(v => v.severity === 'medium').length,
      low: vulnerabilities.filter(v => v.severity === 'low').length,
    };
  }

  /**
   * Calculate overall risk level
   */
  private calculateRiskLevel(vulnerabilities: SecurityVulnerability[]): SecuritySeverity {
    if (vulnerabilities.some(v => v.severity === 'critical')) return 'critical';
    if (vulnerabilities.some(v => v.severity === 'high')) return 'high';
    if (vulnerabilities.some(v => v.severity === 'medium')) return 'medium';
    return 'low';
  }

  /**
   * Calculate OWASP compliance score
   */
  private calculateComplianceScore(vulnerabilities: SecurityVulnerability[]): number {
    const criticalPenalty = vulnerabilities.filter(v => v.severity === 'critical').length * 30;
    const highPenalty = vulnerabilities.filter(v => v.severity === 'high').length * 20;
    const mediumPenalty = vulnerabilities.filter(v => v.severity === 'medium').length * 10;
    const lowPenalty = vulnerabilities.filter(v => v.severity === 'low').length * 5;

    return Math.max(0, 100 - criticalPenalty - highPenalty - mediumPenalty - lowPenalty);
  }

  /**
   * Calculate score deductions based on vulnerabilities
   */
  private calculateDeductions(vulnerabilities: SecurityVulnerability[]): number {
    return vulnerabilities.reduce((sum, v) => {
      switch (v.severity) {
        case 'critical':
          return sum + 25;
        case 'high':
          return sum + 15;
        case 'medium':
          return sum + 8;
        case 'low':
          return sum + 3;
        default:
          return sum;
      }
    }, 0);
  }

  /**
   * Get the internal registry for advanced configuration
   */
  getRegistry(): AntiPatternRegistry {
    return this.registry;
  }
}

// ============================================================================
// Utility Functions
// ============================================================================

/**
 * Analyze security from source code
 */
export function analyzeSecurityFromSource(
  code: string,
  config?: SecurityAnalyzerConfig
): SecurityAnalyzerResult {
  const project = new Project({
    useInMemoryFileSystem: true,
    compilerOptions: {
      jsx: 4, // React JSX
      target: 99, // ESNext
      module: 99, // ESNext
    },
  });

  const sourceFile = project.createSourceFile('temp.tsx', code);
  const analyzer = new SecurityAnalyzer(sourceFile, config);
  return analyzer.analyze();
}

/**
 * Check if there are critical security vulnerabilities
 */
export function hasCriticalVulnerabilities(result: SecurityAnalyzerResult): boolean {
  return result.bySeverity.critical > 0;
}

/**
 * Get high-priority vulnerabilities (critical + high)
 */
export function getHighPriorityVulnerabilities(
  result: SecurityAnalyzerResult
): SecurityVulnerability[] {
  return result.vulnerabilities.filter(v => v.severity === 'critical' || v.severity === 'high');
}
```

### Task 5.5: Security Formatter

**Model:** Sonnet (formatting logic)  
**File:** `src/formatters/security-formatter.ts`  
**Estimated Time:** 45 minutes

Create a formatter for security analysis results.

```typescript
/**
 * Security Formatter
 *
 * Formats security analysis results for console output.
 */

import consola from 'consola';
import type { SecurityAnalyzerResult, SecurityVulnerability } from '../types/security';

export function formatSecurityReport(result: SecurityAnalyzerResult): void {
  consola.box('🔒 Security Analysis Report');

  // Overall score and risk
  const riskEmoji = {
    critical: '🔴',
    high: '🟠',
    medium: '🟡',
    low: '🟢',
  };

  consola.info(`Security Score: ${result.score}/100`);
  consola.info(`Risk Level: ${riskEmoji[result.riskLevel]} ${result.riskLevel.toUpperCase()}`);
  consola.info(`OWASP Compliance: ${result.complianceScore}/100`);
  consola.log('');

  // Summary by severity
  consola.start('Vulnerabilities by Severity:');
  if (result.bySeverity.critical > 0) {
    consola.error(`  🔴 Critical: ${result.bySeverity.critical}`);
  }
  if (result.bySeverity.high > 0) {
    consola.warn(`  🟠 High: ${result.bySeverity.high}`);
  }
  if (result.bySeverity.medium > 0) {
    consola.log(`  🟡 Medium: ${result.bySeverity.medium}`);
  }
  if (result.bySeverity.low > 0) {
    consola.log(`  🟢 Low: ${result.bySeverity.low}`);
  }
  consola.log('');

  // Summary by type
  const typesWithIssues = Object.entries(result.byType).filter(([_, count]) => count > 0);
  if (typesWithIssues.length > 0) {
    consola.start('Vulnerabilities by Type:');
    typesWithIssues.forEach(([type, count]) => {
      consola.log(`  ${type}: ${count}`);
    });
    consola.log('');
  }

  // Detailed vulnerabilities
  if (result.vulnerabilities.length > 0) {
    consola.start('Detected Vulnerabilities:');
    result.vulnerabilities.forEach((vuln, index) => {
      formatVulnerability(vuln, index + 1);
    });
  } else {
    consola.success('✅ No security vulnerabilities detected!');
  }
}

function formatVulnerability(vuln: SecurityVulnerability, index: number): void {
  const severityColors = {
    critical: 'red',
    high: 'yellow',
    medium: 'blue',
    low: 'gray',
  };

  consola.log('');
  consola.log(`${index}. ${vuln.name} [${vuln.severity.toUpperCase()}]`);
  consola.log(`   Location: Line ${vuln.location.line}, Column ${vuln.location.column}`);
  consola.log(`   Type: ${vuln.vulnerabilityType}`);
  if (vuln.cwe) {
    consola.log(`   CWE: ${vuln.cwe}`);
  }
  if (vuln.owasp) {
    consola.log(`   OWASP: ${vuln.owasp}`);
  }
  consola.log(`   Description: ${vuln.description}`);
  consola.log(`   Impact: ${vuln.impact}`);
  consola.log(`   Fix: ${vuln.fix}`);

  if (vuln.references && vuln.references.length > 0) {
    consola.log(`   References:`);
    vuln.references.forEach(ref => consola.log(`     - ${ref}`));
  }
}

export function formatSecurityJSON(result: SecurityAnalyzerResult): string {
  return JSON.stringify(result, null, 2);
}
```

### Task 5.6: Update Type Exports

**Model:** Haiku (simple update)  
**File:** `src/types/index.ts`  
**Estimated Time:** 5 minutes

Add security type exports:

```typescript
// Add to existing exports:
export type {
  SecurityVulnerabilityType,
  SecuritySeverity,
  SecurityVulnerability,
  SecurityMetrics,
} from './security';

export { toSecurityVulnerability } from './security';
```

### Task 5.7: Update Rules Index

**Model:** Haiku (simple update)  
**File:** `src/rules/anti-patterns/index.ts`  
**Estimated Time:** 5 minutes

Add security rules to exports:

```typescript
// Add imports and exports
export { SECURITY_RULES } from './security-rules';
export type {} from './security-rules';
export {
  dynamicUrlRule,
  sensitiveDataExposureRule,
  sensitiveDataLoggingRule,
  sensitiveDataStorageRule,
  evalUsageRule,
} from './security-rules';
```

## Testing Strategy

### Task 5.8: Security Analyzer Tests

**Model:** Sonnet (test implementation)  
**File:** `src/analyzers/security-analyzer.test.ts`  
**Estimated Time:** 1.5 hours

```typescript
/**
 * Security Analyzer Tests
 */

import { Project, SourceFile } from 'ts-morph';
import { describe, it, expect } from 'vitest';
import {
  SecurityAnalyzer,
  analyzeSecurityFromSource,
  hasCriticalVulnerabilities,
  getHighPriorityVulnerabilities,
} from './security-analyzer';

function createSourceFile(code: string): SourceFile {
  const project = new Project({
    useInMemoryFileSystem: true,
    compilerOptions: {
      jsx: 4,
      target: 99,
      module: 99,
    },
  });
  return project.createSourceFile('test.tsx', code);
}

describe('SecurityAnalyzer', () => {
  describe('clean code', () => {
    it('should score 100 for secure code', () => {
      const code = `
        function Component() {
          const [count, setCount] = useState(0);
          return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
        }
      `;
      const result = analyzeSecurityFromSource(code);

      expect(result.score).toBe(100);
      expect(result.riskLevel).toBe('low');
      expect(result.vulnerabilities).toHaveLength(0);
    });
  });

  describe('XSS vulnerabilities', () => {
    it('should detect dangerouslySetInnerHTML', () => {
      const code = `
        function Component({ html }) {
          return <div dangerouslySetInnerHTML={{ __html: html }} />;
        }
      `;
      const result = analyzeSecurityFromSource(code);

      expect(result.vulnerabilities.length).toBeGreaterThan(0);
      expect(result.vulnerabilities.some(v => v.id === 'dangerous-html')).toBe(true);
      expect(result.byType.xss).toBeGreaterThan(0);
    });

    it('should detect dynamic URL injection', () => {
      const code = `
        function Component({ userUrl }) {
          return <a href={userUrl}>Link</a>;
        }
      `;
      const result = analyzeSecurityFromSource(code);

      expect(result.vulnerabilities.some(v => v.id === 'dynamic-url-injection')).toBe(true);
      expect(hasCriticalVulnerabilities(result) || result.bySeverity.high > 0).toBe(true);
    });
  });

  describe('sensitive data exposure', () => {
    it('should detect hardcoded API keys', () => {
      const code = `
        const API_KEY = "sk-1234567890abcdef";
        function Component() {
          return <div>Test</div>;
        }
      `;
      const result = analyzeSecurityFromSource(code);

      expect(result.vulnerabilities.some(v => v.id === 'sensitive-data-exposure')).toBe(true);
      expect(result.bySeverity.critical).toBeGreaterThan(0);
    });

    it('should detect console logging of sensitive data', () => {
      const code = `
        function Component({ apiKey }) {
          console.log(apiKey);
          return <div>Test</div>;
        }
      `;
      const result = analyzeSecurityFromSource(code);

      expect(result.vulnerabilities.some(v => v.id === 'sensitive-data-logging')).toBe(true);
    });

    it('should detect sensitive data in localStorage', () => {
      const code = `
        function Component({ token }) {
          localStorage.setItem('auth_token', token);
          return <div>Test</div>;
        }
      `;
      const result = analyzeSecurityFromSource(code);

      expect(result.vulnerabilities.some(v => v.id === 'sensitive-data-storage')).toBe(true);
    });
  });

  describe('code injection', () => {
    it('should detect eval() usage', () => {
      const code = `
        function Component({ code }) {
          eval(code);
          return <div>Test</div>;
        }
      `;
      const result = analyzeSecurityFromSource(code);

      expect(result.vulnerabilities.some(v => v.id === 'eval-usage')).toBe(true);
      expect(result.bySeverity.critical).toBeGreaterThan(0);
    });

    it('should detect Function constructor', () => {
      const code = `
        function Component() {
          const fn = new Function('a', 'b', 'return a + b');
          return <div>Test</div>;
        }
      `;
      const result = analyzeSecurityFromSource(code);

      expect(result.vulnerabilities.some(v => v.id === 'eval-usage')).toBe(true);
    });
  });

  describe('metrics calculation', () => {
    it('should calculate correct risk level', () => {
      const criticalCode = `
        const secret = "my-secret-key-12345";
        function Component() {
          eval("alert('xss')");
          return <div>Test</div>;
        }
      `;
      const result = analyzeSecurityFromSource(criticalCode);

      expect(result.riskLevel).toBe('critical');
      expect(hasCriticalVulnerabilities(result)).toBe(true);
    });

    it('should calculate compliance score', () => {
      const code = `
        function Component({ userUrl }) {
          return <a href={userUrl}>Link</a>;
        }
      `;
      const result = analyzeSecurityFromSource(code);

      expect(result.complianceScore).toBeLessThan(100);
      expect(result.complianceScore).toBeGreaterThanOrEqual(0);
    });

    it('should group by vulnerability type', () => {
      const code = `
        const apiKey = "sk-1234567890abcdef";
        function Component({ html, url }) {
          console.log(apiKey);
          return (
            <div>
              <div dangerouslySetInnerHTML={{ __html: html }} />
              <a href={url}>Link</a>
            </div>
          );
        }
      `;
      const result = analyzeSecurityFromSource(code);

      expect(result.byType.xss).toBeGreaterThan(0);
      expect(result.byType['sensitive-data']).toBeGreaterThan(0);
    });
  });

  describe('utility functions', () => {
    it('should identify high priority vulnerabilities', () => {
      const code = `
        const secret = "my-secret-12345";
        eval("code");
      `;
      const result = analyzeSecurityFromSource(code);
      const highPriority = getHighPriorityVulnerabilities(result);

      expect(highPriority.length).toBeGreaterThan(0);
      expect(highPriority.every(v => v.severity === 'critical' || v.severity === 'high')).toBe(
        true
      );
    });
  });
});
```

### Task 5.9: Security Rules Tests

**Model:** Sonnet (test implementation)  
**File:** `src/rules/anti-patterns/security-rules.test.ts`  
**Estimated Time:** 1 hour

```typescript
/**
 * Security Rules Tests
 */

import { Project, SourceFile } from 'ts-morph';
import { describe, it, expect } from 'vitest';
import {
  dynamicUrlRule,
  sensitiveDataExposureRule,
  sensitiveDataLoggingRule,
  sensitiveDataStorageRule,
  evalUsageRule,
} from './security-rules';

function createSourceFile(code: string): SourceFile {
  const project = new Project({
    useInMemoryFileSystem: true,
    compilerOptions: { jsx: 4, target: 99, module: 99 },
  });
  return project.createSourceFile('test.tsx', code);
}

describe('Security Rules', () => {
  describe('dynamicUrlRule', () => {
    it('should detect dynamic href', () => {
      const code = `<a href={userUrl}>Link</a>`;
      const violations = dynamicUrlRule.check(createSourceFile(code));
      expect(violations.length).toBeGreaterThan(0);
    });

    it('should allow static URLs', () => {
      const code = `<a href="https://example.com">Link</a>`;
      const violations = dynamicUrlRule.check(createSourceFile(code));
      expect(violations).toHaveLength(0);
    });
  });

  describe('sensitiveDataExposureRule', () => {
    it('should detect hardcoded API key', () => {
      const code = `const apiKey = "sk-1234567890abcdef";`;
      const violations = sensitiveDataExposureRule.check(createSourceFile(code));
      expect(violations.length).toBeGreaterThan(0);
    });

    it('should detect hardcoded password', () => {
      const code = `const password = "mySecretPassword123";`;
      const violations = sensitiveDataExposureRule.check(createSourceFile(code));
      expect(violations.length).toBeGreaterThan(0);
    });
  });

  describe('sensitiveDataLoggingRule', () => {
    it('should detect logging of API key', () => {
      const code = `console.log(apiKey);`;
      const violations = sensitiveDataLoggingRule.check(createSourceFile(code));
      expect(violations.length).toBeGreaterThan(0);
    });

    it('should allow logging safe variables', () => {
      const code = `console.log(count);`;
      const violations = sensitiveDataLoggingRule.check(createSourceFile(code));
      expect(violations).toHaveLength(0);
    });
  });

  describe('sensitiveDataStorageRule', () => {
    it('should detect token in localStorage', () => {
      const code = `localStorage.setItem('auth_token', token);`;
      const violations = sensitiveDataStorageRule.check(createSourceFile(code));
      expect(violations.length).toBeGreaterThan(0);
    });

    it('should allow safe storage keys', () => {
      const code = `localStorage.setItem('theme', 'dark');`;
      const violations = sensitiveDataStorageRule.check(createSourceFile(code));
      expect(violations).toHaveLength(0);
    });
  });

  describe('evalUsageRule', () => {
    it('should detect eval() call', () => {
      const code = `eval(userCode);`;
      const violations = evalUsageRule.check(createSourceFile(code));
      expect(violations.length).toBeGreaterThan(0);
      expect(violations[0]!.severity).toBe('critical');
    });

    it('should detect Function constructor', () => {
      const code = `const fn = new Function('return 42');`;
      const violations = evalUsageRule.check(createSourceFile(code));
      expect(violations.length).toBeGreaterThan(0);
    });
  });
});
```

## Integration

### Existing Integration Points

The security analyzer integrates with existing systems:

1. **Anti-Pattern Registry** ✅ - Already built
2. **Base Analyzer** ✅ - Already built
3. **CLI** - Add to existing CLI commands using `--security` flag
4. **Formatters** - Follows existing formatter pattern

### CLI Integration

Add security analysis to the CLI (`src/cli.ts`):

```typescript
// Add a --security flag or separate security command
program
  .command('security <file>')
  .description('Run security analysis on a React file')
  .action(async (file: string) => {
    const code = await readFile(file, 'utf-8');
    const result = analyzeSecurityFromSource(code);
    formatSecurityReport(result);
  });
```

## Acceptance Criteria

### Detection Accuracy

- [x] Architecture aligns with existing codebase
- [ ] Detect dangerouslySetInnerHTML usage (100%) - Already exists
- [ ] Detect javascript: protocol risks in href (>90%)
- [ ] Detect hardcoded API keys/secrets (>85%)
- [ ] Detect sensitive data logging (>85%)
- [ ] Detect sensitive data in storage (>85%)
- [ ] Detect eval() usage (100%)
- [ ] Low false positive rate (`<15`%)

### Performance

- [ ] Analysis < 200ms for typical file
- [ ] No impact on overall analyzer performance
- [ ] Scales linearly with file size

### Code Quality

- [ ] All tests passing
- [ ] TypeScript strict mode compliance
- [ ] Consistent with existing analyzer patterns
- [ ] Comprehensive JSDoc comments
- [ ] No linter errors

## Testing Instructions

### Manual Testing

1. **Test XSS Detection**

   ```bash
   cat > /tmp/XSSTest.tsx << 'EOF'
   function Component({ userContent, userUrl }) {
     return (
       <div>
         <div dangerouslySetInnerHTML={{ __html: userContent }} />
         <a href={userUrl}>Link</a>
       </div>
     );
   }
   EOF

   # Run with the analyzer (once CLI is integrated)
   npm run analyze:security /tmp/XSSTest.tsx
   ```

2. **Test Sensitive Data Detection**

   ```bash
   cat > /tmp/SecretsTest.tsx << 'EOF'
   const API_KEY = "sk-1234567890abcdef";
   const SECRET = "my-secret-password-123";

   function Component() {
     console.log(API_KEY);
     localStorage.setItem("auth_token", token);
     eval(userCode);
     return <div>Test</div>;
   }
   EOF

   npm run analyze:security /tmp/SecretsTest.tsx
   # Expected: Detects all 5 issues
   ```

3. **Test Clean Code**

   ```bash
   cat > /tmp/CleanTest.tsx << 'EOF'
   function Component() {
     const [count, setCount] = useState(0);
     return (
       <button onClick={() => setCount(c => c + 1)}>
         {count}
       </button>
     );
   }
   EOF

   npm run analyze:security /tmp/CleanTest.tsx
   # Expected: Score 100, no vulnerabilities
   ```

### Automated Testing

```bash
# Run security analyzer tests
npm run test -- security-analyzer.test.ts

# Run security rules tests
npm run test -- security-rules.test.ts

# Run all tests
npm run test
```

## Estimated Effort

| Task                   | Model  | Estimated Time |
| ---------------------- | ------ | -------------- |
| 5.1 Security Types     | Opus   | 45 min         |
| 5.2 Security Rules     | Sonnet | 2.5 hours      |
| 5.3 Registry Update    | Haiku  | 10 min         |
| 5.4 Security Analyzer  | Opus   | 1.5 hours      |
| 5.5 Security Formatter | Sonnet | 45 min         |
| 5.6 Type Exports       | Haiku  | 5 min          |
| 5.7 Rules Index        | Haiku  | 5 min          |
| 5.8 Analyzer Tests     | Sonnet | 1.5 hours      |
| 5.9 Rules Tests        | Sonnet | 1 hour         |
| **Total**              |        | **8-9 hours**  |

## Implementation Order

1. **Day 1 Morning (2.5 hours)**
   - Task 5.1: Security Types (45 min)
   - Task 5.2: Security Rules (2.5 hours) - Start

2. **Day 1 Afternoon (3.5 hours)**
   - Task 5.2: Security Rules (continued)
   - Task 5.3: Registry Update (10 min)
   - Task 5.4: Security Analyzer (1.5 hours)
   - Task 5.5: Security Formatter (45 min)

3. **Day 2 Morning (3 hours)**
   - Task 5.6: Type Exports (5 min)
   - Task 5.7: Rules Index (5 min)
   - Task 5.8: Analyzer Tests (1.5 hours)
   - Task 5.9: Rules Tests (1 hour)

4. **Day 2 Afternoon (1 hour)**
   - Integration testing
   - Documentation updates
   - Manual testing

## Notes for Implementation

### Key Considerations

1. **Existing Rule**: The `dangerous-html` rule already exists in `jsx-rules.ts`. Don't duplicate it.
2. **Category Filtering**: Security rules use `category: 'security'` for filtering.
3. **Pattern Reuse**: Follow the pattern established by existing anti-pattern rules.
4. **ts-morph API**: All AST traversal uses ts-morph, not Babel.
5. **Test Pattern**: Follow the test pattern from `anti-pattern-analyzer.test.ts`.

### Future Enhancements

- Add more security rules (CSRF, clickjacking, etc.)
- Add auto-fix capability for simple issues
- Integrate with CI/CD pipelines
- Add security scoring trends over time
- Integration with security scanning tools

---

**Document Version:** 2.0  
**Created:** January 10, 2026  
**Updated:** January 10, 2026  
**Status:** Ready for Implementation - Aligned with Current Architecture
