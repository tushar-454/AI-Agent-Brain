---
name: production-auth-architecture
description: Implements production-ready authentication using Better Auth with project-aware discovery, database integration (Drizzle or existing ORM), runtime/request-context adaptation, and security verification. Use when adding auth, migrating auth, auditing existing auth, or reproducing a Better Auth + Drizzle + optional Cloudflare/Hyperdrive architecture in a new codebase. Do not use for non-auth tasks or when the user only needs Better Auth API snippets without full system design.
---

# Production Auth Architecture

Reusable workflow for **Better Auth** integrated with the target project's stack. Treat any single repo (including bg-remover) as a **reference implementation**, not a template to copy verbatim.

**Companion:** For Better Auth config field reference and CLI commands, also load `.agents/skills/better-auth-best-practices/SKILL.md` or project equivalent when present.

## When to use

- Greenfield auth on an existing app
- Replacing or extending partial auth
- Auditing auth before changes
- Porting auth patterns across frameworks or deployments

## When not to use

- User only wants docs for one Better Auth option (use better-auth-best-practices)
- Project explicitly uses another auth library and user did not ask to migrate
- Task is unrelated to identity/sessions (API keys only, no users)

## Prerequisites

- Target repo checked out; read `package.json`, deployment config, and existing env examples before editing
- Confirm license/compatibility for `better-auth` and chosen DB driver
- Ability to run typecheck, lint, build, and DB migrations in that project

## Requirement layers

| Layer | Scope | Rule |
|-------|--------|------|
| **A — Universal** | Any production auth | Secrets, HTTPS in prod, session invalidation, least privilege, no secret leakage |
| **B — Better Auth** | Library | Server `betterAuth()`, catch-all handler, client `createAuthClient`, env `BETTER_AUTH_*`, adapters |
| **C — Drizzle** | When Drizzle is the ORM | `drizzleAdapter`, schema tables, `provider` matches dialect, single shared `getDB` |
| **D — Cloudflare / edge** | Workers, OpenNext CF, Hyperdrive | Bindings, `postgres.js` + `prepare: false`, request-scoped resources, no Node-only APIs |
| **E — Project** | This app only | Routes, locales, business hooks, billing fields, anti-abuse — **discover, do not assume** |

State explicitly which layers apply after discovery (e.g. "D not applicable — Vercel Node runtime").

---

## Workflow overview

```
DISCOVER → ASK → DESIGN → IMPLEMENT → VERIFY → AUDIT
```

Never skip discovery. If auth already exists, **audit first** and propose minimal deltas.

---

## Phase 1 — Project discovery

Inspect (as applicable; skip missing artifacts):

| Area | What to find |
|------|----------------|
| **Framework** | App Router vs Pages vs other; API route conventions |
| **Runtime** | Node server, serverless, Workers, edge middleware |
| **Deployment** | Vercel, Cloudflare Workers/OpenNext, Docker, etc. |
| **Database** | Provider, existing client, connection lifecycle, pooling |
| **ORM / migrations** | Drizzle, Prisma, Kysely; migration commands and folders |
| **Existing auth** | NextAuth, custom JWT, Clerk, partial Better Auth |
| **Env** | `.env*`, typed env module, binding types, public vs secret vars |
| **Request context** | ALS, `getCloudflareContext`, custom scope per request |
| **Routing** | Locale prefixes, proxy/middleware matchers, API path for auth |
| **Session usage** | Server `getSession`, client hooks, protected layouts/actions |
| **Email** | Provider, templates, existing send helpers |

**Decision statements to produce internally:**

- "Auth exists → audit mode"
- "Not Cloudflare → Hyperdrive pattern N/A"
- "Prisma present → keep Prisma adapter unless user requests Drizzle"
- "Workers runtime → avoid Node-only drivers and globals"
- "Request context exists → extend it; else add minimal scope"

---

## Phase 2 — Project-specific questions

Ask **only** what the repo cannot answer. Use `AskQuestion` or a short numbered list.

### Authentication

