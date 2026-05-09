---
title: Phase 24 - Licensing
description: Licensing model, feature gating, and tier-based access control for Vipr
---

# Phase 24: Licensing

## User Story

As a Vipr user, I want a clear licensing model so that I understand which features are available to me and how to unlock additional capabilities as the product evolves.

## Licensing Model

| Tier        | Key Features                                       | Update Channel   |
| ----------- | -------------------------------------------------- | ---------------- |
| Open Source | Core analysis, CLI, stable updates                 | `stable`         |
| Pro         | Beta updates, scheduled analysis, priority support | `stable`, `beta` |
| Enterprise  | Custom plugins, SSO, dedicated support             | `stable`, `beta` |

## Feature Gating Strategy

Feature gates are checked at runtime via `LicenseService.hasFeature(feature)`. The service returns a `LicenseInfo` object containing the active tier and its unlocked feature list.

```
LicenseService.getCurrentLicense() -> { type, features, expiresAt? }
LicenseService.hasFeature('beta-updates') -> boolean
LicenseService.canAccessChannel('beta') -> boolean
```

Gates are enforced at two levels:

1. **Backend (main process)** - Services check license before executing gated operations
2. **Frontend (renderer)** - UI conditionally renders or disables gated controls

## Integration Points

- **UpdateService** - `canAccessChannel()` determines whether beta channel is selectable
- **SchedulerService** - Advanced scheduling triggers gated behind Pro tier
- **MCP Server** - Extended tool surface gated behind Enterprise tier

## Seed Prompt

> Expand this document with: validation flow (offline-first with periodic online check), license key format, migration path from open-source to paid tiers, grace period handling, and UI components for license management (activation dialog, tier badge, expiry warnings).
