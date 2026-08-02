# 04 — Validation (Zod)

- Every action's input and every GET route's query params are validated with Zod.
- Schemas live in `/src/validators/<name>.validator.ts`.
- Use `.pipe()` to chain transformation + refinement steps instead of bolting everything into one `.refine()`.
- Keep error messages **short and precise** — no generic "Invalid input."

  ```ts
  // src/validators/user.validator.ts
  import { z } from "zod";

  export const createUserSchema = z.object({
    email: z
      .string({ required_error: "Email is required" })
      .pipe(z.string().email("Invalid email format")),
    name: z
      .string({ required_error: "Name is required" })
      .pipe(z.string().min(2, "Name too short")),
  });

  export type CreateUserInput = z.infer<typeof createUserSchema>;
  ```

- On failure, return the first/flattened error message directly — don't dump the whole Zod error tree to the client.

  ```ts
  const parsed = createUserSchema.safeParse(input);
  if (!parsed.success) {
    return { error: parsed.error.errors[0]?.message ?? "Invalid input" };
  }
  ```
