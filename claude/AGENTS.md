# AGENTS.md — Claude Agent Orchestration

You are the coding agent for the **PAI** project. This file coordinates agent behavior;
the enforceable standards live in `CLAUDE.md` and `.claude/rules/*.md`.

## Load order

1. Read `CLAUDE.md` and this file.
2. Project rules in `.claude/rules/` are always active — follow them.
3. Skills in `.claude/skills/` are loaded on demand — check the routing table in `CLAUDE.md`.
4. MCP tools from `.mcp.json` when the task needs external tooling.

**If a rule and a skill conflict, rules win.**

---

## Identity & Stack

- **Stack:** Next.js App Router, TypeScript, Tailwind CSS, shadcn/ui, Zustand, Drizzle ORM, Postgres, Zod.
- **Architecture:** Service (DB) → Action (auth + validation) → Route/Component.
- **Style:** `function` keyword everywhere, `type` over `interface`, named exports, `@/` path alias.
- **Guidance:** Comments explain _why_, not _what_. If ambiguous and no rule/skill covers it, ask one short question — don't guess.

---

## Project Rules (`.claude/rules/`)

All 12 `.md` files in `.claude/rules/` are always active. Start with `00-overview.md` for stack, build order, and architecture.

---

## Skills (`.claude/skills/`)

Skills follow the Agent Skills standard and are loaded when the task domain matches:
- Auth: `better-auth-best-practices`, `production-auth-architecture`
- Billing: `production-stripe-billing`
- Architecture: `codebase-design`
- UI/Design: `frontend-design`, `high-end-visual-design`, `shadcn`
- Performance: `vercel-react-best-practices`, `web-perf`
- Database: `postgresql-code-review`
- Workers/Edge: `workers-best-practices`, `wrangler`
- Workflow/Communication: `caveman`, `caveman-commit`, `caveman-review`, `cavecrew`, `git-commit`

---

## Pre-flight Checklist

Before finishing: rules followed, `@/` imports, named exports, Zod `.pipe()` on actions, no DB outside services, `"use client"` on leaf only.
