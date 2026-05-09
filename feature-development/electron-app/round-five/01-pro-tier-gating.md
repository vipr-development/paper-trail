---
id: 01-pro-tier-gating
title: 'Pro Tier Feature Gating Infrastructure'
phase: 5
dependencies: []
status: planned
---

# Pro Tier Feature Gating Infrastructure

## Problem Statement

Vipr currently provides all analysis capabilities to all users unconditionally. Round Five introduces a monetization boundary. The model is a **one-time purchase** (Sublime Text model, not a subscription): free tier users get all spatial analysis (live, current-state); Pro tier users additionally unlock temporal analysis (backfill, velocity, trends, projections, comparison, and the commit graph).

The license validation system must work **offline and without a server dependency** at runtime. A user who purchased Pro must not lose access because Vipr cannot reach a license server. The validation mechanism is a signed self-contained token (JWT-style) verified against a public key embedded in the app bundle. There is no revocation in v1—once validated and stored, a license is trusted until the user explicitly clears it.

This phase delivers the infrastructure that all subsequent Round Five phases depend on: the validator, the IPC surface, the React hook, and the two reusable gating components (`ProGate`, `UpgradeCTA`).

## New Files

| File                                                               | Role                                                                                        |
| ------------------------------------------------------------------ | ------------------------------------------------------------------------------------------- |
| `clients/desktop/src/main/licensing/license-validator.ts`          | Validates license keys offline using an embedded public key (cryptographic signature check) |
| `clients/desktop/src/main/licensing/license-store.ts`              | Persists and retrieves license state via SQLite (same DB used by existing settings)         |
| `clients/desktop/src/main/ipc/handlers/licensing.ts`               | IPC handler registrations for `license:validate`, `license:get-state`, `license:clear`      |
| `clients/desktop/src/shared/ipc/licensing-types.ts`                | Shared types consumed by both main process and renderer                                     |
| `clients/desktop/src/renderer/hooks/useLicense.ts`                 | React hook that exposes current license state to components                                 |
| `clients/desktop/src/renderer/components/licensing/UpgradeCTA.tsx` | Reusable upgrade call-to-action rendered inside gated feature areas                         |
| `clients/desktop/src/renderer/components/licensing/ProGate.tsx`    | Wrapper component that conditionally renders children or `UpgradeCTA`                       |

## Modified Files

| File                                                | Changes                                                                                                                                                                                                                                                       |
| --------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `clients/desktop/src/renderer/stores/settings.ts`   | Extend the existing `licenseKey`, `licenseStatus`, and `licenseActivatedAt` fields already present in the store; reconcile `LicenseStatus` type (currently `'inactive' \| 'checking' \| 'active' \| 'invalid'`) with the new licensing types — see note below |
| `clients/desktop/src/main/ipc/handlers/settings.ts` | Add default values for any new license-related settings keys to the `defaults` constant                                                                                                                                                                       |
| `clients/desktop/src/main/index.ts`                 | Register the licensing IPC handlers (import and call the handler setup from `licensing.ts`)                                                                                                                                                                   |
| `clients/desktop/src/shared/ipc/api-types.ts`       | Add `licensing` namespace to `ViprDesktopAPI` with `validate`, `getState`, and `clear` methods                                                                                                                                                                |
| `clients/desktop/src/preload/contexts/`             | Add a preload context file for the licensing API surface                                                                                                                                                                                                      |

## License Validation System

### Token Structure

The license key is a compact signed token. The payload is a base64url-encoded JSON object; the signature covers the payload using RS256 (RSA-SHA256). The public key is embedded in the app bundle at build time—validation never requires a network call.

```typescript
// Payload decoded from the license token.
interface LicensePayload {
  email: string;
  edition: 'pro';
  purchasedAt: number; // Unix timestamp (seconds)
  expiresAt: number | null; // null = perpetual for v1
  licenseId: string; // UUID, for support reference only
}
```

Validation steps in `license-validator.ts`:

