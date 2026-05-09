---
title: Supabase Auth and OAuth Setup
sidebar_label: Supabase Auth and OAuth Setup
description: Step-by-step local-first guide for configuring Vipr Desktop authentication with Supabase, Google, GitHub, email flows, and licensing.
keywords:
  - supabase
  - oauth
  - google
  - github
  - electron
  - desktop
  - authentication
  - licensing
---

# Supabase Auth and OAuth Setup

This guide walks through the full setup for Vipr Desktop authentication and licensing.

It is written for the current stage of the project:

1. Get local development working first.
2. Add production cloud setup second.
3. Defer a dedicated staging environment until you actually need it.

## Recommended Environment Strategy

If you want the shortest path with the least overhead, use this setup:

| Environment       | What to use now             | Why                                                             |
| ----------------- | --------------------------- | --------------------------------------------------------------- |
| Local development | Local Supabase CLI stack    | Fast iteration, local email interception, no risk to production |
| Production        | One hosted Supabase project | Real users, real OAuth providers, real licensing functions      |
| Staging           | Skip for now                | Useful later, but not required to get auth working              |

### Should you create a separate hosted cloud project right now?

Short answer: **not for daily local development**.

For your current stage, the best setup is:

- **Local Supabase** for day-to-day coding, Playwright, email confirmation, and password reset testing.
- **One hosted Supabase project** for production.

Create a second hosted cloud project only if one of these becomes true:

- You want teammates to test shared auth flows without running Supabase locally.
- You need a safe place to test production-like secrets, OAuth settings, or webhooks.
- You want to rehearse deployments without touching production.

If staging requires a paid plan and you want to avoid that for now, that is reasonable. Local plus production is enough to complete this phase.

## What You Are Setting Up

By the end of this guide, you will have:

- Email/password sign-up and sign-in
- Email confirmation
- Password reset
- Google OAuth
- GitHub OAuth
- Local email interception with Mailpit
- Supabase-backed licensing using the same database
- A path to add staging and GitHub deployment workflow later

## Files in This Repo That Matter

You will touch these files during setup:

- [supabase/config.toml](/Users/jamesleebaker/Codespace/vipr-app/supabase/config.toml)
- [.env.supabase.example](/Users/jamesleebaker/Codespace/vipr-app/.env.supabase.example)
- [clients/desktop/.env.example](/Users/jamesleebaker/Codespace/vipr-app/clients/desktop/.env.example)
- [supabase/README.md](/Users/jamesleebaker/Codespace/vipr-app/supabase/README.md)
- [documentation/docs/development/environment-variable-management.md](/Users/jamesleebaker/Codespace/vipr-app/documentation/docs/development/environment-variable-management.md)

## Step 1: Install the Local Prerequisites

Install these first:

- Node.js and pnpm
- Supabase CLI
- Docker Desktop

The local Supabase stack runs inside Docker containers, so Docker must be running before `supabase start`.

## Step 2: Install Project Dependencies

From the repo root:

```bash
pnpm install
```

## Step 3: Decide Your Immediate Environment Layout

For now, use this:

- **Local auth environment**: Supabase CLI stack on your machine
- **Production auth environment**: hosted Supabase project in the cloud

Do **not** create a local-only cloud project unless you specifically need a shared remote test environment.

## Step 4: Prepare the Local Environment Files

Start from the committed examples:

- Copy [.env.supabase.example](/Users/jamesleebaker/Codespace/vipr-app/.env.supabase.example) to `.env.supabase.local`
- Copy [clients/desktop/.env.example](/Users/jamesleebaker/Codespace/vipr-app/clients/desktop/.env.example) to `clients/desktop/.env.local` or your preferred local env file

### What goes into `.env.supabase.local`

This file is for:

- local Supabase auth provider settings
- local Edge Function secrets
- desktop-safe Supabase values if you want one place to reference them

Important: the Supabase CLI does **not** automatically read `.env.supabase.local` for `supabase start`.

Before running local Supabase commands, either:

1. Export the variables into your current shell, or
2. Copy the provider-related values into a root `.env` file for CLI substitution

In `zsh` or `bash`, a common pattern is:

```bash
set -a
source .env.supabase.local
set +a
```

### What goes into `clients/desktop/.env.local`

This file is for the Electron desktop app runtime:

- `VIPR_SUPABASE_URL`
- `VIPR_SUPABASE_PUBLISHABLE_KEY`
- `VIPR_LICENSE_CLAIM_PUBLIC_KEY`

For local development:

- `VIPR_SUPABASE_URL` should usually be `http://127.0.0.1:54321`
- `VIPR_SUPABASE_PUBLISHABLE_KEY` should be set to the local `PUBLISHABLE_KEY` from `supabase status -o env`
- `SUPABASE_ANON_KEY` and `SUPABASE_SERVICE_ROLE_KEY` map from the local `ANON_KEY` and `SERVICE_ROLE_KEY` output by `supabase status -o env`

