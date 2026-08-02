# Reliability

Reliability is the probability that a system produces correct results continuously over time.

Availability asks **"Is it up?"**  
Reliability asks **"Is it working correctly?"**

## Why Reliability Matters

A system can be highly available but still unreliable (for example, always online but returning wrong data).

Reliability is critical for:

- Financial correctness (payments, balances, billing)
- Data integrity (no corruption, no silent loss)
- Predictable behavior under normal and failure conditions

## Reliability Signals and Metrics

- **Failure rate**: How often requests or jobs fail
- **MTTF** (Mean Time To Failure): Average time before failure
- **MTTR** (Mean Time To Recovery): Average time to restore service
- **MTBF** (Mean Time Between Failures): Common for repairable systems

Improving reliability often means reducing both failure frequency and recovery time.

## Common Reliability Failure Modes

- Partial failures in distributed systems
- Retries causing duplicate side effects
- Race conditions and stale writes
- Data inconsistency across replicas/services
- Unhandled edge cases and bad assumptions

## Patterns to Improve Reliability

### Idempotency

- Ensure repeated requests produce the same outcome
- Essential with retries, queues, and at-least-once delivery

### Timeouts, Retries, and Backoff

- Set timeouts on all remote calls
- Retry only transient failures
- Use exponential backoff and jitter

### Isolation and Protection

- Circuit breakers for failing dependencies
- Bulkheads to isolate resource pools
- Dead-letter queues for poison messages

### Data Correctness Controls

- Constraints and invariants at storage layer
- Exactly-once illusions via deduplication/outbox patterns
- Checksums/validation for corruption detection

## Reliability Testing and Operations

- Failure injection / chaos testing
- Load and stress testing
- Runbooks and incident response drills
- Postmortems with actionable follow-ups

## Reliability vs Complexity

More components can improve availability but may hurt reliability if coordination becomes fragile.

Design goal: keep reliability mechanisms explicit and simple enough to operate.

## Interview Talking Points

- Reliability is about **correctness over time**, not only uptime.
- Highlight idempotency, retries, and data consistency controls.
- Discuss failure modes explicitly, then show mitigation strategy.
- Mention detection + recovery, not only prevention.

## Reference Materials

- [Google SRE Book - Addressing Cascading Failures](https://sre.google/sre-book/addressing-cascading-failures/)
- [Release It! Stability Patterns](https://pragprog.com/titles/mnee2/release-it-second-edition/)
