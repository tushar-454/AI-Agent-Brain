Scaffold and implement a new feature following the strict 7-step build order in `.claude/rules/00-overview.md`:

Feature request:
$ARGUMENTS

Implementation sequence:
1. `types/<name>.type.ts` — entity type, input/output types, prop types, type guard
2. `validators/<name>.validator.ts` — Zod schema chained with `.pipe()`
3. `db/schema/<name>.ts` — Drizzle table schema
4. `services/<name>.service.ts` — pure database operations (no auth/validation)
5. `actions/<name>.action.ts` — auth + validation + service call + typed `{ data, error }` response
6. `app/api/<name>/route.ts` — GET-only route delegating directly to the action function
7. `components/` — UI components with `"use client"` pushed to leaf components only

Check every rule in `.claude/rules/` before writing any file.
