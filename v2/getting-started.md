---
id: getting-started
title: Getting Started
sidebar_label: Getting Started
---

# Getting Started

This guide will help you run your first code analysis with Vipr.

## Running Your First Analysis

After installing Vipr, navigate to your React project directory and run:

```bash
vipr analyze ./src
```

This will analyze all React components in your `src` directory.

## Understanding the Output

Vipr provides several types of reports:

- **Security** - Identifies potential security vulnerabilities
- **Accessibility** - Checks for accessibility issues
- **Performance** - Analyzes performance patterns
- **Complexity** - Measures code complexity metrics
- **Maintainability** - Evaluates code maintainability

## Customizing Your Analysis

You can customize which reports to generate:

```bash
vipr analyze ./src --report security,accessibility
```

Or specify output format:

```bash
vipr analyze ./src --format json
```

## Next Steps

- [CLI Commands](./cli/commands) - Learn about all available commands
- [Configuration](./cli/configuration) - Set up a configuration file
- [Architecture](./architecture) - Understand how Vipr works
