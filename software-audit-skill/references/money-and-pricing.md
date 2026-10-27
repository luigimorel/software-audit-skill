# Money and pricing rules

Read this for any app that charges, refunds, credits, discounts, or keeps balances.

## Contents

- Price and total integrity
- Discounts, coupons, and promos
- Payment confirmation and webhooks
- Refunds, chargebacks, and credits
- Wallets, balances, and ledgers
- Subscriptions, trials, and proration
- Rounding and currency
- Stripe-specific checks
- Fix patterns
- Sources

---

## Price and total integrity

**Look for:**

- Request bodies that include `price`, `amount`, `total`, `currency`, `discount`, `tax`, or `shipping` and are used as given
- Checkout sessions created with an amount from the client rather than a server-side lookup by product or price ID
- Totals computed in the browser or app and trusted by the server
- Negative, zero, fractional, or huge `quantity` (negative quantity can produce a refund or credit)
- Cart items from another currency, region, or plan priced as if local
- Price changes between "add to cart" and "pay" with no re-validation
- Free items or $0 totals that skip the payment step but still fulfill
- Items the user should not be able to buy (draft, archived, admin-only, other tenant's) purchasable by ID

**Fix:** the client sends **IDs and quantities only**. The server loads prices, applies discounts and tax from its own data, validates `1 <= quantity <= max`, and creates the payment from that computed total. Store the computed breakdown on the order.

## Discounts, coupons, and promos

**Look for:**

- Coupons with no usage cap, no per-customer limit, or no expiry check on the server
- Case or whitespace variants treated as different codes, or one code redeemable many times via parallel requests
- Stackable discounts that bring the total below zero or below cost
- "First order only" or "new customer only" checked by email string only (plus-addressing and fake emails bypass it)
- Percentage discounts applied before and after fixed ones inconsistently
- Discount applied to items it should not cover (shipping, gift cards, subscriptions)
- Referral and loyalty rewards redeemable on the order that earned them
- Promo logic duplicated in frontend and backend with different results

**Fix:** a single server-side pricing function used by cart, checkout, invoice, and email. Track redemptions in a table with a unique constraint such as `(coupon_id, customer_id)` and an atomic counter for global caps. Clamp totals to a minimum of zero and to a floor price where the business needs one.

## Payment confirmation and webhooks

**Look for:**

- Order marked paid from the **client redirect** (`/success?session_id=...`) instead of a verified webhook or a server-side fetch of the payment
- Webhook endpoint with no signature verification, or signature check present but body parsed before verifying raw bytes
- Webhook handler not idempotent: a retried event grants credits, ships, or emails twice
- Handler that trusts amounts or user IDs from the event metadata that the client could influence
- No handling for out-of-order events (refund arrives before payment event)
- Payment succeeded for one amount, order fulfilled for a bigger one (amount never compared with the order total)
- Fulfillment triggered before payment completes ("pending" treated as paid), especially for mobile money flows with long pending states
- Failed, abandoned, or expired payments that never release reserved stock or credits

**Fix:** verify signatures, record every processed event ID with a unique constraint, compare paid amount and currency with the stored order total, and move the order to `paid` only inside the webhook handler.

## Refunds, chargebacks, and credits

**Look for:**

- Refund amount accepted from the client, or not capped at `paid - already_refunded`
- Multiple partial refunds that sum above the amount paid
- Refunds on orders that were never paid, already fully refunded, or belong to someone else
- Refund issued but entitlement, credits, or stock not reversed (or the reverse: reversed without refund)
- No window or approval step where the stated policy has one
- Store credit or gift cards that can be bought with credit, refunded to cash, and repeated
- Chargeback or dispute events ignored (user keeps access and money)
- Refund permission available to any staff role

**Fix:** compute `refundable = paid - sum(refunds)` server-side inside a transaction, require ownership and role checks, reverse side effects in the same transaction, and log who refunded what.

## Wallets, balances, and ledgers

**Look for:**

- A `balance` column updated by read-modify-write (race condition: two concurrent requests both read the old value; see "Business logic, abuse, and cost" in `injection-and-abuse.md`)
- No ledger: balance changes with no immutable record of why
- Transfers implemented as two separate updates without a transaction
- Negative balances possible (no `CHECK (balance >= 0)`)
- Withdrawals or payouts without a pending or approved state, or without idempotency keys
- Credits for signup, referrals, or promos not capped per person
- Currency mixed in one balance column

**Fix:** append-only ledger plus a derived or cached balance, `CHECK (balance >= 0)`, transfers in one transaction, idempotency key on every money-moving request.

## Subscriptions, trials, and proration

**Look for:**

- Entitlement derived from the client or a stale cached flag instead of the subscription's current server-side status
- Trial restartable by new email, new device, or deleting and recreating the account
- Downgrade or cancel that leaves paid features on until the next manual cleanup
- Plan change mid-cycle with no proration rule, or proration computed in floats
- Failed renewal with no grace-period rule, or an infinite grace period
- Seat counts changeable after billing without a charge
- Mismatch between what the pricing page promises and what the plan table enforces

**Fix:** a single `hasEntitlement(user, feature)` function backed by subscription status from the billing provider (via webhooks), used by every entry point.

## Rounding and currency

**Look for:**

- Money stored or computed as `float` / `double` / JS `number` with decimals
- Per-line rounding that makes the invoice total differ from the sum of the parts
- Currencies with different minor units (JPY has 0 decimals, KWD has 3, UGX has 0 in practice) handled as if they all have 2
- Exchange rates fetched client-side or never refreshed
- Tax computed on a rounded value, or rounding differing between app, invoice PDF, and payment provider

**Fix:** store integer minor units plus a currency code, round once at a defined point with a defined mode, and use one money library or helper everywhere.

## Stripe-specific checks

**Look for:**

- `price_data.unit_amount` or product names taken from the request body. Prefer Stripe **Price IDs** looked up server-side from your own catalog, so the amount is defined in Stripe and cannot be edited by the buyer.
- **Webhook signature check on a parsed body.** Verification needs the raw bytes: in Express register `express.raw({ type: 'application/json' })` for the webhook route *before* `express.json()`; in a Next.js App Router handler use `await request.text()`, not `request.json()`.
- **Checkout session `metadata`** (user id, plan) supplied by the client. Set it on the server when creating the session, otherwise a buyer can claim another user's id or pick a cheaper plan.
- **Subscription status read from a login-time session value, client flag, or stale JWT claim.** Check current status server-side on every protected request, from your database kept in sync by webhooks.

```ts
// Next.js App Router webhook: raw body, then verify
export async function POST(request: Request) {
  const body = await request.text();
  const sig = request.headers.get('stripe-signature')!;
  const event = stripe.webhooks.constructEvent(body, sig, webhookSecret);
  // ... handle event idempotently (see below)
}
```

## Fix patterns

```ts
// Server recomputes everything from IDs
const items = await db.product.findMany({ where: { id: { in: ids }, active: true } });
const subtotal = lines.reduce((sum, l) => sum + priceOf(items, l.productId) * l.qty, 0); // integer minor units
```

```sql
-- Coupon redemption: once per customer, and a global cap, atomically
create unique index on coupon_redemptions (coupon_id, customer_id);

update coupons set redeemed = redeemed + 1
where id = $1 and (max_redemptions is null or redeemed < max_redemptions) and expires_at > now();
-- 0 rows affected => coupon exhausted or expired
```

```go
// Idempotent webhook handling
res, err := tx.Exec(`insert into processed_events (id) values ($1) on conflict do nothing`, evt.ID)
if n, _ := res.RowsAffected(); n == 0 {
    return nil // already handled
}
```

## Sources

- [Stripe: Webhook signatures](https://docs.stripe.com/webhooks#verify-events)
- [Stripe: Prices](https://docs.stripe.com/products-prices/how-products-and-prices-work)
- [OWASP Business Logic Vulnerability](https://owasp.org/www-community/vulnerabilities/Business_logic_vulnerability)
