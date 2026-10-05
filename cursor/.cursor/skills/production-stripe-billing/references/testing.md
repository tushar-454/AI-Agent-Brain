# Testing methodology

Use project's test runner (e.g. Vitest). Prefer unit tests on pure logic; integration tests where DB available.

## Stripe mocking

- Mock Stripe client at service boundary or use stripe-mock / test fixtures
- Handlers accept `StripeWebhookContext` — inject fake `stripe` with stubbed methods

## Event ledger

- Handler failure records root cause in `last_error` (unwrap `error.cause`)
- Failed outcome returns `failed`; event released for retry
- Cancellation reconciliation loop retries after successful recovery
- Duplicate processed event returns `duplicate`
- Concurrent claim: test via integration or SQL behavior documentation

## Checkout handlers

- One-time: paid vs unpaid; missing userId; unknown price id; duplicate session id skips second grant
- Subscription: unpaid syncs row without grant; paid grants once; superseded skips grant
- Async payment succeeded paths mirror completed

## invoice.paid

- billing_reason filtered
- Missing subscription id on creditable invoice throws
- Period match grants; stale/ahead skip
- subscription_create mismatch throws
- subscription_update path (product-specific)
- Duplicate grant blocked by period guard on user update

## Subscription events

- updated ignores irrelevant `previous_attributes`
- item change grants when active/trialing
- deleted downgrades unless other active subscription exists

## Subscription service / slot claim

- Upsert idempotent on same stripe id
- Replacement cancels older subscription (mock Stripe cancel)
- Reconciliation payload updates row when webhook tx aborted

## Period utilities

- `getInvoiceSubscriptionPeriod`: proration lines skipped; pagination when `has_more`
- `comparePaidInvoiceToSubscriptionPeriod`: match, stale, ahead, mismatch cases
- Invoice subscription id from parent vs legacy field

## Cron refill (if applicable)

- Due row grants and advances cursor
- Canceled subscription skipped after user lock recheck
- Duplicate cron pass idempotent (zero row update)
- `endsAt` at period end skips (invoice owns renewal)

## Checkout actions (server)

- Unauthorized rejected
- Invalid plan/price input rejected
- Active subscription blocks new subscription checkout
- past_due routes to payment method portal when product defines that

## Runtime

- Webhook route: missing signature 400; oversize body 413; invalid signature 400
- Optional: constructEvent with known fixture secret

## Verification gate

Before marking work complete:

```
typecheck && lint && test && migrate && build
```

Plus manual Stripe CLI webhook forward for one happy path when first integrating.

## Adversarial test prompts

Write tests or manual cases that answer:

- Double webhook same event id
- Two different events same checkout session
- invoice.paid stale period
- subscription.deleted with replacement active
- DB throw after Stripe cancel in replacement flow → reconciliation
