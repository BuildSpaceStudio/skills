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

## Deploy (dev)

Push the current HEAD to the dev branch (`buildspace/dev`) — the branch the hosted dev environment serves — and sync the hosted dev workspace:

```bash
buildspace deploy            # push + sync dev
buildspace deploy --wait     # also wait for the deployment to finish (non-zero exit on failure)
```

The app slug is auto-detected from the git remote origin (format: `<gitBaseUrl>/<slug>.git`). Override with `--app <slug>`.

If the push is rejected, the remote dev branch has commits you don't have (often from the hosted workspace agent) — run `git fetch origin buildspace/dev && git rebase origin/buildspace/dev` and deploy again.

### Deployment status and logs

```bash
buildspace deploy status                       # View deployment status for dev/prod
buildspace deploy logs --env dev --latest       # View latest dev deployment logs
```

All deploys go to the **dev** environment. Production only changes via `buildspace promote` (below).

## Ship to production

Dev is where you iterate; nothing reaches production until you promote. Roll out whatever is on the dev branch right now:

```bash
buildspace promote --latest --yes --watch
```

- `--latest` promotes the current dev branch head (no deployment id needed). Alternatively pass `--deployment <id>` from `buildspace deploy history --env dev`.
- `--yes` skips the interactive confirmation — **required in non-interactive/agent sessions** (without it, a headless run fails fast instead of hanging).
- `--watch` follows the rollout to a terminal state, prints the production URL on success, and exits non-zero if the rollout fails.

After a successful rollout, verify the app responds (the starter guarantees `GET /api/health`):

```bash
buildspace deploy status --env prod   # shows the prod URL and status
```

If the rollout fails, inspect it with `buildspace deploy logs --env prod --latest`.

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
buildspace deploy --wait         # pushes HEAD to buildspace/dev and waits for the deployment
```

### Ship to production

```bash
buildspace promote --latest --yes --watch    # roll out the current dev branch and follow it
buildspace deploy status --env prod          # confirm prod is live and grab the URL
```

### Add a new env var

```bash
buildspace env set MY_API_KEY=sk-123 --env dev --secret
buildspace env set NEXT_PUBLIC_SITE_URL=https://myapp.com --env prod
```

## Additional resources

For latest upstream docs, fetch: `https://docs.buildspace.studio/llms-full.txt`
