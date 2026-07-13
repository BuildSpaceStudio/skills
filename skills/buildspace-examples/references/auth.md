# Authentication & Route Protection

All of these patterns are already implemented in this project. Point at the real files and extend them — don't recreate them.

## Auth wiring map

| Concern | Lives at |
|---------|----------|
| Client auth state (`useAuth`) | `components/auth-provider.tsx` |
| Provider mounted with server-fetched `initialUser` | `app/layout.tsx` |
| Server session validation | `lib/auth.ts` → `getSession()` |
| Session + local role in one call | `lib/auth.ts` → `getCurrentUser()` |
| Local user mirror (upsert on sign-in) | `lib/db/users.ts` |
| OAuth callback + first-sign-in side effects | `app/api/auth/callback/route.ts` |
| Session check / logout API routes | `app/api/auth/session/route.ts`, `app/api/auth/logout/route.ts` |
| Route-level protection for `/dashboard/*` | `proxy.ts` |
| Auth-aware header (sign in/out, user menu) | `components/site-header.tsx` |

## AuthProvider + useAuth hook

`components/auth-provider.tsx` exposes `user`, `loading`, `signIn`, `signUp`, `signOut` via `useAuth()`. It's mounted in `app/layout.tsx` with `initialUser` fetched server-side (no loading flash). Client components opt in:

```tsx
"use client";
import { useAuth } from "@/components/auth-provider";

function AccountBadge() {
  const { user, signIn } = useAuth();
  if (!user) return <button onClick={signIn}>Sign in</button>;
  return <span>{user.email}</span>;
}
```

`components/site-header.tsx` is the full working example (avatar menu, sign in/up buttons).

## Server-side session checks

Every server component page under `/dashboard` guards itself:

```tsx
const session = await getSession();
if (!session) redirect("/");
```

When you also need the local role or profile record, use `getCurrentUser()` — see `app/dashboard/admin/page.tsx` for the role-gated version (`if (current.role !== "super_admin") redirect("/dashboard")`).

## Local user mirror

The `users` table mirrors BuildSpace identity: `app/api/auth/callback/route.ts` calls `upsertUserFromSession()` (from `lib/db/users.ts`) after the token exchange. On **first** sign-in it also fires the `user_signed_up` event and the welcome email — that callback is the one place for signup side effects. Hang app data (roles, preferences, avatar keys) off this local row, keyed by `buildspaceUserId`.

## Route protection with proxy.ts

`proxy.ts` at the project root (Next.js 16 convention — NOT `middleware.ts`) does a fast cookie-presence check for `/dashboard/:path*` and redirects to `/` when the `bs_session` cookie is missing. No SDK call — full validation happens in pages, server actions, and API routes. If you add a protected area outside `/dashboard`, extend the `matcher` there.

## Protected API route pattern

Use when an API route (streaming, webhooks, external callers) requires authentication:

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
    // ... route logic ...
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
