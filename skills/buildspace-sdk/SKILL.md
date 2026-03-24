---
name: buildspace-sdk
description: Implements features using the BuildSpace SDK (@buildspacestudio/sdk) for authentication, event tracking, file storage, email notifications, and managed database connections. Use when adding or modifying auth flows, tracking user events, handling file uploads, sending transactional emails, or setting up and querying the app database in your Next.js project.
---

# BuildSpace SDK

## Entry points

**Server** (API routes, server actions, server components):

```ts
import { getServerClient } from "@/lib/buildspace";
const bs = getServerClient();
```

**Browser** (client components):

```ts
import { getBrowserClient } from "@/lib/buildspace-client";
const bs = getBrowserClient();
```

Never expose `BUILDSPACE_SECRET_KEY` to the browser. The server singleton uses `bs_sec_*` keys; the browser singleton uses `bs_pub_*` keys.

## Environment variables

```
BUILDSPACE_SECRET_KEY=bs_sec_...
NEXT_PUBLIC_BUILDSPACE_PUBLISHABLE_KEY=bs_pub_...
BUILDSPACE_DB_URL=libsql://...
BUILDSPACE_DB_TOKEN=...
```

`BUILDSPACE_SECRET_KEY` and `NEXT_PUBLIC_BUILDSPACE_PUBLISHABLE_KEY` come from the BuildSpace dashboard. `BUILDSPACE_DB_URL` and `BUILDSPACE_DB_TOKEN` are auto-injected into your Railway service when the app is created — you do not need to set them manually in production.

Copy `.env.example` to `.env.local` and fill in keys from the BuildSpace dashboard. For local database development, you can omit the DB env vars and fall back to a local SQLite file (see Database section below).

## Authentication

### Auth workflow

1. Generate a sign-in or sign-up URL and redirect the user:

```ts
const signInUrl = bs.auth.getSignInUrl({
  redirectUri: `${origin}/api/auth/callback`,
});
const signUpUrl = bs.auth.getSignUpUrl({
  redirectUri: `${origin}/api/auth/callback`,
});
```

2. BuildSpace redirects back to `/api/auth/callback` with an authorization code.

3. The callback route exchanges the code for a token:

```ts
const { access_token, expires_in, user } = await bs.auth.handleCallback(
  request.url,
  { redirectUri: `${origin}/api/auth/callback` },
);
```

4. Store `access_token` in an HTTP-only cookie:

```ts
const jar = await cookies();
jar.set("bs_session", access_token, {
  httpOnly: true,
  secure: process.env.NODE_ENV === "production",
  sameSite: "lax",
  maxAge: expires_in,
  path: "/",
});
```

5. Validate sessions on each request:

```ts
const session = await bs.auth.getSession(token);
// Returns { user: { id, email, name }, appId } or null
```

6. Logout — revoke and delete cookie:

```ts
await bs.auth.revokeSession(token);
```

### Session forwarding

Attach a user session to scope subsequent SDK calls to that user:

```ts
bs.setSession(token);
await bs.storage.getSignedUrl("exports/report.pdf");
bs.clearSession();
```

### Existing auth routes

| Route                            | Purpose                                    |
| -------------------------------- | ------------------------------------------ |
| `app/api/auth/callback/route.ts` | Exchanges auth code for token, sets cookie |
| `app/api/auth/session/route.ts`  | Validates session from cookie              |
| `app/api/auth/logout/route.ts`   | Revokes session, clears cookie             |

## Events

**Browser** — batched with auto-flush:

```ts
bs.events.track("page_viewed", { path: "/pricing" });
await bs.events.flush();
await bs.events.shutdown();
```

**Server** — immediate, awaitable:

```ts
await bs.events.track("user_signed_up", { plan: "pro" }, userId);
await bs.events.batchTrack([
  { event: "item_purchased", properties: { sku: "ABC" }, actor_id: userId },
]);
```

## Storage

**Browser**:

```ts
const { key, url } = await bs.storage.upload(file, { path: "avatars/pic.png" });
const { url } = await bs.storage.getUrl("avatars/pic.png");
const { objects } = await bs.storage.list("avatars/");
await bs.storage.delete("avatars/pic.png");
```

**Server**:

```ts
const { upload_url, key } = await bs.storage.getUploadUrl({
  key: "reports/q4.pdf",
  contentType: "application/pdf",
  size: fileBytes,
});
const { url } = await bs.storage.getSignedUrl("reports/q4.pdf", {
  expiresIn: 3600,
});
const { objects } = await bs.storage.list("reports/", { limit: 50, offset: 0 });
await bs.storage.delete("reports/q4.pdf");
const usage = await bs.storage.getUsage();
```

## Notifications (server only)

