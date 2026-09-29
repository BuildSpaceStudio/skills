---
name: buildspace-cli
description: Reference for the BuildSpace CLI. Use when deploying or promoting apps, managing environment variables, domains, billing, or the dev preview, reading app analytics and reports, authenticating with BuildSpace, or initializing new projects from the command line.
---

# BuildSpace CLI

Command-line interface for managing BuildSpace apps — authentication, deployment, environment variables, domains, billing, and app analytics.

## Authentication

A stored credential is required for every API command (`app`, `deploy`, `env`, ...). Three methods:

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

### Environment token (agents, CI)

```bash
BUILDSPACE_TOKEN=bs_pat_... buildspace whoami --json   # verify auth, no local state needed
```

`BUILDSPACE_TOKEN` takes priority over the saved config file.

### Other auth commands

```bash
buildspace whoami [--json]        # Show the authenticated identity (alias: auth status)
buildspace auth show              # Display the current stored token
buildspace auth clear             # Remove the stored credential
buildspace auth setup-git         # Let plain `git push`/`git pull` use your token in an existing clone
```

## Diagnose and update

```bash
buildspace doctor [--json]            # version, install method, auth, API reachability, git remote; exits 1 on failure
buildspace update [--check] [--json]  # self-update (auto-detects npm/pnpm/bun); --check only reports
```

If a command fails with `CLI_OUTDATED` (HTTP 426), run `buildspace update` and retry.

## Init

Clone a BuildSpace app repo by slug, configure git so plain `git push` works, and pull `.env.local`:

```bash
buildspace init <slug>
```

## Ship to dev

`buildspace deploy` is read-only (status/history/logs) — it does not ship code. Push, then sync the hosted dev workspace so the running preview actually picks up the change:

```bash
git push                # ships your commit to the tracked dev branch
buildspace agent reset  # pulls it into the running dev preview
```

**Both steps matter.** The hosted dev workspace keeps a writable, volume-backed git checkout so edits made directly in the browser survive restarts — which means a `git push` alone updates the repo but not what's currently running. `buildspace agent reset` discards any uncommitted changes in the live workspace and resets its checkout to the latest commit on the tracked branch. If you're not sure whether the workspace has in-browser changes worth keeping, check first:

```bash
buildspace agent inspect   # shows clean/dirty + changed files
buildspace agent accept    # commits the workspace's changes back to git instead of discarding them
```

The app slug is auto-detected from the git remote origin (format: `<gitBaseUrl>/<slug>.git`). Override with `--app <slug>`.

If the push is rejected, the remote dev branch has commits you don't have (often from `buildspace agent accept`) — run `git fetch origin <branch> && git rebase origin/<branch>` and push again.

### Deployment status and logs

```bash
buildspace deploy status                       # View deployment status for dev/prod
buildspace deploy logs --env dev --latest       # View latest dev deployment logs
```

All dev work happens in the **dev** environment. Production only changes via `buildspace promote` (below).

## Ship to production

Dev is where you iterate; nothing reaches production until you promote. Roll out whatever is on the dev branch right now:

```bash
buildspace promote --latest --yes --watch
```

- `--latest` promotes the current dev branch head (no deployment id needed). Alternatively pass `--deployment <id>` from `buildspace deploy history --env dev`.
- `--yes` skips the interactive confirmation — **required in non-interactive/agent sessions** (without it, a headless run fails fast instead of hanging).
- `--watch` follows the rollout to a terminal state, prints the production URL on success, and exits non-zero if the rollout fails. It stops watching after 10 minutes by default (`--timeout <minutes>`, max 60) and exits `2`: the rollout keeps going, so check `buildspace deploy status --env prod` instead of promoting again.

After a successful rollout, verify the app responds (the starter guarantees `GET /api/health`):

```bash
buildspace deploy status --env prod   # shows the prod URL and status
```

If the rollout fails, inspect it with `buildspace deploy logs --env prod --latest`.

### Database migrations on promote

