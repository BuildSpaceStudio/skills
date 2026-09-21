# Database Setup and CRUD

Every BuildSpace app gets a managed Turso (libSQL) database — one per environment. Access it directly via `@libsql/client` + Drizzle ORM (not through the BuildSpace SDK).

Setup already exists in this project — extend it, don't recreate it:

| Concern | Lives at |
|---------|----------|
| Drizzle client (server-only) | `lib/db/index.ts` — import as `import { db, schema } from "@/lib/db"` |
| Table definitions | `lib/db/schema.ts` (`users`, `todos`) |
| User-record helpers + every `users` query | `lib/db/users.ts` |
| Tenant-scoped query helpers | `lib/db/scoped.ts` — `scopedTo(userId)` |
| Drizzle Kit config | `drizzle.config.ts` |
| Versioned migrations | `drizzle/` (generated — never hand-edit) |
| Local-dev seed | `lib/db/seed.ts` (refuses to run against remote DBs) |

## Schema changes

1. Edit `lib/db/schema.ts` using SQLite column types (`integer`, `text`, `real`, `blob`) — Turso is SQLite-compatible. Follow the existing shape: `text` UUID primary keys via `$defaultFn(() => crypto.randomUUID())`, ISO-string timestamps, `integer(..., { mode: "boolean" })` for booleans. Export the inferred types (`$inferSelect` / `$inferInsert`).
2. `bun db:generate` — creates a migration in `drizzle/`. It **refuses destructive migrations** (DROP TABLE/COLUMN, table recreation) and rolls the generated files back, because deploys apply migrations automatically — a column rename that looks harmless in review would delete production data on ship. Renames go add (nullable) → backfill → drop, across two releases. When the data really is expendable (dropping a table you just removed from the schema), pass `--allow-destructive`.
3. `bun db:migrate` — applies it locally (`file:local.db` by default).

Use versioned migrations, not `drizzle-kit push` — deploys run `bun run db:migrate` automatically (see `railway.json` `preDeployCommand`), so committed migrations are how schema reaches production.

## CRUD server actions

`app/dashboard/todos/actions.ts` is the working example: zod input schema (every string bounded with `.max()`), all queries through `scopedTo(ctx.session.user.id)`, `revalidatePath` after each mutation. Copy that file's shape for new tables.

**App code never touches `db` directly** — `test/guardrails.test.ts` fails the build if it does. Tenant scoping is too important to depend on remembering it, so `scopedTo()` injects the owner predicate for you:

```ts
import { scopedTo } from "@/lib/db/scoped";

const mine = scopedTo(ctx.session.user.id);

const todo = await mine.insert(schema.todos, { text: parsedInput.text }); // userId stamped from the session
const one  = await mine.byId(schema.todos, parsedInput.id);               // null if it isn't theirs
await mine.update(schema.todos, parsedInput.id, { completed: true });     // 0 rows if it isn't theirs
await mine.delete(schema.todos, parsedInput.id);
```

For a table to work with `scopedTo()` it needs an `id` primary key and a `userId` column — the convention every slice table follows.

Queries that genuinely aren't user-scoped (admin views, lookups by another key like `users.buildspaceUserId`) go in a `lib/db/*.ts` helper instead, where the comment above the function states who is allowed to call it. `lib/db/users.ts` is the worked example.

## Reading data in server components

Same rule as writes — scope the read (see `app/dashboard/todos/page.tsx`):

```tsx
const todos = await scopedTo(session.user.id)
  .select(schema.todos)
  .orderBy(desc(schema.todos.createdAt));
```

## Database in API routes

For cases where server actions aren't suitable (streaming, external callers):

```ts
import { NextResponse } from "next/server";
import { withAuth } from "@/lib/api-auth";
import { schema } from "@/lib/db";
import { scopedTo } from "@/lib/db/scoped";

export const GET = withAuth(async (_request, { session }) => {
  const userTodos = await scopedTo(session.user.id).select(schema.todos);
  return NextResponse.json(userTodos);
});
```

`proxy.ts` only guards `/dashboard/*`, so an unwrapped route handler is public. See [auth.md](auth.md).

## Key points

- `BUILDSPACE_DB_URL` and `BUILDSPACE_DB_TOKEN` are auto-injected in deployed environments — no manual setup needed
- Dev and prod get separate databases with separate tokens
- `file:local.db` fallback enables local development without a remote Turso connection
- Migrations run automatically on deploy via `railway.json`'s `preDeployCommand`
- Token rotation is available from the Data tab in Creator Studio

## Standalone (non-project) databases

Beyond the per-environment database this project gets, Buildspace can provision **standalone** SQLite databases owned by your organization — for scratch data, prototypes, or data shared across projects. They are not wired into any project's env vars automatically.

```bash
buildspace db create "Scratch data"    # prints the connection URL + token ONCE
buildspace db list
buildspace db shell scratch-data --sql "select count(*) from users"
```

Connect the same way as the project database — the URL and token just come from `buildspace db create` instead of being injected:

```ts
const client = createClient({
  url: process.env.SCRATCH_DB_URL!,
  authToken: process.env.SCRATCH_DB_TOKEN!,
});
```

Creator Studio has a console for these (schema tree, row editor, SQL runner, and a schema-aware AI SQL helper) under **Databases**. Full reference: `https://docs.buildspace.studio/docs/database/standalone-databases`.
