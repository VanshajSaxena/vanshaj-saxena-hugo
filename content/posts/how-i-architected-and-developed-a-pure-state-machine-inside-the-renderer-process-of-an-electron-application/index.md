---
title: "How I Architected and Developed a Pure, Deterministic State Machine inside the Renderer Process of an Electron Application"
date: 2026-04-05T15:06:43+05:30
draft: false
showToc: true
---

### tldr;

> It's this, if you understand this, you don't need to read the rest of the article. But if you want to understand this, read on.

```ts
import { fetchClient } from "@api/client";
import { create } from "zustand";
import { persist } from "zustand/middleware";
import { ObservableExecutor } from "./executor";
import { reducer } from "./reducer";

export const useTransactionStore = create<TransactionStore>()(
  persist(
    (set, get) => {
      const executor = new ObservableExecutor((event) => {
        const [newState, operations] = reducer(get().state, event);
        set({ state: newState });
        operations.forEach((operation) => void executor.execute(operation));
      }, fetchClient);
      return {
        state: { transactions: {}, printQueue: { items: [], status: "idle" } },
        dispatch: (event) => {
          const [newState, operations] = reducer(get().state, event);
          set({ state: newState });
          operations.forEach((operation) => void executor.execute(operation));
        },
      };
    },
    {
      name: "transaction-storage",
      storage: createIdbStorage<Pick<TransactionStore, "state">>(),
      partialize: (store) => ({ state: store.state }),
      onRehydrateStorage(store) {
        return () => {
          store.state.printQueue.status = "idle";
        };
      },
      version: 1,
    },
  ),
);
```

## Why

## Constraints

## Architecture and Design
