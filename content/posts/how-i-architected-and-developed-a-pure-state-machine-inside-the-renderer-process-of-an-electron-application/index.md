---
title: "How I Architected and Developed a Pure, Deterministic State Machine inside the Renderer Process of an Electron Application"
date: 2026-04-05T15:06:43+05:30
draft: false
showToc: true
---

### TLDR;

> This is it, if you understand this, you don't need to read furthermore. If you want to understand the details, read on.

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

Building [**Printit**]({{< relref "projects/printit" >}}) is not easy, to deliver a good user experience, the following challenges had to be solved:

- **Managing state of printers, queues, orders and jobs** that are highly coupled with environments like printers and network requests which can behave highly non-deterministically.
- One of the challenge was to **reliably determine if a piece of paper came out of the printer or not**, only then we can compute the state of an order, job or queue. Printers behavior vary widely and are notoriously difficult to reliably receive updates from, this is not a small problem to solve (although, I did it eventually using some heuristics).
- As we saw, printers can behave differently and can **showcase endlessly unique state transitions**. Managing these states and turning these transitions into events (complete observability) is another challenge.
- Every change the order goes through during its life cycle (from user's device to the printer and back to user) needs to be showcased to the user in **real-time with eventual consistency**.

## Requirements

Here are some of the requirements that I gathered before starting the development:

### Functional Requirements

- Users should be able to add and manage print queues (printers).
- Users should be able to receive and manage print orders.
- Users should be able to start, pause, resume and reject print orders and jobs.
- Users should be able to see analytics and reports about their print queues and orders.
- The system should propagate key milestone events in the life cycle of a print order and job to the user (like when the printer is out of paper, or when a print job is completed).

### Non-Functional Requirements

- Life cycle of a print order and job should be observable to the user in real-time with eventual consistency.
- Every step in the life cycle of a print order and job should be deterministic and replay-able (for debugging and testing).
- The system should be resilient and cleanly propagate errors and exceptions to the user without crashing or freezing the application.

### Constraints

Windows, the operating system that Printit is targeting has a terrible printing API in terms of real-time feedback. On top of that, Electron has its own set of constraints and limitations. Some of the major ones are:

#### System.Printing

You either need to interact with low level _System.Drawing_ API and do all the work yourself or use fairly high-level _System.Printing_ API which is limiting and has little to no support for events and feedback. This means once you submit a job to the spooler, windows does not bother to tell you about it's status unless you explicitly ask for it.

#### The Spooler Cache

Windows likes to keep the status of it's print queues in a cache and it only updates it when you actually try to print a document. This is a huge limitation if you want to show a printer's connection state (you can't, at least not easily). Our architectural design had to take this into account and we had to come up with some heuristics to determine the connection state of a printer (Short Polling).

#### Electron IPC

At a high level, Electron does not allow direct access to Node.js APIs from the renderer process due to various security risks. Instead, it provides **Inter-Process Communication (IPC)**, that allows the renderer processes to request operations from the main process, then main process executes these operations and returns results to the renderer processes. Every message payload that goes through IPC is **Structured Cloned** and stripped of all functions and prototypes (structured cloning requires this), this means you can't send complex objects with methods through IPC, you need to serialize them into plain objects and then deserialize them on the other side. Not a huge challenge but it adds significant complexity to the architecture and design of the application.

## Design

{{< inline-svg src="electron-architecture.svg" >}}
