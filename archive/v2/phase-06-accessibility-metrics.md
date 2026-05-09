# Phase 06: Accessibility Metrics

**Priority:** Medium - Inclusivity & compliance  
**Complexity:** Medium  
**Dependencies:** Phase 00 (Foundation) ✅, Phase 02 (Anti-Pattern Rules Engine) ✅  
**Status:** Ready for implementation

**Phase Duration:** 3-4 days

All code examples use the `ts-morph` API for AST parsing and traversal, following the established patterns from security and anti-pattern analyzers.

## Overview

This phase implements accessibility (a11y) analysis based on WCAG guidelines and jsx-a11y rules. It helps teams ensure inclusive user experiences and maintain accessibility compliance.

## Business Value

- Ensure WCAG compliance
- Improve user experience for all users
- Reduce legal/compliance risk
- Support automated accessibility auditing
- Integrate with existing accessibility workflows

## Agent Assignments

| Agent                  | Role                             | Capacity  |
| ---------------------- | -------------------------------- | --------- |
| react-engineer         | Lead implementer, a11y expertise | Primary   |
| typescript-engineer    | Type system, rule implementation | Secondary |
| vscode-plugin-engineer | Extension diagnostics preview    | Advisory  |

## Execution Strategy

### Milestone 6.1: ARIA Analysis (Day 1-2)

**Parallel Tasks:**

- Missing alt text detection (Sonnet)
- Missing labels detection (Sonnet)
- Invalid ARIA attributes (Sonnet)

### Milestone 6.2: Keyboard Navigation (Day 2-3)

**Parallel Tasks:**

- Missing tabIndex detection (Sonnet)
- Click without keyboard handler (Sonnet)
- Interactive element detection (Sonnet)

### Milestone 6.3: Semantic HTML & Integration (Day 3-4)

**Parallel Tasks:**

- Non-semantic element detection (Sonnet)
- Heading hierarchy analysis (Sonnet)
- Integration with rules engine (typescript-engineer)

## Detailed Tasks

### Task 6.1: Accessibility Types

**Model:** Opus (foundational types)
**File:** `src/types/accessibility.ts`

**Note:** Also update `src/types/index.ts` to export accessibility types:

```typescript
// Export accessibility types
export type {
  WCAGLevel,
  A11ySeverity,
  A11yViolation,
  A11yWarning,
  A11yBestPractice,
  AccessibilityMetrics,
} from './accessibility';

// Export accessibility utilities
export { toA11yViolation, initializeBySeverity } from './accessibility';
```

```typescript
import type { CodeLocation } from '../types';
import type { AntiPattern } from './anti-patterns';

/**
 * WCAG conformance level
 */
export type WCAGLevel = 'A' | 'AA' | 'AAA' | 'non-compliant';

/**
 * Accessibility violation severity
 * Maps to AntiPatternSeverity but provides a11y-specific context
 */
export type A11ySeverity = 'critical' | 'serious' | 'moderate' | 'minor';

/**
 * Accessibility violation
 * Extends AntiPattern with WCAG-specific metadata
 */
export interface A11yViolation extends Omit<AntiPattern, 'severity'> {
  /** WCAG criterion (e.g., "1.1.1") */
  wcagCriterion: string;
  /** Severity level (mapped from anti-pattern severity) */
  severity: A11ySeverity;
  /** Affected element description */
  element: string;
  /** WCAG reference URL */
  wcagReference: string;
}

/**
 * Accessibility warning (not a violation, but worth reviewing)
 */
export interface A11yWarning {
  id: string;
  name: string;
  element: string;
  location: CodeLocation;
  message: string;
  suggestion: string;
}

/**
 * Best practice recommendation
 */
export interface A11yBestPractice {
  id: string;
  name: string;
  description: string;
  implemented: boolean;
  location?: CodeLocation;
}

/**
 * Complete accessibility metrics
 */
export interface AccessibilityMetrics {
  /** Overall accessibility score (0-100) */
  score: number;
  /** WCAG conformance level */
  wcagLevel: WCAGLevel;
  /** Detected violations */
  violations: A11yViolation[];
  /** Warnings to review */
  warnings: A11yWarning[];
  /** Best practices status */
  bestPractices: A11yBestPractice[];
  /** Keyboard navigation score (0-100) */
  keyboardNavigationScore: number;
  /** Screen reader compatibility score (0-100) */
  screenReaderCompatibility: number;
  /** Violation counts by severity */
  bySeverity: Record<A11ySeverity, number>;
}

/**
 * Convert anti-pattern to accessibility violation
 */
export function toA11yViolation(
  antiPattern: AntiPattern,
  wcagCriterion: string,
  wcagReference: string
): A11yViolation {
  const severityMap: Record<string, A11ySeverity> = {
    critical: 'critical',
    high: 'serious',
    medium: 'moderate',
    warning: 'moderate',
    low: 'minor',
    info: 'minor',
  };

  return {
    ...antiPattern,
    severity: severityMap[antiPattern.severity] ?? 'moderate',
    wcagCriterion,
    wcagReference,
    element: antiPattern.description.split(' ')[0] ?? 'element',
  };
}

/**
 * Initialize severity counts record
 */
export function initializeBySeverity(): Record<A11ySeverity, number> {
  return {
    critical: 0,
    serious: 0,
    moderate: 0,
    minor: 0,
  };
}
```

### Task 6.2: Accessibility Rules

**Model:** Sonnet (pattern-based detection)
**File:** `src/rules/anti-patterns/a11y-rules.ts`

