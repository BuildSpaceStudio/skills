# Authentication & Route Protection

## AuthProvider + useAuth hook

Use when your app needs auth state in client components (user name, sign in/out buttons, conditional UI).

Create `components/auth-provider.tsx` — a context provider that:

- Accepts `initialUser` (fetched server-side in the root layout) to avoid loading spinners on first paint
- Exposes `user`, `loading`, `signIn`, `signUp`, `signOut` via `useAuth()`
- Uses `getBrowserClient()` for sign-in/sign-up URLs with `redirectUri` pointing to `/api/auth/callback`
- Calls `POST /api/auth/logout` for sign-out and clears local user state

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
    const bs = getBrowserClient();
    window.location.href = bs.auth.getSignInUrl({
      redirectUri: `${window.location.origin}/api/auth/callback`,
    });
  }, []);

  const signUp = useCallback(() => {
    const bs = getBrowserClient();
    window.location.href = bs.auth.getSignUpUrl({
      redirectUri: `${window.location.origin}/api/auth/callback`,
    });
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

Wrap the provider in `app/layout.tsx` — make the layout `async` so the session is fetched server-side:

```tsx
import { getSession } from "@/lib/auth";
import { AuthProvider } from "@/components/auth-provider";

export default async function RootLayout({
  children,
}: Readonly<{ children: React.ReactNode }>) {
  const session = await getSession();

  return (
    <html lang="en" suppressHydrationWarning>
      <body>
        <AuthProvider initialUser={session?.user ?? null}>
          {children}
        </AuthProvider>
      </body>
    </html>
  );
}
```

Components opt-in to auth by calling `useAuth()`. Pages that don't need auth simply ignore it.

## Route protection with proxy.ts

Use when certain pages should hard-redirect unauthenticated users.

Create `proxy.ts` at the project root (Next.js 16 convention — NOT `middleware.ts`):

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
  matcher: ["/dashboard/:path*"],
};
```

This is a fast cookie-presence check — no SDK call. Full session validation happens in server actions and API routes. The file must be named `proxy.ts` and the export must be `function proxy`.

## Protected API route pattern

Use when an API route (streaming, webhooks, external callers) requires authentication.

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
