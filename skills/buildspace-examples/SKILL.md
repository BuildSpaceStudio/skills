---
name: buildspace-examples
description: Preferred patterns and recipes for building features with the BuildSpace SDK and managed database in your Next.js project. Use when adding features or slices, authentication UI, auth providers, route protection, server actions, event tracking, file storage, email notifications, billing/checkout/subscriptions, database setup and queries, or protected API routes.
---

# BuildSpace Patterns & Recipes

Preferred patterns for building with the BuildSpace SDK. **Projects created from the BuildSpace starter already implement every pattern below** — the references point at the real files. Extend those files and copy their shapes instead of writing from scratch. If your project wasn't created from the starter, treat each file reference as the pattern to create at that path. For SDK method signatures and options, see the [SDK skill](../buildspace-sdk/SKILL.md) and [API reference](../buildspace-sdk/api-reference.md).

## Conventions

- Next.js 16 App Router, TypeScript, Tailwind CSS
- Features are vertical slices under `app/dashboard/<slice>/` — see the [new-feature playbook](references/new-feature-playbook.md)
- `getSession()` / `getCurrentUser()` from `lib/auth.ts` for all server-side auth checks
- `getServerClient()` for server-side SDK access, `getBrowserClient()` for client-side
- Action tiers in `lib/safe-action.ts`: `actionClient` → `authActionClient` → `adminActionClient`
- Platform helpers live in `lib/`: `analytics.ts`, `email.ts`, `billing.ts` — go through them, they encode graceful degradation
- Prefer **server actions** for mutations; use **API routes** for streaming, webhooks, and OAuth callbacks
- Use `proxy.ts` (Next.js 16) for route-level protection — NOT `middleware.ts`
- Catch `BuildspaceError` in server-side code and degrade gracefully (empty state, logged error) — a missing backing service must never crash a page
- Use `bs.setSession(token)` / `bs.clearSession()` to scope SDK calls to the authenticated user

## Recipes

Read the reference file matching the feature being built:

### Adding any new feature

The vertical-slice checklist: schema → migration → actions → page → nav → event → verify.

See [references/new-feature-playbook.md](references/new-feature-playbook.md).

### Authentication & route protection

AuthProvider + useAuth hook, session helpers, the local users mirror, route protection with `proxy.ts`, and protected API route patterns.

See [references/auth.md](references/auth.md).

### Server actions with next-safe-action

The three action tiers, working action examples per slice, client-side usage with `useAction`.

See [references/server-actions.md](references/server-actions.md).

### Database setup and CRUD

Turso/Drizzle client, schema conventions, migrations (generated, applied on deploy), CRUD patterns.

See [references/database.md](references/database.md).

### SDK features: events, storage, notifications

The analytics and email helpers, page-view tracking, browser uploads, server-signed downloads, path-ownership rules.

See [references/sdk-features.md](references/sdk-features.md).

### Billing: checkout, portal, entitlements

Pricing UI states, server-side checkout, customer portal, entitlement gating, test vs live mode.

See [references/billing.md](references/billing.md).

## Server actions vs API routes

| Use server actions for | Use API routes for |
|------------------------|--------------------|
| Form submissions | Streaming responses (AI chat) |
| CRUD mutations | Webhook endpoints (incoming) |
| Triggering notifications | OAuth callbacks (redirects) |
| Any user-initiated action | Endpoints called by external services |

## Additional resources

For latest upstream docs, fetch: `https://docs.buildspace.studio/llms-full.txt`