```typescript
import { SourceFile, Node } from 'ts-morph';
import type { AntiPatternRule, AntiPattern } from '../../types/anti-patterns';
import { createAntiPattern, getNodeLocation } from './utils';

/**
 * Image alt text detection
 * WCAG 1.1.1 - Non-text Content (Level A)
 */
const imgAltRule: AntiPatternRule = {
  id: 'img-alt',
  name: 'Image Alt Text',
  category: 'accessibility',
  defaultSeverity: 'critical',
  description: 'Images must have alt text for screen readers',
  references: ['https://www.w3.org/WAI/WCAG21/Understanding/non-text-content.html'],
  fixable: false,

  check(sourceFile: SourceFile): AntiPattern[] {
    const violations: AntiPattern[] = [];

    sourceFile.forEachDescendant(node => {
      if (!Node.isJsxSelfClosingElement(node) && !Node.isJsxElement(node)) {
        return;
      }

      const openingElement = Node.isJsxElement(node) ? node.getOpeningElement() : node;

      const tagName = openingElement.getTagNameNode().getText();
      if (tagName !== 'img') return;

      const attrs = openingElement.getAttributes();
      const altAttr = attrs.find(attr => {
        if (!Node.isJsxAttribute(attr)) return false;
        return attr.getName() === 'alt';
      });

      if (!altAttr) {
        violations.push(
          createAntiPattern({
            id: 'img-alt',
            name: 'Missing Alt Text',
            category: 'accessibility',
            severity: 'critical',
            node: openingElement,
            description: 'Image is missing alt attribute',
            impact: 'Screen reader users cannot understand image content',
            fix: 'Add alt attribute: <img alt="description" /> or alt="" for decorative images',
            autoFixable: false,
            references: ['https://www.w3.org/WAI/WCAG21/quickref/#non-text-content'],
          })
        );
      } else if (Node.isJsxAttribute(altAttr)) {
        const initializer = altAttr.getInitializer();
        if (!initializer) {
          violations.push(
            createAntiPattern({
              id: 'img-alt',
              name: 'Empty Alt Attribute',
              category: 'accessibility',
              severity: 'warning',
              node: openingElement,
              description: 'Image has alt attribute but no value',
              impact: 'Alt text may be missing or incorrectly empty',
              fix: 'Add descriptive alt text or use alt="" for decorative images',
              autoFixable: false,
              references: [],
            })
          );
        } else if (Node.isStringLiteral(initializer) && initializer.getText() === '""') {
          // Valid for decorative images - no violation
        }
      }
    });

    return violations;
  },
};

/**
 * Form input labels
 * WCAG 1.3.1 - Info and Relationships (Level A)
 */
const inputLabelRule: AntiPatternRule = {
  id: 'input-label',
  name: 'Form Input Labels',
  category: 'accessibility',
  defaultSeverity: 'critical',
  description: 'Form inputs must have associated labels',
  references: ['https://www.w3.org/WAI/WCAG21/Understanding/info-and-relationships.html'],
  fixable: false,

  check(sourceFile: SourceFile): AntiPattern[] {
    const violations: AntiPattern[] = [];

    sourceFile.forEachDescendant(node => {
      if (!Node.isJsxSelfClosingElement(node) && !Node.isJsxElement(node)) {
        return;
      }

      const openingElement = Node.isJsxElement(node) ? node.getOpeningElement() : node;

      const tagName = openingElement.getTagNameNode().getText();
      const inputElements = ['input', 'textarea', 'select'];
      if (!inputElements.includes(tagName)) return;

      const attrs = openingElement.getAttributes();

      // Check for aria-label, aria-labelledby, or id (for associated label)
      const hasAriaLabel = attrs.some(attr => {
        if (!Node.isJsxAttribute(attr)) return false;
        const name = attr.getName();
        return name === 'aria-label' || name === 'aria-labelledby';
      });

      const hasId = attrs.some(attr => {
        if (!Node.isJsxAttribute(attr)) return false;
        return attr.getName() === 'id';
      });

      // Check input type - some don't need labels
      const typeAttr = attrs.find(attr => {
        if (!Node.isJsxAttribute(attr)) return false;
        return attr.getName() === 'type';
      });

      let inputType = 'text';
      if (typeAttr && Node.isJsxAttribute(typeAttr)) {
        const initializer = typeAttr.getInitializer();
        if (Node.isStringLiteral(initializer)) {
          inputType = initializer.getLiteralValue();
        }
      }

      // Hidden and submit inputs don't need labels
      if (['hidden', 'submit', 'reset', 'button', 'image'].includes(inputType)) {
        return;
      }

      if (!hasAriaLabel && !hasId) {
        violations.push(
          createAntiPattern({
            id: 'input-label',
            name: 'Missing Input Label',
            category: 'accessibility',
            severity: 'critical',
            node: openingElement,
            description: `${tagName} element lacks accessible label`,
            impact: 'Screen reader users cannot identify the input purpose',
            fix: 'Add aria-label, aria-labelledby, or use <label> element with matching id',
            autoFixable: false,
            references: ['https://www.w3.org/WAI/tutorials/forms/labels/'],
          })
        );
      }
    });

    return violations;
  },
};

/**
 * Interactive elements keyboard accessibility
 * WCAG 2.1.1 - Keyboard (Level A)
 */
const keyboardAccessRule: AntiPatternRule = {
  id: 'keyboard-access',
  name: 'Keyboard Accessibility',
  category: 'accessibility',
  defaultSeverity: 'critical',
  description: 'Interactive elements must be keyboard accessible',
  references: ['https://www.w3.org/WAI/WCAG21/Understanding/keyboard.html'],
  fixable: true,

  check(sourceFile: SourceFile): AntiPattern[] {
    const violations: AntiPattern[] = [];

    sourceFile.forEachDescendant(node => {
      if (!Node.isJsxSelfClosingElement(node) && !Node.isJsxElement(node)) {
        return;
      }

      const openingElement = Node.isJsxElement(node) ? node.getOpeningElement() : node;

      const tagName = openingElement.getTagNameNode().getText();
      // Check for non-interactive elements with onClick
      const nonInteractive = ['div', 'span', 'p', 'section', 'article', 'aside'];
      if (!nonInteractive.includes(tagName)) return;

      const attrs = openingElement.getAttributes();

      const hasOnClick = attrs.some(attr => {
        if (!Node.isJsxAttribute(attr)) return false;
        return attr.getName() === 'onClick';
      });

      if (!hasOnClick) return;

      const hasOnKeyDown = attrs.some(attr => {
        if (!Node.isJsxAttribute(attr)) return false;
        const name = attr.getName();
        return name === 'onKeyDown' || name === 'onKeyUp' || name === 'onKeyPress';
      });

      const hasRole = attrs.some(attr => {
        if (!Node.isJsxAttribute(attr)) return false;
        return attr.getName() === 'role';
      });

      const hasTabIndex = attrs.some(attr => {
        if (!Node.isJsxAttribute(attr)) return false;
        return attr.getName() === 'tabIndex';
      });

      if (!hasOnKeyDown) {
        violations.push(
          createAntiPattern({
            id: 'keyboard-access',
            name: 'Click Without Keyboard Handler',
            category: 'accessibility',
            severity: 'critical',
            node: openingElement,
            description: `<${tagName}> has onClick but no keyboard handler`,
            impact: 'Keyboard-only users cannot activate this element',
            fix: 'Add onKeyDown handler that triggers on Enter/Space, or use <button> instead',
            autoFixable: true,
            references: [],
          })
        );
      }

      if (!hasRole) {
        violations.push(
          createAntiPattern({
            id: 'keyboard-access',
            name: 'Missing Role Attribute',
            category: 'accessibility',
            severity: 'high',
            node: openingElement,
            description: `Interactive <${tagName}> is missing role attribute`,
            impact: 'Screen readers may not announce element as interactive',
            fix: 'Add role="button" or use semantic <button> element',
            autoFixable: true,
            references: [],
          })
        );
      }

      if (!hasTabIndex) {
        violations.push(
          createAntiPattern({
            id: 'keyboard-access',
            name: 'Missing Tab Index',
            category: 'accessibility',
            severity: 'high',
            node: openingElement,
            description: `Interactive <${tagName}> is not focusable`,
            impact: 'Keyboard users cannot tab to this element',
            fix: 'Add tabIndex={0} or use semantic <button> element',
            autoFixable: true,
            references: [],
          })
        );
      }
    });

    return violations;
  },
};

/**
 * Button text content
 * WCAG 2.4.4 - Link Purpose (Level A)
 */
const buttonTextRule: AntiPatternRule = {
  id: 'button-text',
  name: 'Button Accessible Name',
  category: 'accessibility',
  defaultSeverity: 'critical',
  description: 'Buttons must have discernible text',
  references: ['https://www.w3.org/WAI/WCAG21/Understanding/link-purpose-in-context.html'],
  fixable: false,

  check(sourceFile: SourceFile): AntiPattern[] {
    const violations: AntiPattern[] = [];

    sourceFile.forEachDescendant(node => {
      if (!Node.isJsxElement(node)) return;

      const openingElement = node.getOpeningElement();
      const tagName = openingElement.getTagNameNode().getText();
      if (tagName !== 'button') return;

      const attrs = openingElement.getAttributes();

      // Check for aria-label
      const hasAriaLabel = attrs.some(attr => {
        if (!Node.isJsxAttribute(attr)) return false;
        const name = attr.getName();
        return name === 'aria-label' || name === 'aria-labelledby' || name === 'title';
      });

      if (hasAriaLabel) return;

      // Check for text content
      const children = node.getChildren();
      let hasTextContent = false;
      let onlyHasIcon = false;

      for (const child of children) {
        if (Node.isJsxText(child)) {
          if (child.getText().trim().length > 0) {
            hasTextContent = true;
            break;
          }
        } else if (Node.isJsxExpressionContainer(child)) {
          const expr = child.getExpression();
          if (Node.isStringLiteral(expr) && expr.getLiteralValue().length > 0) {
            hasTextContent = true;
            break;
          }
        } else if (Node.isJsxElement(child) || Node.isJsxSelfClosingElement(child)) {
          const childTagName = Node.isJsxElement(child)
            ? child.getOpeningElement().getTagNameNode().getText()
            : child.getTagNameNode().getText();
          if (/icon|svg|img/i.test(childTagName)) {
            onlyHasIcon = true;
          } else {
            hasTextContent = true;
            break;
          }
        }
      }

      if (!hasTextContent && onlyHasIcon && children.length === 1) {
        violations.push(
          createAntiPattern({
            id: 'button-text',
            name: 'Icon-only Button',
            category: 'accessibility',
            severity: 'critical',
            node: openingElement,
            description: 'Button contains only icon without accessible name',
            impact: 'Screen reader users cannot determine button purpose',
            fix: 'Add aria-label to describe button action',
            autoFixable: false,
            references: [],
          })
        );
      } else if (!hasTextContent && children.length === 0) {
        violations.push(
          createAntiPattern({
            id: 'button-text',
            name: 'Empty Button',
            category: 'accessibility',
            severity: 'critical',
            node: openingElement,
            description: 'Button has no accessible text content',
            impact: 'Screen reader users cannot determine button purpose',
            fix: 'Add text content or aria-label',
            autoFixable: false,
            references: [],
          })
        );
      }
    });

    return violations;
  },
};

/**
 * Heading hierarchy
 * WCAG 1.3.1 - Info and Relationships (Level A)
 */
const headingHierarchyRule: AntiPatternRule = {
  id: 'heading-order',
  name: 'Heading Hierarchy',
  category: 'accessibility',
  defaultSeverity: 'medium',
  description: 'Headings should follow a logical hierarchy',
  references: ['https://www.w3.org/WAI/tutorials/page-structure/headings/'],
  fixable: true,

  check(sourceFile: SourceFile): AntiPattern[] {
    const violations: AntiPattern[] = [];
    const headings: Array<{ level: number; node: Node }> = [];

    sourceFile.forEachDescendant(node => {
      if (!Node.isJsxElement(node) && !Node.isJsxSelfClosingElement(node)) {
        return;
      }

      const openingElement = Node.isJsxElement(node) ? node.getOpeningElement() : node;

      const tagName = openingElement.getTagNameNode().getText();
      const match = tagName.match(/^h([1-6])$/);
      if (!match) return;

      headings.push({
        level: parseInt(match[1], 10),
        node: openingElement,
      });
    });

    // Check for skipped levels
    for (let i = 1; i < headings.length; i++) {
      const prev = headings[i - 1];
      const curr = headings[i];

      if (curr.level > prev.level + 1) {
        violations.push(
          createAntiPattern({
            id: 'heading-order',
            name: 'Skipped Heading Level',
            category: 'accessibility',
            severity: 'medium',
            node: curr.node,
            description: `Heading level skipped from h${prev.level} to h${curr.level}`,
            impact: 'Screen reader users may have difficulty understanding page structure',
            fix: `Use h${prev.level + 1} instead of h${curr.level}`,
            autoFixable: true,
            references: [],
          })
        );
      }
    }

    return violations;
  },
};

// Export all a11y rules
export const A11Y_RULES: AntiPatternRule[] = [
  imgAltRule,
  inputLabelRule,
  keyboardAccessRule,
  buttonTextRule,
  headingHierarchyRule,
];
```