- Login methods: email/password, OAuth providers, magic link, passkeys, anonymous?
- Email verification required? Password reset required? 2FA?
- Account linking rules?

### Database

- Engine and hosting? Existing schema for `user`?
- ORM and migration workflow?
- Reuse one connection factory for app + auth?

### Runtime / deployment

- Where does server code run (Node vs Worker vs edge)?
- Serverless constraints (connection limits, cold starts)?
- Hyperdrive or connection pooler desired?

### Application

- Protected vs public routes? Roles/permissions?
- Extra user fields (plan, credits, org id)?
- Admin surfaces?

### UX / routing

- Sign-in/up URLs? Post-login/logout redirects?
- Locale-aware paths? Client vs server route guards?

### Email

- Provider and env vars? Reuse existing mail layer?

**Do not re-ask** what `package.json`, wrangler, or env examples already prove.

---

## Phase 3 — Architecture proposal

Before large diffs, present a short proposal:

1. **Current state** — framework, runtime, DB, existing auth
2. **Risks** — duplicate DB clients, wrong driver, global singleton auth on Workers, cookie domain mismatch
3. **Better Auth** — enabled methods, plugins, `basePath`, `trustedOrigins`, rate limits
4. **Database** — adapter choice, table strategy (generate vs merge into existing `users`), migration plan
5. **Runtime** — how `env`/bindings reach `betterAuth()` and DB
6. **Routes** — catch-all handler path; public auth pages location
7. **Session** — storage (DB, secondary KV/Redis, cookie cache only); expiration; invalidation on password reset
8. **Env vars** — names aligned with project conventions (document new ones)

Wait for user confirmation on material choices (providers, session storage, ORM) when ambiguous.

---

## Architecture decision rules

### Better Auth server (B)

- **Factory, not global singleton on serverless:** `getAuth(env)` or equivalent, memoized **per request scope** when the runtime reuses isolates (Workers). On classic Node, project may use module-level singleton if one process owns one pool — match existing DB pattern.
- **Config inputs:** `secret`, `baseURL`, `trustedOrigins` from validated env — never hardcode production URLs.
- **Secure cookies:** `useSecureCookies` (or equivalent) tied to HTTPS base URL in production.
- **Database:** Use official adapter for the project's ORM. Pass full schema map if tables are customized or extended.
- **User extensions:** `user.additionalFields` for app-specific columns; keep `input: false` for server-managed fields.
- **Email/password:** Configure `sendResetPassword`, `emailVerification.sendVerificationEmail` via project's mail layer; wire `env` through closures, not `process.env` scattered in handlers.
- **OAuth:** Only configure providers the product needs; credentials from env/bindings.
- **Hooks:** Use for cross-cutting rules (signup abuse, field restrictions). Fail closed on security checks when appropriate.
- **Background work:** On Workers, route Better Auth `backgroundTasks` to `waitUntil`/request background queue — never fire-and-forget unawaited promises that must complete after response.
- **IP / headers:** Set `ipAddressHeaders` to what the deployment actually sends (e.g. CDN connecting IP), not a default from another host.

### Better Auth client (B)

- Separate module from server auth; **no server secrets** on client.
- `baseURL` from **public** env var (project naming may differ; mirror server origin).
- Plugins: match server (`inferAdditionalFields`, anonymous, etc.).
- Framework import: `better-auth/react`, `better-auth/client`, etc., per target UI.

### HTTP surface (B + E)

- Expose Better Auth via framework catch-all: `GET`/`POST` → `auth.handler(request)`.
- Resolve `auth` through the same request context as DB.
- Do not duplicate auth logic in custom API routes unless an exception (large uploads) applies per project rules.

### Drizzle integration (C)

- **One `getDB(env)`** used by app services and `drizzleAdapter`.
- `provider`: `"pg"` | `"mysql"` | `"sqlite"` must match driver.
- `usePlural: true` only when table names are plural — match generated schema.
- Auth tables: `user`, `session`, `account`, `verification` (+ plugin tables). Merge with existing `users` table carefully; prefer CLI `generate` then edit.
- Migrations: follow project's existing drizzle-kit (or other) scripts; separate local/prod configs if the repo does.
- **Workers + Postgres:** prefer `postgres` (postgres.js) with `max: 1`, `prepare: false`, `fetch_types: false` unless project already standardized otherwise.

