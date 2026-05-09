That changes everything. Now we're talking about **commercial software architecture**, not just packaging.

## The Core Challenge

If your code is public and feature gating is client-side, someone will fork it and remove the checks. This is why most commercial desktop/extension software keeps source private.

## Recommended Architecture

### Keep the monorepo private

```
your-private-monorepo/
  apps/
    desktop/          ← paid product
    vscode-extension/ ← paid product
  packages/
    cli/              ← freemium
    core/             ← shared business logic
    licensing/        ← entitlement checks
```

No public repo at all, or a minimal public repo with just docs/marketing.

### Feature gating strategies

**Option 1: License key validation (simpler)**

Users purchase a key, the app validates it locally or against your server. Works offline, but keys can be shared/pirated. Fine for most indie software.

```typescript
// packages/licensing/src/index.ts
export function checkEntitlement(feature: string, licenseKey?: string): boolean {
  if (!licenseKey) return FREE_FEATURES.includes(feature);
  const decoded = validateKey(licenseKey); // JWT, cryptographic signature, etc.
  return decoded.features.includes(feature);
}
```

**Option 2: Server-side entitlement (more robust)**

App calls your API to check what features the user has access to. Harder to bypass, enables subscription models, usage tracking, etc. Requires you to run a service.

**Option 3: Hybrid**

License key for offline use, server validation when online for subscription status, usage limits, etc. This is what JetBrains, Sublime, and similar tools do.

### CLI considerations

If you want the CLI publicly installable but feature-limited, you can still publish to npm from a private repo. The question is whether you want the CLI _source_ visible for community trust/contributions.

**"Open core" model**: CLI source is public with basic features, but advanced features require a license key that unlocks them. People can see the code, but removing the checks violates your license and they lose updates/support.

## Typical Commercial Stack

| Concern            | Solution                                               |
| ------------------ | ------------------------------------------------------ |
| Source code        | Private monorepo                                       |
| Distribution       | npm (CLI), VS Code Marketplace, your website (desktop) |
| Payments           | Stripe, Paddle, Gumroad, LemonSqueezy                  |
| License management | Keygen.sh, your own API, or self-rolled JWT keys       |
| Trial limits       | Time-based, feature-based, or usage-based              |

## My Recommendation for Vipr

Given you're targeting post-seed startups analyzing technical debt:

1. **Private monorepo** — no public source for any of it
2. **Server-side entitlements** — you'll want usage analytics anyway to understand how customers use the tool, and it enables seat-based pricing which is natural for B2B
3. **CLI published to npm** but closed-source — totally normal, many commercial CLIs work this way (Vercel CLI was closed source for years)
4. **Generous free tier** to drive adoption, paid tiers unlock depth of analysis, team features, integrations

Would you like to dig into the licensing architecture (Keygen vs. rolling your own) or the payment/subscription infrastructure?
