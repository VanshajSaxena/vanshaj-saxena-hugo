---
title: "How I Architected and Developed an Electron Application capable of Managing Complex State and Side Effects using a Pure State Machine inside the Renderer Process"
date: 2026-04-05T15:06:43+05:30
draft: false
showToc: true
---

> _This is it, if you understand this, you don't need to read furthermore. If you want to understand the details, read on._

{{< inline-svg src="electron-architecture.svg" >}}

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
- Every step in the life cycle of a print order and job should be deterministic and replay-able (very helpful for debugging and testing such an unpredictable system).
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

Given the requirements and constraints, I came up with a layered architecture where **state**, **events**, **commands**, **side effects**, and **views** are all decoupled, forming a unidirectional event loop where state transitions remain pure and side effects are executed asynchronously.

Events originating from user input or external sources are dispatched to a pure reducer, which produces both the next state and a set of declarative commands. These commands are handled by an executor responsible for performing side effects (such as API calls or interacting with Native APIs) and emitting new events back into the system, closing the loop.

This architecture is heavily inspired by **Command Query Responsibility Segregation (CQRS)** and event-driven patterns, adapted to fit the needs of a client-side application built with Electron.

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

Notice we are using the **persist middleware** to store the application state in IndexedDB, allowing it to survive reloads and unexpected crashes. Combined with rehydration safeguards, this ensures the system can recover into a consistent state and safely resume the event loop without carrying over stale or in-flight side effects.

### Components

The system, as you saw has several components, each with a specific responsibility:

#### Reducer

The reducer is designed as a pure function that takes the current state and an incoming event, and returns the next state alongside a list of declarative operations. State transitions are handled immutably, while any side effects are described but not executed at this stage.

```ts
export const reducer = (
  state: TransactionState,
  event: ReducerEvent,
): [TransactionState, TransactionOperation[]] => {
  let nextState = state;
  const operations: TransactionOperation[] = [];
  switch (event.type) {
    case "user:print:transaction": {
      nextState = produce(state, (draft) => {
        const transaction = draft.transactions[event.transactionId];
        const queueItems: PrintQueueItem[] = transaction.jobs
          .filter((job) => job.status === "PENDING")
          .map((job) => ({
            id: generateQueueItemId(),
            transactionId: event.transactionId,
            jobId: job.id,
            queuedAt: new Date().toISOString(),
          }));
        draft.printQueue.items.push(...queueItems);
        if (draft.printQueue.status === "idle") {
          draft.printQueue.status = "working";
        }
        draft.transactions[event.transactionId].status = "QUEUED";
        draft.transactions[event.transactionId].jobs
          .filter((job) => job.status === "PENDING")
          .forEach((job) => {
            job.status = "QUEUED";
          });
      });
      operations.push({
        operationId: generateOperationId(),
        type: "download-files-to-file-system",
        files: nextState.transactions[event.transactionId].jobs
          .filter((job) => job.status === "QUEUED")
          .map((job) => {
            return { id: job.document.id };
          }),
        transactionId: event.transactionId,
      });
      break;
    }

    // more cases...

    case "job:report:success": {
      nextState = produce(state, (draft) => {
        draft.printQueue.items = draft.printQueue.items.filter(
          (item) =>
            !(
              item.jobId === event.jobId &&
              item.transactionId === event.transactionId
            ),
        );
      });
      break;
    }

    default: {
      break;
    }
  }

  return [nextState, operations];
};
```

This approach keeps all decision-making centralized and predictable. Each event fully describes a state transition, making the system easier to reason about, test, and extend. By returning operations instead of executing them directly, the reducer remains side-effect free, allowing the execution layer to handle asynchronous work independently.

As a result, the system benefits from a clear separation between state transitions and side effects, improved testability of business logic, and a consistent event-driven flow where all changes originate from explicit events and propagate through the same pipeline.

#### Executor

The executor is responsible for handling all side effects produced by the reducer, acting as the boundary between pure state transitions and the outside world. It receives declarative operations and executes them asynchronously, interacting with external systems such as APIs, WebSocket events, and system-level processes.

