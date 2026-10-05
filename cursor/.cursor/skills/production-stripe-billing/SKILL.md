---
name: production-stripe-billing
description: Implements production-ready Stripe payments and billing with project-aware discovery, webhook idempotency, subscription lifecycle, entitlement fulfillment, PostgreSQL concurrency patterns, and optional Cloudflare Workers/Hyperdrive adaptation. Use when adding Stripe Checkout, subscriptions, webhooks, billing portals, recurring entitlements, or auditing existing Stripe integration. Do not use for non-payment tasks or when the user only wants Stripe Dashboard setup without application architecture.
---

# Production Stripe Billing

Reusable workflow for **Stripe** integrated with the target project's stack. Treat any single repo (including a mature reference implementation) as **patterns to extract**, not a template to copy verbatim.

## When to use

- Greenfield Stripe on an existing app
- Extending partial payment code (Checkout only, no webhooks, etc.)
- Auditing billing before changes
- Porting billing across frameworks or deployments

## When not to use

- User only wants Stripe CLI or Dashboard steps with no app code
- Project uses a different payment provider and user did not ask to migrate
- Task is unrelated to payments (pure auth, UI-only pricing pages)

## Prerequisites

- Target repo checked out; read `package.json`, deployment config, env examples, existing schema
- Ability to run typecheck, lint, tests, migrations, and production build
- Stripe test mode keys for local webhook testing when implementing webhooks

## Requirement layers

| Layer | Scope | Rule |
|-------|--------|------|
| **A — Universal Stripe** | Any production integration | Webhook authority, idempotency, ownership validation, period correctness, external side effects ≠ DB transactions |
| **B — Stripe objects & events** | Checkout, PaymentIntent, Customer, Subscription, Invoice, Price | Use only what the product needs; map IDs to durable app state |
| **C — Database** | PostgreSQL (typical) | Event ledger, transactional fulfillment, row/advisory locks, unique constraints |
| **D — Cloudflare / edge** | Workers, OpenNext CF, Hyperdrive | Fetch HTTP client, raw webhook body, request-scoped DB, cron `scheduled` |
| **E — Project** | This app only | Plan names, prices, entitlements, tables, routes, env names — **discover and ask** |

State explicitly which layers apply after discovery.

## Architectural principles (enforce)

1. Stripe is payment authority.
2. Application DB is entitlement/state authority (for app access).
3. Webhook processing is server-side payment confirmation.
4. **Client redirect ≠ payment confirmation.**
5. Webhooks are retryable and must be idempotent.
6. DB transactions provide application atomicity only.
7. **DB rollback does not undo Stripe API calls** — use durable state + reconciliation.
8. Concurrency must be explicit around state transitions.
9. Billing periods must be validated (not event arrival order).
10. Stale events must not corrupt current state.
11. User ownership must be verified on every fulfillment path.
12. Client input must not determine price, plan, or amount.
13. Secrets never reach the client.
14. Cloudflare runtime constraints apply when deployed there.
15. Business rules remain project-specific.
16. Reuse existing infrastructure before adding queues/cron.

---

## Workflow overview

```
DISCOVER → ASK → DESIGN → IMPLEMENT → TEST → VERIFY → AUDIT
```

If payment code already exists: **AUDIT → GAP ANALYSIS → PROPOSE → IMPLEMENT APPROVED CHANGES** — never blind overwrite.

Enable only features the target project requires (one-time, subscription, portal, cron reconciliation, etc.).

---

## Phase 1 — Project discovery

Inspect (skip missing artifacts):

| Area | What to find |
|------|----------------|
| **Framework** | App Router, API routes, Server Actions vs route handlers for Checkout |
| **Runtime** | Node server, Workers, OpenNext, edge |
| **Deployment** | wrangler, Vercel, Docker; cron triggers |
| **Database** | Engine, ORM, migrations, transaction helpers |
| **Existing payment** | Checkout, webhooks, customer IDs on user table, subscription tables |
| **Auth** | How to bind payments to users (`requireUser`, session) |
| **Env** | Secret vs public vars; typed env module |
| **Request context** | Per-request `env`, DB, background work |
| **Entitlements** | Credits, seats, features — where stored and read |
| **Email / UI** | Post-purchase emails, success URLs, billing portal |

**Search terms:** `stripe`, `checkout`, `webhook`, `subscription`, `invoice`, `PaymentIntent`, `customer`, `metadata`, `client_reference_id`, `idempot`, `advisory`, `billing`.

