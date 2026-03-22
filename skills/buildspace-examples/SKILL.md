---
name: buildspace-examples
description: Preferred patterns and recipes for building features with the BuildSpace SDK and managed database in your Next.js project. Use when adding authentication UI, auth providers, route protection, server actions, event tracking, file storage, email notifications, database setup and queries, or protected API routes.
---

# BuildSpace Patterns & Recipes

Preferred patterns for building with the BuildSpace SDK. Each recipe is self-contained. For SDK method signatures and options, see the [SDK skill](../buildspace-sdk/SKILL.md) and [API reference](../buildspace-sdk/api-reference.md).

## Conventions

- Next.js 16 App Router, TypeScript, Tailwind CSS
- `getSession()` from `lib/auth.ts` is the shared session helper — use it in all server-side auth checks
- `getServerClient()` for server-side SDK access, `getBrowserClient()` for client-side
- Prefer **server actions** for mutations; use **API routes** for streaming, webhooks, and OAuth callbacks
- Use `proxy.ts` (Next.js 16) for route-level protection — NOT `middleware.ts`
- Catch `BuildspaceError` in server-side code and return structured error responses
- Use `bs.setSession(token)` / `bs.clearSession()` to scope SDK calls to the authenticated user

## Recipes

Read the reference file matching the feature being built:

### Authentication & route protection

AuthProvider + useAuth hook, route protection with `proxy.ts`, and protected API route patterns.

See [references/auth.md](references/auth.md).

### Server actions with next-safe-action

Safe-action client setup (public and authenticated), action patterns, client-side usage with `useAction`.

See [references/server-actions.md](references/server-actions.md).

### Database setup and CRUD

Turso/Drizzle setup, schema definition, CRUD server actions, and database access from API routes.

See [references/database.md](references/database.md).

### SDK features: events, storage, notifications

Browser and server-side event tracking, file uploads, signed URLs, and transactional email.

See [references/sdk-features.md](references/sdk-features.md).

## Server actions vs API routes

| Use server actions for | Use API routes for |
|------------------------|--------------------|
| Form submissions | Streaming responses (AI chat) |
| CRUD mutations | Webhook endpoints (incoming) |
| Triggering notifications | OAuth callbacks (redirects) |
| Any user-initiated action | Endpoints called by external services |

## Additional resources

For latest upstream docs, fetch: `https://docs.buildspace.studio/llms-full.txt`
