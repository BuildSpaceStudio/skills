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

## Test vs live mode

`getBillingOverview().status.testMode` is true when the connected Stripe account is in test mode — payments use Stripe test cards, no real charges. Always surface a test-mode banner (the billing page shows the pattern). Mode is configured per environment in Creator Studio, not in this app.

## States to handle (in order)

1. **unavailable** — SDK/API doesn't expose billing → empty state
2. **disabled / setup_required / paused** — Stripe not fully connected in Creator Studio → "enable billing in Creator Studio" empty state
3. **active** — render products, prices, checkout, subscription, entitlements