### Task 6.3: Accessibility Analyzer

**Model:** Opus (integration)
**File:** `src/analyzers/accessibility-analyzer.ts`

```typescript
import { Project, SourceFile } from 'ts-morph';
import {
  BaseAnalyzer,
  type BaseAnalyzerResult,
  type AnalyzerConfig,
} from './base-analyzer-tsmorph';
import { AntiPatternRegistry, createAntiPatternRegistry } from '../rules/anti-pattern-registry';
import { A11Y_RULES } from '../rules/anti-patterns/a11y-rules';
import type { AntiPattern, AntiPatternCategory } from '../types/anti-patterns';
import type {
  AccessibilityMetrics,
  A11yViolation,
  A11yWarning,
  A11yBestPractice,
  A11ySeverity,
  WCAGLevel,
} from '../types/accessibility';
import { toA11yViolation, initializeBySeverity } from '../types/accessibility';

/**
 * Accessibility analyzer configuration
 */
export interface AccessibilityAnalyzerConfig extends AnalyzerConfig {
  /** Custom anti-pattern registry (if not provided, creates a11y-focused one) */
  registry?: AntiPatternRegistry;
  /** Minimum severity to report */
  minSeverity?: A11ySeverity;
}

/**
 * Accessibility analyzer result
 */
export interface AccessibilityAnalyzerResult extends AccessibilityMetrics, BaseAnalyzerResult {}

/**
 * Accessibility Analyzer
 *
 * Focused analyzer for detecting accessibility violations in React code.
 * Uses the anti-pattern registry internally with accessibility-specific configuration.
 */
export class AccessibilityAnalyzer extends BaseAnalyzer<AccessibilityAnalyzerResult> {
  private registry: AntiPatternRegistry;
  private a11yConfig: AccessibilityAnalyzerConfig;

  constructor(ast: SourceFile, config: AccessibilityAnalyzerConfig = {}) {
    super(ast, config);
    this.a11yConfig = config;

    // Create registry focused on accessibility rules
    this.registry = config.registry ?? this.createA11yFocusedRegistry();
  }

  /**
   * Create a registry configured for accessibility analysis
   */
  private createA11yFocusedRegistry(): AntiPatternRegistry {
    // Categories to ignore for focused accessibility analysis
    const nonA11yCategories: AntiPatternCategory[] = [
      'hooks',
      'performance',
      'state-management',
      'lifecycle',
      'jsx',
      'props',
      'security',
      'testing',
    ];

    return createAntiPatternRegistry({
      // Include all accessibility category rules
      ignoreCategories: nonA11yCategories,
      // Catch all accessibility issues by default
      minSeverity: this.a11yConfig.minSeverity
        ? this.mapA11ySeverityToAntiPattern(this.a11yConfig.minSeverity)
        : 'low',
    });
  }

  protected analyzeAST(): Omit<AccessibilityAnalyzerResult, 'metadata'> {
    // Run analysis using the registry
    const antiPatterns = this.registry.analyze(this.ast);

    // Filter to accessibility-related patterns only
    const a11yPatterns = antiPatterns.filter(ap => ap.category === 'accessibility');

    // Convert to accessibility violations
    const violations = this.convertToViolations(a11yPatterns);
    const warnings = this.extractWarnings(a11yPatterns);

    const bySeverity = this.groupBySeverity(violations);
    const wcagLevel = this.determineWCAGLevel(violations);
    const keyboardNavigationScore = this.calculateKeyboardScore(violations);
    const screenReaderCompatibility = this.calculateScreenReaderScore(violations);
    const bestPractices = this.assessBestPractices();

    const score = this.calculateOverallScore(violations);

    return {
      score,
      wcagLevel,
      violations,
      warnings,
      bestPractices,
      keyboardNavigationScore,
      screenReaderCompatibility,
      bySeverity,
    };
  }

  /**
   * Convert anti-patterns to accessibility violations
   */
  private convertToViolations(antiPatterns: AntiPattern[]): A11yViolation[] {
    return antiPatterns
      .filter(ap => ap.severity !== 'info')
      .map(ap => {
        const wcagCriterion = this.getWCAGCriterion(ap.id);
        const wcagReference = ap.references[0] ?? '';
        return toA11yViolation(ap, wcagCriterion, wcagReference);
      });
  }

  /**
   * Extract warnings from info-level anti-patterns
   */
  private extractWarnings(antiPatterns: AntiPattern[]): A11yWarning[] {
    return antiPatterns
      .filter(ap => ap.severity === 'info')
      .map(ap => ({
        id: ap.id,
        name: ap.name,
        element: ap.description.split(' ')[0] ?? '',
        location: ap.location,
        message: ap.description,
        suggestion: ap.fix,
      }));
  }

  /**
   * Get WCAG criterion for a rule ID
   */
  private getWCAGCriterion(ruleId: string): string {
    const mapping: Record<string, string> = {
      'img-alt': '1.1.1',
      'input-label': '1.3.1',
      'keyboard-access': '2.1.1',
      'button-text': '2.4.4',
      'heading-order': '1.3.1',
    };
    return mapping[ruleId] ?? 'unknown';
  }

  /**
   * Map anti-pattern severity to accessibility severity
   */
  private mapA11ySeverityToAntiPattern(severity: A11ySeverity): AntiPattern['severity'] {
    const mapping: Record<A11ySeverity, AntiPattern['severity']> = {
      critical: 'critical',
      serious: 'high',
      moderate: 'medium',
      minor: 'low',
    };
    return mapping[severity] ?? 'medium';
  }

  /**
   * Get the internal registry for advanced configuration
   */
  getRegistry(): AntiPatternRegistry {
    return this.registry;
  }

  /**
   * Enable a specific accessibility rule
   */
  enableRule(ruleId: string): void {
    this.registry.enableRule(ruleId);
  }

  /**
   * Disable a specific accessibility rule
   */
  disableRule(ruleId: string): void {
    this.registry.disableRule(ruleId);
  }

  /**
   * Group violations by severity
   */
  private groupBySeverity(violations: A11yViolation[]): Record<A11ySeverity, number> {
    const result = initializeBySeverity();

    violations.forEach(v => {
      result[v.severity]++;
    });

    return result;
  }

  /**
   * Determine WCAG conformance level
   */
  private determineWCAGLevel(violations: A11yViolation[]): WCAGLevel {
    const criticalViolations = violations.filter(v => v.severity === 'critical').length;

    if (criticalViolations > 0) return 'non-compliant';
    if (violations.length > 5) return 'A';
    if (violations.length > 0) return 'AA';
    return 'AAA';
  }

  /**
   * Calculate keyboard navigation score
   */
  private calculateKeyboardScore(violations: A11yViolation[]): number {
    const keyboardViolations = violations.filter(v => v.id === 'keyboard-access').length;
    return Math.max(0, 100 - keyboardViolations * 20);
  }

  /**
   * Calculate screen reader compatibility score
   */
  private calculateScreenReaderScore(violations: A11yViolation[]): number {
    const srViolations = violations.filter(v =>
      ['img-alt', 'input-label', 'button-text'].includes(v.id)
    ).length;
    return Math.max(0, 100 - srViolations * 15);
  }

  /**
   * Assess best practices (placeholder for future enhancement)
   */
  private assessBestPractices(): A11yBestPractice[] {
    return [
      {
        id: 'landmark-regions',
        name: 'Landmark Regions',
        description: 'Use semantic HTML5 landmarks (main, nav, aside)',
        implemented: false,
      },
      {
        id: 'skip-link',
        name: 'Skip Navigation Link',
        description: 'Provide skip link for keyboard users',
        implemented: false,
      },
    ];
  }

  /**
   * Calculate overall accessibility score
   */
  private calculateOverallScore(violations: A11yViolation[]): number {
    let score = 100;

    violations.forEach(v => {
      switch (v.severity) {
        case 'critical':
          score -= 20;
          break;
        case 'serious':
          score -= 10;
          break;
        case 'moderate':
          score -= 5;
          break;
        case 'minor':
          score -= 2;
          break;
      }
    });

    return Math.max(0, score);
  }
}

/**
 * Analyze accessibility from source code string
 */
export function analyzeAccessibilityFromSource(
  code: string,
  config?: AccessibilityAnalyzerConfig
): AccessibilityAnalyzerResult {
  const project = new Project({
    useInMemoryFileSystem: true,
    compilerOptions: {
      jsx: 4, // React JSX
      target: 99, // ESNext
      module: 99, // ESNext
    },
  });

  const sourceFile = project.createSourceFile('accessibility-analysis.tsx', code);
  const analyzer = new AccessibilityAnalyzer(sourceFile, config);
  return analyzer.analyze();
}
```

