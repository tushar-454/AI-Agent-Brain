# Reference patterns (bg-remover)

Patterns observed in the reference implementation. **Reimplement the idea**, not file paths or product rules.

## Stack snapshot (reference only)

- Next.js App Router, OpenNext on Cloudflare Workers
- Better Auth + `@better-auth/drizzle-adapter` / `drizzleAdapter`
- Drizzle + `postgres` driver, schema includes extended `users` row
- Hyperdrive binding with `DATABASE_URL` fallback
- KV as Better Auth `secondaryStorage` + session user patch sync

## Request-scoped resources

```
ExecutionContext → WeakMap<RequestScope>
  env (merged bindings + process.env for local dev)
  db?  — lazy getDB(env)
  auth? — lazy getAuth(env)
  background — Set<Promise> drained via waitUntil / after()
```

- API auth route: `getRequestAuth()` then `auth.handler(request)`
- Server session: `auth.api.getSession({ headers: await headers() })`
- React `cache()` on per-request user lookup to dedupe RSC calls

## Auth factory highlights

- `getAuth` wrapped in `cache((env) => betterAuth({...}))` at module level, but **instance** stored on request scope via `getRequestAuth`
- `drizzleAdapter(db, { provider: "pg", schema: { ...allSchema }, usePlural: true })`
- `advanced.backgroundTasks.handler` → request background runner
- `advanced.database.generateId: false` when DB defaults UUIDs
- `advanced.ipAddress.ipAddressHeaders` set for CDN
- `session.storeSessionInDatabase: false` when using secondary storage
- `session.cookieCache` enabled with short `maxAge` when acceptable
- `emailAndPassword` + `socialProviders` + plugins (e.g. anonymous) as product requires
- Product-specific: signup hooks (blacklist, disposable email), `user.additionalFields` for billing — **not universal**

## Database connection (Cloudflare + Postgres)

```ts
connectionString = env.HYPERDRIVE?.connectionString ?? env.DATABASE_URL
postgres(connectionString, {
  fetch_types: false,
  prepare: false,
  max: 1,
  connect_timeout: 10,
  idle_timeout: 20,
})
```

## Env merge for dev/prod parity

Overlay `process.env` onto binding env object once per env reference (WeakMap memoized) so `betterAuth` and `getDB` read the same keys in `next dev` and Workers.

## Client

- `createAuthClient({ baseURL: publicAuthUrl, plugins: [...] })`
- `inferAdditionalFields<Auth>()` for typed extended user fields
- Forms call `authClient.signIn.email`, `signUp.email`, `signIn.social`, `signOut`, reset/verify APIs
- Map known error codes (e.g. email not verified) to UX flows

## Authorization layers (defense in depth)

1. **Server:** `requireUser` in layouts/actions; `getRequestSessionUser` for optional auth UI
2. **Client:** `RouteGuards` with `useSession` for redirect UX (not sole security boundary)
3. **Type guard:** `isSessionUser` on session user at boundaries

## Routing notes (reference)

- Catch-all under `/api/auth/[...all]`
- Locale proxy excludes `api` from matcher
- Auth pages under localized `(auth)` group
- `callbackUrl` query param with safe validation

## Migrations (reference)

- Separate drizzle configs for local vs production env files
- Auth schema lives in shared Drizzle schema export used by adapter

## Intentionally not universal

- Google OAuth only as example provider
- Email deliverability / blacklist hooks on signup
- Anonymous plugin
- KV session patching for billing fields
- Stripe-related user columns
- Rate limit custom rules per path
- Cloudflare Email Sending for transactional mail

Copy only what the target product needs after Phase 2 questions.
