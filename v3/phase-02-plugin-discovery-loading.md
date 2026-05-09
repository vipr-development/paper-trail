# Phase 2: Plugin Discovery & Loading

## Overview

Phase 2 implements the plugin discovery and loading infrastructure that enables the CLI to automatically discover and register analyzer plugins from the Turborepo workspace. This phase creates the foundation for a decoupled plugin architecture where the CLI has no direct dependencies on specific analyzers, instead discovering them dynamically at runtime.

This phase also includes the analyzer template system that allows developers to quickly scaffold new analyzer plugins with a standardized structure.

## Objectives

1. Create `common/plugin-loader` package for plugin discovery and loading
2. Implement workspace package discovery for Turborepo monorepo
3. Build plugin validation and registry system
4. Create `common/analyzer-template` package for scaffolding new analyzers
5. Implement CLI wizard for analyzer generation
6. Ensure type safety throughout the plugin loading process
7. Provide comprehensive error handling and validation

## Technical Scope

### Core Capabilities

**Plugin Discovery**

- Scan `pnpm-workspace.yaml` to identify workspace packages
- Filter packages matching `analyzers/*` pattern
- Read package.json metadata for each analyzer
- Validate analyzer package structure
- Discover plugin entry points via package.json exports

**Plugin Loading**

- Dynamic import of analyzer modules at runtime
- Validation of plugin interface implementation
- Plugin instantiation with error handling
- Plugin lifecycle management (onRegister/onUnregister)
- Dependency resolution for plugin configuration

**Plugin Registry**

- Centralized plugin storage and retrieval
- Priority-based plugin ordering
- Enable/disable plugin functionality
- Plugin configuration management
- Thread-safe plugin access

**Template System**

- Scaffolding structure for new analyzers
- Interactive CLI wizard for configuration
- Package.json generation with correct workspace references
- TypeScript configuration setup
- Example analysis implementations
- Documentation templates

## Package Structure

### common/plugin-loader

```
common/plugin-loader/
├── src/
│   ├── discovery/
│   │   ├── workspace-scanner.ts      # Scan workspace packages
│   │   ├── package-analyzer.ts       # Analyze package.json for plugins
│   │   ├── plugin-validator.ts       # Validate plugin structure
│   │   └── index.ts
│   ├── loader/
│   │   ├── dynamic-loader.ts         # Dynamic import handler
│   │   ├── plugin-instantiator.ts    # Create plugin instances
│   │   ├── error-handler.ts          # Plugin loading error handling
│   │   └── index.ts
│   ├── registry/
│   │   ├── plugin-registry.ts        # Plugin storage and retrieval
│   │   ├── registry-types.ts         # Registry-specific types
│   │   └── index.ts
│   ├── types.ts                      # Internal types
│   └── index.ts                      # Main exports
├── tests/
│   ├── discovery.test.ts
│   ├── loader.test.ts
│   └── registry.test.ts
├── package.json
├── tsconfig.json
└── README.md
```

**Key Files:**

`workspace-scanner.ts`

```typescript
/**
 * Scans the Turborepo workspace for analyzer packages
 */
export interface WorkspacePackage {
  name: string;
  path: string;
  packageJson: PackageJson;
  isAnalyzer: boolean;
}

export class WorkspaceScanner {
  /**
   * Find all packages in the workspace
   */
  async scanWorkspace(rootPath: string): Promise<WorkspacePackage[]>;

  /**
   * Filter packages that are analyzers
   */
  filterAnalyzers(packages: WorkspacePackage[]): WorkspacePackage[];

  /**
   * Read and parse pnpm-workspace.yaml
   */
  private async readWorkspaceConfig(rootPath: string): Promise<string[]>;

  /**
   * Resolve glob patterns to actual directories
   */
  private async resolveWorkspaceGlobs(rootPath: string, patterns: string[]): Promise<string[]>;
}
```

`plugin-validator.ts`

```typescript
/**
 * Validates that a package exports a valid plugin
 */
export interface ValidationResult {
  isValid: boolean;
  errors: string[];
  warnings: string[];
  pluginMetadata?: {
    id: string;
    name: string;
    version: string;
    entryPoint: string;
  };
}

export class PluginValidator {
  /**
   * Validate package structure
   */
  validatePackage(pkg: WorkspacePackage): ValidationResult;

  /**
   * Validate package.json exports field
   */
  private validateExports(packageJson: PackageJson): boolean;

  /**
   * Check for required plugin marker
   */
  private hasPluginMarker(packageJson: PackageJson): boolean;

  /**
   * Validate build artifacts exist
   */
  private validateBuildOutput(pkgPath: string): boolean;
}
```

