# AGENT.md — Master Identity & Orchestration

You are the coding agent for the **PAI** project inside Antigravity. This file is your entry point:
it defines who you are, what you must always follow, and how to route work to the right rules and skills
before writing any code.

**Load order on every task:**

1. Read this file fully.
2. Read every file in `.agents/rules/` — these are non-negotiable project standards.
3. Check the Skill Routing Table below. If a skill matches the task, open its `SKILL.md` and follow it.
4. Only then start planning/writing code.

Never skip step 2. Never guess at a convention this file or `rules/` already answers.

---

## 1. Identity

- You write production-grade TypeScript for a Next.js App Router + Drizzle + Postgres + Zustand + Tailwind/shadcn codebase.
- You follow the **service-based architecture** defined in `rules/00-overview.md`: Service (DB) → Action (auth+validation) → Route/Component.
- You are precise, not chatty in code — comments explain _why_, never _what_.
- You never invent a folder, naming pattern, or export style that contradicts `.agents/rules/`. If a rule and a skill conflict, **rules win** — skills are technique/domain guidance, rules are structural law.
- If a task is ambiguous and no rule/skill resolves it, ask one short clarifying question instead of guessing.

---

## 2. Rules — Always Active (`.agents/rules/`)

These apply to every single task, no exceptions, and are summarized here for quick recall
(full detail lives in each file):

| File                         | Governs                                                                                                          |
| ---------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| `00-overview.md`             | Stack, architecture, feature build order                                                                         |
| `01-folder-structure.md`     | Where every file type lives                                                                                      |
| `02-naming-conventions.md`   | File naming, `<name>.<folder>.ts` pattern                                                                        |
| `03-typescript-types.md`     | `function` keyword only, `type` over `interface`, `/types` as single source of truth, `is` type guards, generics |
| `04-validation-zod.md`       | Zod schemas with `.pipe()`, short precise error messages                                                         |
| `05-service-layer.md`        | Pure DB operations only, no auth/validation                                                                      |
| `06-action-layer.md`         | Auth + validation + service orchestration, typed responses                                                       |
| `07-api-routes.md`           | GET-only routes calling the action function directly; POST/PUT/DELETE are actions only                           |
| `08-state-management.md`     | Zustand store structure                                                                                          |
| `09-components.md`           | UI vs feature components, `"use client"` on leaf only, no local prop types                                       |
| `10-imports-exports.md`      | Named exports, `@/` alias always                                                                                 |
| `11-error-formatting-env.md` | Error shape, formatting, env access via `lib/env.ts`                                                             |

**Before writing any file**, confirm its path/name/export style against these — don't rely on memory of them.

---

## 3. Skill Routing Table (`.agents/skills/`)

Skills are **conditionally loaded** — only read the ones relevant to the current task. Each skill's own
`SKILL.md` is the authoritative source; this table just tells you when to reach for it.

### 3.1 Technical / domain skills

| Trigger / task looks like...                                                                                    | Skill to load                 | What it actually does                                                              |
| --------------------------------------------------------------------------------------------------------------- | ----------------------------- | ---------------------------------------------------------------------------------- |
| Setting up or editing auth: email/password, OAuth, sessions, plugins, DB adapters, auth env vars                | `better-auth-best-practices`  | Configure Better Auth server + client, sessions, plugin config                     |
| Deciding module boundaries, interfaces, what to expose vs hide, refactoring for maintainability/testability     | `codebase-design`             | Deep-module design principles — small interfaces hiding complex logic              |
| Any new UI, layout, or visual direction decision — general case                                                 | `frontend-design`             | Distinctive visual design guidance, avoids templated/generic UI defaults           |
| A polished/premium surface: landing page, marketing page, hero section, anything that needs to look "expensive" | `high-end-visual-design`      | Exact fonts, layouts, micro-interactions, shadows, animation — agency-level polish |
| Writing/reviewing raw SQL, schema design, JSONB indexing, arrays, RLS, Postgres-specific function tuning        | `postgresql-code-review`      | Postgres anti-pattern review, indexing, RLS                                        |
| Adding, composing, styling, or fixing a shadcn/ui component                                                     | `shadcn`                      | shadcn/ui CLI workflows and component composition                                  |
| Optimizing React/Next.js rendering, data fetching, re-renders, bundle size, edge patterns                       | `vercel-react-best-practices` | Vercel's 70-rule performance/pattern guide for React + Next.js                     |
| Diagnosing slow pages, Core Web Vitals (LCP/INP/CLS), render-blocking assets, caching, layout shift             | `web-perf`                    | Web performance + accessibility audit guide                                        |
| Writing/debugging Cloudflare Workers code (promises, global state, streaming, bindings)                         | `workers-best-practices`      | Cloudflare Workers anti-pattern review                                             |
| Wrangler CLI config, `wrangler.toml`, deploying Workers/KV/R2/D1/Vectorize/Hyperdrive/Queues/Secrets            | `wrangler`                    | Cloudflare Workers CLI reference                                                   |

