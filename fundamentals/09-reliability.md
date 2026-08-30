---
title: "Reliability"
concepts:
  - idempotency
  - circuit-breakers
  - bulkheads
  - dead-letter-queues
  - exponential-backoff
  - chaos-testing
  - mttf-mttr-mtbf
related:
  - fundamentals/08-availability.md
  - fundamentals/14-resilience.md
  - fundamentals/15-observability.md
  - fundamentals/18-checksums.md
  - fundamentals/26-concurrency-control.md
---

# Reliability

Reliability is the probability that a system produces correct results continuously over time.

Availability asks **"Is it up?"**  
Reliability asks **"Is it working correctly?"**

## Why reliability matters

A system can be highly available but still unreliable (for example, always online but returning wrong data).

Reliability is critical for:

- Financial correctness (payments, balances, billing)
- Data integrity (no corruption, no silent loss)
- Predictable behavior under normal and failure conditions

## Reliability signals and metrics

| Metric                            | Definition                      |
| --------------------------------- | ------------------------------- |
| Failure rate                      | How often requests or jobs fail |
| MTTF (Mean Time To Failure)       | Average time before failure     |
| MTTR (Mean Time To Recovery)      | Average time to restore service |
| MTBF (Mean Time Between Failures) | Common for repairable systems   |

Improving reliability often means reducing both failure frequency and recovery time.

## Common reliability failure modes

- Partial failures in distributed systems
- Retries causing duplicate side effects
- Race conditions and stale writes
- Data inconsistency across replicas/services
- Unhandled edge cases and bad assumptions

## Patterns to improve reliability

### Idempotency

- Ensure repeated requests produce the same outcome
- Essential with retries, queues, and at-least-once delivery

### Timeouts, retries, and backoff

- Set timeouts on all remote calls
- Retry only transient failures
- Use exponential backoff and jitter

### Isolation and protection

- Circuit breakers for failing dependencies
- Bulkheads to isolate resource pools
- Dead-letter queues for poison messages

### Data correctness controls

- Constraints and invariants at the storage layer
- Exactly-once illusions via deduplication/outbox patterns
- Checksums/validation for corruption detection

## Reliability testing and operations

- Failure injection / chaos testing
- Load and stress testing
- Runbooks and incident response drills
- Postmortems with actionable follow-ups

## Reliability vs complexity

More components can improve availability but may hurt reliability if coordination becomes fragile.

Design goal: keep reliability mechanisms explicit and simple enough to operate.

## Interview talking points

- Reliability is about **correctness over time**, not only uptime.
- Highlight idempotency, retries, and data consistency controls.
- Discuss failure modes explicitly, then show mitigation strategy.
- Mention detection and recovery, not only prevention.

## Reference materials

- [Google SRE Book - Addressing Cascading Failures](https://sre.google/sre-book/addressing-cascading-failures/)
- [Release It! Stability Patterns](https://pragprog.com/titles/mnee2/release-it-second-edition/)