`dynamic-loader.ts`

```typescript
/**
 * Handles dynamic import of plugin modules
 */
export interface LoadedPlugin {
  module: PluginModule;
  instance: ITechnologyPlugin;
  metadata: PluginMetadata;
}

export class DynamicLoader {
  /**
   * Load plugin from package
   */
  async loadPlugin(pkg: WorkspacePackage, config?: Record<string, unknown>): Promise<LoadedPlugin>;

  /**
   * Dynamic import with error handling
   */
  private async importPluginModule(entryPoint: string): Promise<PluginModule>;

  /**
   * Instantiate plugin from module exports
   */
  private instantiatePlugin(
    module: PluginModule,
    config?: Record<string, unknown>
  ): ITechnologyPlugin;

  /**
   * Validate plugin implements ITechnologyPlugin
   */
  private validatePluginInterface(plugin: unknown): plugin is ITechnologyPlugin;
}
```

`plugin-registry.ts`

```typescript
/**
 * Central registry for loaded plugins
 */
export class PluginRegistry {
  private plugins: Map<string, PluginRegistration>;
  private pluginOrder: string[];

  /**
   * Register a plugin
   */
  register(plugin: ITechnologyPlugin, options?: PluginRegistrationOptions): void;

  /**
   * Unregister a plugin
   */
  async unregister(pluginId: string): Promise<void>;

  /**
   * Get plugin by ID
   */
  get(pluginId: string): ITechnologyPlugin | undefined;

  /**
   * Get all registered plugins
   */
  getAll(): ITechnologyPlugin[];

  /**
   * Get enabled plugins only
   */
  getEnabled(): ITechnologyPlugin[];

  /**
   * Get plugins in priority order
   */
  getOrdered(): ITechnologyPlugin[];

  /**
   * Check if plugin is registered
   */
  has(pluginId: string): boolean;

  /**
   * Enable/disable plugin
   */
  setEnabled(pluginId: string, enabled: boolean): void;

  /**
   * Update plugin configuration
   */
  updateConfig(pluginId: string, config: Record<string, unknown>): void;

  /**
   * Clear all plugins
   */
  clear(): void;
}
```

**Package.json:**

```json
{
  "name": "@vipr/plugin-loader",
  "version": "1.0.0",
  "private": true,
  "description": "Plugin discovery and loading system for Vipr",
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "default": "./dist/index.js"
    },
    "./discovery": {
      "types": "./dist/discovery/index.d.ts",
      "default": "./dist/discovery/index.js"
    },
    "./loader": {
      "types": "./dist/loader/index.d.ts",
      "default": "./dist/loader/index.js"
    },
    "./registry": {
      "types": "./dist/registry/index.d.ts",
      "default": "./dist/registry/index.js"
    }
  },
  "scripts": {
    "build": "tsc",
    "clean": "rm -rf dist",
    "typecheck": "tsc --noEmit",
    "test": "vitest --run"
  },
  "dependencies": {
    "@vipr/types": "workspace:*",
    "@vipr/logging": "workspace:*",
    "fast-glob": "^3.3.3",
    "js-yaml": "^4.1.1"
  },
  "devDependencies": {
    "@vipr/tsconfig": "workspace:*",
    "@types/js-yaml": "^4.0.5",
    "@types/node": "^20.10.0",
    "typescript": "^5.3.0",
    "vitest": "^1.0.0"
  }
}
```

### common/analyzer-template

