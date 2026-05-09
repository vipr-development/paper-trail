---
id: regression-detection
title: Regression Detection
sidebar_label: Regression Detection
---

# Regression Detection

> **Status:** Planned Feature

This feature is currently in development.

## Overview

Regression detection will identify when code quality degraded by:

- Binary search through git history
- Identifying the commit that introduced quality issues
- Providing before/after comparisons
- Metric-level attribution

## Current Status

The foundation for regression detection is implemented in:

- [SQLite Storage](./sqlite-storage) - Historical snapshots
- [Phase 24 Documentation](../../../feature-development/vscode-extension/phase-24-temporal-regression-detection.md) - Algorithm design

## Coming Soon

- `Vipr: Find Regression` command
- Automated regression detection on save
- Regression reports and notifications
- Integration with CI/CD pipelines

## Related Documentation

- [SQLite Storage](./sqlite-storage)
- [Git Integration](./git-integration)
- [Git Integration Flow Diagram](../../../feature-development/vscode-extension/diagrams/03-git-integration-flow.md)

---

**Last Updated**: 2026-01-25
