# 06 — Action Layer (Auth + Validation + Orchestration)

- `src/actions/<name>.action.ts` — this is the ONLY layer allowed to:
  1. Check authentication/authorization
  2. Validate input via the matching Zod validator
  3. Call the service function
  4. Return a typed, guarded response

- Every action is a Next.js Server Action (`"use server"` at top of file).

  ```ts
  // src/actions/user.action.ts
  "use server";

  import { auth } from "@/lib/auth";
  import { createUserSchema } from "@/validators/user.validator";
  import { createUser, findUserById } from "@/services/user.service";
  import { isUser } from "@/types/user.type";
  import type { ApiResponse } from "@/types/api.type";
  import type { User, CreateUserInput } from "@/types/user.type";

  export async function createUserAction(
    input: CreateUserInput
  ): Promise<ApiResponse<User>> {
    const session = await auth();
    if (!session) {
      return { data: null as never, error: "Unauthorized" };
    }

    const parsed = createUserSchema.safeParse(input);
    if (!parsed.success) {
      return { data: null as never, error: parsed.error.errors[0]?.message ?? "Invalid input" };
    }

    const user = await createUser(parsed.data);
    if (!isUser(user)) {
      return { data: null as never, error: "Unexpected response shape" };
    }

    return { data: user };
  }

  export async function getUserAction(id: string): Promise<ApiResponse<User>> {
    const user = await findUserById(id);
    if (!user || !isUser(user)) {
      return { data: null as never, error: "User not found" };
    }
    return { data: user };
  }
  ```
