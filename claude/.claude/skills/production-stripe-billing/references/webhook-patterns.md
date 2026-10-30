# Webhook patterns

## Route responsibilities

The HTTP adapter only:

- Loads secrets from validated env
- Reads raw body within byte limit (reject oversize)
- Verifies signature
- Delegates to ledger + handlers
- Maps outcomes to HTTP status

No business rules in the route file.

## Signature verification

- Require `stripe-signature` header
- Pass **exact** raw string body to Stripe verification (not re-serialized JSON)
- Use webhook secret from server env only
- 400 on invalid signature (Stripe should not retry forever on bad config)
- 500 on handler/DB failure so Stripe retries

## Handled event types

Maintain an explicit allowlist. Return 200 with `{ ignored: true }` for other types if desired (reduces noise) or 200 empty — match project convention.

Common production set (enable subset):

| Event | Typical purpose |
|-------|-----------------|
| `checkout.session.completed` | Fulfillment entry; subscription row sync |
| `checkout.session.async_payment_succeeded` | Delayed payment methods |
| `customer.subscription.created` | Subscription sync |
| `customer.subscription.updated` | Status/plan/cancel flag changes |
| `customer.subscription.deleted` | Access removal |
| `invoice.paid` | Recurring entitlement grant |
| `invoice.payment_failed` | Past due, notify, restrict access (if product requires) |
| `invoice.finalized` | Rarely needed for entitlement; audit only |

If `invoice.payment_failed` is omitted, document that access changes rely on subscription status webhooks or Stripe dunning only.

## Ledger transaction shape

Inside one database transaction:

```
claim(event) → if duplicate → return
             → if in_flight → return 500 (retry later)
handler(ctx, event)   // uses same tx connection
mark_processed(event)
commit
```

**Never** `mark_processed` before handler succeeds.

## Claim implementation (PostgreSQL pattern)

Elements to combine:

1. **Primary key** on Stripe event id
2. **INSERT … ON CONFLICT** to increment `attempts` only when `processed_at IS NULL` and claim is stale or first delivery
3. **`pg_try_advisory_xact_lock(hashtext(event_id))`** so two concurrent requests do not both pass the UPDATE branch
4. **Stale claim interval** (minutes): in-flight worker crash releases event for retry
5. **`last_error`** on failure with truncated message (root cause preferred)

Outcomes:

| Outcome | HTTP | Stripe behavior |
|---------|------|-----------------|
| processed | 200 | stops retry |
| duplicate | 200 | stops retry |
| in_flight | 500 | retries |
| failed (released) | 500 | retries |

## Handler context

Pass into handlers:

- Stripe client
- DB transaction handle
- Server env (for price maps, URLs, email)
- Optional: set of user ids needing session/cache sync after commit

Keep handlers free of HTTP types.

## Post-commit side effects

If session cache or KV must reflect new entitlements:

- Prefer patching after successful transaction
- Failures in cache patch should **not** fail webhook (log + reconcile)
- Run cache work inside webhook only if product requires immediate consistency

## Optional recovery hook

When handler performs Stripe API cancellation inside a transaction and then must abort DB work, use a typed error with payload and a **separate** reconciliation transaction after rollback (see reconciliation.md).
