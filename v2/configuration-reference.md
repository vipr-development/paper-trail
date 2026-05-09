# Configuration Reference

Complete API reference for Vipr configuration options.

## Table of Contents

- [Root Configuration](#root-configuration)
- [Global Configuration](#global-configuration)
- [Plugin Configuration](#plugin-configuration)
- [Output Configuration](#output-configuration)
- [CI Configuration](#ci-configuration)
- [File Filtering](#file-filtering)
- [Environment Configuration](#environment-configuration)

## Root Configuration

Top-level configuration properties.

### `$schema`

**Type:** `string`
**Optional**

Path or URL to the JSON schema for IDE autocomplete and validation.

```json
{
  "$schema": "https://vipr.dev/schemas/vipr-config.schema.json"
}
```

### `extends`

**Type:** `string | string[]`
**Optional**

Path(s) to base configuration file(s) to extend from. Configs are merged left-to-right.

```json
{
  "extends": "./base-config.json"
}
```

```json
{
  "extends": ["./team-standards.json", "./react-defaults.json"]
}
```

Relative paths are resolved relative to the config file location.

## Global Configuration

System-wide settings under the `global` property.

### `global.debug`

**Type:** `boolean`
**Default:** `false`

Enable debug-level logging for troubleshooting.

```json
{
  "global": {
    "debug": true
  }
}
```

Equivalent CLI flag: `--debug`

### `global.cache`

**Type:** `boolean`
**Default:** `true`

Enable caching of analysis results for faster subsequent runs.

```json
{
  "global": {
    "cache": false
  }
}
```

Equivalent CLI flag: `--no-cache`

### `global.gradeBoundaries`

**Type:** `object`
**Default:** `{ A: 25, B: 45, C: 65, D: 80 }`

Score thresholds for letter grades. Lower scores are better.

```json
{
  "global": {
    "gradeBoundaries": {
      "A": 20,
      "B": 40,
      "C": 60,
      "D": 75
    }
  }
}
```

#### Properties

| Property | Type     | Range | Default | Description               |
| -------- | -------- | ----- | ------- | ------------------------- |
| `A`      | `number` | 0-100 | 25      | Maximum score for grade A |
| `B`      | `number` | 0-100 | 45      | Maximum score for grade B |
| `C`      | `number` | 0-100 | 65      | Maximum score for grade C |
| `D`      | `number` | 0-100 | 80      | Maximum score for grade D |

Scores above `D` threshold receive grade F.

## Plugin Configuration

Plugin-specific configuration under the `plugins` property.

### React Plugin (`plugins.react`)

Configuration for the React analyzer plugin.

#### `plugins.react.thresholds`

Thresholds for React-specific metrics.

##### `plugins.react.thresholds.structural`

Structural complexity thresholds.

```json
{
  "plugins": {
    "react": {
      "thresholds": {
        "structural": {
          "warning": 15,
          "critical": 25
        }
      }
    }
  }
}
```

| Property   | Type     | Default | Description        |
| ---------- | -------- | ------- | ------------------ |
| `warning`  | `number` | 15      | Warning threshold  |
| `critical` | `number` | -       | Critical threshold |

##### `plugins.react.thresholds.hooks`

React Hooks complexity thresholds.

```json
{
  "plugins": {
    "react": {
      "thresholds": {
        "hooks": {
          "total": 10,
          "customHookSuggestion": 8,
          "stateCountWarning": 3,
          "effectCountWarning": 2,
          "callbackCountInfo": 3
        }
      }
    }
  }
}
```

| Property               | Type     | Default | Description                      |
| ---------------------- | -------- | ------- | -------------------------------- |
| `total`                | `number` | 10      | Max total hooks in component     |
| `customHookSuggestion` | `number` | 8       | Threshold to suggest custom hook |
| `stateCountWarning`    | `number` | 3       | Max useState calls               |
| `effectCountWarning`   | `number` | 2       | Max useEffect calls              |
| `callbackCountInfo`    | `number` | 3       | Info threshold for callbacks     |

##### `plugins.react.thresholds.temporal`

Temporal coupling thresholds for useEffect complexity.

```json
{
  "plugins": {
    "react": {
      "thresholds": {
        "temporal": {
          "dependencyWarning": 5,
          "highComplexityEffectCount": 3
        }
      }
    }
  }
}
```

| Property                    | Type     | Default | Description                 |
| --------------------------- | -------- | ------- | --------------------------- |
| `dependencyWarning`         | `number` | 5       | Max dependencies per effect |
| `highComplexityEffectCount` | `number` | 3       | Max high-complexity effects |

##### `plugins.react.thresholds.coupling`

Component coupling thresholds.

```json
{
  "plugins": {
    "react": {
      "thresholds": {
        "coupling": {
          "contextWarning": 3,
          "contextCritical": 5,
          "propsWarning": 10,
          "propsCritical": 15,
          "propsDrillingThreshold": 7
        }
      }
    }
  }
}
```

| Property                 | Type     | Default | Description                           |
| ------------------------ | -------- | ------- | ------------------------------------- |
| `contextWarning`         | `number` | 3       | Contexts before warning               |
| `contextCritical`        | `number` | 5       | Contexts before critical              |
| `propsWarning`           | `number` | 10      | Props before warning                  |
| `propsCritical`          | `number` | 15      | Props before critical                 |
| `propsDrillingThreshold` | `number` | 7       | Threshold for prop drilling detection |

##### `plugins.react.thresholds.dataflow`

Dataflow analysis thresholds.

```json
{
  "plugins": {
    "react": {
      "thresholds": {
        "dataflow": {
          "propDrillingDepthWarning": 3,
          "transformChainWarning": 3,
          "stateUpdatePathsWarning": 10
        }
      }
    }
  }
}
```

| Property                   | Type     | Default | Description                |
| -------------------------- | -------- | ------- | -------------------------- |
| `propDrillingDepthWarning` | `number` | 3       | Max prop drilling depth    |
| `transformChainWarning`    | `number` | 3       | Max transform chain length |
| `stateUpdatePathsWarning`  | `number` | 10      | Max state update paths     |

#### `plugins.react.weights`

Weights for score calculations. Must sum to 1.0.

##### `plugins.react.weights.complexity`

Component complexity weights.

```json
{
  "plugins": {
    "react": {
      "weights": {
        "complexity": {
          "structural": 0.2,
          "hooks": 0.25,
          "temporal": 0.25,
          "coupling": 0.15,
          "identity": 0.15
        }
      }
    }
  }
}
```

| Property     | Type     | Range | Default | Description                  |
| ------------ | -------- | ----- | ------- | ---------------------------- |
| `structural` | `number` | 0-1   | 0.2     | Structural complexity weight |
| `hooks`      | `number` | 0-1   | 0.25    | Hooks complexity weight      |
| `temporal`   | `number` | 0-1   | 0.25    | Temporal coupling weight     |
| `coupling`   | `number` | 0-1   | 0.15    | Component coupling weight    |
| `identity`   | `number` | 0-1   | 0.15    | Identity stability weight    |

**Important:** All weights must sum to exactly 1.0.

#### `plugins.react.analyses`

Enable or disable specific analyses.

```json
{
  "plugins": {
    "react": {
      "analyses": {
        "security": { "enabled": true },
        "accessibility": { "enabled": true },
        "performance": { "enabled": true },
        "reliability": { "enabled": true },
        "react-migration": { "enabled": false },
        "dataflow": { "enabled": true },
        "anti-pattern": { "enabled": true }
      }
    }
  }
}
```

| Analysis          | Default | Description                          |
| ----------------- | ------- | ------------------------------------ |
| `security`        | `true`  | Security vulnerability detection     |
| `accessibility`   | `true`  | Accessibility issue detection        |
| `performance`     | `true`  | Performance optimization suggestions |
| `reliability`     | `true`  | Reliability and error handling       |
| `react-migration` | `true`  | React version migration readiness    |
| `dataflow`        | `true`  | Data flow and state management       |
| `anti-pattern`    | `true`  | Anti-pattern detection               |

### Core Plugin (`plugins.core`)

Configuration for the core analyzer plugin.

#### `plugins.core.thresholds`

Thresholds for core complexity metrics.

##### `plugins.core.thresholds.cyclomaticComplexity`

Cyclomatic complexity thresholds.

```json
{
  "plugins": {
    "core": {
      "thresholds": {
        "cyclomaticComplexity": {
          "warning": 10,
          "critical": 20
        }
      }
    }
  }
}
```

| Property   | Type     | Default | Description        |
| ---------- | -------- | ------- | ------------------ |
| `warning`  | `number` | 10      | Warning threshold  |
| `critical` | `number` | 20      | Critical threshold |

##### `plugins.core.thresholds.halstead`

Halstead complexity thresholds.

```json
{
  "plugins": {
    "core": {
      "thresholds": {
        "halstead": {
          "difficulty": 30,
          "effort": 1000000
        }
      }
    }
  }
}
```

| Property     | Type     | Default | Description                   |
| ------------ | -------- | ------- | ----------------------------- |
| `difficulty` | `number` | 30      | Halstead difficulty threshold |
| `effort`     | `number` | 1000000 | Halstead effort threshold     |

##### `plugins.core.thresholds.maintainability`

Maintainability index thresholds.

```json
{
  "plugins": {
    "core": {
      "thresholds": {
        "maintainability": {
          "warning": 65,
          "critical": 40
        }
      }
    }
  }
}
```

| Property   | Type     | Default | Description                         |
| ---------- | -------- | ------- | ----------------------------------- |
| `warning`  | `number` | 65      | Warning threshold (lower is worse)  |
| `critical` | `number` | 40      | Critical threshold (lower is worse) |

## Output Configuration

Output formatting and display options under the `output` property.

### `output.format`

**Type:** `"cli" | "json" | "json-full" | "markdown"`
**Default:** `"cli"`

Output format for analysis results.

```json
{
  "output": {
    "format": "json"
  }
}
```

Equivalent CLI flag: `--format <type>`

| Format      | Description                                 |
| ----------- | ------------------------------------------- |
| `cli`       | Colored console output with visual elements |
| `json`      | Compact JSON with core metrics              |
| `json-full` | Full JSON with all analysis details         |
| `markdown`  | Markdown format for documentation           |

### `output.minSeverity`

**Type:** `"info" | "warning" | "critical"`
**Default:** `"info"`

Minimum severity level to display in results.

```json
{
  "output": {
    "minSeverity": "warning"
  }
}
```

Equivalent CLI flag: `--min-severity <level>`

### `output.compact`

**Type:** `boolean`
**Default:** `false`

Use compact output format (applies to JSON formats only).

```json
{
  "output": {
    "compact": true
  }
}
```

Equivalent CLI flag: `--compact`

### `output.failThreshold`

**Type:** `number`
**Range:** 0-100
**Default:** `0` (disabled)

Exit with error code 1 if overall score is below this threshold.

```json
{
  "output": {
    "failThreshold": 70
  }
}
```

Equivalent CLI flag: `--fail-threshold <score>`

### `output.failOnCritical`

**Type:** `boolean`
**Default:** `false`

Exit with error code 1 if any critical severity issues are found.

```json
{
  "output": {
    "failOnCritical": true
  }
}
```

Equivalent CLI flag: `--fail-on-critical`

## CI Configuration

CI/CD specific settings under the `ci` property.

### `ci.failThreshold`

**Type:** `number`
**Range:** 0-100
**Optional**

Score threshold for CI failure. Overrides `output.failThreshold` in CI environments.

```json
{
  "ci": {
    "failThreshold": 70
  }
}
```

### `ci.failOnCritical`

**Type:** `boolean`
**Default:** `false`

Fail CI build if critical issues are found. Overrides `output.failOnCritical` in CI environments.

```json
{
  "ci": {
    "failOnCritical": true
  }
}
```

## File Filtering

Control which files are included or excluded from analysis.

### `include`

**Type:** `string[]`
**Default:** `["**/*.{ts,tsx,js,jsx}"]`

Glob patterns for files to include in analysis.

```json
{
  "include": ["src/**/*.{ts,tsx}", "components/**/*.tsx"]
}
```

### `exclude`

**Type:** `string[]`
**Default:** `["**/node_modules/**", "**/dist/**", "**/build/**"]`

Glob patterns for files to exclude from analysis.

```json
{
  "exclude": ["**/node_modules/**", "**/dist/**", "**/build/**", "**/*.test.ts", "**/__tests__/**"]
}
```

## Environment Configuration

Environment-specific configuration overrides under the `env` property.

### `env.<name>`

**Type:** `object`

Define configuration overrides for specific environments.

```json
{
  "output": {
    "format": "cli"
  },
  "env": {
    "ci": {
      "output": {
        "format": "json-full"
      },
      "ci": {
        "failThreshold": 70,
        "failOnCritical": true
      }
    },
    "production": {
      "output": {
        "minSeverity": "critical"
      }
    }
  }
}
```

Activate with CLI flag:

```bash
vipr analyze src/**/*.tsx --environment ci
```

Environment configs can override any top-level configuration except `extends`.

## Configuration Examples

### Minimal Configuration

```json
{
  "$schema": "https://vipr.dev/schemas/vipr-config.schema.json"
}
```

Uses all default values.

### Development Configuration

```json
{
  "$schema": "https://vipr.dev/schemas/vipr-config.schema.json",
  "global": {
    "debug": false,
    "cache": true
  },
  "output": {
    "format": "cli",
    "minSeverity": "info"
  }
}
```

### CI Configuration

```json
{
  "$schema": "https://vipr.dev/schemas/vipr-config.schema.json",
  "output": {
    "format": "json-full",
    "minSeverity": "warning"
  },
  "ci": {
    "failThreshold": 70,
    "failOnCritical": true
  }
}
```

### Strict Configuration

```json
{
  "$schema": "https://vipr.dev/schemas/vipr-config.schema.json",
  "global": {
    "gradeBoundaries": {
      "A": 15,
      "B": 30,
      "C": 50,
      "D": 70
    }
  },
  "plugins": {
    "react": {
      "thresholds": {
        "structural": { "warning": 12 },
        "hooks": { "total": 8 },
        "coupling": { "propsWarning": 6 }
      }
    },
    "core": {
      "thresholds": {
        "cyclomaticComplexity": { "warning": 8, "critical": 15 }
      }
    }
  },
  "output": {
    "minSeverity": "warning"
  },
  "ci": {
    "failThreshold": 80,
    "failOnCritical": true
  }
}
```

### Multi-Environment Configuration

```json
{
  "$schema": "https://vipr.dev/schemas/vipr-config.schema.json",
  "global": {
    "cache": true
  },
  "output": {
    "format": "cli",
    "minSeverity": "info"
  },
  "env": {
    "ci": {
      "output": {
        "format": "json-full",
        "minSeverity": "warning"
      },
      "ci": {
        "failThreshold": 70,
        "failOnCritical": true
      }
    },
    "production": {
      "output": {
        "minSeverity": "critical"
      }
    },
    "development": {
      "global": {
        "debug": true
      }
    }
  }
}
```

## See Also

- [Configuration Guide](./configuration.md) - User guide and best practices
- [CLI Usage](cli/usage) - Command-line interface
- [Plugin Architecture](./plugin-architecture.md) - Plugin system documentation
