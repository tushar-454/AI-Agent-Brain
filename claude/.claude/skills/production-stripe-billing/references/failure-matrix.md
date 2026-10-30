# Failure matrix

For each scenario, define **expected state**, **recovery**, **retry**, **reconciliation**. Adapt rows to enabled features.

## Checkout creation

| Failure | Expected state | Recovery |
|---------|----------------|----------|
| Stripe API error | No session; user unchanged | User retries; optional idempotency key |
| Stripe OK, DB rollback after customer create | Customer may exist in Stripe | Store customer id on retry lookup; reconcile orphan customers periodically |
| User already subscribed | No new session | Show manage billing / portal |

## Payment fulfillment

| Failure | Expected state | Recovery |
|---------|----------------|----------|
| Webhook delayed | User paid in Stripe; app not granted | Webhook retry or manual replay; Checkout success URL shows pending |
| Duplicate webhook | Single grant | Ledger + session idempotency |
| Grant OK, email fails | Entitlement OK | Log; retry email async |
| Grant OK, KV/cache patch fails | DB OK; stale session | Patch on next request or cron |

## Stripe succeeds, DB fails

| Failure | Expected state | Recovery |
|---------|----------------|----------|
| Handler throws mid-tx | Event not processed | Stripe retries webhook |
| mark_processed not reached | Event released | Stripe retries |
| Partial tx (should not happen) | Design single tx boundary | Fix code |

## DB succeeds, Stripe fails later

| Failure | Expected state | Recovery |
|---------|----------------|----------|
| Local cancel flags set; Stripe update failed | Mismatch | Retry Stripe update; admin tool |
| Replacement: new sub active in Stripe, old not canceled | Two actives | Reconciliation cancels old in Stripe + DB |

## Webhook ordering

| Failure | Expected state | Recovery |
|---------|----------------|----------|
| `invoice.paid` before `checkout.session.completed` | Subscription row may exist | Period guards; checkout session idempotency |
| Stale `invoice.paid` | No grant | Period comparison skip |
| `subscription.deleted` before `updated` | Final state from deleted handler | Check for other active sub |

## Subscription replacement

| Failure | Expected state | Recovery |
|---------|----------------|----------|
| Cancel old in Stripe OK; DB tx rolls back | Old canceled in Stripe; DB still active | Typed reconciliation updates old row |
| New sub wins in Stripe; DB shows old | User wrong plan | Webhook upsert + slot claim |
| Both active in Stripe | Violation | List subscriptions; cancel older |

## Invoice / period

| Failure | Expected state | Recovery |
|---------|----------------|----------|
| Missing line period on create | Throw; retry | Stripe may finalize lines later |
| Wrong subscription item on invoice | No grant | Log; alert |
| Proration-only invoice | No recurring grant | Exclude proration lines |

## Concurrency

| Failure | Expected state | Recovery |
|---------|----------------|----------|
| Duplicate concurrent webhook | One processed | Ledger |
| Cron + webhook same period | One grant | Period guard + conditional cron update |
| Deadlock | 500 webhook | Retry |

## Runtime (Workers)

| Failure | Expected state | Recovery |
|---------|----------------|----------|
| Webhook body parsed as JSON before verify | Security break | Always raw body |
| DB connection not closed on cron | Leak | `finally` end connection per job design |
| Node-only Stripe client | Runtime error | Fetch HTTP client |

## Security

| Failure | Expected state | Recovery |
|---------|----------------|----------|
| Forged webhook | 400 | No state change |
| User A supplies B's customer id | Reject at checkout | Auth + row ownership |
| Client tampered price id | Server maps from catalog | Fulfillment uses Stripe line items |

Document project-specific gaps after audit (e.g. no `invoice.payment_failed` handler).
