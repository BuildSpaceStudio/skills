# BuildSpace SDK API Reference

## Contents

- [Initialization](#initialization)
- [Configuration options](#configuration-options)
- [Auth — client methods](#auth--client-methods)
- [Auth — server methods](#auth--server-methods)
- [Events — client methods](#events--client-methods)
- [Events — server methods](#events--server-methods)
- [Storage — client methods](#storage--client-methods)
- [Storage — server methods](#storage--server-methods)
- [Notifications — server methods](#notifications--server-methods)
- [BuildspaceError](#buildspaceerror)

## Initialization

### Server

```ts
import Buildspace from "@buildspacestudio/sdk";

const bs = new Buildspace("bs_sec_...", config?);
```

### Client

```ts
import { createClient } from "@buildspacestudio/sdk/client";

const bs = createClient("bs_pub_...", config?);
```

Key validation is enforced: passing a secret key to `createClient` throws, and passing a publishable key to `new Buildspace` throws.

## Configuration options

Shared options for both `new Buildspace(key, config)` and `createClient(key, config)`:

| Option | Default | Description |
|--------|---------|-------------|
| `baseUrl` | `https://runtime.buildspace.studio` | API endpoint |
| `loginUrl` | `https://login.buildspace.studio` | Auth host |
| `version` | API version header | API version |
| `fetch` | global `fetch` | Custom fetch (server only) |

Client-only options:

| Option | Description |
|--------|-------------|
| `events.flushInterval` | Auto-flush interval in ms |
| `events.maxBatchSize` | Max events per batch |

## Auth — client methods

Available via `bs.auth` on the browser client.

### `getSignInUrl(options)`

Returns a URL string to redirect the user for sign-in.

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `redirectUri` | `string` | yes | Callback URL after auth |
| `appSlug` | `string` | no | App slug override |
| `env` | `"dev" \| "prod"` | no | Target environment |

### `getSignUpUrl(options)`

Same signature as `getSignInUrl`. Returns a sign-up URL.

## Auth — server methods

Available via `bs.auth` on the server client.

### `handleCallback(requestUrl, options)`

Exchanges an authorization code for a session token.

| Param | Type | Required |
|-------|------|----------|
| `requestUrl` | `string` | yes |
| `options.redirectUri` | `string` | yes |

Returns: `{ access_token: string, expires_in: number, user: { id, email, name } }`

### `getSession(token)`

Validates a session token.

| Param | Type | Required |
|-------|------|----------|
| `token` | `string` | yes |

Returns: `{ user: { id, email, name }, appId: string }` or `null`.

### `revokeSession(token)`

Revokes a session token. Returns `void`.

### `setSession(token)` / `clearSession()`

Attach or detach a user session for subsequent SDK calls.

## Events — client methods

Available via `bs.events` on the browser client. Events are batched and auto-flushed.

### `track(event, properties?)`

Queue an event for batched delivery.

| Param | Type | Required |
|-------|------|----------|
| `event` | `string` | yes |
| `properties` | `Record<string, any>` | no |

### `flush()`

Force-flush the current event batch. Returns `Promise<void>`.

### `shutdown()`

Graceful shutdown — flushes remaining events. Returns `Promise<void>`.

## Events — server methods

Available via `bs.events` on the server client. Events are sent immediately.

### `track(event, properties?, actorId?)`

| Param | Type | Required |
|-------|------|----------|
| `event` | `string` | yes |
| `properties` | `Record<string, any>` | no |
| `actorId` | `string` | no |

Returns: `{ event_id: string }`

### `batchTrack(events)`

| Param | Type | Required |
|-------|------|----------|
| `events` | `Array<{ event, properties?, actor_id? }>` | yes |

Returns: `{ count: number, event_ids: string[] }`

## Storage — client methods

Available via `bs.storage` on the browser client.

### `upload(file, options?)`

| Param | Type | Required |
|-------|------|----------|
| `file` | `File` | yes |
| `options.path` | `string` | no |

Returns: `{ key: string, url: string, size: number }`

### `getUrl(key)`

Returns: `{ url: string }`

### `list(prefix?)`

Returns: `{ objects: Array<{ key, size, lastModified }> }`

### `delete(key)`

Returns: `void`

## Storage — server methods

Available via `bs.storage` on the server client.

### `getUploadUrl(options)`

Generate a presigned upload URL.

| Param | Type | Required |
|-------|------|----------|
| `options.key` | `string` | yes |
| `options.contentType` | `string` | yes |
| `options.size` | `number` | yes |

Returns: `{ upload_url: string, key: string, expires_in: number }`

### `getSignedUrl(key, options?)`

Generate a temporary signed download URL.

| Param | Type | Required |
|-------|------|----------|
| `key` | `string` | yes |
| `options.expiresIn` | `number` | no |

Returns: `{ url: string, expiresIn: number }`

### `list(prefix?, options?)`

| Param | Type | Required |
|-------|------|----------|
| `prefix` | `string` | no |
| `options.limit` | `number` | no |
| `options.offset` | `number` | no |

Returns: `{ objects: Array<{ key, size, lastModified }> }`

### `delete(key)`

Returns: `void`

### `getUsage()`

Returns: `{ storageBytes: number, objectCount: number, maxStorageBytes: number }`

## Notifications — server methods

Available via `bs.notifications` on the server client only. Not available in the browser.

### `send(options)`

| Param | Type | Required |
|-------|------|----------|
| `options.to` | `string \| string[]` | yes |
| `options.subject` | `string` | yes |
| `options.html` | `string` | yes |
| `options.text` | `string` | no |
| `options.replyTo` | `string` | no |
| `options.metadata` | `Record<string, string>` | no |

Returns: `{ id: string }`

### `sendTemplate(templateSlug, options)`

| Param | Type | Required |
|-------|------|----------|
| `templateSlug` | `string` | yes |
| `options.to` | `string \| string[]` | yes |
| `options.variables` | `Record<string, string>` | no |
| `options.metadata` | `Record<string, string>` | no |

Returns: `{ id: string }`

## BuildspaceError

All SDK methods throw `BuildspaceError` on failure.

```ts
import { BuildspaceError } from "@buildspacestudio/sdk";
```

| Field | Type | Description |
|-------|------|-------------|
| `code` | `string` | Service-scoped error code |
| `service` | `"auth" \| "events" \| "storage" \| "notifications"` | Which service threw |
| `status` | `number` | HTTP status code |
| `message` | `string` | Human-readable description |

## Database (outside the SDK)

The managed Turso/libSQL database is **not** accessed through `@buildspacestudio/sdk`. Connect directly using `@libsql/client` and [Drizzle ORM](https://orm.drizzle.team). Credentials are injected as `BUILDSPACE_DB_URL` and `BUILDSPACE_DB_TOKEN` env vars. See the [SDK skill Database section](SKILL.md#database) for setup instructions and patterns.
