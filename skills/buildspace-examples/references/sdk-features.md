# SDK Features: Events, Storage, Notifications

## Event tracking

### Browser events

Track directly from client components — events are batched and auto-flushed:

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

For events that must be tracked server-side (after a purchase, sign-up). Requires `next-safe-action` — see [server-actions.md](server-actions.md).

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

For external callers or cases where a server action isn't suitable:

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

## File storage

### Browser upload

Upload directly from the browser — no API route needed:

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
  };

  return <input type="file" onChange={handleUpload} />;
}
```

### Server actions for listing and signed URLs

Requires `next-safe-action` — see [server-actions.md](server-actions.md).

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

## Email notifications

Server-only (requires the secret key). Requires `next-safe-action` — see [server-actions.md](server-actions.md).

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
