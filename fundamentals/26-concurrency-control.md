---
title: "Concurrency control"
concepts:
  - critical-section
  - mutual-exclusion
  - mutex
  - semaphore
  - atomic-operations
  - race-condition
  - deadlock
  - priority-inversion
related:
  - fundamentals/09-reliability.md
  - fundamentals/14-resilience.md
  - fundamentals/15-observability.md
  - fundamentals/25-database-concurrency-control.md
---

# Concurrency control

Concurrency control coordinates multiple threads or processes so that shared state stays correct when they access it at the same time.

This document covers concurrency **inside a single process**: synchronization, ordering, and safe access to shared memory and shared resources. The transaction-level equivalent, where the database enforces the same guarantees across sessions through isolation levels and locking, is covered in [Database concurrency control](./25-database-concurrency-control.md). The problems rhyme (lost updates, deadlocks, contention hot spots); the tools differ.

## Why this belongs with reliability topics

Concurrency is where a large share of reliability failures originate, and it is easy to miss because nothing looks broken: the host is healthy, the service is available, and the dashboard is green while the data quietly goes wrong.

- The failure modes in [Reliability](./09-reliability.md) (race conditions, lost updates, duplicate side effects) are usually concurrency bugs, not infrastructure failures
- Several patterns in [Resilience](./14-resilience.md) are concurrency primitives underneath: a bulkhead is a bounded thread pool, a concurrency limit on a dependency is a semaphore, and backpressure is what a bounded queue does when it is full

So this is the building-block layer. The system-level patterns in those documents assume the primitives here are used correctly.

## Core concepts

- **Critical section**: Code that accesses shared mutable state
- **Mutual exclusion**: Only one execution context enters a critical section at a time
- **Progress/liveness**: The system keeps making forward progress
- **Fairness**: No actor is starved indefinitely

## Synchronization primitives

### Mutex / lock

- Simplest mutual exclusion primitive
- Good for protecting short critical sections

Example:

- Protect a shared in-memory balance map during update (`lock -> read/modify/write -> unlock`)
- If two threads update the same account, the lock ensures one completes before the other starts

### Read-write lock

- Many concurrent readers, exclusive writer
- Useful for read-heavy shared structures

Example:

- In-memory product catalog with frequent reads and occasional updates
- Multiple readers can query simultaneously, but a refresh job takes the write lock to replace the snapshot safely

Be aware that a continuous stream of readers can starve the writer, so most implementations offer a writer-preferring mode.

### Semaphore

- Controls access to a limited pool (for example, at most 20 workers)
- Can be binary (equivalent to a mutex) or counting

Example:

- Limit outbound calls to a third-party API to 50 concurrent requests
- Each worker acquires one permit before the call and releases it after the response
- Prevents connection and resource exhaustion during traffic spikes

This is exactly the bulkhead pattern from [Resilience](./14-resilience.md), implemented in-process.

### Condition variable

- Allows threads to sleep until a condition becomes true
- Typically used with a mutex for producer-consumer patterns

Example:

- Consumer waits while the queue is empty
- Producer pushes an item and signals the condition
- Consumer wakes up, **re-checks** the condition, and processes the item

The re-check matters: threads can wake spuriously, or another consumer may have taken the item first, so always wait in a loop rather than assuming the condition holds on wake-up.

### Atomic operations and compare-and-swap

- Hardware-level operations that complete indivisibly, with no lock to acquire or release
- **Compare-and-swap (CAS)**: Update a value only if it still holds the expected value, otherwise retry
- Cheaper than a mutex under low contention, and they cannot deadlock

Example:

- An atomic increment on a metrics counter removes the read-modify-write race entirely
- A CAS loop on a status field applies a state transition only if no one else changed the state first

Lock-free structures built from CAS are fast but hard to get right, and they can still livelock under heavy contention. Reach for a plain lock first, and for an atomic only when the state is a single word or contention is measured to be a problem.

## Concurrency problems

### Race condition

The program's result depends on thread or process timing, not just on logic.

Example:

- Shared counter starts at `0`
- Thread A reads `0`, thread B reads `0`
- Thread A writes back `1`, thread B writes back `1`
- Two increments happened, but the counter shows `1`: one update was lost

### Deadlock

Two or more threads wait forever on each other.

Example:

- Thread A locks resource X, then waits for resource Y
- Thread B locks resource Y, then waits for resource X
- Neither thread can make progress because each holds what the other needs

A deadlock requires four conditions to hold at once: mutual exclusion, hold-and-wait, no preemption, and circular wait. Breaking any one of them prevents it, which is why a consistent global lock ordering (breaking circular wait) and lock timeouts (breaking no-preemption) are the two most common fixes.

### Livelock

Threads keep reacting, but no useful progress is made.

Example:

- Two workers repeatedly release and retry the same lock "to be polite"
- Both remain active (not blocked), but neither completes work
- Randomized backoff breaks the symmetry, in the same way jitter breaks synchronized retries

### Starvation

One thread never gets CPU time or lock access.

Example:

- Many short high-priority tasks keep taking a shared lock
- A low-priority task waits indefinitely and rarely runs

### Priority inversion

A high-priority task waits on a lower-priority holder.

Example:

- Low-priority thread holds lock `L`
- High-priority thread needs `L` and blocks
- Medium-priority thread keeps running, preventing the low-priority thread from running and releasing `L`
- High-priority work is delayed by lower-priority scheduling

The standard mitigation is priority inheritance: the lock holder temporarily inherits the priority of the highest-priority waiter so it can finish and release.

## Design strategies

- **Avoid sharing in the first place**: Prefer immutable data, thread-local state, or message passing. State that is not shared cannot race
- **Keep critical sections small**: Hold the lock for the shortest possible span, and never perform I/O or a remote call while holding one
- **Use a single global lock order**: Acquiring locks in a consistent order everywhere eliminates circular wait
- **Bound every wait**: Use try-lock with a timeout or explicit cancellation, so a stuck holder degrades into an error rather than a permanent hang
- **Bound every queue**: An unbounded work queue turns overload into memory exhaustion. A bounded queue applies backpressure to producers instead, as described in [Resilience](./14-resilience.md)
- **Cap concurrency per dependency**: A semaphore per downstream service isolates a slow dependency from the rest of the process

## Observability for concurrency issues

Concurrency bugs are timing-dependent and rarely reproduce on demand, so the instrumentation has to be in place before the incident. See [Observability](./15-observability.md) for the general approach.

- Track lock wait time and contention hot spots, not just lock hold time
- Measure queue depth and processing lag, which are the earliest signs of saturation
- Track thread pool utilization and rejected-task counts per pool
- Capture thread dumps during stalls, since a stack snapshot usually identifies a deadlock immediately
- Alert on stuck-worker symptoms: no progress on a queue that still has items

## Interview talking points

- Start from the shared state and the invariant it must preserve, then choose the primitive.
- Match the primitive to the access pattern: read-heavy, write-heavy, or a bounded resource pool.
- State your deadlock-prevention plan explicitly: consistent lock ordering plus bounded waits.
- Mention bounded queues and concurrency limits as the link between local correctness and system-level backpressure.
- Distinguish this from [database concurrency control](./25-database-concurrency-control.md), which solves the same problems across transactions rather than across threads.

## Reference materials

- [The Little Book of Semaphores](https://greenteapress.com/wp/semaphores/)
- [The Go Memory Model](https://go.dev/ref/mem)
