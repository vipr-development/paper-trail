---
id: 01-types
---

# VSCode Extension Type Definitions

**Purpose**: Comprehensive TypeScript type definitions for the Vipr VSCode Extension v2.

**Last Updated**: 2026-01-18

## Overview

This document defines all extension-specific types. These types complement the existing types from `@vipr/common` and are specific to the VSCode extension implementation.

## File Organization

Types are organized into separate files by domain:

```
src/types/
  ├─ config.ts       # Extension configuration types
  ├─ analysis.ts     # Analysis state and cache types
  ├─ webview.ts      # Webview message protocol types
  └─ license.ts      # License validation types
```

## Configuration Types

**File**: `src/types/config.ts`

These types represent the extension's configuration settings stored in VSCode's settings.

```typescript
/**
 * Extension configuration interface.
 * Mapped from package.json contributes.configuration section.
 */
export interface ViprConfiguration {
  /** Enable automatic analysis on file save */
  analyzeOnSave: boolean;

  /** Enable automatic analysis on file open */
  analyzeOnOpen: boolean;

  /** Show inline CodeLens hints above components */
  showInlineHints: boolean;

  /** Show editor decorations for issues */
  showDecorations: boolean;

  /** Minimum severity level for diagnostics */
  diagnosticSeverity: DiagnosticSeverityLevel;

  /** License key for paid features */
  licenseKey?: string;

  /** Enable debug logging to output channel */
  debug: boolean;

  /** Plugins to enable (empty = all plugins) */
  enabledPlugins: string[];

  /** Analysis-specific configuration */
  analyses: Record<string, AnalysisConfiguration>;

  /** Automatically fix issues on save when possible */
  autoFixOnSave: boolean;

  /** Enable AI-assisted fixing */
  enableAiFixing: boolean;

  /** Preferred AI provider (copilot, cursor, clipboard) */
  aiProvider: AiProvider;
}

/**
 * Diagnostic severity level for filtering
 */
export type DiagnosticSeverityLevel = 'all' | 'warning' | 'error';

/**
 * AI provider options
 */
export type AiProvider = 'copilot' | 'cursor' | 'clipboard' | 'auto';

/**
 * Configuration for individual analyses
 */
export interface AnalysisConfiguration {
  /** Whether this analysis is enabled */
  enabled: boolean;

  /** Analysis-specific options */
  options?: Record<string, unknown>;
}

/**
 * Default configuration values
 */
export const DEFAULT_CONFIG: ViprConfiguration = {
  analyzeOnSave: true,
  analyzeOnOpen: false,
  showInlineHints: true,
  showDecorations: true,
  diagnosticSeverity: 'all',
  debug: false,
  enabledPlugins: [],
  analyses: {},
  autoFixOnSave: false,
  enableAiFixing: true,
  aiProvider: 'auto',
};
```

## Analysis State Types

**File**: `src/types/analysis.ts`

These types manage analysis state, caching, and results within the extension.