```ts
export class ObservableExecutor {
  private dispatch: (event: ReducerEvent) => void;
  private fetchClient: FetchClientType;
  private monitoringCancellation = new Map<string, Subject<void>>();
  private reportRetryQueue: {
    operation: ReportJobStatusOperation | ReportTransactionStatusOperation;
    attempts: number;
    lastAttemptAt: number;
  }[] = [];
  private maxRetryAttempts = 5;
  private retryIntervalMs = 5000;
  constructor(
    dispatch: (event: ReducerEvent) => void,
    fetchClient: FetchClientType,
  ) {
    this.dispatch = dispatch;
    this.fetchClient = fetchClient;

    this.startRetryProcessor();
  }

  async execute(operation: TransactionOperation): Promise<void> {
    switch (operation.type) {
      case "emit-user-print-intent": {
        emitWebSocketEvent("milestone:transaction:queued", {
          transactionId: operation.transactionId,
          vendorId: operation.vendorId,
          timestamp: new Date().toISOString(),
        });
        break;
      }
      case "emit-transaction-printing": {
        emitWebSocketEvent("milestone:transaction:printing", {
          transactionId: operation.transactionId,
          vendorId: operation.vendorId,
          timestamp: new Date().toISOString(),
        });
        break;
      }

      // more cases...

      case "report-transaction-and-jobs-status": {
        await this.reportTransactionStatus(operation);
        break;
      }

      default:
        break;
    }
  }
```

Its design centralizes all effectful logic in a single place, allowing the rest of the system to remain deterministic and side-effect free. By mapping operations to concrete implementations, it ensures that external interactions are consistently handled and that their outcomes are fed back into the system as events, maintaining the integrity of the event loop.

Additionally, the executor manages concerns such as retries, long-running processes, and cancellation, which are kept out of the reducer to avoid introducing complexity into state transitions. This separation improves reliability and makes it easier to evolve side-effect handling independently from business logic.

This gives us clear and maintainable execution boundary, better control over asynchronous workflows, and a consistent mechanism for integrating external effects back into the application through events.

#### UI Components

The interaction between components and the store follows a unidirectional, event-driven flow where views remain fully decoupled from business logic. Components subscribe to state changes and derive the data they need for rendering, while all mutations are performed by dispatching events into the system.

```ts
export function TransactionListContainer({
  onTransactionClick,
}: {
  onTransactionClick: (transactionId: number) => void;
}) {
  const dispatch = useTransactionStore((store) => store.dispatch);
  const transactionsMap = useTransactionStore(
    (store) => store.state.transactions,
  );
  const transactionList = useMemo(() => {
    return Object.values(transactionsMap).sort(
      (a, b) =>
        new Date(b.createdAt).getTime() - new Date(a.createdAt).getTime(),
    );
  }, [transactionsMap]);
  useEffect(() => {
    dispatch({ type: "transaction:request" });
  }, [dispatch]);
  useWsEvent("transaction:new", () => {
    dispatch({ type: "transaction:request" });
  });
  return (
    <div className="h-full min-h-fit max-w-full min-w-0 overflow-y-auto rounded-3xl border border-border-default mask-[linear-gradient(black,black)]">
      {transactionList.length === 0 && (
        <div className="flex size-full flex-col items-center justify-center gap-y-6 px-4 py-8">
          <EmptyTrayIcon />
          <p className="text-center text-sm font-normal text-secondary-content">
            This is empty, share the QR Code to start receiving orders.
          </p>
        </div>
      )}
      {transactionList.map((transaction) => {
        return (
          <TransactionItem
            key={transaction.id}
            {...transaction}
            onClick={() => {
              onTransactionClick(transaction.id);
            }}
          />
        );
      })}
    </div>
  );
}
```

User interactions and external signals (such as WebSocket events) are translated into events and dispatched to the store, feeding into the same event loop that drives the entire application. Components do not perform any direct state mutations or side effects; instead, they act as entry points into the system and observers of its outcomes.

This design keeps components simple and focused on presentation, while ensuring that all state changes go through a single, predictable pipeline. It also guarantees consistency, as both user-driven and external updates are handled uniformly through events.

The benefits of this approach are, improved separation of concerns, easier reasoning about data flow, and a more maintainable UI layer where behavior is driven entirely by state and events rather than imperative logic.
