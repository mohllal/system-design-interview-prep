---
title: "Resilience"
concepts:
  - graceful-degradation
  - fault-isolation
  - circuit-breakers
  - bulkheads
  - retries-with-backoff
  - load-shedding
  - failover
  - dead-letter-queues
  - backpressure
related:
  - fundamentals/08-availability.md
  - fundamentals/09-reliability.md
  - fundamentals/15-observability.md
  - fundamentals/31-messaging-patterns.md
  - fundamentals/32-rate-limiting.md
---

# Resilience

Resilience is a system's ability to continue delivering acceptable service during failures and recover quickly afterward.

It focuses on graceful degradation, fault isolation, and fast recovery rather than assuming failures will never happen.

## Resilience vs reliability vs availability

- **Availability**: Is the system reachable right now?
- **Reliability**: Does it produce correct results over time?
- **Resilience**: How well does it absorb failures and recover?

Resilience techniques often improve both availability and reliability in real systems.

## Common failure types

- Dependency timeouts and partial outages
- Traffic spikes and resource saturation
- Network partitions and packet loss
- Bad deploys or misconfiguration
- Regional cloud failures

## Resilience patterns

### Timeouts and retries

- Set explicit timeouts on all remote calls
- Retry only transient failures
- Use exponential backoff and jitter to avoid retry storms

### Circuit breakers

- Open the circuit when a dependency is failing repeatedly
- Fail fast instead of waiting on slow downstream calls
- Probe periodically to close the circuit once the dependency recovers

### Bulkheads

- Isolate resources by workload (thread pools, queues, connection pools)
- Prevent one noisy path from starving all others

### Fallbacks and graceful degradation

- Serve cached or stale data when the live dependency is down
- Disable non-critical features under stress
- Preserve critical user journeys first

### Rate limiting and load shedding

- Bound incoming traffic to protect backend capacity
- Drop low-priority work when saturated
- Keep core SLOs for high-priority traffic

## Recovery and operational readiness

- Health checks (liveness/readiness/startup)
- Automated failover for critical services
- Regular disaster-recovery and game-day drills

## Data and messaging resilience

- Idempotency keys for safe retries
- Dead-letter queues for poison messages
- Outbox/inbox patterns for reliable event delivery
- Backpressure signals to prevent queue collapse

## Measuring resilience

- MTTR (mean time to recovery)
- Incident frequency and severity
- Error budget burn rate
- Recovery success rate during drills

## Common pitfalls

- Infinite retries without deadlines
- Shared pools across unrelated workloads
- No fallback path for critical dependencies
- Untested failover plans

## Interview talking points

- Resilience is about surviving failure, not avoiding all failures.
- Start with failure modes, then map each to a mitigation pattern.
- Explain both prevention and recovery (detect, isolate, restore).
- Mention trade-offs: complexity, cost, and degraded user experience.

## Reference materials

- [Google SRE - Addressing Cascading Failures](https://sre.google/sre-book/addressing-cascading-failures/)
- [AWS Builders Library - Timeouts, retries, and backoff](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/)
