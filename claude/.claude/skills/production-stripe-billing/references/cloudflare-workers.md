# Cloudflare Workers and OpenNext

Apply this file only when discovery confirms Workers, OpenNext on Cloudflare, or similar fetch-based serverless runtime.

## Stripe SDK

- Use `Stripe.createFetchHttpClient()` in client options
- Avoid Node `http`/`https` agents
- Keep `server-only` (or equivalent) on modules that import secret key
- Pin `apiVersion`; retest webhooks after upgrades

## Webhook route on Workers

- Read body via streaming reader with max bytes (do not use middleware that consumes body twice)
- `constructEventAsync` when sync construct unavailable
- Return JSON responses; map ledger `failed` / `in_flight` to 500

## Environment

- Secrets: `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET` via bindings or env module — names vary by project
- Public: publishable key only in `NEXT_PUBLIC_*` or client-safe binding
- Price/product ids: server-side map; validate at startup with Zod or generated binding types

Do not hardcode reference project's price env names — discover or define new schema.

## Request context

- Resolve `env` and DB per request (WeakMap + `ExecutionContext`, ALS, or OpenNext `getCloudflareContext`)
- Webhook handler uses same `getRequestDB()` as Server Actions
- Pass `env` into Stripe factory: `getStripe(env)`

## Hyperdrive + PostgreSQL + Drizzle

When Hyperdrive binding present:

- Connection string from binding in production; direct URL locally
- Driver: often `postgres.js` with `max: 1`, `prepare: false` for serverless
- One `getDB(env)` shared by services and webhook
- Transactions: webhook ledger + handler in one `db.transaction`

Read target project's existing DB module — extend, do not duplicate pools.

## Cron (`scheduled`)

When recurring entitlement repair runs on schedule:

- Export handler on custom worker wrapper: `scheduled` → dynamic import job module
- Pass `env` and `scheduledTime` as logical `now` for idempotency tests
- Close DB client in `finally` if job opens dedicated connection (`$client.end`)

Configure cron in wrangler; frequency is product-specific.

## Background work

- Use `ctx.waitUntil` for non-blocking post-response work when appropriate
- Critical entitlement grants stay **awaited** inside webhook before 200 unless product explicitly accepts async fulfillment

## Compatibility checklist

- [ ] No `fs` / `child_process` in Stripe path
- [ ] Webhook raw body preserved
- [ ] Stripe uses fetch client
- [ ] DB credentials not exposed to client bundle
- [ ] OpenNext build produces worker entry referenced by custom worker

## Not on Cloudflare

Skip Hyperdrive and Worker cron sections. Use host-recommended Postgres pooling (Neon serverless, RDS proxy, etc.).