## Step 5: Find or Create the Licensing Keys

You need two related values:

- `LICENSE_SIGNING_PRIVATE_KEY`: used by Supabase Edge Functions to sign claims
- `VIPR_LICENSE_CLAIM_PUBLIC_KEY`: used by the desktop app to verify those claims

### If you already have licensing keys

Use the existing pair. The public key must match the private key already used by your licensing functions.

### If you need a new pair

Run:

```bash
pnpm --filter @vipr/licensing generate-keypair
```

This writes:

- `packages/licensing/keys/private-ed25519.pem`
- `packages/licensing/keys/public-ed25519.pem`

Use them like this:

- `LICENSE_SIGNING_PRIVATE_KEY` = contents of `private-ed25519.pem`
- `VIPR_LICENSE_CLAIM_PUBLIC_KEY` = contents of `public-ed25519.pem`

## Recommended One-Command Local Dev Flow

Once `.env.supabase.local` is filled in, the fastest path is:

```bash
pnpm desktop:dev:local-auth
```

This command:

- loads `.env.supabase.local` and `clients/desktop/.env.local` when present
- opens Docker Desktop if it is not already running
- starts the local Supabase stack
- reads `supabase status -o env` and injects the local API URL plus publishable, anon, and service-role keys
- starts local Edge Functions for licensing when needed
- launches the desktop app against the local stack

Use the manual steps below when you want to debug a specific layer independently.

## Step 6: Start the Local Supabase Stack

Make sure Docker is running, then:

```bash
set -a
source .env.supabase.local
set +a

pnpm supabase:local:start
pnpm supabase:local:status
```

After `supabase status`, note these values:

- API URL
- publishable key
- anon key
- service role key
- Studio URL
- Mailpit URL

For the default local setup, they are usually:

- API: `http://127.0.0.1:54321`
- Studio: `http://127.0.0.1:54323`
- Mailpit: `http://127.0.0.1:54324`

## Step 7: Put the Local Supabase Values into the Desktop App

In your desktop env file, set:

```bash
VIPR_SUPABASE_URL=http://127.0.0.1:54321
VIPR_SUPABASE_PUBLISHABLE_KEY=<local-publishable-key-from-supabase-status>
VIPR_LICENSE_CLAIM_PUBLIC_KEY="-----BEGIN PUBLIC KEY-----\n...\n-----END PUBLIC KEY-----"
```

## Step 8: Understand the Local Email Flow

Local email confirmation and password reset do **not** need a real SMTP service.

Supabase CLI intercepts those emails and exposes them in Mailpit.

Use Mailpit to test:

- sign-up confirmation links
- forgot-password links
- recovery flow timing and link contents

Open Mailpit in the browser:

```text
http://127.0.0.1:54324
```

This is the best local workflow for development and Playwright.

## Step 9: Configure Local Auth Redirects

This repo already expects these redirect targets:

- `vipr://auth/callback`
- `vipr://auth/recovery`

These are **desktop deep links**, not provider callback URLs.

### Important distinction

Use this mental model:

1. Google or GitHub sends the user back to **Supabase**
2. Supabase completes auth
3. Supabase redirects back to the desktop app using `vipr://...`

### Provider callback URLs

When you configure Google or GitHub, the callback URL is:

- Local: `http://127.0.0.1:54321/auth/v1/callback`
- Production: `https://<project-ref>.supabase.co/auth/v1/callback`

Do **not** put `vipr://auth/callback` directly into Google or GitHub as the provider callback URL.

## Step 10: Set Up Local Google OAuth

You only need this when you want real local Google login testing.

### In Google Cloud Console

1. Open the Google Cloud Console.
2. Select your project, or create one for auth.
3. Go to **Google Auth platform**.
4. Open **Clients**.
5. Create a new OAuth client of type **Web application**.
6. Add these authorized redirect URIs:
   - `http://127.0.0.1:54321/auth/v1/callback`
   - `https://<project-ref>.supabase.co/auth/v1/callback`
7. Save the client.
8. Copy the **Client ID** and **Client secret**.

### Where to store them locally

Put them in `.env.supabase.local`:

```bash
SUPABASE_AUTH_EXTERNAL_GOOGLE_CLIENT_ID=<google-client-id>
SUPABASE_AUTH_EXTERNAL_GOOGLE_CLIENT_SECRET=<google-client-secret>
```

## Step 11: Set Up Local GitHub OAuth

You only need this when you want real local GitHub login testing.

### In GitHub

