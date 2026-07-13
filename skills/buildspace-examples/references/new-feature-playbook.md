# New Feature Playbook

The exact checklist for adding a feature to this app. Features are **vertical slices**: one folder under `app/dashboard/<slice>/` owning its page, actions, and components, plus (usually) one table. `app/dashboard/todos/` is the reference slice — when in doubt, copy its shape.

## Checklist

1. **Schema** — add the table to `lib/db/schema.ts` (SQLite types, text UUID PK via `$defaultFn(() => crypto.randomUUID())`, `userId` column, ISO-string `createdAt`). Export `$inferSelect`/`$inferInsert` types.
2. **Migration** — `bun db:generate`, review the SQL in `drizzle/`, then `bun db:migrate`. Never hand-write migration files.
3. **Actions** — `app/dashboard/<slice>/actions.ts` with `"use server"`. Use `authActionClient` (or `adminActionClient` for admin features) + zod `.inputSchema(...)`. Scope every query by `ctx.session.user.id`. Call `revalidatePath("/dashboard/<slice>")` after mutations.
4. **Page** — `app/dashboard/<slice>/page.tsx` as an async server component: `getSession()` guard (`if (!session) redirect("/")`), query via `db`, render `<PageHeader>` + content. Use `<EmptyState>` for the zero state.
5. **Client components** — colocate in the slice folder, `"use client"` only where there's interactivity. Call actions with `useAction` from `next-safe-action/hooks`; toast on success/error; disable controls on `isPending`.
6. **Loading state** — `app/dashboard/<slice>/loading.tsx` with `Skeleton` blocks matching the page layout (copy an existing one).
7. **Nav entry** — add the route to `components/dashboard-nav.tsx`. Nothing else is needed for it to appear.
8. **Track an event** — `await trackEvent({ event: "<thing>_created", properties, userId: ctx.session.user.id })` from `lib/analytics.ts` in the primary mutation.
9. **UI kit first** — check `components/ui/` before writing markup; add missing primitives there in the same cva style. Never install a component library.
10. **Verify** — `bun lint && bun typecheck && bun build` must all pass.

## Guardrails

- Reuse `getSession()` / `getCurrentUser()` from `lib/auth.ts` — never parse cookies yourself.
- SDK calls that render UI must degrade gracefully: catch `BuildspaceError`, log, show an empty state (see `app/dashboard/files/page.tsx`).
- New env vars go through `lib/env.ts` **and** `.env.example` with a comment.
- Deleting a feature = delete the slice folder, its nav entry, and its table (via a new migration). Keep slices self-contained so this stays true.
