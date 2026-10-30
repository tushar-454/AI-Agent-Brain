# Concurrency and idempotency

## Layers of defense

Use multiple layers; pick what fits the product:

| Layer | Protects against |
|-------|------------------|
| Webhook event ledger | Duplicate Stripe event delivery |
| Processed checkout session table | Duplicate checkout completion + async event |
| User row `SELECT … FOR UPDATE` | Concurrent grants/cancel for same user |
| Partial unique index (one active sub per user) | Two active subscription rows |
| Period guard on entitlement update | Duplicate renewal grant same period |
| Conditional UPDATE (`WHERE ends_at <= now AND …`) | Cron duplicate grants |
| Advisory xact lock on event id | Concurrent webhook workers same event |

Do not add locks everywhere — only around transitions that must serialize.

## Event ledger concurrency

Two webhook workers receiving the same event id:

- Both attempt INSERT/UPDATE claim
- Advisory lock ensures one proceeds; other gets `in_flight` or waits for stale claim

Configure stale claim minutes > p99 handler duration.

## User row locking

Before:

- Granting entitlements
- Downgrading on cancel
- Upserting subscription tied to user

Execute `FOR UPDATE` on the application user/account row (or dedicated billing row).

Recheck subscription/entitlement state **after** lock when cron and webhooks can race deletion flows.

## One active subscription slot

When product allows only one active subscription:

- Partial unique index on `user_id` where status in (`active`, `trialing`)
- On incoming active subscription: cancel older Stripe subscriptions that still occupy slot (compare `created` timestamps)
- If Stripe cancel succeeds but DB transaction must abort: reconciliation job updates old row status outside webhook tx

## Entitlement idempotency (period-based)

Update user entitlement only when:

- `current_period_end` is null or **less than** new period end (new billing period), **or**
- Explicit same-period plan change flag when product allows mid-period tier change

If zero rows updated, grant already applied — safe no-op.

Optional metadata (card brand): separate small transaction; failure logged, not rolled back with grant.

## Checkout vs webhook race

User completes Checkout while webhook processes:

- Session idempotency table prevents double grant
- Subscription upsert uses stripe id conflict target
- Active slot logic handles second subscription attempt

Checkout creation path should check DB **and** Stripe subscription list (first page often enough when newest-first) when webhooks lag.

## Cron vs webhook

When both exist:

- Cron selects due rows with strict predicates
- Inside tx: lock user, re-read subscription, verify still eligible
- Advance cursor (`ends_at`, `credit_reset_at`) in same tx as grant
- Use conditional UPDATE so second cron pass updates zero rows

## Stripe API idempotency keys

Database idempotency does not replace Stripe idempotency for **outbound** API calls.

Use deterministic keys when retrying:

- Checkout session create: e.g. hash(userId, priceId, mode, logical checkout attempt id)
- Subscription create/update when app retries after timeout

Random keys on each retry defeat Stripe deduplication.

## In-memory idempotency is insufficient

Do not rely on:

- Process memory sets
- Single-instance mutex on serverless/Workers

Always durable store for webhook and checkout session keys.