Trace full paths: Checkout creation → Stripe → webhook → DB → entitlement read path.

---

## Phase 2 — Project-specific questions

Ask **only** what the repo cannot answer. See question bank in [references/subscription-lifecycle.md](references/subscription-lifecycle.md#project-question-bank).

Minimum decisions:

- Payment models: one-time, subscription, or both
- Intervals: monthly, yearly, custom
- What successful payment **grants** (entitlement model)
- Cancellation: immediate vs end of period
- Upgrades/downgrades/replacements required?
- Payment failure behavior
- Customer Portal, trials, coupons, tax
- Which tables hold users, subscriptions, entitlements, events

---

## Phase 3 — Architecture proposal

Before large diffs, document:

1. **Stripe object model** — which IDs are persisted
2. **DB model** — subscriptions, ledger, processed checkouts/sessions
3. **Flows** — one-time, subscription signup, renewal, cancel, replace
4. **Webhook pipeline** — verify → ledger claim → handler → mark processed
5. **Entitlement flow** — when and how grants happen (which events)
6. **Idempotency** — event ledger + secondary keys (checkout session, period guards)
7. **Concurrency** — user row lock, partial uniques, advisory locks
8. **Reconciliation** — replacement cancellation, cron backfill, Stripe list guards
9. **Failure recovery** — compensating actions when Stripe succeeds and DB fails
10. **Runtime** — Stripe SDK init, webhook raw body, DB lifecycle on Workers

Wait for user confirmation on ambiguous product choices.

---

## Phase 4 — Implementation checklist

### Stripe client (A + D)

- Initialize with secret from validated env/bindings only on server.
- On Workers/serverless fetch runtimes: use Stripe's **fetch HTTP client**, not Node-only agents.
- Pin or document `apiVersion`; retest webhooks when upgrading.

### Checkout & PaymentIntent (B + E)

- Create Checkout from **authenticated** server code; never trust client price IDs.
- Resolve price IDs from server config/env map validated at startup.
- Set `client_reference_id` and/or `metadata.userId` to application user id (consistent scheme).
- For subscriptions: mirror metadata on `subscription_data.metadata`.
- Use `mode: "payment"` vs `mode: "subscription"` appropriately.
- Guard duplicate subscriptions: local DB **and** Stripe list when webhooks lag.
- Consider Stripe **idempotency keys** on create calls when retries are possible (deterministic key per logical operation).
- Success/cancel URLs are UX only — fulfillment via webhooks.

### Customer (B)

- Store Stripe customer id on user/account row after first create.
- Create customer with metadata linking application user id.
- Validate customer belongs to authenticated user on every portal/checkout path.

### Webhook route (A + B)

Pipeline (details: [references/webhook-patterns.md](references/webhook-patterns.md)):

1. Read **raw** body (size limit).
2. Verify `stripe-signature` with webhook secret.
3. Parse event; optionally ignore unhandled types with 200.
4. Run **event ledger** claim inside a DB transaction.
5. Execute business handler in same transaction when possible.
6. Mark event processed **after** successful handler.
7. Return 200 on success; 4xx on bad signature; 5xx on retryable failure.

Use `constructEventAsync` when required by runtime.

### Event ledger (A + C)

Durable table keyed by Stripe **event id**. Claim pattern:

- Insert or bump attempts on conflict when not yet processed and not stale in-flight.
- Use **transaction-scoped advisory lock** on event id to serialize concurrent deliveries.
- Stale claim timeout so crashed workers release events.
- On handler failure: record error and release claim for Stripe retry — **do not** mark processed.

### Handlers (B)

Split responsibilities:

| Concern | Typical handlers |
|---------|------------------|
| Checkout completed | One-time fulfillment; subscription sync (may run before paid) |
| Async payment succeeded | Delayed payment methods |
| Subscription created/updated | Sync subscription row; selective credit on item change |
| Subscription deleted | End access if no other active subscription |
| Invoice paid | Recurring entitlement when billing reason + period validates |

Register only events the product needs ([references/subscription-lifecycle.md](references/subscription-lifecycle.md)).

### Subscription state (B + C)

- Persist stripe subscription id, status, price, period boundaries, cancel-at-period-end.
- Derive period from **subscription item** fields when API places them there.
- Partial unique index (or equivalent) for **one active slot per user** when product requires it.
- Replacement: cancel superseded Stripe subscription; if DB tx cannot commit after Stripe cancel, run **out-of-band reconciliation** (see [references/reconciliation.md](references/reconciliation.md)).

### Entitlement grants (A + E)

- Separate **sync subscription row** from **grant entitlement** when renewals and checkout overlap.
- One-time: idempotent key = checkout session id (dedicated table or ledger).
- Recurring: guard with **billing period** (user or subscription `current_period_end` must advance logically).
- For `invoice.paid`: filter `billing_reason`; compare invoice line period to subscription item period — skip stale/ahead/mismatch ([references/subscription-lifecycle.md](references/subscription-lifecycle.md)).
- Optional enrichment (card metadata, email): **non-critical** — failure must not undo grant.

### Cancellation (B)

- User-initiated: `cancel_at_period_end` when product keeps access until period end.
- Webhook `customer.subscription.deleted`: downgrade only if no other active subscription row.

### Cron / reconciliation (D + E)

Use only when entitlements need scheduled repair (e.g. yearly plan with monthly grants). Design:

- Idempotent selection query + user row `FOR UPDATE` + recheck state inside transaction.
- Never grant to canceled/expired/wrong-period rows.
- Coordinate with webhook grants to avoid double grant ([references/reconciliation.md](references/reconciliation.md)).

### Security (A)

- Webhook signature mandatory; no JSON parser before verify.
- Never trust client-supplied price, plan, customer, subscription ids without ownership checks.
- Validate paid line items against server price map on fulfillment.
- Short webhook error messages to client; log details server-side.

### Cloudflare (D)

When Workers/OpenNext + Hyperdrive: [references/cloudflare-workers.md](references/cloudflare-workers.md).

---

## Phase 5 — Testing

Required categories: [references/testing.md](references/testing.md).

Run project's test command after changes. Mock Stripe at service boundaries; test ledger, period comparison, and idempotency guards directly.

---

## Phase 6 — Verification standard

Not complete until:

- [ ] Typecheck, lint, tests, migrations, production build pass
- [ ] Webhook signature verified in integration test or documented manual check
- [ ] Duplicate webhook / concurrent delivery tests for critical grants
- [ ] Subscription lifecycle paths covered
- [ ] Failure matrix reviewed for target flows ([references/failure-matrix.md](references/failure-matrix.md))
- [ ] Worker compatibility confirmed when applicable

---

## Phase 7 — Adversarial audit (Greptile-style)

Answer **no** unless proven with code/tests:

- Can entitlement be granted twice for one payment?
- Can fulfillment happen without verified Stripe payment?
- Can user A receive user B's entitlement?
- Can stale invoice grant current period?
- Can canceled subscription receive future grants?
- Can concurrent webhooks duplicate state?
- Can cron + webhook race duplicate grants?
- Can Stripe succeed while DB fails without recovery path?
- Can replacement leave two active subscriptions in Stripe or DB?
- Can stale webhook overwrite newer state?
- Can duplicate event bypass ledger?
- Can attacker pick arbitrary price/customer id?
- Can webhook verification be skipped?
- Can secrets leak to client bundle?

Document any **accepted gaps** explicitly (e.g. unhandled `invoice.payment_failed` if product defers to Stripe dunning only).

---

## Reference implementation patterns (non-normative)

A mature reference may include:

- Event ledger with `pg_try_advisory_xact_lock(hashtext(event_id))` and stale claim window
- `processed_checkouts` / session idempotency table
- `SubscriptionCancellationReconciliationNeeded` + post-webhook reconciliation transaction
- Invoice line period vs subscription item period comparison before recurring grant
- Checkout transaction with user `FOR UPDATE` + Stripe subscription list guard
- Worker `scheduled` hook calling entitlement reconciliation job
- Optional session cache patch after billing (KV) — only if product uses cached session user JSON

Do **not** copy reference plan names, credit math, env var names, or cron business filters.

---

## Additional resources

| Topic | File |
|-------|------|
| Webhook pipeline & ledger | [references/webhook-patterns.md](references/webhook-patterns.md) |
| Subscriptions, invoices, cancellation | [references/subscription-lifecycle.md](references/subscription-lifecycle.md) |
| Locks, races, idempotency layers | [references/concurrency.md](references/concurrency.md) |
| Stripe↔DB failure recovery | [references/failure-matrix.md](references/failure-matrix.md) |
| Replacement & cron reconciliation | [references/reconciliation.md](references/reconciliation.md) |
| Workers, Hyperdrive, cron | [references/cloudflare-workers.md](references/cloudflare-workers.md) |
| Test matrix | [references/testing.md](references/testing.md) |

Read reference files when implementing or auditing that topic — not all files apply to every project.
