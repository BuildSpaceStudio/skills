# Billing: Checkout, Portal, Entitlements

Billing runs on the creator's connected Stripe account, managed through Creator Studio. The app talks to it via `bs.billing.*` on the server SDK. A working slice exists in this project:

| Concern | Lives at |
|---------|----------|
| Billing helpers + graceful degradation | `lib/billing.ts` |
| Pricing/status page (all states) | `app/dashboard/billing/page.tsx` |
| Checkout + portal server actions | `app/dashboard/billing/actions.ts` |
| Redirect buttons (client) | `app/dashboard/billing/checkout-button.tsx` |
| Promo codes (`?promo=` + form) | `app/dashboard/billing/page.tsx`, `lib/billing.ts` |

## Always go through `lib/billing.ts`

The helpers feature-detect the SDK's billing namespace and catch `BuildspaceError`, so an app without billing enabled (or an older SDK) renders the "billing isn't enabled" empty state instead of crashing. Never call `bs.billing.*` directly from pages or actions — add a helper.

- `getBillingOverview()` → `{ state: "unavailable" | "disabled" | "active", ... }` with products + prices when active
- `createCheckout({ userId, priceId, successUrl, cancelUrl, promotionCode?, allowPromotionCodes? })` → `{ url }` or `{ error }` for a rejected promo code
- `createPortalSession({ userId, returnUrl })` → `{ url }`
- `getSubscription({ userId })` → subscription or null
- `hasEntitlement({ userId })` → boolean, never throws
- `formatPrice(price)` → display string like `$29/month`

## Checkout is server-side

The blessed integration creates the Stripe Checkout session in a server action bound to the session user — billing state always lands on the right identity:

```ts
export const startCheckout = authActionClient
  .inputSchema(
    z.object({
      priceId: z.string().min(1).max(128),
      promotionCode: z.string().trim().max(40).regex(PROMO_CODE_PATTERN).optional(),
    }),
  )
  .action(async ({ parsedInput, ctx }) => {
    const origin = await getAppOrigin();
    return createCheckout({
      userId: ctx.session.user.id,
      priceId: parsedInput.priceId,
      promotionCode: parsedInput.promotionCode,
      successUrl: `${origin}/dashboard/billing?checkout=success`,
      cancelUrl: `${origin}/dashboard/billing?checkout=cancelled`,
    });
  });
```

The action returns `{ url }`, or `{ error }` when a promo code is rejected. The client redirects with `window.location.href = data.url` (see `checkout-button.tsx`). Redirect URLs must be absolute: prefer `NEXT_PUBLIC_APP_URL`, fall back to the request `origin` header (see `getAppOrigin` in the actions file).

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
buildspace app billing promos create --code FRIEND --percent-off 100 --max-redemptions 1   # single-use, free
buildspace app billing promos --json            # verify
```

The billing slice already applies codes. The whole flow:

| Piece | Lives at |
|-------|----------|
| `createCheckout({ ..., promotionCode?, allowPromotionCodes? })`, `normalizePromoCode`, `PROMO_CODE_PATTERN` | `lib/billing.ts` |
| `startCheckout` accepts an optional, validated `promotionCode` | `app/dashboard/billing/actions.ts` |
| `?promo=CODE` links + "Have a promo code?" form (a GET back to the page) | `PromoCodeCard` in `app/dashboard/billing/page.tsx` |
| Buttons carry the code; a rejected code shows as a toast | `app/dashboard/billing/checkout-button.tsx` |
| Tests for the promo paths | `app/dashboard/billing/actions.test.ts` |

Share a link like `https://your-app.com/dashboard/billing?promo=LAUNCH20` and the code rides along on every checkout button.

**Rejected codes are returned, not thrown.** `createCheckout` returns `{ error }` when a code is unknown, expired, used up, or doesn't apply to the product, because next-safe-action replaces thrown messages with a generic one. Handle both shapes in `onSuccess`:

```ts
onSuccess: ({ data }) => {
  if (data && "error" in data) toast.error(data.error);
  else if (data?.url) window.location.href = data.url;
},
```

**Let customers type a code on Stripe's page instead (subscriptions only).** Pass `allowPromotionCodes: true` to `createCheckout` from a server action for the prices or users where it makes sense. It's rejected for one-time prices; use `promotionCode` for those.

**Codes for specific people** (staff, partners, beta users): don't put them in links or accept them from the client. Look the user up in the server action and pass `promotionCode` only when they qualify.

Rules:
- Pass `allowPromotionCodes` or `promotionCode`, never both.
- Codes are created per environment. Recreate them with `--env prod` when going live — `billing sync` doesn't copy them.
- A code only covers the products that existed when it was created (unless `--products` says otherwise); new products need a new code.

## Test vs live mode

`getBillingOverview().status.testMode` is true when the connected Stripe account is in test mode — payments use Stripe test cards, no real charges. Always surface a test-mode banner (the billing page shows the pattern). Mode is configured per environment in Creator Studio, not in this app.

## States to handle (in order)

1. **unavailable** — SDK/API doesn't expose billing → empty state
2. **disabled / setup_required / paused** — Stripe not fully connected in Creator Studio → "enable billing in Creator Studio" empty state
3. **active** — render products, prices, checkout, subscription, entitlements