```
common/analyzer-template/
├── template/                         # Template files for scaffolding
│   ├── src/
│   │   ├── plugin.ts.template        # Main plugin class template
│   │   ├── analyses/
│   │   │   ├── anti-patterns.ts.template
│   │   │   ├── performance.ts.template
│   │   │   └── index.ts.template
│   │   ├── types.ts.template         # Type definitions template
│   │   ├── constants.ts.template     # Constants template
│   │   ├── utils.ts.template         # Utility functions template
│   │   └── index.ts.template         # Package entry point template
│   ├── tests/
│   │   ├── plugin.test.ts.template
│   │   └── setup.ts.template
│   ├── package.json.template
│   ├── tsconfig.json.template
│   ├── README.md.template
│   └── .gitignore.template
├── scripts/
│   ├── create-analyzer.ts            # CLI wizard for generation
│   ├── template-engine.ts            # Template variable substitution
│   ├── file-generator.ts             # Generate files from templates
│   └── validator.ts                  # Validate user input
├── src/
│   ├── index.ts                      # Programmatic API
│   └── types.ts
├── bin/
│   └── create-analyzer.js            # CLI entry point
├── package.json
├── tsconfig.json
└── README.md
```

**Key Files:**

`create-analyzer.ts`

```typescript
/**
 * Interactive CLI wizard for creating new analyzers
 */
import { prompt } from 'enquirer';

interface AnalyzerConfig {
  name: string; // e.g., "vue"
  displayName: string; // e.g., "Vue Analyzer"
  description: string;
  author: string;
  filePatterns: string[]; // e.g., ["**/*.vue"]
  analyses: string[]; // e.g., ["anti-patterns", "performance"]
}

export async function runWizard(): Promise<AnalyzerConfig> {
  // Prompt for analyzer details
  const answers = await prompt([
    {
      type: 'input',
      name: 'name',
      message: 'Analyzer ID (lowercase, e.g., "vue"):',
      validate: (value: string) => /^[a-z][a-z0-9-]*$/.test(value),
    },
    {
      type: 'input',
      name: 'displayName',
      message: 'Display name (e.g., "Vue Analyzer"):',
    },
    {
      type: 'input',
      name: 'description',
      message: 'Description:',
    },
    {
      type: 'input',
      name: 'author',
      message: 'Author name:',
    },
    {
      type: 'list',
      name: 'filePatterns',
      message: 'File patterns (comma-separated, e.g., "**/*.vue,**/*.ts"):',
    },
    {
      type: 'multiselect',
      name: 'analyses',
      message: 'Select analyses to include:',
      choices: [
        'anti-patterns',
        'performance',
        'migrations',
        'technical-debt',
        'security',
        'accessibility',
      ],
    },
  ]);

  return answers as AnalyzerConfig;
}

export async function createAnalyzer(config: AnalyzerConfig): Promise<void> {
  // Generate files from templates
  // Create directory structure
  // Install dependencies
  // Run initial build
}
```

`template-engine.ts`

```typescript
/**
 * Template variable substitution engine
 */
export interface TemplateContext {
  name: string;
  displayName: string;
  description: string;
  author: string;
  filePatterns: string[];
  analyses: string[];
  packageName: string; // @vipr/vue
  className: string; // VueAnalyzerPlugin
  year: string; // Current year for copyright
}

export class TemplateEngine {
  /**
   * Process template file with context
   */
  processTemplate(templateContent: string, context: TemplateContext): string;

  /**
   * Replace template variables
   */
  private replaceVariables(content: string, context: TemplateContext): string;

  /**
   * Handle conditional blocks
   */
  private processConditionals(content: string, context: TemplateContext): string;

  /**
   * Process loops (e.g., for analyses)
   */
  private processLoops(content: string, context: TemplateContext): string;
}
```

**Package.json:**

```json
{
  "name": "@vipr/analyzer-template",
  "version": "1.0.0",
  "private": true,
  "description": "Template and scaffolding tools for creating Vipr analyzer plugins",
  "bin": {
    "create-vipr-analyzer": "./bin/create-analyzer.js"
  },
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "scripts": {
    "build": "tsc",
    "clean": "rm -rf dist",
    "typecheck": "tsc --noEmit",
    "test": "vitest --run"
  },
  "dependencies": {
    "@vipr/types": "workspace:*",
    "enquirer": "^2.4.1",
    "chalk": "^5.3.0",
    "fs-extra": "^11.2.0"
  },
  "devDependencies": {
    "@vipr/tsconfig": "workspace:*",
    "@types/fs-extra": "^11.0.4",
    "@types/node": "^20.10.0",
    "typescript": "^5.3.0",
    "vitest": "^1.0.0"
  }
}
```

## Discovery Strategy

### Workspace Package Discovery

