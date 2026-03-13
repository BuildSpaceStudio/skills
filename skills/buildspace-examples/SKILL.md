---
name: buildspace-examples
description: Preferred patterns and recipes for building features with the BuildSpace SDK and managed database in your Next.js project. Use when adding authentication UI, route protection, server actions, event tracking, file storage, email notifications, database setup and queries, or protected API routes.
---

# BuildSpace Patterns & Recipes

Preferred patterns for building with the BuildSpace SDK. Each recipe is self-contained with complete file contents. For SDK method signatures and options, see the [SDK skill](../buildspace-sdk/SKILL.md) and [API reference](../buildspace-sdk/api-reference.md).

## Conventions

- Assumes Next.js 16 App Router, TypeScript, Tailwind CSS
- `getSession()` from `lib/auth.ts` is the shared session helper -- use it in all server-side auth checks
- `getServerClient()` for server-side SDK access, `getBrowserClient()` for client-side
- Prefer **server actions** for mutations; use **API routes** for streaming responses, webhooks, and OAuth callbacks
- Use `proxy.ts` (Next.js 16) for route-level protection -- NOT `middleware.ts`
- Always catch `BuildspaceError` in server-side code and return structured error responses
- Use `bs.setSession(token)` / `bs.clearSession()` to scope SDK calls to the authenticated user

---

## Recipe: AuthProvider + useAuth hook

**When to use:** Your app needs auth state in client components (e.g. showing user name, sign in/out buttons, conditional UI).

**Create `components/auth-provider.tsx`:**

```tsx
"use client";

import { createContext, useCallback, useContext, useState } from "react";
import { getBrowserClient } from "@/lib/buildspace-client";

interface User {
  id: string;
  email: string;
  name: string | null;
}

interface AuthContextValue {
  user: User | null;
  loading: boolean;
  signIn: () => void;
  signUp: () => void;
  signOut: () => Promise<void>;
}

const AuthContext = createContext<AuthContextValue>({
  user: null,
  loading: true,
  signIn: () => {},
  signUp: () => {},
  signOut: async () => {},
});

export function useAuth() {
  return useContext(AuthContext);
}

export function AuthProvider({
  children,
  initialUser,
}: {
  children: React.ReactNode;
  initialUser: User | null;
}) {
  const [user, setUser] = useState<User | null>(initialUser);
  const [loading] = useState(false);

  const signIn = useCallback(() => {
    try {
      const bs = getBrowserClient();
      window.location.href = bs.auth.getSignInUrl({
        redirectUri: `${window.location.origin}/api/auth/callback`,
      });
    } catch {
      console.error("Set NEXT_PUBLIC_BUILDSPACE_PUBLISHABLE_KEY to enable auth");
    }
  }, []);

  const signUp = useCallback(() => {
    try {
      const bs = getBrowserClient();
      window.location.href = bs.auth.getSignUpUrl({
        redirectUri: `${window.location.origin}/api/auth/callback`,
      });
    } catch {
      console.error("Set NEXT_PUBLIC_BUILDSPACE_PUBLISHABLE_KEY to enable auth");
    }
  }, []);

  const signOut = useCallback(async () => {
    await fetch("/api/auth/logout", { method: "POST" });
    setUser(null);
  }, []);

  return (
    <AuthContext value={{ user, loading, signIn, signUp, signOut }}>
      {children}
    </AuthContext>
  );
}
```

**Update `app/layout.tsx`** to wrap with the provider. Make the layout async so the session is fetched server-side (no client-side loading spinner on first paint):

```tsx
import type { Metadata } from "next";
import { getSession } from "@/lib/auth";
import { AuthProvider } from "@/components/auth-provider";
import { ThemeProvider } from "@/components/theme-provider";
// ... font imports ...

export default async function RootLayout({
  children,
}: Readonly<{ children: React.ReactNode }>) {
  const session = await getSession();

  return (
    <html lang="en" suppressHydrationWarning>
      <body className={/* font variables */}>
        <ThemeProvider>
          <AuthProvider initialUser={session?.user ?? null}>
            {children}
          </AuthProvider>
        </ThemeProvider>
      </body>
    </html>
  );
}
```

**Usage in any client component:**

