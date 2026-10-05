# Reconciliation

## External side effects vs database

**Invariant:** PostgreSQL transaction rollback does **not** undo:

- Stripe customer/subscription create or cancel
- Checkout Session creation
- PaymentIntent confirmation

Design:

1. Perform Stripe mutations only when necessary
2. Persist intent/outcome in DB when possible
3. On DB failure after Stripe success, leave event **unprocessed** for webhook retry **or** run compensating reconciliation

Never assume one distributed transaction across Stripe and Postgres.

## Subscription replacement cancellation

**Problem:** Webhook handler cancels old Stripe subscription (external success), then DB work fails → old row still `active` in DB.

**Pattern:**

1. During slot claim, cancel superseded subscription via Stripe API
2. Throw typed error carrying `{ userId, subscriptionRowId, stripeId, stripeStatus, canceledAt }`
3. Webhook outer layer catches; runs **separate** DB transaction:
   - Lock user row
   - Verify row still matches stripe id and was active
   - Update status to canceled if Stripe payload confirms non-active
4. Retry original webhook event (ledger releases on failure)

Cap reconciliation attempts to avoid infinite loops.

## Stripe as source of truth guards

When local DB may lag webhooks:

- Before new subscription Checkout: list customer subscriptions (bounded page) for blocking `active` / `past_due`
- On upsert: retrieve subscription if webhook payload incomplete
- On cancel: verify remote status if API errors with `resource_missing`

## Orphan Stripe customers

If customer created then DB update failed:

- Next checkout: read `stripeId` from user if set; else create customer again only after checking Stripe by email/metadata (product policy)

Optional admin script: list customers with metadata userId not in DB.

## Cron reconciliation (entitlements)

Use when billing interval ≠ entitlement reset interval (product-specific).

**Selection query:** subscriptions due by cursor timestamp, status allowlist, plan allowlist — **all project-defined**

**Per row:**

1. Open transaction
2. `FOR UPDATE` user
3. Re-select subscription; verify still refillable (not canceled, period not ended, cursor within year)
4. Conditional update subscription cursor
5. Grant entitlement in same transaction
6. Commit

**Failures:** log per row; fail job if any hard error when product requires all-or-nothing

**Coordination with invoice.paid:** yearly renewal may reset cursor; cron handles intra-year resets — define single owner for each period boundary to avoid double grant.

## Webhook replay

Manual replay in Stripe Dashboard:

- Safe if ledger deduplicates processed event ids
- Unsafe if event id changed (new delivery id) but same business effect — rely on session/period guards

## Session / cache sync

If app caches user entitlements in KV:

- After webhook commit, patch cached sessions from DB row
- Cron job may call same patch helper without Next request context
- KV failure is non-critical; DB remains authoritative