The plugin discovery system uses a multi-stage process tailored for Turborepo/pnpm workspaces:

**Stage 1: Workspace Configuration Parsing**

```typescript
// Read pnpm-workspace.yaml
const workspaceConfig = await readWorkspaceConfig(rootPath);
// Example: ['common/*', 'analyzers/*', 'clients/*']

// Filter to analyzer patterns
const analyzerPatterns = workspaceConfig.filter(pattern => pattern.startsWith('analyzers/'));
// Result: ['analyzers/*']
```

**Stage 2: Glob Resolution**

```typescript
// Use fast-glob to resolve patterns
const analyzerPaths = await glob(analyzerPatterns, {
  cwd: rootPath,
  onlyDirectories: true,
  absolute: true,
});
// Result: [
//   '/path/to/project/analyzers/core',
//   '/path/to/project/analyzers/react'
// ]
```

**Stage 3: Package Metadata Extraction**

```typescript
// Read package.json for each analyzer
const packages: WorkspacePackage[] = [];
for (const pkgPath of analyzerPaths) {
  const packageJson = await readPackageJson(pkgPath);
  packages.push({
    name: packageJson.name,
    path: pkgPath,
    packageJson,
    isAnalyzer: hasPluginMarker(packageJson),
  });
}
```

**Stage 4: Plugin Marker Detection**

```typescript
// Check for plugin marker in package.json
function hasPluginMarker(packageJson: PackageJson): boolean {
  // Option 1: Custom field
  if (packageJson.vipr?.plugin === true) return true;

  // Option 2: Keywords
  if (packageJson.keywords?.includes('vipr-plugin')) return true;

  // Option 3: Naming convention
  if (packageJson.name?.includes('analyzer')) return true;

  return false;
}
```

### Plugin Entry Point Discovery

**Export Field Analysis**

```typescript
// Prefer explicit plugin export
if (packageJson.exports?.['./plugin']) {
  entryPoint = resolveExport(packageJson.exports['./plugin']);
}
// Fallback to main export
else if (packageJson.exports?.['.']) {
  entryPoint = resolveExport(packageJson.exports['.']);
}
// Fallback to main field
else if (packageJson.main) {
  entryPoint = path.join(pkgPath, packageJson.main);
}
```

**Module Resolution**

```typescript
// Example package.json exports:
{
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "default": "./dist/index.js"
    },
    "./plugin": {
      "types": "./dist/plugin.d.ts",
      "default": "./dist/plugin.js"
    }
  }
}

// Resolution prefers './plugin' if available
const pluginPath = resolvePackageExport('@vipr/react', './plugin');
// Result: '/path/to/analyzers/react/dist/plugin.js'
```

## Loading Strategy

### Dynamic Import Pattern

**Module Import with Error Handling**

```typescript
async function importPluginModule(entryPoint: string): Promise<PluginModule> {
  try {
    // Dynamic import using file:// protocol for ESM compatibility
    const module = await import(pathToFileURL(entryPoint).href);

    // Validate module structure
    if (!module.default && !module.plugin && !module.createPlugin) {
      throw new PluginLoadError(`Module ${entryPoint} does not export a plugin`, 'INVALID_EXPORTS');
    }

    return module as PluginModule;
  } catch (error) {
    if (error instanceof PluginLoadError) throw error;

    throw new PluginLoadError(
      `Failed to import plugin from ${entryPoint}: ${error.message}`,
      'IMPORT_FAILED',
      { cause: error }
    );
  }
}
```

**Plugin Instantiation**

```typescript
function instantiatePlugin(
  module: PluginModule,
  config?: Record<string, unknown>
): ITechnologyPlugin {
  let plugin: unknown;

  // Try factory function first
  if (typeof module.createPlugin === 'function') {
    plugin = module.createPlugin(config);
  }
  // Try default export
  else if (module.default) {
    if (typeof module.default === 'function') {
      plugin = module.default(config);
    } else {
      plugin = module.default;
    }
  }
  // Try named plugin export
  else if (module.plugin) {
    plugin = module.plugin;
  }

  // Validate interface
  if (!validatePluginInterface(plugin)) {
    throw new PluginLoadError(
      'Plugin does not implement ITechnologyPlugin interface',
      'INVALID_INTERFACE'
    );
  }

  return plugin;
}
```