1. Open GitHub.
2. Go to **Settings**.
3. Open **Developer settings**.
4. Open **OAuth Apps**.
5. Create a new OAuth app.
6. Set the callback URL to:
   - `http://127.0.0.1:54321/auth/v1/callback`
7. After creation, also keep the production callback in mind for your production app:
   - `https://<project-ref>.supabase.co/auth/v1/callback`
8. Copy the **Client ID**.
9. Generate and copy the **Client secret**.

### Where to store them locally

Put them in `.env.supabase.local`:

```bash
SUPABASE_AUTH_GITHUB_CLIENT_ID=<github-client-id>
SUPABASE_AUTH_GITHUB_SECRET=<github-client-secret>
```

## Step 12: Manual Debug Path for Local Edge Functions

When you want licensing and Edge Function behavior against the same local stack:

```bash
set -a
source .env.supabase.local
set +a

pnpm supabase:functions:serve
```

`pnpm desktop:dev:local-auth` already starts this for you. Use the manual command when you need to inspect or restart the function layer by itself.

## Step 13: Manual Debug Path for the Desktop App

Use the app-only local auth script:

```bash
VIPR_SUPABASE_PUBLISHABLE_KEY="<local-publishable-key>" pnpm desktop:dev:local-auth:app
```

If your desktop env file already includes the local values, you can just run the normal desktop dev flow that loads your local env.

## Step 14: Verify the Local Flows in This Order

Do them in this order so you isolate problems quickly:

1. Start the local Supabase stack
2. Start local Edge Functions
3. Launch the desktop app
4. Verify the sign-in screen appears
5. Test **email sign-up**
6. Open Mailpit and confirm the email
7. Test **email sign-in**
8. Test **forgot password**
9. Open Mailpit and complete the reset flow
10. Test **Google**
11. Test **GitHub**
12. Test a licensing validation path

If email/password works but OAuth does not, the issue is usually in provider configuration, not the desktop app.

## Step 15: Run the Local Playwright Auth Suite

There are now two Playwright modes:

### Mock auth suite

This is deterministic and does not need Supabase:

```bash
pnpm --filter @vipr/desktop test:e2e
```

### Real local auth suite

This is opt-in and uses the local Supabase stack:

```bash
PLAYWRIGHT_REAL_AUTH_ENABLED=1 \
PLAYWRIGHT_SUPABASE_URL=http://127.0.0.1:54321 \
PLAYWRIGHT_SUPABASE_PUBLISHABLE_KEY="<local-publishable-key>" \
VIPR_LICENSE_CLAIM_PUBLIC_KEY="<public-key>" \
pnpm --filter @vipr/desktop test:e2e:auth
```

If you want local Google and GitHub Playwright coverage, the local provider credentials must also be configured.

## Step 16: Set Up the Production Supabase Project

Once local is working, move to production.

### In the Supabase dashboard

Open your production project and gather:

- Project URL
- publishable key

You will use them as:

- `VIPR_SUPABASE_URL=https://<project-ref>.supabase.co`
- `VIPR_SUPABASE_PUBLISHABLE_KEY=<production-publishable-key>`

### Where to find them

In Supabase, look in the project **Connect** dialog or **Settings -> API Keys**.

## Step 17: Configure Production Google OAuth

In your Google OAuth client:

1. Add the hosted callback URL:
   - `https://<project-ref>.supabase.co/auth/v1/callback`
2. If you want one Google client to work for both local and production, also keep:
   - `http://127.0.0.1:54321/auth/v1/callback`
3. Save the client.

In Supabase production auth settings, enter:

- Google client ID
- Google client secret

### Recommendation

For now, you can use one Google OAuth client that has both local and production callback URLs.

Later, when you add staging, split them into separate OAuth clients:

- local/dev
- staging
- production

## Step 18: Configure Production GitHub OAuth

In your GitHub OAuth app:

1. Set or update the callback URL to:
   - `https://<project-ref>.supabase.co/auth/v1/callback`
2. If you want one GitHub OAuth app to work for both local and production, also create or maintain a local/dev app with:
   - `http://127.0.0.1:54321/auth/v1/callback`
3. Save the app.

In Supabase production auth settings, enter:

- GitHub client ID
- GitHub client secret

## Step 19: Configure Production Secrets

You need secrets in two places:

### Desktop production runtime

Set:

- `VIPR_SUPABASE_URL`
- `VIPR_SUPABASE_PUBLISHABLE_KEY`
- `VIPR_LICENSE_CLAIM_PUBLIC_KEY`

### Supabase production secrets

Set:

- `SUPABASE_URL`
- `SUPABASE_ANON_KEY`
- `SUPABASE_SERVICE_ROLE_KEY`
- `LICENSE_SIGNING_PRIVATE_KEY`
- `LEMON_WEBHOOK_SECRET`
- `LEMON_VARIANT_MAP`

