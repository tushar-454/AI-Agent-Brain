# 05 — Service Layer (Pure DB Operations)

- `src/services/<name>.service.ts` — **only** raw database operations via Drizzle. No auth checks, no Zod validation, no business rules here.
- Services are the only files allowed to import the Drizzle client directly.

  ```ts
  // src/services/user.service.ts
  import { db } from "@/db";
  import { users } from "@/db/schema/user";
  import { eq } from "drizzle-orm";
  import type { User, CreateUserInput } from "@/types/user.type";

  export async function findUserById(id: string): Promise<User | undefined> {
    return db.query.users.findFirst({ where: eq(users.id, id) });
  }

  export async function createUser(input: CreateUserInput): Promise<User> {
    const [user] = await db.insert(users).values(input).returning();
    return user;
  }
  ```