### Task 6.4: Register Rules in Registry

**Model:** Sonnet (integration)
**File:** `src/rules/anti-pattern-registry.ts`

Update the registry to import and register accessibility rules:

```typescript
// Add to imports
import { A11Y_RULES } from './anti-patterns/a11y-rules';

// Update registerBuiltinRules method
private registerBuiltinRules(): void {
  const allRules = [
    ...HOOKS_RULES,
    ...PERFORMANCE_RULES,
    ...STATE_RULES,
    ...JSX_RULES,
    ...SECURITY_RULES,
    ...A11Y_RULES, // Add this line
  ];

  allRules.forEach(rule => this.register(rule));
}
```

### Task 6.5: Accessibility Formatter

**Model:** Sonnet (formatting)
**File:** `src/formatters/accessibility-formatter.ts`

Create a formatter following the pattern from `security-formatter.ts`:

```typescript
import { createConsola } from 'consola';
import type { ConsolaInstance } from 'consola';
import { colors } from 'consola/utils';
import type { A11yViolation, A11ySeverity, WCAGLevel } from '../types/accessibility';
import type { AccessibilityAnalyzerResult } from '../analyzers/accessibility-analyzer';

// ============================================================================
// Formatter Options
// ============================================================================

export interface AccessibilityFormatterOptions {
  /** Disable colors in output */
  noColor?: boolean;
  /** Show verbose output including all violation details */
  verbose?: boolean;
  /** Show compact single-line output */
  compact?: boolean;
  /** Show JSON output instead of formatted text */
  json?: boolean;
  /** Maximum number of violations to display (default: all) */
  maxViolations?: number;
}

// ============================================================================
// Accessibility Formatter Class
// ============================================================================

/**
 * Accessibility Formatter
 *
 * Provides formatted console output for accessibility analysis results.
 * Uses the same styling patterns as SecurityFormatter for consistency.
 */
export class AccessibilityFormatter {
  private options: AccessibilityFormatterOptions;
  private logger: ConsolaInstance;

  constructor(options: AccessibilityFormatterOptions = {}) {
    this.options = {
      noColor: options.noColor ?? false,
      verbose: options.verbose ?? false,
      compact: options.compact ?? false,
      json: options.json ?? false,
      maxViolations: options.maxViolations,
    };

    this.logger = createConsola({
      level: this.options.verbose ? 4 : 3,
      fancy: !this.options.noColor,
      formatOptions: {
        colors: !this.options.noColor,
        compact: this.options.compact,
        date: false,
      },
    });
  }

  /**
   * Format and output accessibility analysis results
   */
  format(result: AccessibilityAnalyzerResult, filename?: string): void {
    if (this.options.json) {
      this.formatJSON(result);
      return;
    }

    if (this.options.compact) {
      this.formatCompact(result, filename);
      return;
    }

    this.formatFull(result, filename);
  }

  /**
   * Compact single-line output
   */
  private formatCompact(result: AccessibilityAnalyzerResult, filename?: string): void {
    const wcagIcon = this.getWCAGIcon(result.wcagLevel);
    const wcagText = this.colorByWCAGLevel(result.wcagLevel.toUpperCase(), result.wcagLevel);
    const scoreText =
      result.score >= 80
        ? this.green(`${result.score}/100`)
        : result.score >= 50
          ? this.yellow(`${result.score}/100`)
          : this.red(`${result.score}/100`);

    const file = filename ? `${filename}: ` : '';
    const violations = result.violations.length;
    const violationsText =
      violations > 0 ? this.red(` ${violations} violations`) : this.green(' No violations');

    this.logger.log(`${file}${wcagIcon} ${wcagText} | Score: ${scoreText}${violationsText}`);
  }

  /**
   * Full formatted output
   */
  private formatFull(result: AccessibilityAnalyzerResult, filename?: string): void {
    // Header
    this.logger.log('');
    this.logger.box({
      title: '♿ Accessibility Analysis',
      message: `WCAG Level: ${this.colorByWCAGLevel(result.wcagLevel.toUpperCase(), result.wcagLevel)} | File: ${filename ?? 'Unknown'}`,
    });

    // Summary
    this.logger.log('');
    this.logger.log('Summary');
    const scoreColor = result.score >= 80 ? 'green' : result.score >= 50 ? 'yellow' : 'red';
    this.logger.log(
      `  Accessibility Score:     ${this.colorByScore(result.score.toFixed(0), scoreColor)}/100`
    );
    this.logger.log(
      `  WCAG Compliance:         ${this.colorByWCAGLevel(result.wcagLevel, result.wcagLevel)}`
    );
    this.logger.log(`  Violations:              ${result.violations.length}`);
    this.logger.log(`  Keyboard Navigation:     ${result.keyboardNavigationScore}/100`);
    this.logger.log(`  Screen Reader Compat:    ${result.screenReaderCompatibility}/100`);

    // Violations by severity
    if (result.violations.length > 0) {
      this.logger.log('');
      this.logger.log('Violations by Severity');
      this.logger.log(`  ├─ Critical: ${result.bySeverity.critical}`);
      this.logger.log(`  ├─ Serious:  ${result.bySeverity.serious}`);
      this.logger.log(`  ├─ Moderate: ${result.bySeverity.moderate}`);
      this.logger.log(`  └─ Minor:    ${result.bySeverity.minor}`);
    }

    // Detected violations
    if (result.violations.length > 0) {
      this.logger.log('');
      this.logger.log('Detected Violations');

      const violationsToShow = this.options.maxViolations
        ? result.violations.slice(0, this.options.maxViolations)
        : result.violations;

      violationsToShow.forEach((violation, index) => {
        const isLast = index === violationsToShow.length - 1;
        const prefix = isLast ? '  └─' : '  ├─';
        const severityIcon = this.getSeverityIcon(violation.severity);

        this.logger.log(`${prefix} ${severityIcon} ${violation.name}`);
        this.logger.log(
          `     Location: Line ${violation.location.line}, Column ${violation.location.column}`
        );
        this.logger.log(`     WCAG: ${violation.wcagCriterion} | Element: ${violation.element}`);
        this.logger.log(`     Impact: ${violation.impact}`);
        this.logger.log(`     → ${violation.fix}`);

        if (this.options.verbose && violation.references.length > 0) {
          this.logger.log('     References:');
          violation.references.forEach(ref => {
            this.logger.log(`       - ${ref}`);
          });
        }
      });
    }

    // Warnings
    if (result.warnings.length > 0) {
      this.logger.log('');
      this.logger.log('Warnings');
      result.warnings.forEach((warning, index) => {
        const isLast = index === result.warnings.length - 1;
        const prefix = isLast ? '  └─' : '  ├─';
        this.logger.log(`${prefix} ⚠ ${warning.name}`);
        this.logger.log(`     ${warning.message}`);
        this.logger.log(`     → ${warning.suggestion}`);
      });
    }

    // Best practices
    if (result.bestPractices.length > 0) {
      this.logger.log('');
      this.logger.log('Best Practices');
      result.bestPractices.forEach((practice, index) => {
        const isLast = index === result.bestPractices.length - 1;
        const prefix = isLast ? '  └─' : '  ├─';
        const status = practice.implemented ? this.green('✓') : this.yellow('○');
        this.logger.log(`${prefix} ${status} ${practice.name}`);
        if (!practice.implemented) {
          this.logger.log(`     ${practice.description}`);
        }
      });
    }

    this.logger.log('');
  }

  /**
   * JSON output
   */
  private formatJSON(result: AccessibilityAnalyzerResult): void {
    console.log(JSON.stringify(result, null, 2));
  }

  // Helper methods for colors and icons
  private green(text: string): string {
    return this.options.noColor ? text : colors.green(text);
  }

  private yellow(text: string): string {
    return this.options.noColor ? text : colors.yellow(text);
  }

  private red(text: string): string {
    return this.options.noColor ? text : colors.red(text);
  }

  private colorByScore(text: string, color: string): string {
    if (this.options.noColor) return text;
    return color === 'green'
      ? colors.green(text)
      : color === 'yellow'
        ? colors.yellow(text)
        : colors.red(text);
  }

  private colorByWCAGLevel(text: string, level: WCAGLevel): string {
    if (this.options.noColor) return text;
    switch (level) {
      case 'AAA':
        return colors.green(text);
      case 'AA':
        return colors.yellow(text);
      case 'A':
        return colors.yellow(text);
      case 'non-compliant':
        return colors.red(text);
      default:
        return text;
    }
  }

  private getWCAGIcon(level: WCAGLevel): string {
    switch (level) {
      case 'AAA':
        return '♿';
      case 'AA':
        return '⚠';
      case 'A':
        return '⚠';
      case 'non-compliant':
        return '✗';
      default:
        return '?';
    }
  }

  private getSeverityIcon(severity: A11ySeverity): string {
    switch (severity) {
      case 'critical':
        return '●';
      case 'serious':
        return '○';
      case 'moderate':
        return '◐';
      case 'minor':
        return '◯';
      default:
        return '•';
    }
  }
}

// ============================================================================
// Convenience Functions
// ============================================================================

/**
 * Format and print accessibility report to console
 *
 * @param result - Accessibility analysis result
 * @param filename - Optional filename being analyzed
 * @param options - Formatter options
 */
export function formatAccessibilityReport(
  result: AccessibilityAnalyzerResult,
  filename?: string,
  options?: AccessibilityFormatterOptions
): void {
  const formatter = new AccessibilityFormatter(options);
  formatter.format(result, filename);
}

/**
 * Get accessibility report as JSON string
 *
 * @param result - Accessibility analysis result
 * @returns JSON string representation
 */
export function formatAccessibilityJSON(result: AccessibilityAnalyzerResult): string {
  return JSON.stringify(result, null, 2);
}
```

