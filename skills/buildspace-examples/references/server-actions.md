# Server Actions with next-safe-action

## Setup

Install dependencies:

```bash
npm install next-safe-action zod
# or: pnpm add next-safe-action zod | bun add next-safe-action zod
```

Create `lib/safe-action.ts`:

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

- `actionClient` — no auth required (public forms)
- `authActionClient` — validates session, injects `ctx.session` (with `user` and `token`)

## Authenticated action pattern

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
    // ... SDK calls scoped to authenticated user ...
    bs.clearSession();
    return { success: true };
  });
```

## Client usage with useAction

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
    <form
      onSubmit={(e) => {
        e.preventDefault();
        execute({ name: "Jane" });
      }}
    >
      <input name="name" />
      <button type="submit" disabled={isExecuting}>
        {isExecuting ? "Saving..." : "Save"}
      </button>
      {result.validationErrors && <p>Check your input</p>}
    </form>
  );
}
```
