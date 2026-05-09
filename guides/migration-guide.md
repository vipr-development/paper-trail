# Migration Guide: v2 to v3

## Overview

This guide helps you migrate from Vipr v2 to v3. The v3 architecture introduces a plugin-based system with discrete analysis units, improved performance, and better extensibility.

## Key Changes

### Architecture Changes

**v2**: Monolithic analyzer with hardcoded analysis logic
**v3**: Plugin-based architecture with discrete `IAnalysis` units

### API Changes

#### Before (v2)

```typescript
import { analyzeReactComplexity } from '@vipr/react';

const result = analyzeReactComplexity(code);
console.log(result.score);
```

#### After (v3)

```typescript
import { AnalysisEngine } from '@vipr/core';
import { ReactAnalyzerPlugin } from '@vipr/react';

const engine = new AnalysisEngine();
engine.registerPlugin(new ReactAnalyzerPlugin());

const result = await engine.analyzeFile('Component.tsx');
console.log(result.overallScore);
```

### Plugin Registration

**v2**: Analyzers were imported and used directly
**v3**: Plugins must be registered with the AnalysisEngine

```typescript
// v2
import { ReactAnalyzer } from '@vipr/react';
const analyzer = new ReactAnalyzer();

// v3
import { AnalysisEngine } from '@vipr/core';
import { ReactAnalyzerPlugin } from '@vipr/react';

const engine = new AnalysisEngine();
engine.registerPlugin(new ReactAnalyzerPlugin());
```

### Result Structure

**v2**: Single result object with all metrics
**v3**: Aggregated result with plugin breakdown

```typescript
// v2
interface Result {
  score: number;
  structural: StructuralMetrics;
  hooks: HookMetrics;
  // ...
}

// v3
interface AggregatedResult {
  overallScore: number;
  overallGrade: string;
  pluginResults: Map<string, PluginResult>;
  insights: ComplexityInsight[];
}
```

## Migration Steps

### Step 1: Update Dependencies

```json
{
  "dependencies": {
    "@vipr/core": "^3.0.0",
    "@vipr/react": "^3.0.0",
    "@vipr/types": "^3.0.0"
  }
}
```

### Step 2: Update Imports

Replace direct analyzer imports with plugin imports:

```typescript
// Before
import { ReactAnalyzer } from '@vipr/react';

// After
import { ReactAnalyzerPlugin } from '@vipr/react';
import { AnalysisEngine } from '@vipr/core';
```

### Step 3: Update Analysis Calls

```typescript
// Before
const analyzer = new ReactAnalyzer();
const result = analyzer.analyze(code);

// After
const engine = new AnalysisEngine();
engine.registerPlugin(new ReactAnalyzerPlugin());
const result = await engine.analyzeFile(filePath);
```

### Step 4: Update Result Access

```typescript
// Before
const score = result.score;
const structural = result.structural;

// After
const score = result.overallScore;
const reactResult = result.pluginResults.get('react');
const structural = reactResult?.metrics.structural;
```

## Configuration Changes

### Analyzer Config

**v2**:

```typescript
const config = {
  enableCache: true,
  thresholds: {
    /* ... */
  },
};
```

**v3**:

```typescript
const config = {
  enableCache: true,
  analyzerConfig: {
    analyses: {
      'react-structural': { enabled: true },
      'react-hooks': { enabled: true },
    },
  },
};
```

## Breaking Changes

### Removed APIs

- `analyzeReactComplexity()` - Use `AnalysisEngine` instead
- Direct analyzer classes - Use plugins instead
- `AnalyzerResult` - Use `AggregatedResult` instead

### Changed Types

- `Result` → `AggregatedResult`
- `AnalyzerConfig` → Extended with `analyses` property
- Plugin interfaces updated

## Troubleshooting

### Common Issues

**Issue**: "Cannot find module '@vipr/react'"
**Solution**: Ensure you're using v3 packages

**Issue**: "Plugin not registered"
**Solution**: Call `engine.registerPlugin()` before analyzing

**Issue**: "Result structure changed"
**Solution**: Access results via `pluginResults` map

## Need Help?

- Check the [Plugin Development Guide](./plugin-development.md)
- Review [Architecture Documentation](../architecture/plugin-architecture.md)
- Open an issue on GitHub