### Cloudflare / Hyperdrive (D — conditional)

Apply only when discovery confirms Cloudflare Workers/OpenNext + Hyperdrive (or user requests it):

- Connection string: `hyperdriveBinding.connectionString ?? DATABASE_URL` (binding names vary — read wrangler/types).
- Local dev: Hyperdrive local connection string env if documented in that project.
- Merge bindings with `process.env` for `next dev` parity if the reference pattern exists.
- **Not on Cloudflare:** use direct URL, pooler (Neon, Supabase), or host's recommended serverless driver — do not add Hyperdrive.

### Request context (A + D + E)

Server auth and DB must see the **same request-scoped `env`**:

- WeakMap keyed by `ExecutionContext` (or ALS) holding `{ env, db?, auth?, background tasks }`.
- `getRequestAuth()` / `getRequestDB()` read Cloudflare context or project equivalent.
- Cleanup: `after()` / `waitUntil` to drain background work and release scope.
- **No** mutable global current-request state.

### Session architecture (decide per project)

Do **not** copy reference settings blindly. Choose explicitly:

| Concern | Options / notes |
|---------|------------------|
| Primary session store | DB, secondary storage (KV/Redis), or cookie-only |
| `storeSessionInDatabase` | `true` if DB is source of truth; `false` if secondary storage holds sessions |
| `cookieCache` | Short TTL reduces DB/KV reads; tune `maxAge` vs freshness needs |
| Server session | `auth.api.getSession({ headers })` with framework `headers()` |
| Client session | `authClient.useSession()` / `getSession()` |
| Protected routes | Server: layout/action `requireUser`; Client: optional guard for UX |
| Logout | `signOut`; revoke sessions on password reset if required |
| User data freshness | If sessions cache user JSON in KV, define patch/sync when profile changes |

Reference pattern (optional): sessions in secondary storage, not DB; cookie cache ~2 min; KV sync on profile updates — **only if product needs it**.

### Environment (E)

- Discover naming: server URL, public URL, OAuth secrets, DB URL.
- Better Auth convention: `BETTER_AUTH_SECRET`, `BETTER_AUTH_URL`; client often needs a public mirror — name per project (`NEXT_PUBLIC_*` or other).
- Parse and validate in the project's env module (Zod or generated binding types).
- Document every new variable in env example files.

### Routing (E)

- Auth API path must not conflict with locale proxy matchers (exclude `api` in middleware/proxy if needed).
- Auth pages: follow existing route groups (`(auth)`, localized segments).
- Redirects: preserve `callbackUrl` with safe allowlist (no open redirects).

### Server authorization helpers (E)

Patterns to adapt (names/locations vary):

- `getSessionUser(auth)` + type guard at boundary
- `getRequestSessionUser` cached per request (React `cache` on Next)
- `requireUser(pathname)` → redirect to sign-in with return URL
- `redirectIfAuthenticated` on auth pages

Implement using target project's redirect and i18n utilities.

### Actions and API routes (E)

- Server Actions: call `requireUser` or `getSessionUser` at start; return `{ data, error }` per project rules.
- GET routes: same session helpers as actions.

---

## Phase 4 — Implementation

Order (adjust to project feature-build rules):

1. Types + validators for auth forms (if UI in scope)
2. DB schema / migrations for auth tables
3. `getDB` / request context (if missing)
4. Server `getAuth` factory + adapter
5. API catch-all route
6. Client auth module
7. Auth pages and forms (minimal first: sign-in, sign-out)
8. Protected layouts/actions
9. Email flows (verify, reset) if enabled
10. OAuth buttons if enabled
11. Optional: secondary storage, rate limits, hooks (abuse), plugins (anonymous)

**Rules:**

