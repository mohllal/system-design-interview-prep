# Resilience

Resilience is a system's ability to continue delivering acceptable service during failures and recover quickly afterward.

It focuses on graceful degradation, fault isolation, and fast recovery rather than assuming failures will never happen.

## Resilience vs Reliability vs Availability

- Availability: Is the system reachable right now?
- Reliability: Does it produce correct results over time?
- Resilience: How well does it absorb failures and recover?

Resilience techniques often improve both availability and reliability in real systems.

## Common Failure Types

- Dependency timeouts and partial outages
- Traffic spikes and resource saturation
- Network partitions and packet loss
- Bad deploys or misconfiguration
- Regional cloud failures

## Resilience Patterns

### Timeouts and Retries

- Set explicit timeouts on all remote calls
- Retry only transient failures
- Use exponential backoff + jitter to avoid retry storms

### Circuit Breakers

- Open circuit when a dependency is failing repeatedly
- Fail fast instead of waiting on slow downstream calls
- Probe periodically to close circuit when dependency recovers

### Bulkheads

- Isolate resources by workload (thread pools, queues, connection pools)
- Prevent one noisy path from starving all others

### Fallbacks and Graceful Degradation

- Serve cached/stale data when live dependency is down
- Disable non-critical features under stress
- Preserve critical user journeys first

### Rate Limiting and Load Shedding

- Bound incoming traffic to protect backend capacity
- Drop low-priority work when saturated
- Keep core SLOs for high-priority traffic

## Recovery and Operational Readiness

- Health checks (liveness/readiness/startup)
- Automated failover for critical services
- Regular disaster-recovery and game-day drills

## Data and Messaging Resilience

- Idempotency keys for safe retries
- Dead-letter queues for poison messages
- Outbox/inbox patterns for reliable event delivery
- Backpressure signals to prevent queue collapse

## Measuring Resilience

- MTTR (mean time to recovery)
- Incident frequency and severity
- Error budget burn rate
- Recovery success rate during drills

## Common Pitfalls

- Infinite retries without deadlines
- Shared pools across unrelated workloads
- No fallback path for critical dependencies
- Untested failover plans

## Interview Talking Points

- Resilience is about surviving failure, not avoiding all failures.
- Start with failure modes, then map each to a mitigation pattern.
- Explain both prevention and recovery (detect, isolate, restore).
- Mention trade-offs: complexity, cost, and degraded user experience.

## Reference Materials

- [Google SRE - Addressing Cascading Failures](https://sre.google/sre-book/addressing-cascading-failures/)
- [AWS Builders Library - Timeouts, retries, and backoff](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/)