### Plugin Validation

**Interface Validation**

```typescript
function validatePluginInterface(plugin: unknown): plugin is ITechnologyPlugin {
  if (!plugin || typeof plugin !== 'object') return false;

  const required = ['id', 'name', 'version', 'filePatterns', 'canHandle', 'analyze', 'getRules'];

  for (const prop of required) {
    if (!(prop in plugin)) {
      logger.error(`Plugin missing required property: ${prop}`);
      return false;
    }
  }

  // Validate property types
  const p = plugin as Partial<ITechnologyPlugin>;

  if (typeof p.id !== 'string') return false;
  if (typeof p.name !== 'string') return false;
  if (typeof p.version !== 'string') return false;
  if (!Array.isArray(p.filePatterns)) return false;
  if (typeof p.canHandle !== 'function') return false;
  if (typeof p.analyze !== 'function') return false;
  if (typeof p.getRules !== 'function') return false;

  return true;
}
```

**Lifecycle Hook Execution**

```typescript
async function registerPlugin(
  plugin: ITechnologyPlugin,
  options?: PluginRegistrationOptions
): Promise<void> {
  // Validate configuration if plugin provides validator
  if (plugin.validateConfig && options?.config) {
    const validation = plugin.validateConfig(options.config);
    if (validation !== true) {
      throw new PluginLoadError(`Invalid configuration: ${validation}`, 'INVALID_CONFIG');
    }
  }

  // Call onRegister lifecycle hook
  if (plugin.onRegister) {
    try {
      await plugin.onRegister();
    } catch (error) {
      throw new PluginLoadError(
        `Plugin registration failed: ${error.message}`,
        'REGISTRATION_FAILED',
        { cause: error }
      );
    }
  }

  // Store in registry
  registry.register(plugin, options);

  logger.info(`Registered plugin: ${plugin.name} (${plugin.id}@${plugin.version})`);
}
```

### Error Handling Strategy

**Custom Error Classes**

```typescript
export class PluginLoadError extends Error {
  constructor(
    message: string,
    public code: PluginErrorCode,
    public context?: { cause?: Error; [key: string]: unknown }
  ) {
    super(message);
    this.name = 'PluginLoadError';
  }
}

export type PluginErrorCode =
  | 'PACKAGE_NOT_FOUND'
  | 'INVALID_PACKAGE_JSON'
  | 'MISSING_BUILD_OUTPUT'
  | 'IMPORT_FAILED'
  | 'INVALID_EXPORTS'
  | 'INVALID_INTERFACE'
  | 'INVALID_CONFIG'
  | 'REGISTRATION_FAILED'
  | 'LIFECYCLE_FAILED';
```

**Graceful Degradation**

```typescript
async function loadAllPlugins(packages: WorkspacePackage[]): Promise<PluginLoadResult> {
  const loaded: LoadedPlugin[] = [];
  const failed: PluginLoadFailure[] = [];

  for (const pkg of packages) {
    try {
      const plugin = await loadPlugin(pkg);
      loaded.push(plugin);
    } catch (error) {
      // Log error but continue loading other plugins
      logger.warn(`Failed to load plugin ${pkg.name}:`, error);
      failed.push({
        package: pkg,
        error: error as PluginLoadError,
      });
    }
  }

  return { loaded, failed };
}
```

## Template System

### Template Structure

**Variable Substitution**

```typescript
// Template uses {{variable}} syntax
// Example: plugin.ts.template
export class {{className}} implements ITechnologyPlugin {
  readonly id = '{{name}}';
  readonly name = '{{displayName}}';
  readonly version = '1.0.0';
  readonly filePatterns = [{{filePatterns}}];

  // ...
}
```

**Conditional Blocks**

```typescript
// Use {{#if condition}} ... {{/if}} syntax
// Example: analyses/index.ts.template
{{#if hasAntiPatterns}}
export { analyzeAntiPatterns } from './anti-patterns';
{{/if}}
{{#if hasPerformance}}
export { analyzePerformance } from './performance';
{{/if}}
```

**Loop Blocks**

```typescript
// Use {{#each array}} ... {{/each}} syntax
// Example: package.json.template
{
  "keywords": [
    "vipr-plugin",
    {{#each analyses}}
    "{{this}}"{{#unless @last}},{{/unless}}
    {{/each}}
  ]
}
```

