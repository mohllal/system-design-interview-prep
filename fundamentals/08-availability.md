---
title: "Availability"
concepts:
  - sli-slo-sla
  - availability-nines
  - redundancy-and-failover
  - multi-az-multi-region
  - graceful-degradation
  - disaster-recovery
  - rto-rpo
  - error-budgets
related:
  - fundamentals/09-reliability.md
  - fundamentals/14-resilience.md
  - fundamentals/27-cap-and-pacelc-theorems.md
  - advanced/02-multi-region-replication.md
---

# Availability

Availability is the percentage of time a system is reachable and able to serve requests.

In interviews and production, availability answers: **"Can users access the system right now?"**

## Key terms

- **SLI (Service Level Indicator)**: Measured metric (for example, successful request rate)
- **SLO (Service Level Objective)**: Target for that metric (for example, 99.9% monthly availability)
- **SLA (Service Level Agreement)**: External contract and consequences if the objective is missed

## Availability levels ("nines")

| Availability         | Downtime per year |
| -------------------- | ----------------- |
| 99% (two nines)      | 3.65 days         |
| 99.9% (three nines)  | 8.77 hours        |
| 99.99% (four nines)  | 52.6 minutes      |
| 99.999% (five nines) | 5.3 minutes       |

Higher availability usually costs disproportionately more.

## Patterns for high availability

### Redundancy and failover

- Run multiple instances behind a load balancer
- Use active-active or active-passive patterns
- Remove unhealthy nodes quickly with health checks

### Multi-AZ and multi-region

- Spread critical services across availability zones
- Use multi-region for disaster resilience when needed
- Define failover criteria and practice failover drills

### Graceful degradation

- Prioritize core user flows during incidents
- Disable non-critical features under stress
- Serve stale cache when fresh data is temporarily unavailable

### Operational guardrails

- Timeouts, retries with backoff, circuit breakers
- Autoscaling and rate limiting
- Progressive delivery (canary, blue/green)

## Disaster recovery

- **RTO (Recovery Time Objective)**: Maximum acceptable outage duration
- **RPO (Recovery Point Objective)**: Maximum acceptable data loss window

Availability targets should align with RTO/RPO expectations.

## Error budgets

The error budget is the amount of acceptable failure for a service over some period of time (often a month).

- `Error Budget = 1 - SLO` (e.g., `1 - 99% = 1%`)
- Example: A 99.9% monthly SLO allows about 43.8 minutes of unavailability per month

Teams use error budgets to balance reliability work against feature delivery.
As long as a service hasn't spent its error budget for the month, the development team is free (within reason) to launch new features, updates, and so on.

## Interview talking points

- Availability is about **uptime and reachability**, not correctness.
- Start with realistic SLOs and map architecture choices to them.
- Mention failure domains, redundancy, failover, and graceful degradation.
- Tie design choices to cost and operational complexity.

## Reference materials

- [The Calculus of Service Availability](https://queue.acm.org/detail.cfm?id=3096459)
- [Google SRE Book - Service Level Objectives](https://sre.google/sre-book/service-level-objectives/)
