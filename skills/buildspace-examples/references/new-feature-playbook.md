# New Feature Playbook

The exact checklist for adding a feature to this app. Features are **vertical slices**: one folder under `app/dashboard/<slice>/` owning its page, actions, and components, plus (usually) one table. `app/dashboard/todos/` is the reference slice — when in doubt, copy its shape.

## Checklist

1. **Schema** — add the table to `lib/db/schema.ts` (SQLite types, text UUID PK via `$defaultFn(() => crypto.randomUUID())`, `userId` column, ISO-string `createdAt`). Export `$inferSelect`/`$inferInsert` types.
2. **Migration** — `bun db:generate`, review the SQL in `drizzle/`, then `bun db:migrate`. Never hand-write migration files. The command refuses destructive migrations; if you're renaming, go add → backfill → drop instead.
3. **Actions** — `app/dashboard/<slice>/actions.ts` with `"use server"`. Use `authActionClient` (or `adminActionClient` for admin features) + zod `.inputSchema(...)` with every string/array bounded by `.max()`. Query through `scopedTo(ctx.session.user.id)` — never `db` directly. Call `revalidatePath("/dashboard/<slice>")` after mutations.
4. **Page** — `app/dashboard/<slice>/page.tsx` as an async server component: `getSession()` guard (`if (!session) redirect("/")`), query via `scopedTo(session.user.id)`, render `<PageHeader>` + content. Use `<EmptyState>` for the zero state.
5. **Client components** — colocate in the slice folder, `"use client"` only where there's interactivity. Call actions with `useAction` from `next-safe-action/hooks`; toast on success/error; disable controls on `isPending`.
6. **Loading state** — `app/dashboard/<slice>/loading.tsx` with `Skeleton` blocks matching the page layout (copy an existing one).
7. **Nav entry** — add the route to `components/dashboard-nav.tsx`. Nothing else is needed for it to appear.
8. **Track an event** — `await trackEvent({ event: "<thing>_created", properties, userId: ctx.session.user.id })` from `lib/analytics.ts` in the primary mutation.
9. **UI kit first** — check `components/ui/` before writing markup; add missing primitives there in the same cva style. Never install a component library.
10. **Verify** — `bun run verify` must pass. It runs lint, types, build, your tests, and the guardrail conformance tests below.

## Guardrails

Some of these are conventions; the first six **fail `bun run verify`** via `test/guardrails.test.ts`, which names the file, the line, and the fix. That's deliberate — a rule that only lives in a doc gets skipped on step 40 of a build.

Enforced:

1. **Route handlers under `app/api/` are wrapped** in `withAuth()`/`withAdmin()` from `lib/api-auth.ts`, or listed in `test/public-routes.ts` with a reason. `proxy.ts` only guards `/dashboard/*`.
2. **App code never builds queries on `db` directly** — `scopedTo(userId)` for user-owned tables, a `lib/db/*.ts` helper for anything else.
3. **Every `z.string()`/`z.array()` in an action input is bounded** with `.max()`.
4. **Client components never import server-only modules** (`@/lib/db`, `@/lib/billing`, `@/lib/email`, `@/lib/log`, …).
5. **No dependency that duplicates a platform capability** — no second auth system, ORM, UI kit, mailer, or storage client.
6. **App code reads config through `lib/env.ts`**, not raw `process.env`.

Plus, by convention:

- Reuse `getSession()` / `getCurrentUser()` from `lib/auth.ts` — never parse cookies yourself.
- SDK calls that render UI must degrade gracefully: catch `BuildspaceError`, log, show an empty state (see `app/dashboard/files/page.tsx`).
- Log through `log.*` from `lib/log.ts` (structured + secret-redacting), not `console.*` — biome enforces this.
- New env vars go through `lib/env.ts` **and** `.env.example` with a comment.
- Every dashboard slice has a `loading.tsx` (enforced).
- Deleting a feature = delete the slice folder, its nav entry, and its table (via a new migration, with `--allow-destructive` since dropping a table is destructive by definition). Keep slices self-contained so this stays true.

**Escape hatch.** Any enforced rule can be excused on a specific line with `// guardrail-ok: <reason>` on or directly above it (see `app/api/health/route.ts`). Use it when you're genuinely right. A false positive is a bug in the rule — fix the rule, don't delete the test.