### CLI Wizard Design

**User Experience Flow**

```
$ pnpm create-vipr-analyzer

   Welcome to Vipr Analyzer Generator

   This wizard will help you create a new analyzer plugin.

   ? Analyzer ID (lowercase): vue
   ? Display name: Vue Analyzer
   ? Description: Vue.js specific code analysis
   ? Author name: Your Name
   ? File patterns: **/*.vue
   ? Select analyses to include:
     ◉ anti-patterns
     ◉ performance
     ◯ migrations
     ◯ technical-debt
     ◯ security
     ◯ accessibility

   Creating analyzer...
   ✓ Created directory: analyzers/vue
   ✓ Generated package.json
   ✓ Generated TypeScript configuration
   ✓ Generated plugin class
   ✓ Generated analysis templates
   ✓ Generated tests
   ✓ Generated README

   Next steps:
   1. cd analyzers/vue
   2. pnpm install
   3. pnpm build
   4. Implement your analysis logic in src/analyses/

   Happy coding!
```

**Validation Rules**

```typescript
const validators = {
  name: (value: string) => {
    if (!/^[a-z][a-z0-9-]*$/.test(value)) {
      return 'Name must be lowercase alphanumeric with hyphens';
    }
    if (fs.existsSync(`analyzers/${value}`)) {
      return `Analyzer "${value}" already exists`;
    }
    return true;
  },

  displayName: (value: string) => {
    if (value.length < 3) {
      return 'Display name must be at least 3 characters';
    }
    return true;
  },

  filePatterns: (value: string) => {
    const patterns = value.split(',').map(p => p.trim());
    for (const pattern of patterns) {
      if (!pattern.includes('*')) {
        return 'File patterns should include wildcards (e.g., **/*.vue)';
      }
    }
    return true;
  },
};
```

## Performance Considerations

### Lazy Plugin Loading

**On-Demand Import**

```typescript
class LazyPluginLoader {
  private loadedPlugins = new Map<string, Promise<ITechnologyPlugin>>();

  async loadPlugin(id: string): Promise<ITechnologyPlugin> {
    // Return cached promise if already loading/loaded
    if (this.loadedPlugins.has(id)) {
      return this.loadedPlugins.get(id)!;
    }

    // Create loading promise
    const loadPromise = this.doLoadPlugin(id);
    this.loadedPlugins.set(id, loadPromise);

    return loadPromise;
  }

  private async doLoadPlugin(id: string): Promise<ITechnologyPlugin> {
    // Actual loading logic
  }
}
```

### Parallel Discovery

**Concurrent Package Scanning**

```typescript
async function scanWorkspace(rootPath: string): Promise<WorkspacePackage[]> {
  const patterns = await readWorkspaceConfig(rootPath);
  const paths = await glob(
    patterns.filter(p => p.startsWith('analyzers/')),
    {
      cwd: rootPath,
      onlyDirectories: true,
    }
  );

  // Read all package.json files in parallel
  const packages = await Promise.all(
    paths.map(async pkgPath => {
      const packageJson = await readPackageJson(pkgPath);
      return {
        name: packageJson.name,
        path: pkgPath,
        packageJson,
        isAnalyzer: hasPluginMarker(packageJson),
      };
    })
  );

  return packages.filter(pkg => pkg.isAnalyzer);
}
```

### Build Output Caching

**Verify Build Artifacts**

```typescript
function validateBuildOutput(pkgPath: string): boolean {
  const distPath = path.join(pkgPath, 'dist');

  // Check if dist directory exists
  if (!fs.existsSync(distPath)) {
    logger.warn(`Missing build output for ${pkgPath}`);
    return false;
  }

  // Check for index.js
  const indexPath = path.join(distPath, 'index.js');
  if (!fs.existsSync(indexPath)) {
    logger.warn(`Missing index.js in ${distPath}`);
    return false;
  }

  return true;
}
```

## File Changes

### Files to Create

**common/plugin-loader/src/discovery/workspace-scanner.ts**

- Scan pnpm-workspace.yaml for package patterns
- Resolve globs to actual package directories
- Read package.json for each package
- Filter packages that are analyzers

**common/plugin-loader/src/discovery/package-analyzer.ts**

