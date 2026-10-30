Perform a concise code and diff review on the specified changes or unstaged diffs:

1. Follow the one-line-per-finding format:
   - `<filepath>:<line> — <problem> — <fix>`
2. Check strictly against the project rules in `.claude/rules/`:
   - `function` keyword used everywhere (no arrow functions)
   - `type` used over `interface`
   - Types live in `/types`, never inline in components
   - `@/` path alias used (no relative `../../`)
   - Named exports everywhere (except Next.js special files)
   - Zod validation with `.pipe()`
   - No DB calls outside `services/`, no auth/validation outside `actions/`
   - `"use client"` pushed to deepest leaf components only
3. Prioritize bugs, architecture breaches, security concerns, and performance anti-patterns.
4. If `$ARGUMENTS` are provided, focus review on that scope.

Scope:
$ARGUMENTS
