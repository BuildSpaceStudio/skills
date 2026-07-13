# Server Actions with next-safe-action

Already set up in this project — extend the existing clients, don't recreate them.

## Action clients (`lib/safe-action.ts`)

Three authorization tiers, stacked as middleware:

| Client | Guarantees | ctx |
|--------|-----------|-----|
| `actionClient` | nothing (public forms) | — |
| `authActionClient` | valid session | `ctx.session` (`user`, `token`) |
| `adminActionClient` | session + local `super_admin` role | `ctx.session`, `ctx.user` (local record) |

To add a new tier (e.g. an "owner" check), copy the `adminActionClient` shape: extend the previous tier with `.use()` and throw when the check fails.

## Working examples in-repo

| Pattern | File |
|---------|------|
| CRUD actions with zod + `revalidatePath` | `app/dashboard/todos/actions.ts` |
| Read-modify-write on the users table | `app/dashboard/settings/actions.ts` |
| Admin-gated action (`adminActionClient`) | `app/dashboard/admin/actions.ts` |
| SDK call inside an action (signed URL, delete) | `app/dashboard/files/actions.ts` |
| Action returning a redirect URL (checkout) | `app/dashboard/billing/actions.ts` |

Conventions these files follow:

- `"use server"` at the top of a colocated `actions.ts` inside the slice folder
- `.inputSchema(z.object({ ... }))` on every action that takes input
- Scope every query by the caller: `eq(table.userId, ctx.session.user.id)` — never trust an id from the client alone
- `revalidatePath("/dashboard/<slice>")` after mutations so the server component re-renders

## Client usage with useAction

`app/dashboard/todos/todo-form.tsx` is the canonical example:

```tsx
"use client";
import { useAction } from "next-safe-action/hooks";
import { toast } from "sonner";
import { createTodo } from "./actions";

const { execute, isPending } = useAction(createTodo, {
  onSuccess: () => toast.success("Todo added"),
  onError: ({ error }) => toast.error(error.serverError ?? "Failed to add todo"),
});
```

Always pair `onSuccess`/`onError` with toasts, and disable the submit control on `isPending`.
