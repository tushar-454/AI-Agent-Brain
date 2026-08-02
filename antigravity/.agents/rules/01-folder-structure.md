# 01 — Folder Structure

Group files by **type**. Each type-folder uses the naming pattern `<filename>.<folderName>.ts` for files inside it.

```
src/
├── app/                        # Next.js App Router (pages, layouts, GET route handlers only)
│   ├── (routes)/
│   │   └── dashboard/
│   │       └── page.tsx        # default export required (Next.js convention)
│   └── api/
│       └── user/
│           └── route.ts        # GET only — imports & calls the function from actions/user.action.ts
│
├── components/
│   ├── ui/                     # shadcn/ui primitives — multiple exports per file OK (card.tsx, table.tsx)
│   └── shared/                 # Feature/section components — one component per file, colocated pieces
│
├── hooks/                      # hooks/use-xxx.ts
│
├── stores/                     # Zustand — stores/user.store.ts
│
├── actions/                    # Server Actions — actions/user.action.ts
│                                # auth check → input validation (Zod) → calls service → typed response
│
├── services/                   # Pure DB operations only — services/user.service.ts
│                                # no auth, no validation, no business rules — just DB reads/writes
│
├── validators/                 # Zod schemas — validators/user.validator.ts
│
├── types/                      # ALL types: entities, API shapes, component props, type guards
│   ├── user.type.ts
│   ├── product.type.ts
│   └── api.type.ts
│
├── db/                         # Database layer (NOT under lib/)
│   ├── index.ts                # Drizzle client instance
│   └── schema/
│       └── user.ts
│
├── lib/                        # Generic helpers, config, env parsing — nothing DB-related
│   ├── env.ts
│   └── utils.ts
│
└── constants/
    └── index.ts
```

**Rule:** Never create a `/features` folder. Everything is grouped by type, and every type-folder's files follow `<name>.<folderName>.ts`.