### Task 6.6: CLI Integration

**Model:** Opus (integration)
**File:** `src/cli.ts`

Add accessibility analysis to CLI:

1. **Add import:**

```typescript
import {
  analyzeAccessibilityFromSource,
  type AccessibilityAnalyzerResult,
} from './analyzers/accessibility-analyzer';
import { formatAccessibilityReport } from './formatters/accessibility-formatter';
```

2. **Add CLI flag:**

```typescript
const enableAccessibility = args.includes('--accessibility');
```

3. **Add to help text:**

```typescript
  --accessibility      Analyze accessibility violations (WCAG compliance)
```

**Add detailed help section:**

```typescript
${colorize('Phase 06 - Accessibility Analysis (--accessibility):', 'cyan')}
  Detects accessibility violations and WCAG compliance issues:
  - Missing alt text on images (WCAG 1.1.1)
  - Missing form input labels (WCAG 1.3.1)
  - Keyboard accessibility issues (WCAG 2.1.1)
  - Empty or icon-only buttons without labels (WCAG 2.4.4)
  - Heading hierarchy violations (WCAG 1.3.1)
  - Overall accessibility score (0-100, higher = more accessible)
  - WCAG conformance level (A, AA, AAA, non-compliant)
  - Keyboard navigation score and screen reader compatibility
  - Severity-based violation counts (critical, serious, moderate, minor)
```