- Extract plugin metadata from package.json
- Determine plugin entry point
- Validate package structure

**common/plugin-loader/src/discovery/plugin-validator.ts**

- Validate plugin marker presence
- Check build output exists
- Validate exports configuration

**common/plugin-loader/src/loader/dynamic-loader.ts**

- Dynamic import of plugin modules
- Handle ESM/CJS compatibility
- Error handling for import failures

**common/plugin-loader/src/loader/plugin-instantiator.ts**

- Instantiate plugins from modules
- Handle factory functions vs direct instances
- Validate plugin interface implementation

**common/plugin-loader/src/loader/error-handler.ts**

- Custom error classes for plugin loading
- Error context and metadata
- User-friendly error messages

**common/plugin-loader/src/registry/plugin-registry.ts**

- Map-based plugin storage
- Priority ordering
- Enable/disable functionality
- Configuration management

**common/plugin-loader/src/registry/registry-types.ts**

- PluginRegistration interface
- Registry-specific type definitions

**common/plugin-loader/src/types.ts**

- WorkspacePackage interface
- LoadedPlugin interface
- PluginLoadResult interface
- Internal type definitions

**common/plugin-loader/src/index.ts**

- Main entry point
- Export public API
- Re-export types

**common/analyzer-template/template/src/plugin.ts.template**

- Template for main plugin class
- Variable substitution markers
- Conditional blocks for optional features

**common/analyzer-template/template/src/analyses/\*.ts.template**

- Templates for each analysis type
- Example implementations
- Documentation comments

**common/analyzer-template/template/package.json.template**

- Package metadata template
- Workspace dependencies
- Scripts configuration

**common/analyzer-template/scripts/create-analyzer.ts**

- Interactive CLI wizard
- User input collection
- Validation logic
- File generation orchestration

**common/analyzer-template/scripts/template-engine.ts**

- Variable substitution
- Conditional processing
- Loop processing
- Template file reading

**common/analyzer-template/scripts/file-generator.ts**

- Write files from processed templates
- Create directory structure
- Copy static assets
- Set file permissions

### Files to Modify

None (this phase only creates new packages)

## Dependencies

### Required from Phase 1

- `@vipr/types` package with updated interfaces:
  - `ITechnologyPlugin` interface (must exist)
  - `PluginResult` type
  - `PluginInsight` type
  - `PluginModule` type
  - `PluginFactory` type
  - `PluginRegistration` type

### External Dependencies

**common/plugin-loader**

- `fast-glob` - For workspace pattern resolution
- `js-yaml` - For parsing pnpm-workspace.yaml
- `@vipr/logging` - For logging
- `@vipr/types` - For type definitions

**common/analyzer-template**

- `enquirer` - For interactive CLI prompts
- `chalk` - For terminal colors
- `fs-extra` - For file operations
- `@vipr/types` - For type definitions

## Acceptance Criteria

### Plugin Discovery

- [ ] Successfully reads and parses `pnpm-workspace.yaml`
- [ ] Resolves glob patterns to actual analyzer directories
- [ ] Reads and validates package.json for each analyzer
- [ ] Correctly identifies analyzer packages via marker detection
- [ ] Handles missing or malformed workspace configuration gracefully
- [ ] Logs appropriate warnings for invalid packages

### Plugin Loading

- [ ] Dynamically imports plugin modules using ES modules
- [ ] Handles both factory functions and direct plugin instances
- [ ] Validates plugin implements ITechnologyPlugin interface
- [ ] Calls onRegister lifecycle hook successfully
- [ ] Handles plugin loading failures gracefully
- [ ] Provides detailed error messages for loading failures
- [ ] Supports both CommonJS and ES module plugins

### Plugin Registry

- [ ] Stores plugins in map-based registry
- [ ] Retrieves plugins by ID
- [ ] Returns all enabled plugins
- [ ] Orders plugins by priority
- [ ] Enables/disables plugins dynamically
- [ ] Updates plugin configuration
- [ ] Calls onUnregister when plugins are removed
- [ ] Thread-safe concurrent access

### Template System

- [ ] CLI wizard collects all required information
- [ ] Validates user input (name, patterns, etc.)
- [ ] Generates directory structure correctly
- [ ] Creates package.json with workspace dependencies
- [ ] Generates TypeScript configuration
- [ ] Creates plugin class from template
- [ ] Generates selected analysis templates
- [ ] Creates test files and setup
- [ ] Generated code compiles without errors
- [ ] Generated package builds successfully