1. Split the token into `header.payload.signature` (standard JWT structure).
2. Decode and parse the header — reject if `alg` is not `RS256`.
3. Decode and parse the payload — reject if `edition` is not `'pro'`.
4. Verify the signature of `header.payload` against the embedded public key using Node.js `crypto.verify`.
5. If `expiresAt` is non-null and in the past, return `expired` status (reserved for future use; v1 always issues perpetual licenses).
6. Return a `LicenseState` object on success.

### LicenseState Type

```typescript
// clients/desktop/src/shared/ipc/licensing-types.ts

export type LicenseTier = 'free' | 'pro';

// Note: The renderer settings store already defines LicenseStatus = 'inactive' | 'checking' | 'active' | 'invalid'.
// This new type extends it with 'expired' for future use. Reconcile by updating the existing
// LicenseStatus in stores/settings.ts to use the unified type from this file.
export type LicenseStatus = 'active' | 'inactive' | 'checking' | 'invalid' | 'expired';

export interface LicenseState {
  tier: LicenseTier;
  status: LicenseStatus; // 'active' replaces 'valid', 'inactive' replaces 'not-set' — matches existing store
  email: string | null;
  purchasedAt: number | null; // Unix timestamp
  licenseId: string | null;
}

export interface ValidateLicenseResult {
  success: boolean;
  state: LicenseState;
  errorMessage: string | null;
}
```

### Persistence

`license-store.ts` reads and writes to the existing SQLite database via `getSettingsDb()` (the same database used by the settings handler in `clients/desktop/src/main/ipc/handlers/settings.ts`). The raw license key string is stored under a key in the settings table. On app start, the main process calls `license-store.loadLicense()`, which re-validates the stored key and initialises the in-memory `LicenseState`. If validation fails (e.g., the stored key has been tampered with), the store resets to the free tier state.

**Important:** The renderer settings store (`clients/desktop/src/renderer/stores/settings.ts`) already has `licenseKey: string`, `licenseStatus: LicenseStatus`, `licenseActivatedAt: number | null`, and `bypassLicensing: boolean` fields, plus a `useIsPro` selector. The new licensing system must integrate with these existing fields rather than creating parallel structures. Update the existing `LicenseStatus` type to import from the shared `licensing-types.ts` file.

## Feature Flag Registry

The feature flag registry is a static constant — no dynamic server config in v1. Each flag carries a `tier` and a `category` so that UI components can group and filter flags consistently.

```typescript
// clients/desktop/src/shared/ipc/licensing-types.ts

export interface FeatureFlag {
  id: string;
  label: string;
  tier: LicenseTier;
  category: 'spatial' | 'temporal';
}

export const FEATURE_FLAGS: FeatureFlag[] = [
  { id: 'live-analysis', label: 'Live Analysis', tier: 'free', category: 'spatial' },
  { id: 'issues', label: 'Issues & Anti-Patterns', tier: 'free', category: 'spatial' },
  { id: 'file-detail', label: 'File Detail View', tier: 'free', category: 'spatial' },
  { id: 'dependencies', label: 'Dependency Graph', tier: 'free', category: 'spatial' },
  { id: 'static-widgets', label: 'Static Dashboard Widgets', tier: 'free', category: 'spatial' },
  { id: 'backfill', label: 'Historical Backfill', tier: 'pro', category: 'temporal' },
  { id: 'velocity', label: 'Velocity Dashboard', tier: 'pro', category: 'temporal' },
  { id: 'trend-projections', label: 'Trend Projections', tier: 'pro', category: 'temporal' },
  { id: 'commit-graph', label: 'Commit Graph', tier: 'pro', category: 'temporal' },
  { id: 'ab-comparison', label: 'A/B Snapshot Comparison', tier: 'pro', category: 'temporal' },
  {
    id: 'temporal-widgets',
    label: 'Temporal Dashboard Widgets',
    tier: 'pro',
    category: 'temporal',
  },
];
```

The helper `isFeatureEnabled(flagId: string, licenseState: LicenseState): boolean` resolves a flag against the current tier: free tier enables only `tier: 'free'` flags; pro tier enables all flags.

## IPC Additions

