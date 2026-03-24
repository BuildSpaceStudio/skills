---
name: buildspace-cli
description: Reference for the BuildSpace CLI. Use when deploying apps, managing environment variables, authenticating with BuildSpace, or initializing new projects from the command line.
---

# BuildSpace CLI

Command-line interface for managing BuildSpace apps — authentication, deployment, and environment variables.

## Authentication

A stored credential is required for `deploy` and `env` commands. Two methods:

### Browser login (recommended)

```bash
buildspace auth login
```

Opens a browser for PKCE-based authentication, runs a local callback server, and stores the session automatically.

### Manual PAT

```bash
buildspace auth set
```

Prompts for a personal access token to store locally.

### Other auth commands

```bash
buildspace auth show    # Display the current stored token
buildspace auth clear   # Remove the stored credential
```

## Init

Clone a BuildSpace app repo by slug:

```bash
buildspace init <slug>
```

## Deploy

Push the current HEAD to `origin/main`, which triggers a dev deployment:

```bash
buildspace deploy
```

The app slug is auto-detected from the git remote origin (format: `<gitBaseUrl>/<slug>.git`). Override with `--app <slug>`.

### Deployment status and logs

```bash
buildspace deploy status                       # View deployment status for dev/prod
buildspace deploy logs --env dev --latest       # View latest dev deployment logs
```

All deploys go to the **dev** environment by default. The repository is the source of truth for what gets deployed.

## Environment variables

Manage env vars for your app's dev and prod environments. Requires authentication.

### List

```bash
buildspace env list [--env dev|prod]
```

Shows all active env vars for the target environment with masked values. System-managed keys (`BUILDSPACE_*`) are listed separately from custom variables. System-managed vars include SDK keys (`BUILDSPACE_SECRET_KEY`) and database credentials (`BUILDSPACE_DB_URL`, `BUILDSPACE_DB_TOKEN`) — these are auto-injected and cannot be modified via the CLI.

### Set

```bash
buildspace env set KEY=VALUE [--env dev|prod] [--secret | --no-secret]
```

Creates or updates a custom env var. Key is auto-uppercased. `NEXT_PUBLIC_*` keys default to non-secret. The `BUILDSPACE_*` prefix is reserved and will be rejected.

### Unset

```bash
buildspace env unset KEY [--env dev|prod]
```

Removes a custom env var. System-managed vars cannot be removed.

### Pull

```bash
buildspace env pull [--env dev|prod] [--output .env.local]
```

Writes non-secret variable keys and masked previews to `.env.local` (or the `--output` path). Useful for bootstrapping a local dev environment with the correct key names.

## Config

Set the API base URL and git base URL:

```bash
buildspace config
```

## App resolution

When run inside a BuildSpace app directory (cloned via `buildspace init`), the app slug is read automatically from the git remote. Override with `--app <slug>`.

## Common workflows

### First-time setup for an existing app

```bash
buildspace auth login
buildspace env pull --env dev
# Edit .env.local to fill in any secret values
# Database env vars (BUILDSPACE_DB_URL, BUILDSPACE_DB_TOKEN) are pulled automatically
npm install && npm run dev
# or: pnpm install && pnpm dev | bun install && bun dev
```

### Deploy after making changes

```bash
npm run build                    # or pnpm run build / bun run build — verify the build passes first
git add . && git commit -m "feat: ..."
buildspace deploy                # pushes HEAD to origin/main
buildspace deploy status         # check deployment progress
```

### Add a new env var

```bash
buildspace env set MY_API_KEY=sk-123 --env dev --secret
buildspace env set NEXT_PUBLIC_SITE_URL=https://myapp.com --env prod
```

## Additional resources

For latest upstream docs, fetch: `https://docs.buildspace.studio/llms-full.txt`