Every promote runs the `preDeployCommand` from `railway.json` (the starter's is `bun run db:migrate`) against the production database **before** the new release starts. If it fails, the new release never starts and the previous one keeps serving. `promote --watch` and `deploy status --env prod` print the result:

```
Migrations: ran before start (`bun run db:migrate`)
```

- `Migrations: FAILED` — exit code 1. The output includes the last log lines and a `Fix:` line. Reproduce locally (`bun run db:migrate`), fix the migration (never edit an already-applied migration — add a new one with `bun db:generate`), commit, push, promote again.
- `Migrations: NOT RUN (no pre-deploy command)` — the rollout succeeded but the database wasn't migrated. Add `"preDeployCommand": ["bun run db:migrate"]` under `deploy` in `railway.json`, then promote again.
- `Migrations: could not be confirmed` — check `buildspace deploy logs --env prod --latest` before assuming the schema is current.

With `--json`, the same report is under `migrations` (`status`: `passed` | `failed` | `not_configured` | `unknown`).

## Build configuration (`railway.json`)

How the app builds and runs is controlled by the `railway.json` at the repo root — not by any platform setting. The starter uses Railway's `RAILPACK` builder (zero-config for Bun + Next.js), a `preDeployCommand` that runs Drizzle migrations, and a `/api/health` healthcheck. Keep Railpack unless you need system packages.

### System packages → use a Dockerfile

Switch to a Dockerfile only when the app needs OS-level packages Railpack can't install (`ffmpeg`, Chromium/Playwright, ImageMagick, fonts):

1. Rename the starter's `Dockerfile.example` to `Dockerfile` and add packages in the runtime stage.
2. Point `railway.json` at it:

```json
"build": { "builder": "DOCKERFILE" }
```

The `deploy` block is builder-independent — migrations, start command, and healthcheck keep working. Two gotchas when moving off Railpack:

- `NEXT_PUBLIC_*` vars are inlined at **build** time — pass them as Docker build args (`ARG` in the Dockerfile).
- Server secrets are runtime-only — build with `SKIP_ENV_VALIDATION=1` (the starter's `lib/env.ts` honors it) so `next build` doesn't fail validating them.

Full reference: `https://docs.buildspace.studio/docs/hosting/build-config`.

## App management

```bash
buildspace app list [--json]                  # your apps, active one marked
buildspace app create --name "My App" --json  # returns slug, environments, keys, provisioningSession
buildspace app use <slug>                     # set the default app
buildspace app status [--json]                # hosting, dev preview sleep state, Terms & Privacy checklist (`legal`)
buildspace app hosting <status|enable|disable> [--env dev|prod]
buildspace app delete <slug> --confirm <slug>
```

### Dev preview sleep and wake

The hosted dev preview sleeps when idle (`app status` shows it). Wake it before relying on the preview URL:

```bash
buildspace app wake [--no-wait] [--json]   # blocks until reachable unless --no-wait
buildspace app sleep [--json]              # sleep now instead of waiting to idle out
```

### Billing

```bash
buildspace app billing status [--env dev|prod] [--json]
buildspace app billing enable --env dev                   # requires Stripe Connect first (Studio Settings)
buildspace app billing overview [--json]                  # Stripe connection, readiness checklist, counts, recent activity
buildspace app billing products [--all] [--env dev|prod]  # products with their prices and ids
buildspace app billing products create --name "Pro" --amount 9.99 --interval month [--lookup-key pro-monthly]
buildspace app billing products create --name "Credits" --amount-cents 500 --type one_time
buildspace app billing products archive <id|name>         # archives the product and deactivates its prices
buildspace app billing prices [--all]                     # flat price list
buildspace app billing prices deactivate <priceId>        # or: activate
buildspace app billing sync                               # copy dev products/prices into prod
```

`products create` makes the product and its first price together. `--amount` is in major units (`9.99`); use `--amount-cents` for zero-decimal currencies like JPY. Recurring products need `--interval day|week|month|year`. Set `--lookup-key` so app code can start checkout with `createCheckout({ lookupKey })` instead of hardcoding a price id. All catalog commands accept `--app <slug>`, `--env dev|prod` (default dev) and `--json`.

**Set up billing end to end (agents):**

1. `buildspace app billing overview --json` — confirm `stripe.test` is connected (the creator connects Stripe in Studio Settings; the CLI cannot do that step).
2. `buildspace app billing enable --env dev` if `environments[].enabled` is false.
3. `buildspace app billing products create ... --json` for each plan, then `buildspace app billing products --json` to verify.
4. When ready for production: connect live Stripe, `buildspace app billing enable --env prod`, then `buildspace app billing sync`. Check `readiness.checks` in `overview --json` for anything still missing.

### Analytics and reports

Read-only; each command accepts `--app <slug>` and `--json`. Use these to see how an app is doing and to feed data to an AI summary:

```bash
buildspace app insights [--days 30]             # event totals, unique actors, top events
buildspace app emails [--days 30] [--limit 10]  # sent, delivery/open/click rates, recent sends
buildspace app users                            # users and last logins
buildspace app report [--days 30] --json        # insights + emails + users + billing in one snapshot
```

`--days` is 1–365 (default 30). In `report`, each section is `{ data }` or `{ error }`, so one failing section doesn't hide the rest. For deployments use `buildspace deploy status|history|logs`.

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

Writes non-secret variables with their real values, plus the managed database credentials `BUILDSPACE_DB_URL` and `BUILDSPACE_DB_TOKEN`, to `.env.local` (or the `--output` path). Secret variables are not written — fill those in by hand.

## Custom domains

`buildspace domains` points a domain the creator owns at an app environment. Buildspace attaches it to the runtime service and returns the DNS records the creator must create at their **own** DNS provider — Buildspace never touches their registrar.

```bash
buildspace domains add app.theirbrand.com                 # prod (default)
buildspace domains add dev.theirbrand.com --env dev
buildspace domains verify app.theirbrand.com              # re-check DNS + certificate
buildspace domains list --json                            # both envs, status + pending records
buildspace domains remove app.theirbrand.com
```

`add` prints a `CNAME` (traffic) and a `TXT` (verification) record. Nothing is live until both resolve; until then the app keeps serving on `<slug>.apps.buildspace.studio`, which stays as a fallback afterward. Re-run `verify` until status is `active` — DNS propagation can take minutes to hours, so do not loop tightly.

Apex domains (`theirbrand.com`) cannot take a plain `CNAME`; the creator needs their provider's ALIAS/ANAME/CNAME-flattening record, or should use a subdomain and redirect the apex.

Login redirects to the domain are allowed by default so the OAuth callback works there; toggle with `buildspace domains auth <domain> on|off`. The app sets its own first-party session cookie on that domain (`Secure`, `HttpOnly`, `SameSite=Lax`).

Full reference: `https://docs.buildspace.studio/docs/hosting/custom-domains`.

## Standalone databases

`buildspace db` manages SQLite databases owned by your organization rather than by a project — useful for scratch data, prototypes, or data shared across projects. They are separate from the per-environment database each project already gets.

```bash
buildspace db list                                  # list databases + quota usage
buildspace db create "Scratch data"                 # prints the connection URL + token ONCE
buildspace db show scratch-data                     # usage + token inventory
buildspace db delete scratch-data --yes             # destroys the database and its data
```

Run SQL server-side (read-only unless `--write` is passed):

```bash
buildspace db shell scratch-data --sql "select count(*) from users"
buildspace db shell scratch-data --write --sql "delete from sessions"
cat migration.sql | buildspace db shell scratch-data --write
```

Tokens are per-database; mint one per consumer so you can revoke narrowly:

```bash
buildspace db token create scratch-data --label analytics --read-only
buildspace db token revoke scratch-data <tokenId>
```

Every subcommand accepts the database slug or its UUID, and supports `--json`. Connect from code with `@libsql/client` using the printed URL plus the token in an env var — never hardcode the token.

Full reference: `https://docs.buildspace.studio/docs/database/standalone-databases`.

## Pages

`buildspace pages` publishes a single HTML file to a hosted, gated URL under your creator handle — no project, build, or deploy required. Buildspace stores and serves the file as-is; it never generates, rewrites, or executes it.

```bash
buildspace pages publish index.html                              # first run prompts you to claim a handle
buildspace pages publish report.html --visibility login --open   # public | login | org | private (default: private)
```

Publishing is idempotent by content — republishing unchanged bytes reports `already published (vN)` instead of creating a new version, so it's safe from CI or a watch script. A publish is blocked if the file appears to contain a credential (`--allow-secrets` overrides).

```bash
buildspace pages list                        # your pages, visibility, version, size, views, quota
buildspace pages open my-report              # open the hosted URL in a browser
buildspace pages versions my-report          # every immutable version
buildspace pages rollback my-report --version 2   # pointer flip, not a rewrite
buildspace pages visibility my-report public
buildspace pages delete my-report --yes
buildspace pages handle [<handle>]           # show or claim your creator handle
```

Every subcommand supports `--json`. Full reference: `https://docs.buildspace.studio/docs/cli/pages`.

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
git push                         # ships the commit to the tracked dev branch
buildspace agent reset --yes     # sync it into the running dev preview (--yes for non-interactive)
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