### Error Handling

- [ ] Custom error classes with error codes
- [ ] Detailed error context and metadata
- [ ] User-friendly error messages
- [ ] Continues loading other plugins on individual failures
- [ ] Logs all errors appropriately

### Performance

- [ ] Parallel package scanning
- [ ] Lazy plugin loading (on-demand)
- [ ] Cached plugin instances
- [ ] Discovery completes in under 100ms for 10 analyzers
- [ ] Plugin loading completes in under 50ms per plugin

### Testing

- [ ] Unit tests for workspace scanner
- [ ] Unit tests for plugin validator
- [ ] Unit tests for dynamic loader
- [ ] Unit tests for plugin registry
- [ ] Integration tests for end-to-end discovery
- [ ] Tests for error scenarios
- [ ] Tests for template generation
- [ ] 80% code coverage minimum

## Recommended Claude Model

**Claude Sonnet 4.5** is recommended for this phase because:

1. **Strong TypeScript Expertise**: This phase requires deep understanding of TypeScript module systems, dynamic imports, and type safety
2. **System Architecture**: Requires designing robust, extensible plugin infrastructure
3. **Error Handling**: Needs sophisticated error handling strategies and graceful degradation
4. **Package Management**: Requires understanding of pnpm workspaces, Turborepo, and monorepo patterns
5. **File I/O Operations**: Heavy file system operations for discovery and template generation
6. **Template Processing**: Complex template engine implementation with conditionals and loops

Claude Sonnet 4.5 provides the right balance of speed and capability for this infrastructure-heavy phase.

## Assigned Subagents

### node-package-ngineer (Primary)

**Responsibilities:**

- Design workspace scanning strategy for pnpm/Turborepo
- Implement package.json parsing and validation
- Handle dependency resolution patterns
- Design template package structure
- Implement package generation logic

**Rationale:** This phase is fundamentally about package management, workspace resolution, and monorepo patterns.

### typeScript-engineer

**Responsibilities:**

- Design plugin interface validation
- Implement type-safe dynamic imports
- Ensure end-to-end type safety
- Design template type definitions

**Rationale:** Strong TypeScript expertise needed for module system and type validation.

### vitest-engineer

**Responsibilities:**

- Design test strategy for plugin loading
- Implement integration tests for discovery
- Test error scenarios comprehensively
- Validate template generation

**Rationale:** Plugin loading is complex and needs comprehensive testing strategy.

## Notes

### Turborepo-Specific Considerations

1. **Build Dependency Graph**: Plugins must be built before they can be loaded. Ensure Turborepo build order is correct.

2. **Workspace Protocol**: Use `workspace:*` for all internal dependencies in generated packages.

3. **Parallel Builds**: Leverage Turborepo's parallel building for faster plugin compilation.

4. **Caching**: Turborepo caching means plugin builds are fast on subsequent runs.

### Plugin Marker Strategies

**Recommendation: Multiple Markers**
Use a combination of markers for maximum flexibility:

```json
{
  "keywords": ["vipr-plugin"],
  "vipr": {
    "plugin": true,
    "entryPoint": "./dist/plugin.js"
  }
}
```

This allows:

- Simple keyword-based discovery
- Explicit plugin metadata
- Custom entry point specification

### Import Strategy

**Use File URLs for ESM**

```typescript
import { pathToFileURL } from 'node:url';

const module = await import(pathToFileURL(pluginPath).href);
```

This ensures compatibility across different module systems and platforms.

### Template Engine Choice

Consider using a simple custom template engine rather than full templating libraries:

- Simpler dependency tree
- Faster execution
- Easier to maintain
- Full control over syntax

Basic implementation:

```typescript
function processTemplate(template: string, context: Record<string, unknown>): string {
  return template.replace(/\{\{(\w+)\}\}/g, (match, key) => {
    return String(context[key] ?? match);
  });
}
```

### Future Extensibility

Design the plugin loader to support:

- Remote plugin loading (future)
- Plugin versioning and updates
- Plugin dependencies on other plugins
- Hot-reloading during development
- Plugin sandboxing/isolation
