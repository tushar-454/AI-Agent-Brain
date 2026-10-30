# CLAUDE.md — Master Identity & Orchestration

You are the coding agent for the **PAI** project inside Claude Code. This file is your entry point:
it defines who you are, what you must always follow, and how to route work to the right rules, skills,
and MCP tools before writing any code.

**Load order on every task:**

1. Read this file fully.
2. Read every rule in `.claude/rules/` — these are non-negotiable project standards.
3. Check the Skill Routing Table below. If a skill matches the task, open its `.claude/skills/<name>/SKILL.md` and follow it.
4. Check available MCP tools configured in `.mcp.json` when the task requires external tool access.
5. Only then start planning/writing code.

Never skip step 2. Never guess at a convention this file or `.claude/rules/` already answers.

---

## 1. Identity & Architecture

- You write production-grade TypeScript for a **Next.js App Router + Drizzle ORM + Postgres + Zustand + Tailwind / shadcn/ui + Zod** codebase.
- You follow the **service-based architecture** defined in `.claude/rules/00-overview.md`:
  `Service (pure DB) → Action (auth + validation + orchestration) → Route / Component (consumption)`.
- You are precise and intentional in code — comments explain _why_, never _what_.
- You never invent a folder, naming pattern, or export style that contradicts `.claude/rules/`. If a rule and a skill conflict, **rules win** — skills are technique/domain guidance, rules are structural law.
- If a task is ambiguous and no rule/skill resolves it, ask one short clarifying question instead of guessing.

---

## 2. Rules — Always Active (`.claude/rules/`)

These apply to every single task, no exceptions:

| File | Governs |
| --- | --- |
| `00-overview.md` | Stack, service-based architecture, and exact 7-step feature build order |
| `01-folder-structure.md` | Strict type-based folder organization and where every file type lives |
| `02-naming-conventions.md` | Kebab-case naming, `<name>.<folder-singular>.ts` pattern |
| `03-typescript-types.md` | `function` keyword only, `type` over `interface`, `/types` single source of truth, type guards |
| `04-validation-zod.md` | Zod schemas with `.pipe()`, short precise error messages |
| `05-service-layer.md` | Pure DB operations only (Drizzle), no auth or validation |
| `06-action-layer.md` | Auth + validation + service orchestration, typed `{ data, error }` responses |
| `07-api-routes.md` | GET-only routes calling the action directly; mutations are actions only |
| `08-state-management.md` | Zustand store structure, client-only state |
| `09-components.md` | UI primitives vs feature components, `"use client"` on deepest leaf only, no local prop types |
| `10-imports-exports.md` | Named exports everywhere, `@/` alias always |
| `11-error-formatting-env.md` | Error shape, formatting, type-safe env access via `lib/env.ts` |

**Before writing any file**, confirm its path, name, and export style against these rules.

---

## 3. Skill Routing Table (`.claude/skills/`)

Skills follow the Agent Skills standard and are **conditionally loaded** — only read the ones relevant to the current task. Each skill's `SKILL.md` is the authoritative guide.

### 3.1 Technical / Domain Skills

| Trigger / Task Looks Like... | Skill to Load | What It Actually Does |
| --- | --- | --- |
| Setting up or editing auth: email/password, OAuth, sessions, plugins, DB adapters | `better-auth-best-practices` | Configure Better Auth server + client, sessions, plugin config |
| Greenfield auth, migration, or auditing full auth architecture (Better Auth + Drizzle) | `production-auth-architecture` | End-to-end production auth integration, request context, and security verification |
| Adding Stripe Checkout, subscriptions, webhooks, billing portals, recurring entitlements | `production-stripe-billing` | Production-grade Stripe integration, webhook idempotency, Postgres concurrency patterns |
| Deciding module boundaries, interfaces, what to expose vs hide, refactoring for maintainability | `codebase-design` | Deep-module design principles — small interfaces hiding complex logic |
| Any new UI, layout, or visual direction decision — general case | `frontend-design` | Distinctive visual design guidance, avoiding generic defaults |
| A polished/premium surface: landing page, marketing hero, executive dashboard | `high-end-visual-design` | Agency-level visual polish, micro-interactions, typography, animations |
| Writing/reviewing raw SQL, schema design, JSONB indexing, arrays, RLS, Postgres tuning | `postgresql-code-review` | Postgres anti-pattern review, indexing, RLS security |
| Adding, composing, styling, or fixing a shadcn/ui component | `shadcn` | shadcn/ui CLI workflows and component composition |
| Optimizing React/Next.js rendering, data fetching, re-renders, bundle size | `vercel-react-best-practices` | Vercel's 70-rule performance and pattern guide for React + Next.js |
| Diagnosing slow pages, Core Web Vitals (LCP/INP/CLS), render-blocking assets, caching | `web-perf` | Web performance and accessibility audit guide |
| Writing/debugging Cloudflare Workers code (promises, global state, streaming, bindings) | `workers-best-practices` | Cloudflare Workers anti-pattern review |
| Wrangler CLI config, `wrangler.toml`, deploying Workers/KV/R2/D1/Hyperdrive/Queues | `wrangler` | Cloudflare Workers CLI and deployment reference |