```tsx
"use client";
import { useAuth } from "@/components/auth-provider";

export function UserGreeting() {
  const { user, signIn, signOut } = useAuth();

  if (!user) {
    return <button onClick={signIn}>Sign in</button>;
  }

  return (
    <div>
      <span>Hello, {user.name ?? user.email}</span>
      <button onClick={signOut}>Sign out</button>
    </div>
  );
}
```

The AuthProvider is flexible -- it does not force authentication on any page. Components opt-in by calling `useAuth()`. Pages that don't need auth simply ignore it.

---

## Recipe: Route protection with proxy.ts

**When to use:** Certain pages should hard-redirect unauthenticated users to the home page (or a login page).

**Create `proxy.ts` at the project root** (Next.js 16 convention):

```ts
import { NextResponse } from "next/server";

export function proxy(request: Request) {
  const url = new URL(request.url);
  const cookie = request.headers.get("cookie") ?? "";
  const hasSession = cookie.includes("bs_session=");

  if (!hasSession) {
    return NextResponse.redirect(new URL("/", url.origin));
  }

  return NextResponse.next();
}

export const config = {
  matcher: [
    "/dashboard/:path*",
    // Add more protected routes here:
    // "/settings/:path*",
    // "/account/:path*",
  ],
};
```

This is a fast cookie-presence check -- no SDK call. Full session validation happens when server actions or API routes are called. The proxy runs on the Node.js runtime by default in Next.js 16.

**Important:** The file must be named `proxy.ts` (not `middleware.ts`) and the export must be `function proxy` (not `function middleware`).

---

## Recipe: Server actions with next-safe-action

**When to use:** Your app needs validated mutations called from client components -- form submissions, data updates, or any operation that should run on the server with type-safe input validation.

**Install dependencies:**

```bash
bun add next-safe-action zod
```

**Create `lib/safe-action.ts`:**

```ts
import { createSafeActionClient } from "next-safe-action";
import { getSession } from "@/lib/auth";

export const actionClient = createSafeActionClient();

export const authActionClient = actionClient.use(async ({ next }) => {
  const session = await getSession();
  if (!session) throw new Error("Unauthorized");
  return next({ ctx: { session } });
});
```

`actionClient` requires no auth (use for public forms). `authActionClient` validates the session and injects `ctx.session` (with `user` and `token`) into every action.

**Example server action (`app/actions/example.ts`):**

```ts
"use server";

import { z } from "zod";
import { authActionClient } from "@/lib/safe-action";
import { getServerClient } from "@/lib/buildspace";

export const updateProfile = authActionClient
  .inputSchema(
    z.object({
      name: z.string().min(1).max(100),
    }),
  )
  .action(async ({ parsedInput, ctx }) => {
    const bs = getServerClient();
    bs.setSession(ctx.session.token);
    // ... use the SDK with user-scoped access ...
    bs.clearSession();
    return { success: true };
  });
```

**Client usage with `useAction` hook:**

```tsx
"use client";
import { useAction } from "next-safe-action/hooks";
import { updateProfile } from "@/app/actions/example";

export function ProfileForm() {
  const { execute, isExecuting, result } = useAction(updateProfile, {
    onSuccess: ({ data }) => {
      // handle success
    },
    onError: ({ error }) => {
      // handle validation or server errors
    },
  });

  return (
    <form onSubmit={(e) => { e.preventDefault(); execute({ name: "Jane" }); }}>
      <input name="name" />
      <button type="submit" disabled={isExecuting}>
        {isExecuting ? "Saving..." : "Save"}
      </button>
      {result.validationErrors && <p>Check your input</p>}
    </form>
  );
}
```

### When to use server actions vs API routes

| Use server actions for | Use API routes for |
|------------------------|--------------------|
| Form submissions | Streaming responses (AI chat) |
| CRUD mutations | Webhook endpoints (incoming) |
| Triggering notifications | OAuth callbacks (redirects) |
| Any user-initiated action | Endpoints called by external services |

---

## Recipe: Event tracking

**When to use:** Your app needs analytics or user behavior tracking.

### Browser events (simplest)

Track directly from client components -- events are batched and auto-flushed:

```tsx
"use client";
import { getBrowserClient } from "@/lib/buildspace-client";

function TrackableButton() {
  const handleClick = () => {
    const bs = getBrowserClient();
    bs.events.track("button_clicked", { page: "pricing", variant: "cta" });
  };

  return <button onClick={handleClick}>Get Started</button>;
}
```

### Server-side events via server action

For events that must be tracked server-side (e.g. after a purchase, sign-up). Requires `next-safe-action` (see recipe above).

**Create `app/actions/events.ts`:**

```ts
"use server";

import { z } from "zod";
import { authActionClient } from "@/lib/safe-action";
import { getServerClient } from "@/lib/buildspace";

export const trackEvent = authActionClient
  .inputSchema(
    z.object({
      event: z.string().min(1),
      properties: z.record(z.unknown()).optional(),
    }),
  )
  .action(async ({ parsedInput, ctx }) => {
    const bs = getServerClient();
    bs.setSession(ctx.session.token);
    const result = await bs.events.track(
      parsedInput.event,
      parsedInput.properties,
      ctx.session.user.id,
    );
    bs.clearSession();
    return result;
  });
```

### Server-side events via API route

For external callers or cases where a server action isn't suitable.

**Create `app/api/events/route.ts`:**

```ts
import { type NextRequest, NextResponse } from "next/server";
import { getSession } from "@/lib/auth";
import { getServerClient } from "@/lib/buildspace";
import { BuildspaceError } from "@buildspacestudio/sdk";

export async function POST(request: NextRequest) {
  const session = await getSession();
  if (!session) {
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
  }

  const body = await request.json();
  if (!body.event || typeof body.event !== "string") {
    return NextResponse.json({ error: "Missing required field: event" }, { status: 400 });
  }

  try {
    const bs = getServerClient();
    bs.setSession(session.token);
    const result = await bs.events.track(body.event, body.properties, session.user.id);
    bs.clearSession();
    return NextResponse.json(result);
  } catch (err) {
    if (err instanceof BuildspaceError) {
      return NextResponse.json({ error: err.message, code: err.code }, { status: err.status });
    }
    throw err;
  }
}
```

---

## Recipe: File storage

**When to use:** Your app needs file uploads, downloads, or file listing.

### Browser upload (simplest)

Upload directly from the browser -- no API route needed:

```tsx
"use client";
import { getBrowserClient } from "@/lib/buildspace-client";

function FileUploader() {
  const handleUpload = async (e: React.ChangeEvent<HTMLInputElement>) => {
    const file = e.target.files?.[0];
    if (!file) return;

    const bs = getBrowserClient();
    const { key, url } = await bs.storage.upload(file, {
      path: `uploads/${file.name}`,
    });
    console.log("Uploaded:", key, url);
  };

  return <input type="file" onChange={handleUpload} />;
}
```

### Server actions for listing and signed URLs

Requires `next-safe-action` (see recipe above).

**Create `app/actions/storage.ts`:**

```ts
"use server";

import { z } from "zod";
import { authActionClient } from "@/lib/safe-action";
import { getServerClient } from "@/lib/buildspace";

export const listFiles = authActionClient
  .inputSchema(
    z.object({
      prefix: z.string().optional(),
      limit: z.number().optional(),
      offset: z.number().optional(),
    }),
  )
  .action(async ({ parsedInput, ctx }) => {
    const bs = getServerClient();
    bs.setSession(ctx.session.token);
    const result = await bs.storage.list(parsedInput.prefix, {
      limit: parsedInput.limit,
      offset: parsedInput.offset,
    });
    bs.clearSession();
    return result;
  });

export const getUploadUrl = authActionClient
  .inputSchema(
    z.object({
      key: z.string().min(1),
      contentType: z.string().min(1),
      size: z.number().positive(),
    }),
  )
  .action(async ({ parsedInput, ctx }) => {
    const bs = getServerClient();
    bs.setSession(ctx.session.token);
    const result = await bs.storage.getUploadUrl(parsedInput);
    bs.clearSession();
    return result;
  });

export const getSignedUrl = authActionClient
  .inputSchema(
    z.object({
      key: z.string().min(1),
      expiresIn: z.number().optional(),
    }),
  )
  .action(async ({ parsedInput, ctx }) => {
    const bs = getServerClient();
    bs.setSession(ctx.session.token);
    const result = await bs.storage.getSignedUrl(parsedInput.key, {
      expiresIn: parsedInput.expiresIn,
    });
    bs.clearSession();
    return result;
  });
```