Three new channels are registered in `clients/desktop/src/main/ipc/handlers/licensing.ts`. Follow the typed handler pattern used in `clients/desktop/src/main/ipc/handlers/settings.ts`.

| Channel             | Direction       | Payload           | Response                |
| ------------------- | --------------- | ----------------- | ----------------------- |
| `license:validate`  | renderer → main | `{ key: string }` | `ValidateLicenseResult` |
| `license:get-state` | renderer → main | none              | `LicenseState`          |
| `license:clear`     | renderer → main | none              | `{ success: boolean }`  |

`license:validate` calls `license-validator.ts`, then persists the result via `license-store.ts` if valid, then emits `license:state-changed` back to all renderer windows so that `useLicense` can update synchronously.

`license:clear` removes the stored key and resets to the free tier state. It is exposed for support and testing workflows—not surfaced in the production settings UI by default.

## React Hook

```typescript
// clients/desktop/src/renderer/hooks/useLicense.ts

import { useEffect, useState } from 'react';
import { useViprDesktopAPI } from '../providers/ViprAPIProvider';
import type { LicenseState } from '../../shared/ipc/licensing-types';

const FREE_STATE: LicenseState = {
  tier: 'free',
  status: 'inactive',
  email: null,
  purchasedAt: null,
  licenseId: null,
};

export function useLicense(): LicenseState {
  const api = useViprDesktopAPI();
  const [state, setState] = useState<LicenseState>(FREE_STATE);

  useEffect(() => {
    api.licensing.getState().then(setState);

    const unsub = api.licensing.onStateChanged(setState);
    return unsub;
  }, [api]);

  return state;
}
```

The hook uses `useViprDesktopAPI()` — the standard pattern across all renderer hooks. The `licensing` namespace must be added to `ViprDesktopAPI` in `clients/desktop/src/shared/ipc/api-types.ts` and wired through the preload context layer.

Components call `useLicense()` to read the current tier. `ProGate` uses this hook internally—consumers of `ProGate` do not need to call `useLicense` themselves.

## UI Gating Patterns

### ProGate Wrapper

`ProGate` wraps any section of the UI that requires Pro. If the user is on the free tier, it renders `UpgradeCTA` in place of `children`. If the user is Pro, it renders `children` transparently.

```
+-------------------------------------------------+
|  ProGate (free tier)                            |
|                                                 |
|  +-----------------------------------------+   |
|  |  UpgradeCTA                             |   |
|  |                                         |   |
|  |  [Pro feature icon]  Velocity Dashboard |   |
|  |  This feature requires Vipr Pro.        |   |
|  |                                         |   |
|  |  [Upgrade to Pro]                       |   |
|  +-----------------------------------------+   |
+-------------------------------------------------+

+-------------------------------------------------+
|  ProGate (pro tier)                             |
|                                                 |
|  [children rendered normally]                   |
+-------------------------------------------------+
```

Usage:

```tsx
<ProGate featureId="velocity" featureLabel="Velocity Dashboard">
  <VelocityDashboard />
</ProGate>
```

### UpgradeCTA Component

`UpgradeCTA` uses the existing `Alert` component (`variant: 'notification'`) from `@vipr/ui`. It does not introduce new UI primitives.

```
+----------------------------------------------------------+
|  Alert (variant: notification)                           |
|                                                          |
|  [Badge: Pro]  Velocity Dashboard                        |
|  Historical velocity tracking requires Vipr Pro.         |
|  One-time purchase. No subscription.                     |
|                                                          |
|  [Upgrade to Pro ->]                (Button, primary)    |
+----------------------------------------------------------+
```

The `Upgrade to Pro` button opens the purchase URL in the system browser via `shell.openExternal`. The URL is a build-time constant — not fetched from a server.

### Disabled Controls with Pro Tooltip

For controls that are present but inactive (e.g., a widget in the library that cannot be added), use a `Tooltip` wrapping a disabled `Button`:

```
[Add widget]  <- disabled, tooltip: "Requires Vipr Pro"
              [Pro]  <- Badge variant inline
```

### License Entry in Settings

