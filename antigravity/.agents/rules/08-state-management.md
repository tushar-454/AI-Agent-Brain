# 08 — State Management (Zustand)

- `src/stores/<name>.store.ts` — client state only, one store per domain.

  ```ts
  // src/stores/cart.store.ts
  import { create } from "zustand";
  import type { CartState } from "@/types/cart.type";

  export const useCartStore = create<CartState>(function (set) {
    return {
      items: [],
      addItem: function (item) {
        set(function (state) {
          return { items: [...state.items, item] };
        });
      },
    };
  });
  ```

- Store state/action types also live in `types/cart.type.ts`, not inline.
- Don't mirror server state in Zustand unless it needs to persist across navigation — prefer fetching fresh via actions/server components.