The public key in the desktop app must match the private key used in the Supabase functions.

## Step 20: Apply Production Database and Functions

From the repo root:

```bash
supabase link --project-ref <your-project-ref>
supabase db push
supabase functions deploy license-activate
supabase functions deploy license-validate
supabase functions deploy license-deactivate
supabase functions deploy purchase-webhook
```

## Step 21: Verify Production Carefully

Do not start with every flow at once.

Test in this order:

1. Desktop app loads sign-in page
2. Existing user can sign in with email/password
3. New user can sign up and confirm email
4. Forgot password works
5. Google sign-in works
6. GitHub sign-in works
7. Licensing validation works after sign-in
8. Sign-out clears access correctly

## Step 22: What to Do When You Are Ready for Staging

When you are ready for a real staging environment, move to this layout:

| Environment | Purpose                                                |
| ----------- | ------------------------------------------------------ |
| Local       | daily development and Playwright                       |
| Staging     | shared QA, deployment rehearsal, provider verification |
| Production  | real users                                             |

At that point, split these per environment:

- Supabase project
- Google OAuth client
- GitHub OAuth app
- Supabase secrets
- desktop runtime env values

## Step 23: How This Should Evolve with GitHub Deployment Workflow

You mentioned GitHub deployment workflow as a future step. The clean progression is:

### Now

- Keep local and production mostly manual
- Validate auth locally first
- Deploy Supabase changes manually

### Next

Add GitHub Actions that:

1. Run lint and tests on pull requests
2. Optionally run mock Playwright auth coverage
3. Deploy to staging on a non-production branch
4. Deploy to production only from the protected production branch

### Later

When staging exists, wire GitHub workflow like this:

- `pull_request`:
  - unit tests
  - renderer tests
  - mock Playwright
- `push` to `develop` or staging branch:
  - deploy staging Supabase functions
  - publish staging desktop env bundle if you add one
- `push` to `main`:
  - deploy production functions
  - publish production desktop build

## Quick Reference: Where to Find Each Value

| Value                           | Where to get it                                                                                             |
| ------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| `VIPR_SUPABASE_URL`             | Supabase dashboard, project URL                                                                             |
| `VIPR_SUPABASE_PUBLISHABLE_KEY` | Supabase dashboard, publishable key; locally set it to `PUBLISHABLE_KEY` from `supabase status -o env`      |
| `VIPR_LICENSE_CLAIM_PUBLIC_KEY` | Existing licensing secret store, or `packages/licensing/keys/public-ed25519.pem` if you generate a new pair |
| Google client ID/secret         | Google Cloud Console, Google Auth platform, OAuth client                                                    |
| GitHub client ID/secret         | GitHub Settings, Developer settings, OAuth Apps                                                             |
| Local Supabase anon key         | `supabase status`                                                                                           |

## Common Problems

### Problem: Email sign-up works but confirmation does not return to the app

**Cause:** Supabase redirect URLs are incomplete.

**Solution:**

1. Check `supabase/config.toml`
2. Verify `vipr://auth/callback` is in the allowed redirect URLs
3. Restart the local stack

### Problem: Google or GitHub login fails before returning to the app

**Cause:** Provider callback URL mismatch.

**Solution:**

1. Verify the provider callback URL points to Supabase
2. Verify local uses `http://127.0.0.1:54321/auth/v1/callback`
3. Verify production uses `https://<project-ref>.supabase.co/auth/v1/callback`

### Problem: Licensing fails even though auth works

**Cause:** Public and private signing keys do not match.

**Solution:**

1. Confirm the desktop app has the correct `VIPR_LICENSE_CLAIM_PUBLIC_KEY`
2. Confirm Supabase functions use the matching `LICENSE_SIGNING_PRIVATE_KEY`

## Recommended First Execution Plan

If you want the cleanest path, do this next:

1. Get local email/password working
2. Confirm email flows in Mailpit
3. Get local password reset working
4. Get local licensing working
5. Add local Google and GitHub
6. Repeat the same provider setup in production
7. Add staging later when you need shared QA and deployment rehearsal

## External References

- [Supabase local development](https://supabase.com/docs/guides/local-development)
- [Supabase API keys](https://supabase.com/docs/guides/api/api-keys)
- [Supabase Google auth](https://supabase.com/docs/guides/auth/social-login/auth-google)
- [Supabase GitHub auth](https://supabase.com/docs/guides/auth/social-login/auth-github)
- [Supabase managing environments](https://supabase.com/docs/guides/deployment/managing-environments)
- [Google OAuth credentials](https://developers.google.com/workspace/guides/create-credentials)
- [GitHub OAuth apps](https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/creating-an-oauth-app)
