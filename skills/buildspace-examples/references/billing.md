# Billing: Checkout, Portal, Entitlements

Billing runs on the creator's connected Stripe account, managed through Creator Studio. The app talks to it via `bs.billing.*` on the server SDK. A working slice exists in this project:

| Concern | Lives at |
|---------|----------|
| Billing helpers + graceful degradation | `lib/billing.ts` |
| Pricing/status page (all states) | `app/dashboard/billing/page.tsx` |
| Checkout + portal server actions | `app/dashboard/billing/actions.ts` |
| Redirect buttons (client) | `app/dashboard/billing/checkout-button.tsx` |

## Always go through `lib/billing.ts`

The helpers feature-detect the SDK's billing namespace and catch `BuildspaceError`, so an app without billing enabled (or an older SDK) renders the "billing isn't enabled" empty state instead of crashing. Never call `bs.billing.*` directly from pages or actions — add a helper.

- `getBillingOverview()` → `{ state: "unavailable" | "disabled" | "active", ... }` with products + prices when active
- `createCheckout({ userId, priceId, successUrl, cancelUrl })` → `{ url }`
- `createPortalSession({ userId, returnUrl })` → `{ url }`
- `getSubscription({ userId })` → subscription or null
- `hasEntitlement({ userId })` → boolean, never throws
- `formatPrice(price)` → display string like `$29/month`

## Checkout is server-side

The blessed integration creates the Stripe Checkout session in a server action bound to the session user — billing state always lands on the right identity:

```ts
export const startCheckout = authActionClient
  .inputSchema(z.object({ priceId: z.string().min(1) }))
  .action(async ({ parsedInput, ctx }) => {
    const { url } = await createCheckout({
      userId: ctx.session.user.id,
      priceId: parsedInput.priceId,
      successUrl: `${origin}/dashboard/billing?checkout=success`,
      cancelUrl: `${origin}/dashboard/billing?checkout=cancelled`,
    });
    return { url };
  });
```

The client redirects with `window.location.href = data.url` (see `checkout-button.tsx`). Redirect URLs must be absolute: prefer `NEXT_PUBLIC_APP_URL`, fall back to the request `origin` header (see `getAppOrigin` in the actions file).

## Manage subscription = Stripe customer portal

`openBillingPortal` in the actions file mints a portal session with a `returnUrl` back to the billing page. Render the button only when the user has a subscription.

## Gating paid features

```ts
import { hasEntitlement } from "@/lib/billing";

const entitled = await hasEntitlement({ userId: session.user.id });
if (!entitled) {
  // render upsell / hide the feature
}
```

The "Pro features" card at the bottom of `app/dashboard/billing/page.tsx` is the in-repo example. `hasEntitlement` resolves `false` when billing is unavailable, so gated features degrade to "locked" rather than erroring.

## Promotion codes

Requires `@buildspacestudio/sdk` >= 0.7.0 and `@buildspacestudio/cli` >= 0.21.0. Codes live on the creator's Stripe account; create them with the CLI (dev first, it's test mode):

```bash
buildspace app billing promos create --code LAUNCH20 --percent-off 20 --max-redemptions 100 --expires 2026-12-31
buildspace app billing promos --json            # verify
```

The app decides per checkout how a code gets applied. Extend the `createCheckout` helper in `lib/billing.ts` to forward the option you need rather than calling `bs.billing` directly:

```ts
export async function createCheckout(opts: {
  userId: string;
  priceId: string;
  successUrl: string;
  cancelUrl: string;
  allowPromotionCodes?: boolean;
  promotionCode?: string;
}): Promise<{ url: string }> {
  return getServerClient().billing.createCheckout(opts);
}
```

**Let customers type a code (subscriptions).** Pass `allowPromotionCodes: true` and Stripe Checkout shows an "Add promotion code" field. Turn it on only where it makes sense, e.g. for users who arrived from a campaign.

**Apply a code from a link or your own form (any price type).** Validate the input and pass it as `promotionCode` from the server action. Don't take the `promotionCode` value from the client blindly for codes meant for specific people — decide on the server:

```ts
export const startCheckout = authActionClient
  .inputSchema(
    z.object({
      priceId: z.string().min(1).max(128),
      // From ?promo=LAUNCH20 or a "Have a code?" input.
      promo: z.string().trim().regex(/^[A-Za-z0-9_-]{3,40}$/).optional(),
    })
  )
  .action(async ({ parsedInput, ctx }) => {
    const origin = await getAppOrigin();
    const { url } = await createCheckout({
      userId: ctx.session.user.id,
      priceId: parsedInput.priceId,
      promotionCode: parsedInput.promo,
      successUrl: `${origin}/dashboard/billing?checkout=success`,
      cancelUrl: `${origin}/dashboard/billing?checkout=cancelled`,
    });
    return { url };
  });
```

**Codes for specific people** (staff, partners, beta users): look the user up on the server and pass the code only when they qualify — never ship the code to the browser.

Rules:
- Pass `allowPromotionCodes` or `promotionCode`, never both.
- `allowPromotionCodes` is rejected for one-time prices; use `promotionCode` for those.
- A bad, expired, or used-up code throws `BuildspaceError` with `Promotion code not found` (or a Stripe restriction message). Surface it next to the code input via `extractActionError`; don't silently retry without the code.
- Codes are created per environment. Recreate them with `--env prod` when going live — `billing sync` doesn't copy them.

## Test vs live mode

`getBillingOverview().status.testMode` is true when the connected Stripe account is in test mode — payments use Stripe test cards, no real charges. Always surface a test-mode banner (the billing page shows the pattern). Mode is configured per environment in Creator Studio, not in this app.

## States to handle (in order)

1. **unavailable** — SDK/API doesn't expose billing → empty state
2. **disabled / setup_required / paused** — Stripe not fully connected in Creator Studio → "enable billing in Creator Studio" empty state
3. **active** — render products, prices, checkout, subscription, entitlements