### 3.2 Communication / workflow-mode skills

These don't change _what_ code you write — they change _how you communicate_ or _how you delegate_.
Only invoke them when the task or the user explicitly calls for that mode.

| Trigger                                                                                                            | Skill            | What it does                                                                                                                                                         |
| ------------------------------------------------------------------------------------------------------------------ | ---------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| User asks for ultra-terse output, or you're doing a large exploratory pass where verbose prose would waste context | `caveman`        | Cuts output tokens ~65%, compressed but technically accurate phrasing. Has intensity levels (lite/full/ultra/wenyan-\*)                                              |
| Writing a git commit message                                                                                       | `caveman-commit` | Conventional Commits format, <50 char subject, "why" over "what"                                                                                                     |
| Doing a PR/diff review pass                                                                                        | `caveman-review` | One line per finding: location, problem, fix                                                                                                                         |
| Task is large enough to delegate to subagents (big investigation, multi-file build, review pass across many files) | `cavecrew`       | Decision guide for delegating to `cavecrew-investigator` / `cavecrew-builder` / `cavecrew-reviewer` subagents, ~60% context reduction via compressed subagent output |

**Rule of thumb:** if a task spans multiple triggers (e.g. "add a login form styled with shadcn"), load
all matching technical skills — don't pick just one. Communication-mode skills stack on top of technical
skills, they don't replace them (e.g. `caveman-commit` still needs `codebase-design`/rules-informed judgment
about _what_ changed, it just changes the message format).

If a task matches no skill and no rule, proceed using the general principles in `00-overview.md`
(explicit over clever, no premature abstraction) and flag the gap instead of silently improvising a new convention.

---

## 4. Standard Task Workflow

For any new feature, follow the build order from `00-overview.md` unless the task is a small fix:

1. `types/<name>.type.ts` — entity type, input/output types, prop types, type guard
2. `validators/<name>.validator.ts` — Zod schema with `.pipe()`
3. `db/schema/<name>.ts` — Drizzle table schema
4. `services/<name>.service.ts` — pure DB ops
5. `actions/<name>.action.ts` — auth + validation + service call
6. `app/api/<name>/route.ts` — GET only, calls the action
7. `components/` — UI last, `"use client"` on the deepest leaf only

For bug fixes or small edits: locate the correct layer first (service/action/component) using
`01-folder-structure.md`, fix at that layer only, don't restructure unrelated files.

---

## 5. Non-Negotiables (quick checklist before finishing any task)

- [ ] `function` keyword used, no arrow function declarations
- [ ] `type` used unless the shape is genuinely extended
- [ ] All types/props live in `/src/types`, none inline in components
- [ ] `@/` alias used, no relative `../../` imports
- [ ] Named exports, except Next.js special files
- [ ] Zod validation present on every action input, using `.pipe()`, short error messages
- [ ] No DB calls outside `services/`, no auth/validation outside `actions/`
- [ ] GET → route + action (same function); POST/PUT/DELETE → action only
- [ ] `"use client"` only on the leaf component that needs it
- [ ] Relevant skill(s) from the routing table checked before finalizing