4. **Add analysis call:**

```typescript
if (enableAccessibility) {
  entry.accessibility = analyzeAccessibilityFromSource(source);
}
```

5. **Add formatter call:**

```typescript
if (enableAccessibility && accessibility) {
  formatAccessibilityReport(accessibility, file, {
    noColor,
    verbose,
    compact,
  });
}
```

6. **Add JSON output handling:**

```typescript
if (enableAccessibility && results.length > 0 && results[0].accessibility) {
  // Accessibility JSON output
  if (results.length === 1) {
    const output = {
      file: results[0].file,
      analyzedAt: new Date().toISOString(),
      complexity: results[0].result,
      accessibility: results[0].accessibility,
    };
    console.log(JSON.stringify(output, null, pretty ? 2 : 0));
    return;
  }
  // ... multi-file JSON output
}
```

7. **Add summary section:**

```typescript
if (enableAccessibility && results.some(r => r.accessibility)) {
  const avgScore =
    results.reduce((sum, r) => sum + (r.accessibility?.score || 100), 0) / results.length;
  const totalViolations = results.reduce(
    (sum, r) => sum + (r.accessibility?.violations.length || 0),
    0
  );
  const totalCritical = results.reduce(
    (sum, r) => sum + (r.accessibility?.bySeverity.critical || 0),
    0
  );
  const totalSerious = results.reduce(
    (sum, r) => sum + (r.accessibility?.bySeverity.serious || 0),
    0
  );
  const avgKeyboardScore =
    results.reduce((sum, r) => sum + (r.accessibility?.keyboardNavigationScore || 100), 0) /
    results.length;
  const avgScreenReaderScore =
    results.reduce((sum, r) => sum + (r.accessibility?.screenReaderCompatibility || 100), 0) /
    results.length;
  const criticalFiles = results
    .filter(r => (r.accessibility?.bySeverity.critical || 0) > 0)
    .map(r => r.file);

  console.log('');
  console.log(colorize('ACCESSIBILITY SUMMARY', 'bright'));
  const scoreColor = avgScore >= 80 ? 'green' : avgScore >= 50 ? 'yellow' : 'red';
  console.log(`Average accessibility score: ${colorize(avgScore.toFixed(0), scoreColor)}/100`);
  console.log(
    `Total violations: ${totalViolations} (${colorize(`${totalCritical} critical`, 'red')}, ${colorize(`${totalSerious} serious`, 'red')})`
  );
  console.log(`Keyboard navigation: ${colorize(avgKeyboardScore.toFixed(0), scoreColor)}/100`);
  console.log(
    `Screen reader compatibility: ${colorize(avgScreenReaderScore.toFixed(0), scoreColor)}/100`
  );
  if (criticalFiles.length > 0) {
    console.log(
      `${colorize('⚠ Files with critical violations:', 'red')} ${criticalFiles.join(', ')}`
    );
  }
}
```