```ts
await bs.notifications.send({
  to: "user@example.com",
  subject: "Welcome!",
  html: "<h1>Hello</h1>",
  text: "Hello (plain text)",
  replyTo: "support@you.com",
  metadata: { userId: "123" },
});

await bs.notifications.sendTemplate("welcome-email", {
  to: "user@example.com",
  variables: { name: "Jane", plan: "Pro" },
});
```

## Database

Every Buildspace app gets a managed [Turso](https://turso.tech) (libSQL) database — one per environment (`dev` and `prod`). The database is **not** accessed through the Buildspace SDK; you connect directly using `@libsql/client` and optionally [Drizzle ORM](https://orm.drizzle.team).

### How it works

1. **Provisioning** — A Turso database is created for each environment when you create an app in Creator Studio.
2. **Credentials** — Two env vars are injected into your Railway service automatically:
   - `BUILDSPACE_DB_URL` — libSQL connection URL
   - `BUILDSPACE_DB_TOKEN` — auth token (JWT)
3. **Connection** — Your app connects directly to Turso using `@libsql/client`. No SDK wrapper needed.
4. **Schema ownership** — You define and manage your own tables and migrations. The platform does not inject or manage any tables.

### Install dependencies

```bash
npm install @libsql/client drizzle-orm
npm install -D drizzle-kit
# or: pnpm add @libsql/client drizzle-orm && pnpm add -D drizzle-kit
# or: bun add @libsql/client drizzle-orm && bun add -D drizzle-kit
```

### Create the database client

Create `lib/db.ts`:

```ts
import { createClient } from "@libsql/client";
import { drizzle } from "drizzle-orm/libsql";

const client = createClient({
  url: process.env.BUILDSPACE_DB_URL ?? "file:local.db",
  authToken: process.env.BUILDSPACE_DB_TOKEN,
});

export const db = drizzle(client);
```

The `file:local.db` fallback lets you develop locally without connecting to Turso.

### Define a schema

Create `schema.ts` (or `lib/schema.ts`):

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

Pass the schema to Drizzle for type-safe queries:

```ts
import { drizzle } from "drizzle-orm/libsql";
import * as schema from "./schema";

export const db = drizzle(client, { schema });
```

### Drizzle config

Create `drizzle.config.ts` at the project root:

```ts
import { defineConfig } from "drizzle-kit";

export default defineConfig({
  schema: "./schema.ts",
  dialect: "turso",
  dbCredentials: {
    url: process.env.BUILDSPACE_DB_URL!,
    authToken: process.env.BUILDSPACE_DB_TOKEN!,
  },
});
```

### Push schema or run migrations

```bash
npx drizzle-kit push       # Apply schema directly (no migration files)
npx drizzle-kit generate   # Generate a migration file
npx drizzle-kit migrate    # Apply pending migrations
```

### Key points

- Turso uses libSQL (SQLite-compatible) — use SQLite column types (`integer`, `text`, `real`, `blob`)
- Dev and prod are separate databases with separate tokens
- Token rotation is available from the Data tab in Creator Studio; rotated tokens auto-sync to Railway

## Error handling

```ts
import { BuildspaceError } from "@buildspacestudio/sdk";

try {
  await bs.auth.getSession(token);
} catch (err) {
  if (err instanceof BuildspaceError) {
    console.error(err.code, err.service, err.status, err.message);
  }
}
```

`BuildspaceError` fields: `code` (string), `service` (`"auth" | "events" | "storage" | "notifications"`), `status` (HTTP status), `message`.

## Project files

| Path                             | Purpose                                                |
| -------------------------------- | ------------------------------------------------------ |
| `lib/buildspace.ts`              | Server SDK singleton                                   |
| `lib/buildspace-client.ts`       | Browser SDK singleton                                  |
| `lib/auth.ts`                    | Shared `getSession()` helper (cookie + SDK validation) |
| `lib/db.ts`                      | Drizzle database client (Turso/libSQL)                 |
| `schema.ts` (or `lib/schema.ts`) | Drizzle table definitions                              |
| `drizzle.config.ts`              | Drizzle Kit configuration for push/migrations          |
| `app/api/auth/callback/route.ts` | OAuth callback handler                                 |
| `app/api/auth/session/route.ts`  | Session validation                                     |
| `app/api/auth/logout/route.ts`   | Logout + revocation                                    |

## Additional resources

For preferred patterns and recipes (AuthProvider, route protection, server actions, event tracking, storage, notifications, database CRUD), see the [buildspace-examples skill](../buildspace-examples/SKILL.md).

For complete method signatures, configuration options, and client vs server availability, see [api-reference.md](api-reference.md).

For latest upstream docs, fetch: `https://docs.buildspace.studio/llms-full.txt`
