---
trigger: always_on
---

# 00 - Overview

Stack: **Next.js (App Router) + TypeScript + Tailwind + shadcn/ui + Zustand + Drizzle ORM + Postgres + Zod**

Architecture: **Service-based** — every domain has a Service (DB) → Action (auth + validation + orchestration) → Route/Component (consumption) chain.

Other rule files in this folder:

- `01-folder-structure.md`
- `02-naming-conventions.md`
- `03-typescript-types.md`
- `04-validation-zod.md`
- `05-service-layer.md`
- `06-action-layer.md`
- `07-api-routes.md`
- `08-state-management.md`
- `09-components.md`
- `10-imports-exports.md`
- `11-error-formatting-env.md`

---

## Feature Build Order

For every new feature, build in this exact order:

1. `types/<name>.type.ts` — entity type + input/output types + prop types + type guard
2. `validators/<name>.validator.ts` — Zod schema using `.pipe()`
3. `db/schema/<name>.ts` — Drizzle table schema
4. `services/<name>.service.ts` — pure DB operations
5. `actions/<name>.action.ts` — auth + validation + service call + typed response
6. `app/api/<name>/route.ts` — **GET only**, calls the action function directly
7. `components/` — UI, with `"use client"` pushed to the deepest leaf only

## General Principles

- Prefer explicit over clever. Optimize for the next reader (human or AI).
- No premature abstraction — don't build a generic system for one use case.
- When in doubt about where a file goes, check `01-folder-structure.md` before creating a new folder pattern.
