---
title: "Reliability"
concepts:
  - correctness-over-time
  - mttf-mttr-mtbf
  - idempotency
  - deduplication
  - outbox-pattern
  - data-integrity-controls
  - failure-injection
related:
  - fundamentals/08-availability.md
  - fundamentals/14-resilience.md
  - fundamentals/15-observability.md
  - fundamentals/18-checksums.md
  - fundamentals/25-database-concurrency-control.md
  - fundamentals/26-concurrency-control.md
  - fundamentals/31-messaging-patterns.md
---

# Reliability

Reliability is the probability that a system produces correct results, continuously, over a period of time.

Availability asks **"Is it up?"**. Reliability asks **"Is what it returned actually right?"**. See [Availability](./08-availability.md) for the full comparison of availability, reliability, and resilience.

## Why reliability matters

A system can be highly available and still be unreliable: online every second of the month, and returning a stale balance, double-charging a card, or silently dropping one message in ten thousand. Users notice the wrong answer long after the dashboard says everything is green.

Reliability is the property that matters most for:

- **Financial correctness**: Payments, balances, billing, ledgers
- **Data integrity**: No corruption, no silent loss, no partially applied writes
- **Predictable behavior**: The same request produces the same outcome under normal *and* failure conditions

## Reliability signals and metrics

| Metric                            | Definition                                                | Improved by                           |
| --------------------------------- | --------------------------------------------------------- | ------------------------------------- |
| Failure rate                      | Fraction of requests or jobs that fail or return bad data | Removing failure modes                |
| MTTF (Mean Time To Failure)       | Average time a system runs before it fails                | Fewer defects, less fragile coupling  |
| MTTR (Mean Time To Recovery)      | Average time to restore correct service after a failure   | Detection, automation, rollback speed |
| MTBF (Mean Time Between Failures) | Full failure-to-failure cycle for a repairable system     | Both of the above                     |

These connect back to availability:

```text
MTBF = MTTF + MTTR
Availability = MTTF / (MTTF + MTTR)

Example: fails once every 30 days (720h), takes 1h to recover
  720 / (720 + 1) = 99.86%

Halve recovery time to 30 minutes, change nothing else
  720 / (720 + 0.5) = 99.93%
```

The second line is the practical lesson: cutting recovery time is usually cheaper than eliminating the failure, so invest in detection and rollback before you invest in perfection.

## Common reliability failure modes

- **Partial failure**: One service in a chain fails while the rest succeed, leaving the operation half-applied
- **Duplicate side effects**: A retry or redelivery re-runs an operation that was not safe to repeat
- **Race conditions and lost updates**: Concurrent writers overwrite each other, in-process ([Concurrency control](./26-concurrency-control.md)) or in the database ([Database concurrency control](./25-database-concurrency-control.md))
- **Replica divergence**: Reads hit a lagging or diverged replica and return stale or conflicting data
- **Silent corruption**: Bit rot or a bad write is stored and replicated because nothing validates it
- **Unhandled edge cases**: Assumptions that hold in testing (ordered delivery, unique input, clock monotonicity) but not in production

Notice that most of these do not cause downtime. That is exactly why reliability needs its own metrics and its own tests rather than riding on the availability dashboard.

## Patterns to improve reliability

### Idempotency

An operation is idempotent when performing it more than once has the same effect as performing it once. This is the single most important reliability property in a distributed system, because retries, at-least-once queues, and client double-submits all make repeat delivery inevitable.

Ways to get it:

- **Naturally idempotent operations**: `SET status = 'shipped'` is safe to repeat; `balance = balance - 10` is not
- **Idempotency keys**: The caller supplies a unique key per logical operation, and the server stores the result against that key and replays it on a repeat
- **Conditional writes**: Compare-and-set on a version or `ETag` so a stale retry is rejected rather than applied

Example (payment capture):

