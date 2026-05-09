---
id: configuration
title: CLI Configuration
sidebar_label: Configuration
---

# CLI Configuration

Configure Vipr to match your project's needs.

## Configuration File

Create a `vipr.config.json` file in your project root:

```json
{
  "include": ["src/**/*.tsx", "src/**/*.ts"],
  "exclude": ["**/*.test.tsx", "**/__tests__/**"],
  "reports": ["security", "accessibility", "performance"],
  "output": {
    "format": "text",
    "path": "./vipr-report.txt"
  },
  "plugins": {
    "react": {
      "enabled": true
    },
    "core": {
      "enabled": true
    }
  }
}
```

## Configuration Options

### include

Array of glob patterns for files to analyze.

```json
{
  "include": ["src/**/*.tsx", "src/**/*.ts"]
}
```

### exclude

Array of glob patterns for files to exclude.

```json
{
  "exclude": ["**/*.test.tsx", "**/__tests__/**", "**/*.spec.tsx"]
}
```

### reports

Array of report types to generate.

```json
{
  "reports": ["security", "accessibility", "performance", "cyclomatic"]
}
```

### output

Output configuration.

```json
{
  "output": {
    "format": "json",
    "path": "./reports/vipr.json"
  }
}
```

### plugins

Plugin-specific configuration.

```json
{
  "plugins": {
    "react": {
      "enabled": true,
      "minComponentSize": 50
    }
  }
}
```

## Environment Variables

| Variable         | Description         | Default            |
| ---------------- | ------------------- | ------------------ |
| `VIPR_CONFIG`    | Path to config file | `vipr.config.json` |
| `VIPR_LOG_LEVEL` | Logging level       | `info`             |

## Next Steps

- [Commands](./commands) - Learn about all available commands
- [Usage](./usage) - Basic CLI usage
