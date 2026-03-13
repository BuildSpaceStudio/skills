---
name: buildspace-sdk
description: Implements features using the BuildSpace SDK (@buildspacestudio/sdk) for authentication, event tracking, file storage, and email notifications. Use when adding or modifying auth flows, tracking user events, handling file uploads, or sending transactional emails in your Next.js project.
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
```

Copy `.env.example` to `.env.local` and fill in keys from the BuildSpace dashboard.

## Authentication

### Auth workflow

1. Generate a sign-in or sign-up URL and redirect the user:

```ts
const signInUrl = bs.auth.getSignInUrl({ redirectUri: `${origin}/api/auth/callback` });
const signUpUrl = bs.auth.getSignUpUrl({ redirectUri: `${origin}/api/auth/callback` });
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

| Route | Purpose |
|-------|---------|
| `app/api/auth/callback/route.ts` | Exchanges auth code for token, sets cookie |
| `app/api/auth/session/route.ts` | Validates session from cookie |
| `app/api/auth/logout/route.ts` | Revokes session, clears cookie |

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
const { url } = await bs.storage.getSignedUrl("reports/q4.pdf", { expiresIn: 3600 });
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

| Path | Purpose |
|------|---------|
| `lib/buildspace.ts` | Server SDK singleton |
| `lib/buildspace-client.ts` | Browser SDK singleton |
| `lib/auth.ts` | Shared `getSession()` helper (cookie + SDK validation) |
| `app/api/auth/callback/route.ts` | OAuth callback handler |
| `app/api/auth/session/route.ts` | Session validation |
| `app/api/auth/logout/route.ts` | Logout + revocation |

## Additional resources

For preferred patterns and recipes (AuthProvider, route protection, server actions, event tracking, storage, notifications), see the [buildspace-examples skill](../buildspace-examples/SKILL.md).

For complete method signatures, configuration options, and client vs server availability, see [api-reference.md](api-reference.md).

For latest upstream docs, fetch: `https://docs.buildspace.studio/llms-full.txt`
