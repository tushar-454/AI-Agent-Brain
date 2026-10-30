# Subscription lifecycle

## Stripe object relationships

```
Customer
  → Checkout Session / PaymentIntent
  → Subscription
  → Subscription Item(s)
  → Price / Product
  → Invoice
  → Invoice Line Item(s)
```

Persist only IDs and fields the application reads. Typical:

- Customer id on user/account
- Subscription id, status, price id, period start/end, cancel flags
- Subscription item ids when invoice line matching is required

## Project question bank

Ask when not evident from repo:

**Stripe**

1. One-time, subscription, or both?
2. Intervals: monthly, yearly, custom?
3. What does successful payment grant?
4. Entitlements: renewable, consumable, period-based, non-expiring?
5. Cancellation: immediate or end of period?
6. Upgrades/downgrades?
7. Subscription replacement (new sub replaces old)?
8. Behavior on payment failure?
9. Customer Portal required?
10. Trials?
11. Coupons/discounts?
12. Tax?
13. Invoices required for access?

**Database**

14. Which tables: users, subscriptions, payments, entitlements, events?
15. ORM and transaction helpers?
16. Existing idempotency tables?

**Infrastructure**

17. Cloudflare Workers?
18. Hyperdrive?
19. Request-scoped DB?
20. Cron infrastructure?
21. Queue infrastructure?

## Checkout.session.completed

**Mode `payment` (one-time)**

- Require `payment_status === paid` (or handle async event separately)
- Resolve user from `client_reference_id` and/or metadata
- Load line items from Stripe; map **paid price id** to server catalog (never metadata amount)
- Idempotency: insert processed session id before granting

**Mode `subscription`**

- Retrieve subscription from Stripe (expanded if needed)
- Upsert subscription row even when unpaid (metadata sync)
- Grant entitlement only when `payment_status === paid` and subscription not superseded
- Idempotency: same session id guard before grant

## customer.subscription.updated

Filter noise: ignore updates whose `previous_attributes` only touch irrelevant fields.

When `items` change and status is `active` or `trialing`, product may grant immediately; when payment still pending (`past_due`, `incomplete`), prefer `invoice.paid`.

## customer.subscription.deleted

- Upsert final subscription state if price still mappable
- If another **active** subscription row exists for user (replacement), **do not** revoke access
- Else apply project's end-of-access rules (downgrade entitlements)

## invoice.paid

**billing_reason allowlist (typical)**

- `subscription_create`
- `subscription_cycle`
- `subscription_update`

Skip other reasons unless product explicitly defines behavior.

**Steps**

1. Resolve subscription id from invoice (API versions may use `parent.subscription_details` or legacy `subscription` field)
2. Retrieve live subscription from Stripe
3. Upsert subscription row; exit if superseded replacement
4. For `subscription_update`: product-specific grant rules (often same-period plan change)
5. For create/cycle: load invoice line items (paginate if `has_more`)
6. Find non-proration line for target subscription item id
7. Compare line period to subscription item `current_period_start/end`
8. Grant only on **match**; skip stale/ahead; throw or alert on mismatch for create

Do not grant recurring entitlement for every `invoice.paid` without period validation.

## Period comparison semantics

| Result | Meaning | Typical action |
|--------|---------|----------------|
| match | Invoice pays current subscription period | Grant |
| stale | Invoice period ends before subscription period starts | Skip (delayed webhook) |
| ahead | Invoice period starts at/after subscription period end | Skip (early or wrong invoice) |
| mismatch | Overlap without alignment | Skip or manual review; fail on create |

Grant using invoice line period bounds when match (not only subscription snapshot) so grants align with paid service window.

## Cancellation

**User requests cancel**

- Stripe: `cancel_at_period_end: true` (if product keeps access until period end)
- Update local cancel flags from Stripe response

**Stripe confirms end**

- `customer.subscription.deleted` or status `canceled`
- Set access cutoff from verified period end / product rules

**cancel_at_period_end**

- Access continues until period end in Stripe; local `ends_at` or `current_period_end` drives app checks

## Payment failure

If not handling `invoice.payment_failed`:

- Rely on `customer.subscription.updated` status (`past_due`, `unpaid`) for access restrictions
- Document gap in audit

If handling:

- Restrict entitlements per product policy
- Do not double-revoke on duplicate events (ledger + status guards)

## Metadata conventions

- `userId` on subscription metadata (set at Checkout creation)
- `client_reference_id` as fallback only when metadata missing
- Validate resolved user owns Stripe customer on grant paths

## Non-transferable reference concepts

Do not assume in new projects:

- Specific plan tiers or enum values
- Credit amounts per plan
- PAYG package catalog
- Yearly subscription with monthly in-period refills
- Application-specific column names on `users` table

Ask: **"What does successful Stripe payment grant in this project?"**