---

## Recipe: Email notifications

**When to use:** Your app needs to send transactional emails (welcome emails, receipts, alerts).

Notifications are server-only (requires the secret key). Requires `next-safe-action` (see recipe above).

**Create `app/actions/notifications.ts`:**

```ts
"use server";

import { z } from "zod";
import { authActionClient } from "@/lib/safe-action";
import { getServerClient } from "@/lib/buildspace";

export const sendNotification = authActionClient
  .inputSchema(
    z.object({
      to: z.union([z.string().email(), z.array(z.string().email())]),
      subject: z.string().min(1),
      html: z.string().min(1),
      text: z.string().optional(),
      replyTo: z.string().email().optional(),
      metadata: z.record(z.string()).optional(),
    }),
  )
  .action(async ({ parsedInput }) => {
    const bs = getServerClient();
    return await bs.notifications.send(parsedInput);
  });

export const sendTemplateNotification = authActionClient
  .inputSchema(
    z.object({
      templateSlug: z.string().min(1),
      to: z.union([z.string().email(), z.array(z.string().email())]),
      variables: z.record(z.string()).optional(),
      metadata: z.record(z.string()).optional(),
    }),
  )
  .action(async ({ parsedInput }) => {
    const bs = getServerClient();
    const { templateSlug, ...options } = parsedInput;
    return await bs.notifications.sendTemplate(templateSlug, options);
  });
```

---

## Recipe: Database setup and CRUD

**When to use:** Your app needs to persist data. Every Buildspace app gets a managed Turso (libSQL) database — one per environment. The database is accessed directly via `@libsql/client` + Drizzle ORM, not through the Buildspace SDK.

**Install dependencies:**

```bash
bun add @libsql/client drizzle-orm
bun add -D drizzle-kit
```

**Create `lib/db.ts`:**

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

**Create `lib/schema.ts`:**

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

Use SQLite column types (`integer`, `text`, `real`, `blob`) — Turso is SQLite-compatible.

**Create `drizzle.config.ts` at the project root:**

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

**Push your schema:**

```bash
npx drizzle-kit push
```

### CRUD server actions

Requires `next-safe-action` (see recipe above).

**Create `app/actions/todos.ts`:**

```ts
"use server";

import { z } from "zod";
import { eq } from "drizzle-orm";
import { authActionClient } from "@/lib/safe-action";
import { db } from "@/lib/db";
import { todos } from "@/lib/schema";

export const getTodos = authActionClient
  .action(async ({ ctx }) => {
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

### Database in API routes

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

### Key points

- `BUILDSPACE_DB_URL` and `BUILDSPACE_DB_TOKEN` are auto-injected in deployed environments — no manual setup needed
- Dev and prod get separate databases with separate tokens
- Use `npx drizzle-kit push` to apply schema changes, or `generate` + `migrate` for versioned migrations
- Token rotation is available from the Data tab in Creator Studio

---

## Recipe: Protect an API route with auth

**When to use:** You need an API route (for streaming, webhooks, or external callers) and it must require authentication.

**Pattern:**

```ts
import { type NextRequest, NextResponse } from "next/server";
import { getSession } from "@/lib/auth";
import { getServerClient } from "@/lib/buildspace";
import { BuildspaceError } from "@buildspacestudio/sdk";

export async function POST(request: NextRequest) {
  const session = await getSession();
  if (!session) {
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
  }

  try {
    const bs = getServerClient();
    bs.setSession(session.token);

    // ... your logic here ...

    bs.clearSession();
    return NextResponse.json({ success: true });
  } catch (err) {
    if (err instanceof BuildspaceError) {
      return NextResponse.json(
        { error: err.message, code: err.code },
        { status: err.status },
      );
    }
    throw err;
  }
}
```

Key points:
- Call `getSession()` first and return 401 if null
- Use `bs.setSession(token)` to scope SDK operations to the authenticated user
- Always call `bs.clearSession()` after (the server client is a singleton)
- Catch `BuildspaceError` and return structured JSON with the error code and HTTP status

---

## Additional resources

For latest upstream docs, fetch: `https://docs.buildspace.studio/llms-full.txt`
