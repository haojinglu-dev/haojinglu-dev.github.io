---
title: "When Service Fabric Transactions Start Fighting"
description: "How two batch workflows created a circular lock wait—and why deterministic ordering fixed it."
pubDate: 2026-08-18
tags:
  - service-fabric
  - distributed-systems
  - reliability
---

Reliable Collections make state management in Azure Service Fabric feel
straightforward: put reads and writes inside a transaction, then let Service
Fabric handle replication and consistency.

Our problems started when two batch workflows began updating the same items at
the same time.

## The conflicting batches

One workflow was a batch API. Another was an internal worker. Both accepted a
list of items and updated each item inside a transaction.

Imagine the API receives this batch:

```text
[A, B]
```

At the same time, the worker receives:

```text
[B, A]
```

The API locks A, then tries to lock B. The worker locks B, then tries to lock A.

```text
Batch API:       lock A -> wait for B
Internal worker: lock B -> wait for A
```

Each transaction holds the lock the other transaction needs. Neither can move
forward.

## How the failure appeared

Reliable Collection transactions use two-phase locking. Once a transaction
acquires a lock, it generally holds that lock until commit or abort.

By default, a Reliable Collection operation waits up to four seconds to acquire
a lock. If it cannot, Service Fabric throws a `TimeoutException`. This is a
lock-acquisition timeout, not necessarily a limit on the entire transaction
([Microsoft: transactions and lock modes in Reliable Collections](https://learn.microsoft.com/en-us/azure/service-fabric/service-fabric-reliable-services-reliable-collections-transactions-locks#locks)).

In our case, one transaction eventually timed out and aborted. The other could
then continue.

That made the problem look like an ordinary transient failure.

### Why Service Fabric uses a timeout

Reliable Collections use rigorous two-phase locking: a transaction keeps its
locks until it commits or aborts. Holding locks to the end preserves the
transaction's isolation and atomicity, but it also means circular waits are
possible when transactions acquire the same resources in different orders.

Service Fabric cannot know whether a transaction is waiting briefly or is part
of a cycle that will never resolve. The timeout provides an escape. Microsoft
describes the timeout argument as being used for
[deadlock detection](https://learn.microsoft.com/en-us/azure/service-fabric/service-fabric-reliable-services-reliable-collections-transactions-locks#locks):
one or both conflicting operations time out, allowing a transaction to abort
and release its locks instead of waiting forever.

### Does Service Fabric retry the transaction?

No. Reliable Collections surface the `TimeoutException`; the application
decides whether the operation is safe to retry. A retry must execute the whole
unit of work again in a **new transaction**, not continue from the transaction
that timed out
([Microsoft: working with Reliable Collections](https://learn.microsoft.com/en-us/azure/service-fabric/service-fabric-work-with-reliable-collections#common-pitfalls-and-how-to-avoid-them)).

That behavior is deliberate. Service Fabric cannot assume that every operation
is safe to replay, choose an appropriate delay, or know whether another attempt
would amplify contention. Those decisions belong to the application.

## Why retries were misleading

We caught the timeout and ran the operation again in a new transaction. Most
retries succeeded because the timing had changed: one workflow had already
released its locks.

But retrying did not remove the circular-wait condition. The same two batches
could collide again on the next request.

Immediate retries could also make contention worse by sending failed work
straight back to the same hot keys. Backoff and jitter reduced pressure, but
they still treated the symptom rather than the cause.

In production, we still wrap the **entire transaction operation** in a bounded
retry policy. Each attempt creates a new transaction, retries only transient
failures, uses exponential backoff with jitter, respects cancellation, and
stops after a small retry budget. We also record retry counts and exhausted
retries so persistent contention cannot remain invisible.

The distinction matters: the retry policy makes occasional conflicts
recoverable; deterministic lock ordering removes this repeatable circular
wait. We need both, but they solve different problems.

The fact that a retry succeeds does not prove that a failure was random.

## Why debugging changed the result

The issue was timing-dependent. Attaching a debugger slowed one workflow enough
to change the lock-acquisition order, so the conflict often disappeared during
investigation.

Useful telemetry made the pattern visible. For each transaction, we recorded:

- the workflow and transaction name
- the keys accessed, in order
- transaction start time and duration
- timeout and retry count
- a correlation or trace ID

The key detail was not simply that both workflows touched A and B. It was that
they touched them in opposite orders.

## The fix: deterministic ordering

Before opening the transaction, both workflows now sort their item keys using
the same stable rule.

```text
Input batch:  [B, A]
Sorted batch: [A, B]
```

Both transactions acquire locks in the same order:

```text
Batch API:       lock A -> lock B
Internal worker: wait for A -> lock A -> lock B
```

One transaction may still wait, but the circular dependency is gone. A
transaction holding A will never wait for another transaction that holds B
while waiting for A.

The particular ordering does not matter. It can be alphabetical, numeric, or
based on another stable key. What matters is that every code path uses the same
rule.

In our API, a batch was limited to roughly 100 items. Sorting a collection that
small is negligible compared with the transaction and replication work.

This approach is not an argument for putting a large dataset in one
transaction. A large transaction holds more locks for longer, increases
contention, and makes timeout recovery more expensive. Large workloads are
usually divided into bounded chunks, with each chunk processed in its own
short, idempotent transaction and progress checkpointed between chunks. That
changes the atomicity boundary, so the surrounding job must define how partial
completion and retries are handled.

### Enforcing one order everywhere

Relying on each caller to remember the rule is fragile. We defined one
application-wide comparer and routed batch updates through a shared transaction
helper.

Each workflow must know its complete key set before starting the transaction,
remove duplicates, and sort using that comparer:

```csharp
var orderedIds = itemIds
    .Distinct()
    .OrderBy(id => id, StringComparer.Ordinal)
    .ToList();

using var tx = stateManager.CreateTransaction();

foreach (var id in orderedIds)
{
    var current = await dictionary.TryGetValueAsync(
        tx,
        id,
        LockMode.Update);

    // Apply the update for this item.
}

await tx.CommitAsync();
```

When a transaction touches several collections, the ordering key can be a
tuple such as `(collectionName, itemKey)`. Every workflow sorts by collection
first and item key second.

The important enforcement boundary is the API: callers use the ordered batch
helper instead of accessing the underlying Reliable Dictionary directly. If a
workflow discovers another required key after locking begins, it should abort,
rebuild the complete ordered key set, and start a new transaction.

We also kept the transactions short and moved non-transactional work outside
their boundaries. The less time a transaction holds a lock, the smaller the
window for contention.

## The lesson

The timeout was not the root problem. The retry policy was not the root problem
either. Two batch workflows acquired the same locks in inconsistent order.

For multi-item transactions:

1. Determine every key the transaction needs.
2. Sort those keys deterministically.
3. Acquire or update them in that order.
4. Keep the transaction as short as possible.
5. Log access order so future contention can be diagnosed.

Retries remain useful for genuinely transient conflicts, but they should not be
used to hide a repeatable circular wait.

## References

- [Transactions and lock modes in Reliable Collections](https://learn.microsoft.com/en-us/azure/service-fabric/service-fabric-reliable-services-reliable-collections-transactions-locks)
- [Guidelines for Reliable Collections](https://learn.microsoft.com/en-us/azure/service-fabric/service-fabric-reliable-services-reliable-collections-guidelines)
