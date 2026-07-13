# Database Setup and CRUD

Every BuildSpace app gets a managed Turso (libSQL) database — one per environment. Access it directly via `@libsql/client` + Drizzle ORM (not through the BuildSpace SDK).

Setup already exists in this project — extend it, don't recreate it:

| Concern | Lives at |
|---------|----------|
| Drizzle client (server-only) | `lib/db/index.ts` — import as `import { db, schema } from "@/lib/db"` |
| Table definitions | `lib/db/schema.ts` (`users`, `todos`) |
| User-record helpers | `lib/db/users.ts` |
| Drizzle Kit config | `drizzle.config.ts` |
| Versioned migrations | `drizzle/` (generated — never hand-edit) |
| Local-dev seed | `lib/db/seed.ts` (refuses to run against remote DBs) |

## Schema changes

1. Edit `lib/db/schema.ts` using SQLite column types (`integer`, `text`, `real`, `blob`) — Turso is SQLite-compatible. Follow the existing shape: `text` UUID primary keys via `$defaultFn(() => crypto.randomUUID())`, ISO-string timestamps, `integer(..., { mode: "boolean" })` for booleans. Export the inferred types (`$inferSelect` / `$inferInsert`).
2. `bun db:generate` — creates a migration in `drizzle/`.
3. `bun db:migrate` — applies it locally (`file:local.db` by default).

Use versioned migrations, not `drizzle-kit push` — deploys run `bun run db:migrate` automatically (see `railway.json` `preDeployCommand`), so committed migrations are how schema reaches production.

## CRUD server actions

`app/dashboard/todos/actions.ts` is the working example: zod input schema, insert/update/delete scoped by `ctx.session.user.id`, `revalidatePath` after each mutation. Copy that file's shape for new tables.

```ts
const [todo] = await db
  .insert(schema.todos)
  .values({ text: parsedInput.text, userId: ctx.session.user.id })
  .returning();
```

Always scope updates and deletes to the owner:

```ts
.where(and(eq(schema.todos.id, parsedInput.id), eq(schema.todos.userId, ctx.session.user.id)))
```

## Reading data in server components

Query directly in the page (see `app/dashboard/todos/page.tsx`):

```tsx
const todos = await db
  .select()
  .from(schema.todos)
  .where(eq(schema.todos.userId, session.user.id))
  .orderBy(desc(schema.todos.createdAt));
```

## Database in API routes

For cases where server actions aren't suitable (streaming, external callers):

```ts
import { type NextRequest, NextResponse } from "next/server";
import { eq } from "drizzle-orm";
import { getSession } from "@/lib/auth";
import { db, schema } from "@/lib/db";

export async function GET(request: NextRequest) {
  const session = await getSession();
  if (!session) {
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
  }

  const userTodos = await db
    .select()
    .from(schema.todos)
    .where(eq(schema.todos.userId, session.user.id));

  return NextResponse.json(userTodos);
}
```

## Key points

- `BUILDSPACE_DB_URL` and `BUILDSPACE_DB_TOKEN` are auto-injected in deployed environments — no manual setup needed
- Dev and prod get separate databases with separate tokens
- `file:local.db` fallback enables local development without a remote Turso connection
- Migrations run automatically on deploy via `railway.json`'s `preDeployCommand`
- Token rotation is available from the Data tab in Creator Studio
