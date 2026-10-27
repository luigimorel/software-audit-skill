# Rule Register: format and extraction

Read this in Step 1 and Step 2 when discovering and documenting rules.

## Contents

- Register format
- Rule categories to look for
- Extraction tips by source
- Cross-checking stated vs implemented
- Writing good rules

---

## Register format

One row per rule. Keep columns short; details go in findings.

| ID | Rule (one testable sentence) | Source | Enforced where | Server-side | Atomic | Tested | Status |
|---|---|---|---|:-:|:-:|:-:|---|

- **Source:** Stated, Implemented, Inferred, or Missing (definitions in `SKILL.md`)
- **Enforced where:** file and line, DB constraint name, policy name, or "UI only"
- **Atomic:** `n/a` for rules that do not involve concurrent read-modify-write
- **Status:** Solid, Broken (finding ID), Unverified, or Open question

## Rule categories to look for

Walk this list so nothing is skipped. Most apps have rules in at least six of these.

| Category | Typical rules |
| --- | --- |
| **Pricing and totals** | Price comes from the catalog; tax and shipping rules; minimum order; currency; rounding |
| **Discounts and promos** | One coupon per order; expiry; usage cap; first-purchase only; stackability; minimum spend |
| **Payments** | Order is fulfilled only after confirmed payment; partial payments; retries; chargebacks |
| **Refunds and credits** | Refund window; cap at amount paid; one refund per item; who may approve |
| **Balances and wallets** | Balance never negative; transfers are atomic; ledger is append-only |
| **Plans and entitlements** | Features per tier; seat counts; project or storage limits; downgrade behavior |
| **Quotas and rate rules** | Per-day or per-month caps; reset timing; overage behavior |
| **Inventory and capacity** | Stock never negative; seat, room, or slot booked once; hold expiry |
| **Workflow and state** | Allowed status transitions; required approvals; steps that cannot be skipped |
| **Roles and approvals** | Who can do what; separation of duties; tenant isolation |
| **Time rules** | Trial length; expiry; cancellation cutoffs; business hours; time zone used |
| **Identity and eligibility** | Age, region, verified email or phone, KYC tier, one account per person |
| **Referrals and rewards** | Self-referral blocked; reward after qualifying action; caps |
| **Content and community** | Ownership; edit windows; moderation states; visibility rules |
| **Lifecycle and retention** | What happens on cancel, delete, downgrade, ownership transfer; retention periods |
| **Notifications and side effects** | Emails and SMS sent once; webhooks delivered at least once and handled idempotently |

## Extraction tips by source

**Code search** (adapt to the stack):

```bash
# conditions that usually encode rules
grep -rInE "(plan|tier|role|status|state)\s*(===?|!==?|==|!=)\s*['\"]" --exclude-dir={node_modules,.git,dist,build,.next,vendor} .
grep -rInE "\b(MAX|MIN|LIMIT|QUOTA|TRIAL|EXPIR|DISCOUNT|FEE|TAX|REFUND)[A-Z_]*\b" --exclude-dir={node_modules,.git,dist,build,.next,vendor} .

# money and counters being changed
grep -rInE "(balance|credits|stock|quantity|inventory|seats|count)\s*(\+|-)?=|increment|decrement" --exclude-dir={node_modules,.git,dist,build,.next,vendor} .

# status changes
grep -rInE "status\s*[:=]\s*['\"][a-z_]+['\"]|UPDATE .* SET status" --exclude-dir={node_modules,.git,dist,build,.next,vendor} .
```

**Schema:** list every `CHECK`, `UNIQUE`, `NOT NULL`, foreign key, enum, and trigger. Each is a rule the database already enforces. Tables with *none* of these and money or counters in them are suspicious.

**Routes and handlers:** list every endpoint that writes. For each, note which rules it should enforce. Then diff across entry points: web API, mobile API, admin, webhook, cron, queue worker, CLI scripts, edge functions. Rules enforced in one and absent in another are the classic gap.

**UI copy:** search for numbers and promises in the UI, pricing page, emails, and error messages (`"up to"`, `"unlimited"`, `"free"`, `"within"`, `"only"`, `"once"`, `"per month"`). Each is a Stated rule.

**Payment provider config:** products, prices, trial days, coupon settings, webhook event list. Compare against what the app assumes.

**Tests:** existing assertions are rules the author cared about. Missing tests for a Stated rule are an Info finding.

## Cross-checking stated vs implemented

For each Stated rule, find the code that enforces it. Three outcomes:

1. **Found, server-side, correct:** Solid.
2. **Found only in UI or client:** Broken (bypassable).
3. **Not found:** Broken (unenforced) or Open question (maybe done elsewhere, such as the payment provider).

For each Implemented rule with no matching Stated rule, add it to the Open questions list: the code may be doing something the business does not know about.

## Writing good rules

- **Testable:** "A coupon can be used once per customer" yes; "coupons are validated" no.
- **One idea per row:** split "orders over $50 ship free and must be paid first" into two rules.
- **Name the actor and the object:** "A member cannot approve their own expense" beats "no self-approval".
- **State the invariant, not the implementation:** "Stock is never negative" beats "decrement stock if > 0".
- **Say when unsure:** mark Inferred and ask.