```typescript
import type { AggregatedResult, PluginResult, ComplexityInsight } from '@vipr/common';
import type { Uri, Diagnostic } from 'vscode';

/**
 * Analysis state for a file
 */
export interface FileAnalysisState {
  /** File URI */
  uri: Uri;

  /** Analysis result */
  result: AggregatedResult;

  /** Generated diagnostics for Problems panel */
  diagnostics: Diagnostic[];

  /** Timestamp of analysis */
  analyzedAt: Date;

  /** Content hash when analyzed (for cache invalidation) */
  contentHash: string;

  /** Whether analysis is in progress */
  isAnalyzing: boolean;

  /** Analysis error if failed */
  error?: Error;
}

/**
 * Workspace analysis summary
 */
export interface WorkspaceAnalysisSummary {
  /** Total files analyzed */
  totalFiles: number;

  /** Files with issues */
  filesWithIssues: number;

  /** Average score across all files */
  averageScore: number;

  /** Total issues by severity */
  issueBySeverity: {
    critical: number;
    warning: number;
    info: number;
  };

  /** Top 10 files with lowest scores */
  hotspots: HotspotFile[];

  /** Analysis timestamp */
  analyzedAt: Date;
}

/**
 * Hotspot file with low score
 */
export interface HotspotFile {
  /** File URI */
  uri: Uri;

  /** Overall score */
  score: number;

  /** Issue count */
  issueCount: number;

  /** Plugin breakdown */
  pluginScores: PluginScore[];
}

/**
 * Plugin score for a file
 */
export interface PluginScore {
  /** Plugin ID */
  pluginId: string;

  /** Plugin name */
  name: string;

  /** Score for this plugin */
  score?: number;

  /** Issue count for this plugin */
  issueCount: number;
}

/**
 * Cache entry for analysis results
 */
export interface AnalysisCacheEntry {
  /** Cached result */
  result: AggregatedResult;

  /** Content hash */
  contentHash: string;

  /** Cache timestamp */
  cachedAt: Date;

  /** Time to live in milliseconds */
  ttl: number;
}

/**
 * Analysis cache manager options
 */
export interface CacheOptions {
  /** Maximum cache entries */
  maxEntries: number;

  /** Default TTL in milliseconds */
  defaultTtl: number;

  /** Enable cache */
  enabled: boolean;
}

/**
 * Default cache options
 */
export const DEFAULT_CACHE_OPTIONS: CacheOptions = {
  maxEntries: 100,
  defaultTtl: 5 * 60 * 1000, // 5 minutes
  enabled: true,
};
```

## Webview Message Types

**File**: `src/types/webview.ts`

These types define the message protocol between the extension and webview dashboard.

```typescript
import type { PluginResult } from '@vipr/common';

/**
 * Base message type for webview communication
 */
export interface WebviewMessage {
  /** Message type discriminator */
  type: string;

  /** Message payload */
  payload?: unknown;
}

/**
 * Message from extension to webview
 */
export type ExtensionToWebviewMessage =
  | UpdateAnalysisMessage
  | UpdateThemeMessage
  | UpdateLicenseMessage
  | ShowErrorMessage;

/**
 * Message from webview to extension
 */
export type WebviewToExtensionMessage =
  | NavigateToIssueMessage
  | RefreshAnalysisMessage
  | AnalyzeWorkspaceMessage
  | OpenSettingsMessage;

/**
 * Update analysis result in dashboard
 */
export interface UpdateAnalysisMessage extends WebviewMessage {
  type: 'updateAnalysis';
  payload: DashboardData;
}

/**
 * Update theme (light/dark)
 */
export interface UpdateThemeMessage extends WebviewMessage {
  type: 'updateTheme';
  payload: {
    theme: 'light' | 'dark';
  };
}

/**
 * Update license status
 */
export interface UpdateLicenseMessage extends WebviewMessage {
  type: 'updateLicense';
  payload: {
    tier: 'free' | 'pro' | 'enterprise';
    validUntil?: string;
  };
}

/**
 * Show error in dashboard
 */
export interface ShowErrorMessage extends WebviewMessage {
  type: 'showError';
  payload: {
    message: string;
    details?: string;
  };
}

/**
 * Navigate to specific issue in editor
 */
export interface NavigateToIssueMessage extends WebviewMessage {
  type: 'navigateToIssue';
  payload: {
    filePath: string;
    line: number;
    column: number;
  };
}

/**
 * Refresh current file analysis
 */
export interface RefreshAnalysisMessage extends WebviewMessage {
  type: 'refreshAnalysis';
}

/**
 * Analyze entire workspace
 */
export interface AnalyzeWorkspaceMessage extends WebviewMessage {
  type: 'analyzeWorkspace';
}

/**
 * Open extension settings
 */
export interface OpenSettingsMessage extends WebviewMessage {
  type: 'openSettings';
}

/**
 * Dashboard data model
 */
export interface DashboardData {
  /** Current file URI */
  fileUri: string;

  /** Overall score */
  overallScore: number;

  /** Score level (excellent, good, fair, poor) */
  scoreLevel: ScoreLevel;

  /** Plugin results breakdown */
  plugins: DashboardPlugin[];

  /** Top issues to address */
  topIssues: DashboardIssue[];

  /** Analysis timestamp */
  analyzedAt: string;

  /** Whether analysis is in progress */
  isAnalyzing: boolean;

  /** License tier */
  licenseTier: 'free' | 'pro' | 'enterprise';
}

/**
 * Plugin data for dashboard
 */
export interface DashboardPlugin {
  /** Plugin ID */
  id: string;

  /** Plugin name */
  name: string;

  /** Overall plugin score */
  score?: number;

  /** Score level */
  scoreLevel?: ScoreLevel;

  /** Available reports from this plugin */
  reports: DashboardReport[];

  /** Total issues from this plugin */
  issueCount: number;
}

/**
 * Report data for dashboard
 */
export interface DashboardReport {
  /** Report type */
  reportType: string;

  /** Display label */
  label: string;

  /** Icon (emoji or name) */
  icon?: string;

  /** Report score */
  score?: number;

  /** Score level */
  scoreLevel?: ScoreLevel;

  /** Issue count for this report */
  issueCount: number;

  /** Whether this report requires paid license */
  requiresLicense: boolean;

  /** Whether user has access to this report */
  hasAccess: boolean;
}

/**
 * Issue data for dashboard
 */
export interface DashboardIssue {
  /** Issue message */
  message: string;

  /** Severity */
  severity: 'critical' | 'warning' | 'info';

  /** Category */
  category: string;

  /** File path */
  filePath: string;

  /** Line number */
  line: number;

  /** Column number */
  column: number;

  /** Whether auto-fixable */
  autoFixable: boolean;

  /** Suggestion */
  suggestion?: string;
}

/**
 * Score level classification
 */
export type ScoreLevel = 'excellent' | 'good' | 'fair' | 'poor';

/**
 * Classify score into level
 */
export function getScoreLevel(score: number): ScoreLevel {
  if (score >= 80) return 'excellent';
  if (score >= 60) return 'good';
  if (score >= 40) return 'fair';
  return 'poor';
}
```

