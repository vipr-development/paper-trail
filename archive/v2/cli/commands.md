---
id: commands
title: CLI Commands
sidebar_label: Commands
---

# CLI Commands

Complete reference for all Vipr CLI commands.

## analyze

Analyze React code and generate reports.

```bash
vipr analyze <path> [options]
```

### Options

| Option      | Description         | Type               | Default            |
| ----------- | ------------------- | ------------------ | ------------------ |
| `--report`  | Reports to generate | `string`           | All reports        |
| `--format`  | Output format       | `json\|text\|html` | `text`             |
| `--output`  | Output file path    | `string`           | stdout             |
| `--config`  | Config file path    | `string`           | `vipr.config.json` |
| `--exclude` | Patterns to exclude | `string`           | -                  |
| `--include` | Patterns to include | `string`           | -                  |

### Examples

```bash
# Basic analysis
vipr analyze ./src

# Specific reports
vipr analyze ./src --report security,accessibility

# JSON output to file
vipr analyze ./src --format json --output report.json

# With exclusions
vipr analyze ./src --exclude "**/*.test.tsx,**/__tests__/**"
```

## init

Initialize a Vipr configuration file.

```bash
vipr init
```

Creates a `vipr.config.json` file in the current directory with default settings.

## version

Display the current version of Vipr.

```bash
vipr --version
```

## help

Display help information.

```bash
vipr --help
vipr analyze --help
```

## Next Steps

- [Configuration](./configuration) - Configure Vipr
- [Usage](./usage) - Learn more about using the CLI
