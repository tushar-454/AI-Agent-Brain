# AGENTS.md — Cursor Agent Orchestration

You are the coding agent for the **PAI** project. This file tells you **how to work**;
the enforceable standards live in `.cursor/rules/*.mdc`.

## Load order

1. Read this file.
2. Project rules in `.cursor/rules/` are always active — follow them.
3. Skills in `.cursor/skills/` are loaded on demand — Cursor auto-selects by task; use the routing table below when unsure.
4. MCP tools from `.cursor/mcp.json` when the task needs external tooling.

**If a rule and a skill conflict, rules win.**

---

## Identity

- Production TypeScript: Next.js App Router, Drizzle, Postgres, Zustand, Tailwind/shadcn.
- Architecture: Service (DB) → Action (auth + validation) → Route/Component.
- Comments explain _why_, not _what_.
- If ambiguous and no rule/skill covers it, ask one short question — don't guess.

---

## Project rules (`.cursor/rules/`)

All 12 `.mdc` files have `alwaysApply: true`. Start with `00-overview.mdc` for stack, build order, and architecture. Do not duplicate those rules here.

---

## Skills (`.cursor/skills/`)

Cursor discovers skills automatically. Load `SKILL.md` when the task matches:

| Task domain | Skill |
|---|---|
| Better Auth setup, OAuth, sessions, plugins | `better-auth-best-practices` |
| Module boundaries, deep interfaces | `codebase-design` |
| New UI / visual direction | `frontend-design` |
| Premium / agency-level polish | `high-end-visual-design` |
| Postgres SQL, schema, RLS | `postgresql-code-review` |
| shadcn/ui components | `shadcn` |
| React/Next.js performance | `vercel-react-best-practices` |
| Core Web Vitals, page speed | `web-perf` |
| Cloudflare Workers code | `workers-best-practices` |
| Wrangler CLI, deploy Workers/KV/R2/D1 | `wrangler` |

**Communication / workflow** (only when user asks or mode is explicit):

| Trigger | Skill |
|---|---|
| Ultra-terse output | `caveman` |
| Commit message | `caveman-commit` |
| PR/diff review | `caveman-review` |
| Delegate to subagents | `cavecrew` |

---

## MCP (`.cursor/mcp.json`)

| Server | Use when |
|---|---|
| `chrome-devtools` | Web perf audits, Core Web Vitals |
| `shadcn` | Add/search/fix shadcn components |
| `better-auth` | Better Auth config and plugins |

---

## Feature workflow

New features: follow build order in `00-overview.mdc` (types → validators → schema → service → action → GET route → components).

Bug fixes: find the correct layer via `01-folder-structure.mdc`, fix there only.

---

## Pre-flight checklist

Before finishing: rules followed, `@/` imports, named exports, Zod on actions, no DB outside services, `"use client"` on leaf only.
