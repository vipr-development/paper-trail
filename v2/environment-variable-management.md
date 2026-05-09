# Environment Variable Management

This guide explains the contributor contract for environment variables across the monorepo.

## Contract Files

Each surface keeps three sources in sync:

1. Schema keys (runtime validation in code)
2. `.env.example` keys (committed template)
3. `env.manifest.json` keys (metadata contract)

Current manifests:

- `clients/desktop/env.manifest.json`
- `clients/vscode-extension/env.manifest.json`
- `clients/cli/env.manifest.json`
- `documentation/env.manifest.json`
- `env.manifest.json` (root, Supabase/functions + shared keys)

If you add, remove, or rename an environment key, update all three sources.

## Manifest Format

`env.manifest.json` is machine-readable and required for checks and guardrails.

```json
{
  "VIPR_SUPABASE_URL": {
    "required": true,
    "category": "public",
    "delivery": "build_constant",
    "targets": ["vscode_extension", "desktop_renderer"]
  }
}
```

Field rules:

- `required`: `true | false`
- `category`: `public | secret`
- `delivery`: `build_constant | runtime_env | secret_store`
- `targets`: non-empty string array

Validation rules:

- `env:check:examples` fails if schema, `.env.example`, and manifest keys differ.
- `env:check:bundle` fails if a `build_constant` is not `category=public`.

## Runtime Boundaries

`@vipr/env` is Node-only by design.

- Node targets (desktop main, CLI, VS Code build scripts) use `@vipr/env` loaders.
- Supabase Edge Functions (Deno) use local schema validation in `supabase/functions/_shared/env.ts`.
- Do not import `@vipr/env` into Deno functions.

VS Code extension boundary:

- `loadVscodeBuildEnv()` is for build/CI/local-dev scripts only.
- Extension activation must not depend on `process.env`; use build constants and `context.secrets`.

Electron boundary:

- Renderer exposure is allowlisted from manifest metadata.
- Main-process secrets stay in main and cross to renderer only through explicit IPC.

## Commands

Run these from the repository root.

### `pnpm env:check`

Validates required keys for runtime loaders.

Examples:

```bash
# Check all Node loader targets against process.env
pnpm env:check

# Check one target
pnpm env:check --target=desktop

# Check multiple targets
pnpm env:check --target=desktop,cli

# Validate using .env.example files instead of process.env
pnpm env:check --target=vscode-build --source=example
```

Supported `--target` values:

- `desktop`
- `cli`
- `vscode-build`

Supported `--source` values:

- `process` (default)
- `example`

### `pnpm env:check:examples`

Validates key parity:

- schema keys
- `.env.example` keys
- `env.manifest.json` keys

Use this after changing any env key contract.

### `pnpm env:check:bundle`

Runs bundle leak guardrails from manifest metadata.

Fails if a build-time injected key is marked secret.

### `pnpm env:doctor`

Readiness check across targets.

```bash
# All checks
pnpm env:doctor

# Supabase-only check
pnpm env:doctor --target=supabase --project-ref=YOUR_PROJECT_REF
```

Supported `--target` values:

- `desktop`
- `cli`
- `vscode-build`
- `supabase`

Supabase doctor behavior:

- Verifies `supabase` CLI availability.
- Calls `supabase secrets list --project-ref <ref> --output json`.
- Reports missing required secrets with actionable errors.

## Contributor Workflow

When adding a new env key:

1. Add/update schema key in the relevant runtime package.
2. Add key to the matching `.env.example`.
3. Add key metadata to `env.manifest.json`.
4. Run:
   - `pnpm env:check:examples`
   - `pnpm env:check --source=example --target=<affected-targets>`
   - `pnpm env:check:bundle`

For platform-managed secrets (for example Supabase), also verify with `pnpm env:doctor --target=supabase`.