1. Client generates `Idempotency-Key: 8f3c-...` and sends the charge request
2. Server records the key with the charge result inside the same transaction as the charge
3. Network times out, so the client retries with the same key
4. Server finds the key, skips the charge, and returns the original result

The card is charged once, and the client still gets a definitive answer. Without step 2 being atomic with the charge, a crash between the two writes reintroduces the double charge.

### Deduplication and "exactly-once"

True exactly-once delivery is not achievable across an unreliable network. What is achievable is **at-least-once delivery plus idempotent processing**, which is indistinguishable from exactly-once from the outside. Common building blocks:

- **Dedup store**: A key set with a retention window long enough to cover the maximum retry horizon
- **Transactional outbox**: Write the business change and the outgoing event in one local transaction, then publish from the outbox, so the event cannot be lost or emitted for a rolled-back change
- **Inbox table**: Record processed message IDs on the consumer side to reject redeliveries

The messaging-side view of these patterns (acknowledgments, delivery semantics, idempotent consumers) is covered in [Messaging patterns](./31-messaging-patterns.md).

### Data correctness controls

- **Constraints and invariants at the storage layer**: Unique keys, foreign keys, and check constraints catch bugs that application code forgets to
- **Validation at boundaries**: Reject malformed input at ingress rather than storing it and discovering it later
- **[Checksums](./18-checksums.md)**: Detect corruption in transit and at rest, since replication happily copies bad bytes
- **Reconciliation jobs**: Periodically compare systems that should agree (ledger versus payment provider, cache versus source of truth) and alert on drift

Reconciliation matters because prevention is never complete. Assume some fraction of writes will be wrong and build the job that finds them.

### Safe retries and failure containment

Retries improve reliability only when the operation is idempotent and the retry is bounded; otherwise they multiply duplicate side effects and turn a slow dependency into an outage.
The mechanics of timeouts, exponential backoff with jitter, circuit breakers, bulkheads, and load shedding are covered in [Resilience](./14-resilience.md).
The reliability-specific rule is the pairing: **never add a retry to an operation you have not made idempotent.**

Messages that fail every attempt should end in a dead-letter queue rather than blocking the partition or being dropped. See [Messaging patterns](./31-messaging-patterns.md) for the mechanics.

## Testing and operating for reliability

Reliability bugs rarely show up in unit tests, because they live in timing, retries, and partial failure. Useful practices:

- **Property and invariant tests**: Assert that a ledger balances or that replaying a message twice changes nothing
- **Load and stress testing**: Many correctness bugs only appear under contention
- **Failure injection**: Kill dependencies, add latency, and drop packets to see what the system actually does. The broader chaos and game-day practice is covered in [Resilience](./14-resilience.md)
- **Runbooks and incident drills**: The fastest MTTR comes from a procedure someone has run before
- **Blameless postmortems with tracked follow-ups**: A postmortem without an owned action item does not reduce failure rate

Detection depends on telemetry: correctness errors are usually invisible unless you deliberately measure them. See [Observability](./15-observability.md).

## Reliability vs complexity

Every reliability mechanism is itself code that can fail. Adding replicas, queues, and coordination raises availability but can lower reliability if the coordination logic is fragile or poorly understood.

The design goal is to keep reliability mechanisms **explicit, few, and simple enough to operate at 3am**. A single well-understood idempotency key beats three overlapping deduplication layers that nobody can reason about.

## Interview talking points

- Reliability is about **correctness over time**, not uptime. Name the distinction explicitly.
- Lead with idempotency whenever retries, queues, or client resubmits are in the design.
- Say "at-least-once delivery plus idempotent processing" rather than claiming exactly-once.
- Enumerate failure modes first, then map each one to a mitigation.
- Cover detection and recovery, not only prevention: reducing MTTR moves the number as much as reducing failures.

## Reference materials

- [Google SRE Book - Addressing Cascading Failures](https://sre.google/sre-book/addressing-cascading-failures/)
- [Release It! Stability Patterns](https://pragprog.com/titles/mnee2/release-it-second-edition/)
