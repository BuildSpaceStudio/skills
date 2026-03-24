# Database Setup and CRUD

Every Buildspace app gets a managed Turso (libSQL) database — one per environment. Access it directly via `@libsql/client` + Drizzle ORM (not through the Buildspace SDK).

## Setup

Install dependencies:

```bash
npm install @libsql/client drizzle-orm
npm install -D drizzle-kit
# or: pnpm add @libsql/client drizzle-orm && pnpm add -D drizzle-kit
# or: bun add @libsql/client drizzle-orm && bun add -D drizzle-kit
```

Create `lib/db.ts`:

```ts
import { createClient } from "@libsql/client";
import { drizzle } from "drizzle-orm/libsql";
import * as schema from "./schema";

const client = createClient({
  url: process.env.BUILDSPACE_DB_URL ?? "file:local.db",
  authToken: process.env.BUILDSPACE_DB_TOKEN,
});

export const db = drizzle(client, { schema });
```

The `file:local.db` fallback enables local development without a remote Turso connection.

## Schema definition

Create `lib/schema.ts` using SQLite column types (`integer`, `text`, `real`, `blob`) — Turso is SQLite-compatible:

```ts
import { integer, sqliteTable, text } from "drizzle-orm/sqlite-core";

export const todos = sqliteTable("todos", {
  id: integer("id").primaryKey({ autoIncrement: true }),
  userId: text("user_id").notNull(),
  text: text("text").notNull(),
  completed: integer("completed").notNull().default(0),
  createdAt: text("created_at")
    .notNull()
    .$defaultFn(() => new Date().toISOString()),
});
```

## Drizzle config

Create `drizzle.config.ts` at the project root:

```ts
import { defineConfig } from "drizzle-kit";

export default defineConfig({
  schema: "./lib/schema.ts",
  dialect: "turso",
  dbCredentials: {
    url: process.env.BUILDSPACE_DB_URL!,
    authToken: process.env.BUILDSPACE_DB_TOKEN!,
  },
});
```

Push schema:

```bash
npx drizzle-kit push
```

## CRUD server actions

Requires `next-safe-action` — see [server-actions.md](server-actions.md) for setup.

```ts
"use server";

import { z } from "zod";
import { eq } from "drizzle-orm";
import { authActionClient } from "@/lib/safe-action";
import { db } from "@/lib/db";
import { todos } from "@/lib/schema";

export const getTodos = authActionClient.action(async ({ ctx }) => {
  return await db
    .select()
    .from(todos)
    .where(eq(todos.userId, ctx.session.user.id));
});

export const createTodo = authActionClient
  .inputSchema(z.object({ text: z.string().min(1).max(500) }))
  .action(async ({ parsedInput, ctx }) => {
    const [todo] = await db
      .insert(todos)
      .values({ userId: ctx.session.user.id, text: parsedInput.text })
      .returning();
    return todo;
  });

export const toggleTodo = authActionClient
  .inputSchema(z.object({ id: z.number(), completed: z.boolean() }))
  .action(async ({ parsedInput }) => {
    await db
      .update(todos)
      .set({ completed: parsedInput.completed ? 1 : 0 })
      .where(eq(todos.id, parsedInput.id));
    return { success: true };
  });

export const deleteTodo = authActionClient
  .inputSchema(z.object({ id: z.number() }))
  .action(async ({ parsedInput }) => {
    await db.delete(todos).where(eq(todos.id, parsedInput.id));
    return { success: true };
  });
```

## Database in API routes

For cases where server actions aren't suitable (streaming, external callers):

```ts
import { type NextRequest, NextResponse } from "next/server";
import { eq } from "drizzle-orm";
import { getSession } from "@/lib/auth";
import { db } from "@/lib/db";
import { todos } from "@/lib/schema";

export async function GET(request: NextRequest) {
  const session = await getSession();
  if (!session) {
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
  }

  const userTodos = await db
    .select()
    .from(todos)
    .where(eq(todos.userId, session.user.id));

  return NextResponse.json(userTodos);
}
```

## Key points

- `BUILDSPACE_DB_URL` and `BUILDSPACE_DB_TOKEN` are auto-injected in deployed environments — no manual setup needed
- Dev and prod get separate databases with separate tokens
- Use `npx drizzle-kit push` to apply schema changes, or `generate` + `migrate` for versioned migrations
- Token rotation is available from the Data tab in Creator Studio