- Match project naming, folder structure, and `function` keyword conventions.
- Minimal diff; no unrelated refactors.
- Strict TypeScript; type guards at session boundaries; no `any` without justification.
- Reuse existing email, logging, and validation utilities.

---

## Phase 5 — Verification

Run project scripts (examples):

- `tsc` / `npm run build` / `npm run lint` / `npm test`
- DB: `drizzle-kit check`, migrate against dev DB

**Manual / automated checks:**

| Check | Expected |
|-------|----------|
| `GET /api/auth/ok` (or configured base) | `{ status: "ok" }` |
| Sign up | User row + account; verification if enabled |
| Sign in | Session cookie set; server `getSession` works |
| Sign out | Session cleared |
| Protected route | Redirect or 401 per design |
| Bad password | Safe error message |
| Invalid/expired session | Treated as anonymous |
| OAuth | Callback completes if enabled |
| Password reset | Token email + login with new password |
| Production build | No Node-only imports in Worker bundles |

---

## Phase 6 — Final audit

- [ ] No secrets in repo; examples use placeholders only
- [ ] Env vars documented; public vs private correct
- [ ] Single DB client path for app + auth
- [ ] No duplicate auth routes or legacy auth left half-enabled
- [ ] Cookies: `Secure`, `SameSite`, domain/path correct for deployment
- [ ] `trustedOrigins` matches real origins
- [ ] OAuth redirect URIs match Better Auth base URL
- [ ] Migrations applied; schema matches adapter map
- [ ] Worker/edge bundle free of `fs`, `pg` Pool, etc., if applicable
- [ ] Existing features still work (regression smoke)
- [ ] Types and lint clean

---

## Security checklist

- HTTPS in production; secure cookies when on HTTPS
- `BETTER_AUTH_SECRET` strong (≥32 chars); rotation plan documented
- `trustedOrigins` minimal
- CSRF: follow Better Auth defaults; no custom endpoints bypassing CSRF without reason
- OAuth state/redirect validation via library
- Password hashing: Better Auth only — no custom crypto
- Session expiration enforced; revoke on password reset if required
- Rate limit sensitive endpoints (sign-in, sign-up, reset, verify email)
- Open redirect: sanitize `callbackUrl`
- Fail closed on authz failures in actions (no partial mutations)
- Separate dev/staging/prod credentials

---

## Error handling

| Source | Approach |
|--------|----------|
| Better Auth API | Map `error.code` / `message` to UI toasts or form errors; handle `EMAIL_NOT_VERIFIED` explicitly if verification required |
| DB | Log server-side; user sees generic message per project rules |
| Invalid session | Anonymous; redirect if route requires user |
| OAuth | Surface provider errors safely; log details server-side |
| Email send failures | Log; optional retry; do not claim success to user |

Adapt to project's action `{ data, error }` or API JSON shape.

---

## Documentation deliverables

After implementation, add or update project docs (README or `docs/auth.md`):

- Required env vars and bindings
- Auth routes (API + pages)
- Enabled providers and plugins
- DB tables and migration commands
- Local dev setup (DB, OAuth localhost URLs)
- Deployment notes (Hyperdrive, KV, cron if any)
- Architectural decisions (session storage, guards server vs client)

---

## Failure and recovery

- **Migration failed:** Roll back migration file; fix schema; never leave partial auth tables without adapter alignment.
- **Sessions not sticking:** Check `baseURL`, cookie domain, `trustedOrigins`, HTTPS mismatch, proxy stripping cookies.
- **Worker DB errors:** Verify driver (`postgres.js`), `prepare: false`, connection string source, `max` connections.
- **OAuth redirect mismatch:** Align provider console URLs with `BETTER_AUTH_URL` + callback path.
- **Duplicate users:** Review sign-up hooks and provider linking settings.

---

## Reference implementation patterns

For concrete patterns extracted from the bg-remover reference repo (request scope, Hyperdrive fallback, KV secondary storage, locale-aware guards), see [reference-patterns.md](reference-patterns.md).

When invoked **in another repository**, do not import paths from bg-remover — re-derive equivalents from that repo's structure.
