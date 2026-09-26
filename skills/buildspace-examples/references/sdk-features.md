# SDK Features: Events, Storage, Notifications

Each capability has a working example in this project. Extend the existing files; only fall back to from-scratch snippets for patterns that aren't baked in.

## Event tracking

| Pattern | Lives at |
|---------|----------|
| Server-side helper (never throws) | `lib/analytics.ts` → `trackEvent({ event, properties, userId })` |
| Client page-view tracking (batched) | `components/analytics.tsx`, mounted in `app/layout.tsx` |
| Tracking inside a server action | `app/dashboard/todos/actions.ts` (`todo_created`) |
| Tracking from a client component | `app/dashboard/files/file-uploader.tsx` (`file_uploaded`) |
| First-sign-in event | `app/api/auth/callback/route.ts` (`user_signed_up`) |

Rules:

- Server-side, always go through `trackEvent` from `lib/analytics.ts` — it catches `BuildspaceError` and logs instead of failing the request. Analytics must never break the app.
- Client-side, call `getBrowserClient().events.track(...)` directly — the browser SDK batches and auto-flushes.

```ts
// server (actions, routes)
await trackEvent({ event: "report_exported", properties: { reportId }, userId: ctx.session.user.id });
```

```tsx
// client
getBrowserClient().events.track("button_clicked", { page: "pricing" });
```

## File storage

| Pattern | Lives at |
|---------|----------|
| Browser upload | Prefer `createUploadRoute` + `useUpload` (see the buildspace-sdk skill). `app/dashboard/files/file-uploader.tsx` still uses `bs.storage.upload`, which only works when the client has a session token. |
| Server-side listing scoped to a prefix | `app/dashboard/files/page.tsx` (`bs.storage.list`) |
| Server-signed download URL via action | `app/dashboard/files/actions.ts` (`getSignedUrl`, 5-min expiry) |
| Delete with ownership check | `app/dashboard/files/actions.ts` (`deleteFile`) |
| Avatar upload + key stored in DB | `app/dashboard/settings/avatar-upload.tsx` + `actions.ts` |

Conventions the files slice encodes:

- **Path ownership**: user files live under `files/{userId}/…`, avatars at `avatars/{userId}`. Every server action validates the key prefix against `ctx.session.user.id` before touching storage.
- **Store keys, not URLs**: signed URLs expire — persist the storage key (see `users.avatarUrl`) and mint a signed URL server-side on read (see `app/dashboard/settings/page.tsx`).
- **Degrade gracefully**: catch `BuildspaceError` on reads and render an empty state (see `listFiles` in `app/dashboard/files/page.tsx`).

## Email notifications

Server-only (requires the secret key). `lib/email.ts` is the home for all transactional email: one exported function per message type with a small inline HTML template, failures logged but never thrown. `sendWelcomeEmail` (called from the auth callback on first sign-in) is the working example — add new message types beside it:

```ts
export async function sendReceiptEmail({ to, amount }: { to: string; amount: string }) {
  try {
    const bs = getServerClient();
    await bs.notifications.send({
      to,
      subject: "Your receipt",
      html: `<p>Thanks! We charged ${amount}.</p>`,
      text: `Thanks! We charged ${amount}.`,
    });
  } catch (err) {
    if (err instanceof BuildspaceError) {
      console.error(`[email] receipt failed: ${err.code} (${err.status})`);
      return;
    }
    console.error("[email] receipt failed", err);
  }
}
```

For dashboard-managed templates, use `bs.notifications.sendTemplate(templateSlug, { to, variables })` inside the same `lib/email.ts` wrapper shape.
