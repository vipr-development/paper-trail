# Phase 0: Infrastructure

**Purpose**: Establish the foundational infrastructure for the Vipr VSCode Extension, including plugin loading, engine integration, and configuration management.

**Dependencies**: None (foundation phase)

**Deliverables**: Core infrastructure components that all subsequent phases depend on

## Overview

Phase 0 sets up the core infrastructure required for all other features:

1. Extension activation and lifecycle management
2. Plugin loader (adapted from CLI's `CliPluginLoader`)
3. AnalysisEngine integration
4. PresenterRegistry setup
5. Configuration management
6. License validation system
7. Analysis cache manager

This phase ensures the extension follows the same plugin architecture as the CLI, enabling dynamic discovery of analyzers and presenters.

## Architecture Diagram

```mermaid
---
title: Phase 0 Infrastructure Components
config:
  theme: forest
---
graph TB
    subgraph Extension["Extension Activation"]
        Activate[extension.ts<br/>activate function]
    end

    subgraph Core["Core Infrastructure"]
        PluginLoader[VscodePluginLoader<br/>discoverBundledPlugins]
        Engine[AnalysisEngine<br/>from @vipr/engine]
        Registry[PresenterRegistry<br/>from @vipr/common]
        ConfigMgr[ConfigManager<br/>read VSCode settings]
        LicenseMgr[LicenseValidator<br/>validate tier]
        CacheMgr[AnalysisManager<br/>cache results]
    end

    subgraph Plugins["Loaded Plugins"]
        ReactPlugin[@vipr/react<br/>ReactAnalyzerPlugin]
        CorePlugin[@vipr/core<br/>CoreAnalyzerPlugin]
    end

    Activate --> PluginLoader
    Activate --> ConfigMgr
    Activate --> LicenseMgr
    Activate --> CacheMgr

    PluginLoader --> Engine
    PluginLoader --> Registry
    PluginLoader -.dynamic import.-> ReactPlugin
    PluginLoader -.dynamic import.-> CorePlugin

    Engine --> ReactPlugin
    Engine --> CorePlugin

    Registry --> ReactPlugin
    Registry --> CorePlugin

    classDef activation fill:#2563eb,stroke:#1e40af,color:#fff
    classDef core fill:#16a34a,stroke:#15803d,color:#fff
    classDef plugin fill:#dc2626,stroke:#b91c1c,color:#fff

    class Activate activation
    class PluginLoader,Engine,Registry,ConfigMgr,LicenseMgr,CacheMgr core
    class ReactPlugin,CorePlugin plugin
```

## File Changes

### 1. Extension Entry Point

**File**: `src/extension.ts`

Main entry point for extension activation and deactivation.

```typescript
import * as vscode from 'vscode';
import { AnalysisEngine } from '@vipr/engine';
import { VscodePluginLoader } from './core/plugin-loader';
import { ConfigManager } from './core/config-manager';
import { LicenseValidator } from './core/license-validator';
import { AnalysisManager } from './core/analysis-manager';

/**
 * Extension state shared across the lifecycle
 */
interface ExtensionState {
  engine: AnalysisEngine;
  pluginLoader: VscodePluginLoader;
  configManager: ConfigManager;
  licenseValidator: LicenseValidator;
  analysisManager: AnalysisManager;
  outputChannel: vscode.OutputChannel;
}

let state: ExtensionState | undefined;

/**
 * Called when extension is activated
 */
export async function activate(context: vscode.ExtensionContext): Promise<void> {
  const outputChannel = vscode.window.createOutputChannel('Vipr');
  outputChannel.appendLine('Vipr extension activating...');

  try {
    // Initialize core components
    const engine = new AnalysisEngine();
    const configManager = new ConfigManager();
    const licenseValidator = new LicenseValidator();
    const analysisManager = new AnalysisManager();

    // Load plugins
    const pluginLoader = new VscodePluginLoader(engine, outputChannel);
    await pluginLoader.loadBundledPlugins();

    // Store state
    state = {
      engine,
      pluginLoader,
      configManager,
      licenseValidator,
      analysisManager,
      outputChannel,
    };

    // Register state for disposal
    context.subscriptions.push({
      dispose: () => {
        outputChannel.dispose();
      },
    });

    outputChannel.appendLine('Vipr extension activated successfully');
  } catch (error) {
    const message = error instanceof Error ? error.message : String(error);
    outputChannel.appendLine(`Failed to activate extension: ${message}`);
    vscode.window.showErrorMessage(`Vipr: Failed to activate - ${message}`);
  }
}

/**
 * Called when extension is deactivated
 */
export function deactivate(): void {
  state?.outputChannel.appendLine('Vipr extension deactivating...');
  state = undefined;
}

/**
 * Get current extension state
 */
export function getExtensionState(): ExtensionState {
  if (!state) {
    throw new Error('Extension not activated');
  }
  return state;
}
```

### 2. VscodePluginLoader

**File**: `src/core/plugin-loader.ts`

Adapted from `clients/cli/src/plugins/loader.ts` for VSCode environment.

```typescript
import * as vscode from 'vscode';
import { AnalysisEngine } from '@vipr/engine';
import type { ITechnologyPlugin } from '@vipr/common';
import { PresenterRegistry } from '@vipr/common';

/**
 * Plugin loader for VSCode extension.
 * Dynamically discovers and loads analyzer plugins.
 *
 * Adapted from CliPluginLoader pattern.
 */
export class VscodePluginLoader {
  private engine: AnalysisEngine;
  private loadedPlugins: Map<string, ITechnologyPlugin> = new Map();
  private registry: PresenterRegistry = new PresenterRegistry();
  private outputChannel: vscode.OutputChannel;

  constructor(engine: AnalysisEngine, outputChannel: vscode.OutputChannel) {
    this.engine = engine;
    this.outputChannel = outputChannel;
  }

  /**
   * Load all bundled analyzer plugins
   */
  async loadBundledPlugins(options?: { quiet?: boolean; plugins?: string[] }): Promise<void> {
    this.outputChannel.appendLine('Loading Vipr analyzer plugins...');

    // Dynamically import bundled analyzers
    let discoveredPlugins = await this.discoverBundledPlugins();

    // Filter plugins if specified
    if (options?.plugins && options.plugins.length > 0) {
      discoveredPlugins = discoveredPlugins.filter(p =>
        options.plugins!.some(
          id =>
            p.id.toLowerCase() === id.toLowerCase() || p.id.toLowerCase().includes(id.toLowerCase())
        )
      );
    }

    for (const plugin of discoveredPlugins) {
      this.registerPlugin(plugin);
    }

    if (!options?.quiet) {
      this.outputChannel.appendLine(`Loaded ${this.loadedPlugins.size} analyzer plugin(s)`);
    }
  }

  /**
   * Discover plugins bundled with the extension.
   * Dynamically imports from @vipr/react and @vipr/core.
   */
  private async discoverBundledPlugins(): Promise<ITechnologyPlugin[]> {
    const plugins: ITechnologyPlugin[] = [];

    // Import React analyzer
    try {
      const { ReactAnalyzerPlugin } = await import('@vipr/react');
      plugins.push(new ReactAnalyzerPlugin());
      this.outputChannel.appendLine('  ✓ Loaded React analyzer');
    } catch (error) {
      const message = error instanceof Error ? error.message : String(error);
      this.outputChannel.appendLine(`  ✗ React analyzer not available: ${message}`);
    }

    // Import Core analyzer
    try {
      const { CoreAnalyzerPlugin } = await import('@vipr/core');
      plugins.push(new CoreAnalyzerPlugin());
      this.outputChannel.appendLine('  ✓ Loaded Core analyzer');
    } catch (error) {
      const message = error instanceof Error ? error.message : String(error);
      this.outputChannel.appendLine(`  ✗ Core analyzer not available: ${message}`);
    }

    return plugins;
  }

  /**
   * Register a plugin with the engine and presenter registry
   */
  private registerPlugin(plugin: ITechnologyPlugin): void {
    // Register with engine
    this.engine.registerPlugin(plugin);
    this.loadedPlugins.set(plugin.id, plugin);

    // Register presenters with registry
    this.registry.registerFromPlugin(plugin);

    this.outputChannel.appendLine(`  Registered plugin: ${plugin.name} (${plugin.id})`);
  }

  /**
   * Get all loaded plugins
   */
  getLoadedPlugins(): ITechnologyPlugin[] {
    return Array.from(this.loadedPlugins.values());
  }

  /**
   * Get plugin by ID
   */
  getPlugin(id: string): ITechnologyPlugin | undefined {
    return this.loadedPlugins.get(id);
  }

  /**
   * Get the presenter registry with all registered presenters
   */
  getPresenterRegistry(): PresenterRegistry {
    return this.registry;
  }
}
```

### 3. ConfigManager

**File**: `src/core/config-manager.ts`

Manages extension configuration from VSCode settings.

```typescript
import * as vscode from 'vscode';
import type { ViprConfiguration, AnalysisConfiguration } from '../types/config';
import { DEFAULT_CONFIG } from '../types/config';

/**
 * Manages extension configuration from VSCode settings
 */
export class ConfigManager {
  private readonly configSection = 'vipr';

  /**
   * Get full extension configuration
   */
  getConfiguration(): ViprConfiguration {
    const config = vscode.workspace.getConfiguration(this.configSection);

    return {
      analyzeOnSave: config.get('analyzeOnSave', DEFAULT_CONFIG.analyzeOnSave),
      analyzeOnOpen: config.get('analyzeOnOpen', DEFAULT_CONFIG.analyzeOnOpen),
      showInlineHints: config.get('showInlineHints', DEFAULT_CONFIG.showInlineHints),
      showDecorations: config.get('showDecorations', DEFAULT_CONFIG.showDecorations),
      diagnosticSeverity: config.get('diagnosticSeverity', DEFAULT_CONFIG.diagnosticSeverity),
      licenseKey: config.get('licenseKey'),
      debug: config.get('debug', DEFAULT_CONFIG.debug),
      enabledPlugins: config.get('enabledPlugins', DEFAULT_CONFIG.enabledPlugins),
      analyses: config.get('analyses', DEFAULT_CONFIG.analyses),
      autoFixOnSave: config.get('autoFixOnSave', DEFAULT_CONFIG.autoFixOnSave),
      enableAiFixing: config.get('enableAiFixing', DEFAULT_CONFIG.enableAiFixing),
      aiProvider: config.get('aiProvider', DEFAULT_CONFIG.aiProvider),
    };
  }

  /**
   * Get specific configuration value
   */
  get<K extends keyof ViprConfiguration>(key: K): ViprConfiguration[K] {
    const config = vscode.workspace.getConfiguration(this.configSection);
    return config.get(key, DEFAULT_CONFIG[key]);
  }

  /**
   * Update configuration value
   */
  async set<K extends keyof ViprConfiguration>(
    key: K,
    value: ViprConfiguration[K],
    target: vscode.ConfigurationTarget = vscode.ConfigurationTarget.Global
  ): Promise<void> {
    const config = vscode.workspace.getConfiguration(this.configSection);
    await config.update(key, value, target);
  }

  /**
   * Get analyzer-specific configuration
   */
  getAnalyzerConfig(): Record<string, AnalysisConfiguration> {
    return this.get('analyses');
  }

  /**
   * Check if analysis is enabled
   */
  isAnalysisEnabled(analysisId: string): boolean {
    const analyses = this.getAnalyzerConfig();
    return analyses[analysisId]?.enabled ?? true; // Default to enabled
  }

  /**
   * Watch for configuration changes
   */
  onDidChangeConfiguration(
    callback: (e: vscode.ConfigurationChangeEvent) => void
  ): vscode.Disposable {
    return vscode.workspace.onDidChangeConfiguration(e => {
      if (e.affectsConfiguration(this.configSection)) {
        callback(e);
      }
    });
  }
}
```

### 4. LicenseValidator

**File**: `src/core/license-validator.ts`

Simple prefix-based license validation (no server calls for MVP).

```typescript
import type { LicenseValidationResult, LicenseTier } from '../types/license';
import { parseLicenseKey, REPORT_ACCESS } from '../types/license';

/**
 * License validator for tiered feature access.
 * MVP implementation uses simple prefix validation.
 */
export class LicenseValidator {
  /**
   * Validate a license key
   */
  validate(licenseKey?: string): LicenseValidationResult {
    // No license = free tier
    if (!licenseKey || licenseKey.trim() === '') {
      return {
        valid: true,
        tier: 'free',
        enabledFeatures: REPORT_ACCESS.free,
      };
    }

    // Parse license key
    const parsed = parseLicenseKey(licenseKey);
    if (!parsed) {
      return {
        valid: false,
        tier: 'free',
        error: 'Invalid license key format',
        enabledFeatures: REPORT_ACCESS.free,
      };
    }

    // Validate format: prefix must be followed by identifier
    if (!parsed.identifier || parsed.identifier.length < 8) {
      return {
        valid: false,
        tier: 'free',
        error: 'License key identifier too short',
        enabledFeatures: REPORT_ACCESS.free,
      };
    }

    // Return validation result based on tier
    return {
      valid: true,
      tier: parsed.tier,
      enabledFeatures: REPORT_ACCESS[parsed.tier],
    };
  }

  /**
   * Check if user has access to a specific report
   */
  hasAccess(licenseKey: string | undefined, reportType: string): boolean {
    const result = this.validate(licenseKey);
    return result.enabledFeatures.includes(reportType);
  }

  /**
   * Get current license tier
   */
  getTier(licenseKey?: string): LicenseTier {
    return this.validate(licenseKey).tier;
  }
}
```

### 5. AnalysisManager

**File**: `src/core/analysis-manager.ts`

Manages analysis state and caching across files.

```typescript
import * as vscode from 'vscode';
import * as crypto from 'crypto';
import type { AggregatedResult } from '@vipr/common';
import type { FileAnalysisState, AnalysisCacheEntry, CacheOptions } from '../types/analysis';
import { DEFAULT_CACHE_OPTIONS } from '../types/analysis';

/**
 * Manages analysis state and caching for files
 */
export class AnalysisManager {
  private fileStates: Map<string, FileAnalysisState> = new Map();
  private cache: Map<string, AnalysisCacheEntry> = new Map();
  private cacheOptions: CacheOptions;

  constructor(options?: Partial<CacheOptions>) {
    this.cacheOptions = { ...DEFAULT_CACHE_OPTIONS, ...options };
  }

  /**
   * Get analysis state for a file
   */
  getState(uri: vscode.Uri): FileAnalysisState | undefined {
    return this.fileStates.get(uri.toString());
  }

  /**
   * Set analysis state for a file
   */
  setState(state: FileAnalysisState): void {
    const key = state.uri.toString();
    this.fileStates.set(key, state);

    // Update cache
    if (this.cacheOptions.enabled) {
      this.cache.set(key, {
        result: state.result,
        contentHash: state.contentHash,
        cachedAt: new Date(),
        ttl: this.cacheOptions.defaultTtl,
      });

      // Enforce max entries
      this.evictOldestIfNeeded();
    }
  }

  /**
   * Get cached result if valid
   */
  getCachedResult(uri: vscode.Uri, currentHash: string): AggregatedResult | undefined {
    if (!this.cacheOptions.enabled) {
      return undefined;
    }

    const key = uri.toString();
    const cached = this.cache.get(key);

    if (!cached) {
      return undefined;
    }

    // Check if hash matches
    if (cached.contentHash !== currentHash) {
      this.cache.delete(key);
      return undefined;
    }

    // Check if expired
    const now = Date.now();
    const age = now - cached.cachedAt.getTime();
    if (age > cached.ttl) {
      this.cache.delete(key);
      return undefined;
    }

    return cached.result;
  }

  /**
   * Mark file as analyzing
   */
  setAnalyzing(uri: vscode.Uri, isAnalyzing: boolean): void {
    const state = this.fileStates.get(uri.toString());
    if (state) {
      state.isAnalyzing = isAnalyzing;
    }
  }

  /**
   * Clear state for a file
   */
  clearState(uri: vscode.Uri): void {
    const key = uri.toString();
    this.fileStates.delete(key);
    this.cache.delete(key);
  }

  /**
   * Clear all state
   */
  clearAll(): void {
    this.fileStates.clear();
    this.cache.clear();
  }

  /**
   * Compute content hash for cache validation
   */
  computeContentHash(content: string): string {
    return crypto.createHash('sha256').update(content).digest('hex');
  }

  /**
   * Evict oldest cache entry if max entries exceeded
   */
  private evictOldestIfNeeded(): void {
    if (this.cache.size `<=` this.cacheOptions.maxEntries) {
      return;
    }

    let oldestKey: string | undefined;
    let oldestTime = Date.now();

    for (const [key, entry] of this.cache.entries()) {
      const time = entry.cachedAt.getTime();
      if (time < oldestTime) {
        oldestTime = time;
        oldestKey = key;
      }
    }

    if (oldestKey) {
      this.cache.delete(oldestKey);
    }
  }
}
```

## Configuration

**File**: `package.json` (contributes.configuration section)

```json
{
  "contributes": {
    "configuration": {
      "title": "Vipr",
      "properties": {
        "vipr.analyzeOnSave": {
          "type": "boolean",
          "default": true,
          "description": "Automatically analyze files on save"
        },
        "vipr.analyzeOnOpen": {
          "type": "boolean",
          "default": false,
          "description": "Automatically analyze files when opened"
        },
        "vipr.showInlineHints": {
          "type": "boolean",
          "default": true,
          "description": "Show inline CodeLens hints above components"
        },
        "vipr.showDecorations": {
          "type": "boolean",
          "default": true,
          "description": "Show editor decorations for issues"
        },
        "vipr.diagnosticSeverity": {
          "type": "string",
          "enum": ["all", "warning", "error"],
          "default": "all",
          "description": "Minimum severity level for diagnostics"
        },
        "vipr.licenseKey": {
          "type": "string",
          "default": "",
          "description": "License key for paid features"
        },
        "vipr.debug": {
          "type": "boolean",
          "default": false,
          "description": "Enable debug logging to output channel"
        },
        "vipr.enabledPlugins": {
          "type": "array",
          "items": {
            "type": "string"
          },
          "default": [],
          "description": "List of enabled plugin IDs (empty = all plugins)"
        },
        "vipr.analyses": {
          "type": "object",
          "default": {},
          "description": "Analysis-specific configuration"
        },
        "vipr.autoFixOnSave": {
          "type": "boolean",
          "default": false,
          "description": "Automatically fix issues on save when possible"
        },
        "vipr.enableAiFixing": {
          "type": "boolean",
          "default": true,
          "description": "Enable AI-assisted fixing"
        },
        "vipr.aiProvider": {
          "type": "string",
          "enum": ["copilot", "cursor", "clipboard", "auto"],
          "default": "auto",
          "description": "Preferred AI provider for fixing suggestions"
        }
      }
    }
  }
}
```

## Acceptance Criteria

- [ ] Extension activates successfully on startup
- [ ] VscodePluginLoader dynamically imports React and Core analyzers
- [ ] AnalysisEngine is properly initialized and registered with plugins
- [ ] PresenterRegistry contains all presenters from loaded plugins
- [ ] ConfigManager reads settings from VSCode workspace configuration
- [ ] LicenseValidator correctly parses and validates license keys
- [ ] Free tier returns correct report access list
- [ ] Pro tier returns correct report access list
- [ ] Enterprise tier returns correct report access list
- [ ] AnalysisManager caches analysis results with content hash validation
- [ ] Cache respects TTL and max entries settings
- [ ] Cache invalidation works when content changes
- [ ] Output channel logs plugin loading progress
- [ ] Extension deactivates cleanly without errors

## Testing Strategy

### Unit Tests

**File**: `src/core/plugin-loader.test.ts`

```typescript
import { describe, it, expect, beforeEach } from 'vitest';
import { AnalysisEngine } from '@vipr/engine';
import { VscodePluginLoader } from './plugin-loader';

describe('VscodePluginLoader', () => {
  let engine: AnalysisEngine;
  let mockOutputChannel: any;

  beforeEach(() => {
    engine = new AnalysisEngine();
    mockOutputChannel = {
      appendLine: vi.fn(),
    };
  });

  it('should load bundled plugins', async () => {
    const loader = new VscodePluginLoader(engine, mockOutputChannel);
    await loader.loadBundledPlugins({ quiet: true });

    const plugins = loader.getLoadedPlugins();
    expect(plugins.length).toBeGreaterThan(0);
  });

  it('should register presenters with registry', async () => {
    const loader = new VscodePluginLoader(engine, mockOutputChannel);
    await loader.loadBundledPlugins({ quiet: true });

    const registry = loader.getPresenterRegistry();
    const reports = registry.getAvailableReports();
    expect(reports.length).toBeGreaterThan(0);
  });

  it('should filter plugins by ID', async () => {
    const loader = new VscodePluginLoader(engine, mockOutputChannel);
    await loader.loadBundledPlugins({ quiet: true, plugins: ['react'] });

    const plugins = loader.getLoadedPlugins();
    expect(plugins.every(p => p.id === 'react')).toBe(true);
  });
});
```

**File**: `src/core/license-validator.test.ts`

```typescript
import { describe, it, expect } from 'vitest';
import { LicenseValidator } from './license-validator';

describe('LicenseValidator', () => {
  const validator = new LicenseValidator();

  it('should validate free tier with no key', () => {
    const result = validator.validate();
    expect(result.valid).toBe(true);
    expect(result.tier).toBe('free');
  });

  it('should validate pro tier key', () => {
    const result = validator.validate('VIPR-PRO-12345678');
    expect(result.valid).toBe(true);
    expect(result.tier).toBe('pro');
  });

  it('should validate enterprise tier key', () => {
    const result = validator.validate('VIPR-ENT-12345678');
    expect(result.valid).toBe(true);
    expect(result.tier).toBe('enterprise');
  });

  it('should reject invalid format', () => {
    const result = validator.validate('INVALID-KEY');
    expect(result.valid).toBe(false);
    expect(result.tier).toBe('free');
  });

  it('should reject short identifier', () => {
    const result = validator.validate('VIPR-PRO-123');
    expect(result.valid).toBe(false);
  });
});
```

**File**: `src/core/analysis-manager.test.ts`

```typescript
import { describe, it, expect, beforeEach } from 'vitest';
import * as vscode from 'vscode';
import { AnalysisManager } from './analysis-manager';

describe('AnalysisManager', () => {
  let manager: AnalysisManager;

  beforeEach(() => {
    manager = new AnalysisManager();
  });

  it('should compute consistent content hash', () => {
    const content = 'const x = 1;';
    const hash1 = manager.computeContentHash(content);
    const hash2 = manager.computeContentHash(content);
    expect(hash1).toBe(hash2);
  });

  it('should cache analysis results', () => {
    const uri = vscode.Uri.file('/test.ts');
    const hash = manager.computeContentHash('test content');

    const mockResult: any = {
      filePath: '/test.ts',
      overallScore: 85,
      pluginResults: new Map(),
      insights: [],
      errors: [],
    };

    manager.setState({
      uri,
      result: mockResult,
      diagnostics: [],
      analyzedAt: new Date(),
      contentHash: hash,
      isAnalyzing: false,
    });

    const cached = manager.getCachedResult(uri, hash);
    expect(cached).toBeDefined();
    expect(cached?.overallScore).toBe(85);
  });

  it('should invalidate cache on hash mismatch', () => {
    const uri = vscode.Uri.file('/test.ts');
    const oldHash = manager.computeContentHash('old content');
    const newHash = manager.computeContentHash('new content');

    const mockResult: any = {
      filePath: '/test.ts',
      overallScore: 85,
      pluginResults: new Map(),
      insights: [],
      errors: [],
    };

    manager.setState({
      uri,
      result: mockResult,
      diagnostics: [],
      analyzedAt: new Date(),
      contentHash: oldHash,
      isAnalyzing: false,
    });

    const cached = manager.getCachedResult(uri, newHash);
    expect(cached).toBeUndefined();
  });
});
```

### Integration Tests

Test that all components work together:

```typescript
import { describe, it, expect } from 'vitest';
import { AnalysisEngine } from '@vipr/engine';
import { VscodePluginLoader } from './plugin-loader';
import { ConfigManager } from './config-manager';
import { LicenseValidator } from './license-validator';

describe('Infrastructure Integration', () => {
  it('should initialize complete infrastructure', async () => {
    const engine = new AnalysisEngine();
    const mockOutputChannel: any = { appendLine: vi.fn() };
    const loader = new VscodePluginLoader(engine, mockOutputChannel);
    const configManager = new ConfigManager();
    const licenseValidator = new LicenseValidator();

    await loader.loadBundledPlugins({ quiet: true });

    // Verify plugins loaded
    expect(loader.getLoadedPlugins().length).toBeGreaterThan(0);

    // Verify registry populated
    const registry = loader.getPresenterRegistry();
    expect(registry.getAvailableReports().length).toBeGreaterThan(0);

    // Verify config reads
    const config = configManager.getConfiguration();
    expect(config).toBeDefined();

    // Verify license validation
    const validation = licenseValidator.validate();
    expect(validation.tier).toBe('free');
  });
});
```

## Code Examples

### Accessing Infrastructure from Commands

```typescript
import * as vscode from 'vscode';
import { getExtensionState } from '../extension';

export async function analyzeCurrentFile(): Promise<void> {
  const { engine, configManager, analysisManager } = getExtensionState();

  const editor = vscode.window.activeTextEditor;
  if (!editor) {
    return;
  }

  const document = editor.document;
  const content = document.getText();
  const hash = analysisManager.computeContentHash(content);

  // Check cache
  const cached = analysisManager.getCachedResult(document.uri, hash);
  if (cached) {
    // Use cached result
    return;
  }

  // Run analysis
  analysisManager.setAnalyzing(document.uri, true);
  const result = await engine.analyzeFile(document.uri.fsPath);
  analysisManager.setAnalyzing(document.uri, false);

  // Store result
  analysisManager.setState({
    uri: document.uri,
    result,
    diagnostics: [],
    analyzedAt: new Date(),
    contentHash: hash,
    isAnalyzing: false,
  });
}
```

## Summary

Phase 0 establishes the foundation for all subsequent features:

- Plugin system adapted from CLI with dynamic loading
- Engine and registry properly initialized
- Configuration management integrated with VSCode settings
- License validation with tier-based access control
- Analysis caching for performance optimization

All infrastructure components follow the existing Vipr architecture patterns and avoid anti-patterns like hardcoded report types or direct presenter imports.