### 3.2 Communication & Workflow Skills

| Trigger | Skill | What It Does |
| --- | --- | --- |
| Ultra-terse output requested or exploring large codebases where verbosity wastes context | `caveman` | Cuts output tokens ~65%, compressed but technically accurate phrasing |
| Writing a git commit message | `caveman-commit` | Conventional Commits format, <50 char subject, "why" over "what" |
| Standard git commit formatting | `git-commit` | Commit message guidelines and conventional commit standards |
| Doing a PR/diff review pass | `caveman-review` | One line per finding: location, problem, fix |
| Delegating large tasks to subagents | `cavecrew` | Subagent orchestration and context-compressed task delegation |

---

## 4. Model Context Protocol Tools (`.mcp.json`)

When an external tool is required, leverage the connected MCP servers:

| MCP Server | Trigger / Use When |
| --- | --- |
| `chrome-devtools` | Running performance audits, checking Core Web Vitals, diagnosing render bottlenecks |
| `shadcn` | Searching, adding, or validating shadcn/ui components and registries |
| `better-auth` | Querying Better Auth schemas, plugin options, and authentication configurations |

---

## 5. Standard Task Workflow

For any new feature, follow the build order from `.claude/rules/00-overview.md`:

1. `types/<name>.type.ts` — entity type, input/output types, prop types, type guard
2. `validators/<name>.validator.ts` — Zod schema with `.pipe()`
3. `db/schema/<name>.ts` — Drizzle table schema
4. `services/<name>.service.ts` — pure DB ops (no auth, no validation)
5. `actions/<name>.action.ts` — auth + validation + service call + typed `{ data, error }` response
6. `app/api/<name>/route.ts` — **GET only**, calls the action directly
7. `components/` — UI last, `"use client"` on the deepest leaf only

For bug fixes or edits: locate the correct layer first (service, action, or component) using `01-folder-structure.md`, fix at that layer only, and never restructure unrelated code.

---

## 6. Custom Project Commands (`.claude/commands/`)

Claude Code provides these project-scoped slash commands:

- `/project:commit` — Drafts a conventional git commit message adhering to project standards.
- `/project:review` — Runs a 1-line-per-finding code review pass checking against `.claude/rules/`.
- `/project:build-feature <description>` — Scaffolds a complete feature following the 7-step build order.

---

## 7. Common Development Commands

- `npm run dev` — Start the local development server
- `npm run build` — Build production Next.js bundle
- `npm run lint` — Run ESLint across codebase
- `npm test` — Run test suite
- `npx drizzle-kit generate` — Generate SQL migrations from schema
- `npx drizzle-kit migrate` — Apply database migrations

---

## 8. Non-Negotiables Checklist

Before concluding any coding task, verify every checkbox:

- [ ] `function` keyword used everywhere (no arrow function declarations)
- [ ] `type` used over `interface` unless extension is required
- [ ] All types/props live in `/types`, none declared inline in components
- [ ] `@/` alias used for all internal imports (no relative `../../` paths)
- [ ] Named exports everywhere (except Next.js special files)
- [ ] Zod validation on every action input using `.pipe()` and concise error messages
- [ ] No DB calls outside `services/`, no auth/validation outside `actions/`
- [ ] GET requests use route + action; mutations (POST/PUT/DELETE) use actions only
- [ ] `"use client"` pushed to the deepest leaf component only
- [ ] Relevant skill(s) from the routing table consulted before finalizing
