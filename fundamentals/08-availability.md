---
title: "Availability"
concepts:
  - sli-slo-sla
  - availability-nines
  - availability-composition
  - redundancy-and-failover
  - multi-az-multi-region
  - rto-rpo
  - error-budgets
related:
  - fundamentals/09-reliability.md
  - fundamentals/14-resilience.md
  - fundamentals/15-observability.md
  - fundamentals/13-load-balancing.md
  - fundamentals/27-cap-and-pacelc-theorems.md
  - advanced/02-multi-region-replication.md
---

# Availability

Availability is the percentage of time a system is reachable and able to serve requests.

In interviews and production, availability answers: **"Can users access the system right now?"**

## Availability vs reliability vs resilience

These three words are often used interchangeably, but they answer different questions and are measured differently. This table is the reference definition used across these notes.

| Property                           | Question it answers                                             | Typical measure                                              |
| ---------------------------------- | --------------------------------------------------------------- | ------------------------------------------------------------ |
| Availability (this document)       | Can users reach the system and get a response right now?        | Uptime percentage, or successful-request ratio over a window |
| [Reliability](./09-reliability.md) | Are the responses correct, and do they stay correct over time?  | Failure rate, MTTF, MTBF                                     |
| [Resilience](./14-resilience.md)   | How well does the system absorb failures and recover from them? | MTTR, blast radius, error budget burn rate                   |

The three are related but not substitutes:

- A system can be **available but unreliable**: it always answers, but the answers are wrong or the data is corrupted.
- A system can be **reliable but unavailable**: whatever it returns is correct, but it is down too often to be useful.
- **Resilience** is the set of mechanisms (timeouts, circuit breakers, bulkheads, fallbacks, failover) you build so that availability and reliability stay acceptable while components are failing.

## Key terms

- **SLI (Service Level Indicator)**: The measured metric, for example the ratio of successful requests to total requests
- **SLO (Service Level Objective)**: The internal target for that metric, for example 99.9% monthly availability
- **SLA (Service Level Agreement)**: The external contract, plus the consequences (usually credits) if the target is missed

SLAs are normally set looser than SLOs, so the team is alerted and reacting well before a customer-facing commitment is breached.

Availability is measured over a window, in one of two ways:

- **Time-based**: Uptime divided by total time. Simple, but a partially broken system still counts as "up".
- **Request-based**: Successful requests divided by total requests. Better for services where failure is partial rather than total, and it is what most SLOs use today.

## Availability levels ("nines")

| Availability         | Downtime per year | Downtime per month |
| -------------------- | ----------------- | ------------------ |
| 99% (two nines)      | 3.65 days         | 7.2 hours          |
| 99.9% (three nines)  | 8.77 hours        | 43.8 minutes       |
| 99.99% (four nines)  | 52.6 minutes      | 4.4 minutes        |
| 99.999% (five nines) | 5.3 minutes       | 26 seconds         |

Each additional nine costs disproportionately more, because it removes an entire class of failure (single host, then single zone, then single region, then single deploy or config change). Pick the target the business actually needs rather than the highest number you can describe.

## Composing availability

The availability of a system is not the availability of its best component. It depends on how components are wired together.

**Dependencies in series** (a request needs all of them to succeed) multiply:

```text
Two services at 99.9% each, both required:
0.999 * 0.999 = 0.998 -> 99.8% (roughly 17.5 hours of downtime per year)
```

This is why a long dependency chain quietly erodes an availability target, and why critical paths are kept short.

**Redundant instances in parallel** (any one of them can serve the request) multiply the *failure* probability instead:

```text
Two independent replicas at 99% each:
1 - (0.01 * 0.01) = 0.9999 -> 99.99%
```

The word that matters is *independent*. Two replicas in the same rack, on the same power supply, or behind the same config push share a failure domain, so the real number is far worse than the formula suggests. Redundancy only buys availability when the copies fail for different reasons.

## Patterns for high availability

### Redundancy and failover

- Run multiple instances behind a [load balancer](./13-load-balancing.md)
- Use active-active (all instances serve traffic) or active-passive (a standby takes over on failure)
- Remove unhealthy nodes quickly with health checks, so traffic stops reaching them before users notice

### Multi-AZ and multi-region

- Spread critical services across availability zones so a single zone outage is survivable
- Use multi-region when a regional outage is unacceptable, or when users are geographically spread
- Define explicit failover criteria and practice failover drills, so the runbook is not first used during an incident

Multi-region raises availability but forces a consistency trade-off across the replication link. See [CAP and PACELC](./27-cap-and-pacelc-theorems.md) for the reasoning, and [multi-region replication](../advanced/02-multi-region-replication.md) for the mechanics.

### Graceful degradation

When a dependency is down, serving something is usually better than serving nothing:

- Prioritize core user flows, and disable non-critical features under stress
- Serve stale cached data when fresh data is temporarily unavailable

Degradation is a resilience mechanism used in service of availability. The fallback and load-shedding patterns behind it are covered in [Resilience](./14-resilience.md).

### Release and capacity safety

Most outages are self-inflicted, so the deploy and scaling path deserves as much attention as the failure path:

- Autoscaling to absorb load growth before saturation turns into an outage
- Progressive delivery (canary, blue/green) so a bad build reaches a small blast radius first
- Fast, tested rollback, which usually restores availability sooner than diagnosing the bug

For the runtime protections that keep a failing dependency from taking the whole service down (timeouts, retries with backoff, circuit breakers, bulkheads, load shedding), see [Resilience](./14-resilience.md).

## Disaster recovery

- **RTO (Recovery Time Objective)**: Maximum acceptable outage duration
- **RPO (Recovery Point Objective)**: Maximum acceptable data loss window

RTO drives your failover design (a warm standby recovers in minutes, restoring from backup takes hours). RPO drives your replication and backup design (synchronous replication approaches an RPO of zero, hourly snapshots imply losing up to an hour of writes). An availability target that is stricter than your RTO is not achievable, so the two numbers should be chosen together.

## Error budgets

The error budget is the amount of acceptable failure for a service over some period of time, usually a month.

- `Error budget = 1 - SLO` (for example, `1 - 99.9% = 0.1%`)
- A 99.9% monthly SLO allows about 43.8 minutes of unavailability per month

Error budgets turn "how reliable should we be?" into a spending decision:

- **While budget remains**: the team ships features freely.
- **When the budget is exhausted**: the team stops feature work and spends the time on stability instead.

The *burn rate* (how fast the budget is being consumed) is also a good alerting signal, since it catches slow degradation that a simple threshold alert misses.
See [Observability](./15-observability.md) for how these signals are collected and alerted on.

## Interview talking points

- Availability is about **uptime and reachability**, not correctness. Say which of the three properties you are optimizing.
- Start with a realistic SLO and map architecture choices to it, rather than reaching for five nines by default.
- Walk through failure domains explicitly: host, zone, region, deploy, dependency.
- Show that you understand serial dependencies multiply, and that redundancy only helps when failures are independent.
- Tie every availability decision back to cost and operational complexity.

## Reference materials

- [The Calculus of Service Availability](https://queue.acm.org/detail.cfm?id=3096459)
- [Google SRE Book - Service Level Objectives](https://sre.google/sre-book/service-level-objectives/)