## License Types

**File**: `src/types/license.ts`

These types manage license validation and tier access.

```typescript
/**
 * License tier levels
 */
export type LicenseTier = 'free' | 'pro' | 'enterprise';

/**
 * License validation result
 */
export interface LicenseValidationResult {
  /** Whether license is valid */
  valid: boolean;

  /** License tier */
  tier: LicenseTier;

  /** License expiration date (if applicable) */
  validUntil?: Date;

  /** Validation error message */
  error?: string;

  /** Features enabled by this license */
  enabledFeatures: string[];
}

/**
 * License key structure (for prefix validation)
 */
export interface LicenseKey {
  /** Raw license key string */
  raw: string;

  /** Parsed prefix (VIPR-FREE, VIPR-PRO, VIPR-ENT) */
  prefix: string;

  /** License tier derived from prefix */
  tier: LicenseTier;

  /** Key identifier part */
  identifier: string;
}

/**
 * Report access levels by tier
 */
export const REPORT_ACCESS: Record<LicenseTier, string[]> = {
  free: ['core-overview', 'react-overview'],
  pro: [
    'core-overview',
    'react-overview',
    'security',
    'accessibility',
    'performance',
    'reliability',
    'anti-patterns',
  ],
  enterprise: [
    'core-overview',
    'react-overview',
    'security',
    'accessibility',
    'performance',
    'reliability',
    'anti-patterns',
    'migration',
    'dataflow',
    'technical-debt',
  ],
};

/**
 * License key prefixes
 */
export const LICENSE_PREFIXES: Record<LicenseTier, string> = {
  free: 'VIPR-FREE',
  pro: 'VIPR-PRO',
  enterprise: 'VIPR-ENT',
};

/**
 * Parse license key
 */
export function parseLicenseKey(key: string): LicenseKey | null {
  const trimmed = key.trim().toUpperCase();

  for (const [tier, prefix] of Object.entries(LICENSE_PREFIXES)) {
    if (trimmed.startsWith(prefix)) {
      const identifier = trimmed.substring(prefix.length + 1); // +1 for hyphen
      return {
        raw: key,
        prefix,
        tier: tier as LicenseTier,
        identifier,
      };
    }
  }

  return null;
}

/**
 * Check if report requires license
 */
export function requiresLicense(reportType: string): boolean {
  return !REPORT_ACCESS.free.includes(reportType);
}

/**
 * Check if tier has access to report
 */
export function hasAccess(tier: LicenseTier, reportType: string): boolean {
  return REPORT_ACCESS[tier].includes(reportType);
}
```

