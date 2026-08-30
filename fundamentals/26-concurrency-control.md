---
title: "Concurrency control"
concepts:
  - critical-section
  - mutual-exclusion
  - mutex
  - semaphore
  - race-condition
  - deadlock
  - livelock
  - priority-inversion
related:
  - fundamentals/25-database-concurrency-control.md
  - fundamentals/15-observability.md
  - fundamentals/14-resilience.md
---

# Concurrency control

Concurrency control coordinates multiple threads/processes so shared state stays correct when they access it concurrently.

At the OS/application level, this is about synchronization, ordering, and safe access to shared resources.

## Core concepts

- **Critical section**: Code that accesses shared mutable state
- **Mutual exclusion**: Only one execution context enters a critical section at a time
- **Progress/liveness**: System keeps making forward progress
- **Fairness**: No actor is starved indefinitely

## Synchronization primitives

### Mutex / lock

- Simplest mutual exclusion primitive
- Good for protecting short critical sections

Example:

- Protect shared in-memory balance map during update (`lock -> read/modify/write -> unlock`)
- If two threads update the same account, lock ensures one completes before the other starts

### Read-write lock

- Many concurrent readers, exclusive writer
- Useful for read-heavy shared structures

Example:

- In-memory product catalog with frequent reads and occasional updates
- Multiple readers can query simultaneously, but refresh job takes write lock to replace snapshot safely

### Semaphore

- Controls access to a limited pool (for example, max 20 workers)
- Can be binary or counting

Example:

- Limit outbound calls to a third-party API to 50 concurrent requests
- Each worker acquires one permit before call and releases after response
- Prevents connection/resource exhaustion during traffic spikes

### Condition variable

- Allows threads to sleep until a condition becomes true
- Typically used with a mutex for producer-consumer patterns

Example:

- Consumer waits while queue is empty
- Producer pushes item and signals condition
- Consumer wakes up, re-checks condition, and processes item

## Concurrency problems

### Race condition

The program's result depends on thread or process timing, not just on logic.

Example:

- Shared counter starts at `0`
- Thread A reads `0`, thread B reads `0`
- Thread A writes back `1`, thread B writes back `2`
- The final value depends on the interleaving of the two operations

### Deadlock

Two or more threads wait forever on each other.

Example:

- Thread A locks resource X, then waits for resource Y
- Thread B locks resource Y, then waits for resource X
- Neither thread can make progress because they are waiting on each other

### Livelock

Threads keep reacting, but no useful progress is made.

Example:

- Two workers repeatedly release and retry the same lock "to be polite"
- Both remain active (not blocked), but neither completes work

### Starvation

One thread never gets CPU or lock access.

Example:

- Many short high-priority tasks keep taking a shared lock
- A low-priority task waits indefinitely and rarely runs

### Priority inversion

A high-priority task waits on a lower-priority holder.

Example:

- Low-priority thread holds lock `L`
- High-priority thread needs `L` and blocks
- Medium-priority thread keeps running, preventing low-priority thread from running and releasing `L`
- High-priority work is delayed by lower-priority scheduling

## Design strategies

- Prefer immutable data and message passing when possible
- Keep critical sections small and simple
- Use a single global lock order to reduce deadlocks
- Avoid blocking calls while holding locks
- Add timeouts or cancellation for long waits

## Observability for concurrency issues

- Track lock wait time and contention hot spots
- Measure queue depth and processing lag
- Capture thread dumps during stalls
- Alert on deadlock or stuck-worker symptoms

## Interview talking points

- Start with shared state and invariants
- Choose primitive by access pattern (read-heavy, write-heavy, bounded resource)
- Explain your deadlock-prevention and backpressure plan