The license key input lives in a `SettingCard` (following the Phase 12 pattern) inside a new "License" section of the Settings page:

```
+----------------------------------------------------------+
|  SettingCard                                             |
|  Label: License Key                                      |
|  Description: Enter your Vipr Pro license key            |
|                                                          |
|  [________________________]  [Activate]  (Button)        |
|                                                          |
|  Status: Active  email@example.com  (Badge: Pro)         |
+----------------------------------------------------------+
```

On activation, the renderer calls `license:validate` via IPC. Success updates the status inline without a page reload.

## Existing Components to Reuse

| Component                           | Usage                                                                     |
| ----------------------------------- | ------------------------------------------------------------------------- |
| `Alert` (variant: `'notification'`) | UpgradeCTA container — provides consistent visual treatment               |
| `Badge`                             | "Pro" tier label on gated features and in settings status line            |
| `Modal`                             | License key entry flow if surfaced as a modal rather than inline settings |
| `Button`                            | Activate and Upgrade to Pro actions                                       |
| `EmptyState`                        | Full-page gating for temporal pages (Velocity, Commit Graph) on free tier |

Do not build a custom notification box or status banner — `Alert` covers both use cases.

## Settings Integration

The renderer settings store already has `licenseKey`, `licenseStatus`, and `licenseActivatedAt` fields (see `clients/desktop/src/renderer/stores/settings.ts`). The raw license key is persisted via the main process licensing store (`license-store.ts`) using the SQLite settings database. The decoded `LicenseState` is derived at runtime by the main process validator — the renderer never stores or decodes the raw key.

**No changes to `settings-types.ts` are required for license fields.** The existing `licenseKey` and `licenseActivatedAt` in the renderer store are sufficient. The licensing IPC surface (`licensing.ts` handler) manages persistence independently of the `settings:get`/`settings:set` IPC pattern.

If new license-related settings keys are added to `clients/desktop/src/shared/ipc/settings-types.ts`, corresponding default values must be added to the `defaults` constant in `clients/desktop/src/main/ipc/handlers/settings.ts`.

## Testing Approach

### Unit Tests — `license-validator.ts`

File: `clients/desktop/src/main/licensing/license-validator.test.ts`

| Test case                             | Assertion                                                       |
| ------------------------------------- | --------------------------------------------------------------- |
| Valid Pro key                         | Returns `{ tier: 'pro', status: 'active' }`                     |
| Tampered payload (base64 modified)    | Returns `{ status: 'invalid' }`                                 |
| Wrong algorithm in header             | Returns `{ status: 'invalid' }`                                 |
| `edition` field is not `'pro'`        | Returns `{ status: 'invalid' }`                                 |
| Expired key (`expiresAt` in the past) | Returns `{ status: 'expired' }`                                 |
| Null/empty string input               | Returns `{ status: 'invalid' }` with descriptive `errorMessage` |

Use a test keypair generated at test-suite setup time — do not embed real production keys in test fixtures.

### Integration Tests — IPC Handlers

File: `clients/desktop/src/main/ipc/handlers/licensing.test.ts`

| Test case                                 | Assertion                                                |
| ----------------------------------------- | -------------------------------------------------------- |
| `license:validate` with valid key         | Returns `success: true`; `license:state-changed` emitted |
| `license:validate` with invalid key       | Returns `success: false`; existing state unchanged       |
| `license:get-state` before any activation | Returns free tier state                                  |
| `license:get-state` after activation      | Returns pro tier state                                   |
| `license:clear`                           | Returns free tier state; stored key removed              |

### Component Tests — ProGate and UpgradeCTA

File: `clients/desktop/src/renderer/components/licensing/ProGate.test.tsx`

| Test case               | Assertion                                        |
| ----------------------- | ------------------------------------------------ |
| Free tier license state | Renders `UpgradeCTA`, does not render children   |
| Pro tier license state  | Renders children, does not render `UpgradeCTA`   |
| `UpgradeCTA` renders    | Contains "Upgrade to Pro" button and "Pro" Badge |
| Upgrade button click    | Calls `shell.openExternal` with purchase URL     |
