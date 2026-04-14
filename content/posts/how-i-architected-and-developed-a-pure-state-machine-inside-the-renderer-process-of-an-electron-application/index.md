---
title: "How I Architected and Developed a Pure, Deterministic State Machine inside the Renderer Process of an Electron Application"
date: 2026-04-05T15:06:43+05:30
draft: false
showToc: true
---

### TLDR;

> This is it, if you know what this is, you don't need to read furthermore. Read if you wish to get insight.

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

While building [**Printit,**]({{< relref "projects/printit" >}}) I was faced with some challenges that I never faced before:

- **Managing state of printers, queues, orders and jobs** that are highly coupled with environments like printers and network requests which are external to my system.
- **Reliably knowing if a piece of paper came out of the printer or not**, only then I can tell if an order is complete, printers are notoriously difficult to get updates from, this is not a small problem to solve (although, I did it eventually).
- Every printer can behave differently and can **showcase endlessly unique state transitions**. Managing these states and turning these transitions into system events.
- All this needs to be showcased to the user in real time with **high accuracy and reliability**.

## Constraints

Windows operating system /

## Architecture and Design

{{< inline-svg src="electron-architecture.svg" >}}