## Shared Types

Types that are imported from `@vipr/common` and used throughout the extension:

```typescript
// From @vipr/common
import type {
  // Core types
  ComplexityInsight,
  CodeLocation,
  CodeFix,
  Severity,

  // Plugin types
  ITechnologyPlugin,
  PluginResult,
  AggregatedResult,

  // Presentation types
  IReportPresenter,
  IReportMetadata,
  IPresenterRegistry,
  ReportPresentation,
  PresentationSection,
  PresentationItem,
  PresentationMetric,

  // Analysis types
  AnalyzerConfig,
  IAnalysis,
  AnalysisResult,
} from '@vipr/common';
```

## VSCode API Types

VSCode types used throughout the extension:

```typescript
import type {
  // Extension lifecycle
  ExtensionContext,

  // Commands
  Command,

  // Diagnostics
  Diagnostic,
  DiagnosticCollection,
  DiagnosticSeverity,

  // Providers
  CodeLensProvider,
  CodeActionProvider,

  // UI
  StatusBarItem,
  OutputChannel,
  WebviewView,
  WebviewViewProvider,

  // Editor
  TextDocument,
  TextEditor,
  Range,
  Position,
  TextEditorDecorationType,

  // Workspace
  Uri,
  WorkspaceConfiguration,
} from 'vscode';
```

## Type Utility Helpers

**File**: `src/types/utils.ts`

Utility types for common patterns:

```typescript
/**
 * Make specific properties required
 */
export type RequireFields<T, K extends keyof T> = T & Required<Pick<T, K>>;

/**
 * Make specific properties optional
 */
export type OptionalFields<T, K extends keyof T> = Omit<T, K> & Partial<Pick<T, K>>;

/**
 * Extract promise result type
 */
export type Awaited<T> = T extends Promise<infer U> ? U : T;

/**
 * Non-nullable type
 */
export type NonNullable<T> = T extends null | undefined ? never : T;

/**
 * Result type for operations that can fail
 */
export type Result<T, E = Error> = { success: true; value: T } | { success: false; error: E };

/**
 * Create success result
 */
export function success<T>(value: T): Result<T> {
  return { success: true, value };
}

/**
 * Create error result
 */
export function error<E = Error>(error: E): Result<never, E> {
  return { success: false, error };
}
```

## Type Guards

**File**: `src/types/guards.ts`

Runtime type checking utilities:

```typescript
import type {
  WebviewMessage,
  ExtensionToWebviewMessage,
  WebviewToExtensionMessage,
} from './webview';

/**
 * Check if message is from extension to webview
 */
export function isExtensionToWebviewMessage(msg: WebviewMessage): msg is ExtensionToWebviewMessage {
  return ['updateAnalysis', 'updateTheme', 'updateLicense', 'showError'].includes(msg.type);
}

/**
 * Check if message is from webview to extension
 */
export function isWebviewToExtensionMessage(msg: WebviewMessage): msg is WebviewToExtensionMessage {
  return ['navigateToIssue', 'refreshAnalysis', 'analyzeWorkspace', 'openSettings'].includes(
    msg.type
  );
}

/**
 * Check if value is LicenseTier
 */
export function isLicenseTier(value: unknown): value is LicenseTier {
  return value === 'free' || value === 'pro' || value === 'enterprise';
}

/**
 * Check if value is AiProvider
 */
export function isAiProvider(value: unknown): value is AiProvider {
  return value === 'copilot' || value === 'cursor' || value === 'clipboard' || value === 'auto';
}
```

## Summary

These types provide a complete type-safe foundation for the extension:

- **Configuration**: User settings with defaults
- **Analysis State**: File-level and workspace-level analysis tracking
- **Webview Protocol**: Bidirectional message types for dashboard communication
- **License**: Tier validation and access control
- **Utilities**: Helper types and type guards

All types are designed to integrate seamlessly with `@vipr/common` types and VSCode API types.