## Acceptance Criteria

### Detection Requirements

- [ ] Detect missing alt text on images
- [ ] Detect missing form input labels
- [ ] Detect click handlers without keyboard support
- [ ] Detect empty/icon-only buttons without labels
- [ ] Detect heading hierarchy issues

### WCAG Mapping

- [ ] Each rule maps to WCAG success criterion
- [ ] Severity correlates with WCAG impact

### Integration Requirements

- [ ] Rules registered in AntiPatternRegistry
- [ ] Analyzer extends BaseAnalyzer correctly
- [ ] Formatter displays results clearly
- [ ] CLI flag `--accessibility` works
- [ ] JSON output includes accessibility metrics

## Testing Instructions

### Manual Testing

1. **Test Alt Text Detection**

   ```bash
   cat > /tmp/A11yTest.tsx << 'EOF'
   function Component() {
     return (
       <div>
         <img src="photo.jpg" />
         <button><Icon /></button>
         <div onClick={() => {}}>Click me</div>
       </div>
     );
   }
   EOF

   npm run analyze -- /tmp/A11yTest.tsx --accessibility
   # Expected: Detects missing alt, icon-only button, non-semantic click handler
   ```

2. **Test with JSON output:**
   ```bash
   npm run analyze -- src/ --accessibility --json
   ```

## Estimated Effort

| Task                        | Model  | Estimated Time  |
| --------------------------- | ------ | --------------- |
| 6.1 Accessibility Types     | Opus   | 1 hour          |
| 6.2 A11y Rules (5 rules)    | Sonnet | 3 hours         |
| 6.3 Accessibility Analyzer  | Opus   | 2 hours         |
| 6.4 Register Rules          | Sonnet | 0.5 hours       |
| 6.5 Accessibility Formatter | Sonnet | 2 hours         |
| 6.6 CLI Integration         | Opus   | 1 hour          |
| Unit Tests                  | Sonnet | 2 hours         |
| Integration Tests           | Sonnet | 1 hour          |
| Documentation               | Haiku  | 0.5 hours       |
| **Total**                   |        | **12-13 hours** |

---

**Document Version:** 2.0
**Created:** January 10, 2026
**Last Updated:** January 2026
**Status:** Ready for Implementation

**Changes from v1.0:**

- Updated to use ts-morph instead of Babel
- Aligned with current BaseAnalyzer architecture
- Integrated with AntiPatternRegistry system
- Added formatter and CLI integration tasks
- Updated file paths to match current structure
- Added types export task
