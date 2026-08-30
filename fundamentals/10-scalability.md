---
title: "Scalability"
concepts:
  - vertical-scaling
  - horizontal-scaling
  - stateless-services
  - data-partitioning
  - async-processing
  - read-write-separation
  - capacity-planning
  - hot-keys
related:
  - fundamentals/03-latency-and-throughput.md
  - fundamentals/09-reliability.md
  - fundamentals/11-caching.md
  - fundamentals/12-proxies.md
  - fundamentals/13-load-balancing.md
  - fundamentals/23-database-replication.md
  - fundamentals/24-database-partitioning.md
  - fundamentals/32-rate-limiting.md
  - fundamentals/33-back-of-the-envelope-calculations.md
  - fundamentals/34-cdn.md
---

# Scalability

Scalability is the ability of a system to handle growth in traffic, data, and workload without unacceptable performance or cost.

It is not the same thing as performance. Performance is how fast one request is served today; scalability is what happens to that number when the load multiplies. A system can be fast at 100 RPS and unusable at 10,000 RPS, and the fix is usually structural rather than a faster machine.

## Types of scalability

### Vertical scaling (scale up)

- Add more CPU/RAM/storage to a single machine
- Simple to start with: no architecture changes, no coordination
- Hits a hardware ceiling, gets expensive fast, and leaves a single point of failure

### Horizontal scaling (scale out)

- Add more machines and distribute work across them
- Better long-term growth and fault isolation: one node dying removes a fraction of capacity, not all of it
- Requires coordination — something must distribute the work ([load balancing](./13-load-balancing.md)), split the data ([partitioning](./24-database-partitioning.md)), and reconcile the results

Most designs start vertical because it is cheap and immediate, then go horizontal once the ceiling or the availability risk becomes the binding constraint.

## Dimensions of growth

Scale is not one number. Name which dimension is actually growing before choosing a technique:

- **Traffic growth**: More requests per second
- **Data growth**: More records, bigger indexes, longer history
- **Compute growth**: More expensive business logic or model inference per request
- **Geographic growth**: More users across more regions

Each dimension has a different answer. Traffic growth is solved with more stateless replicas; data growth is solved with partitioning; geographic growth is solved at the edge.

## Common bottlenecks

Scaling work starts with finding the constraint, not with applying patterns:

- A single database primary absorbing every write
- Chatty synchronous service-to-service calls that multiply latency
- Hot keys and uneven partitioning that concentrate load on one shard
- Shared locks and contention on a common row, queue, or counter
- CPU-heavy endpoints served without caching

## Scaling by layer

Work through the request path in order. Each layer removes load from the one behind it, so fixing the outermost layer first is usually the cheapest win.

### Edge: serve it before it reaches you

- Push static and cacheable content to a [CDN](./34-cdn.md) so most reads never touch your infrastructure
- Terminate TLS and apply routing at a [reverse proxy](./12-proxies.md) rather than in application code

### Service tier: keep it stateless

- A stateless server keeps no per-client data between requests, so any replica can serve any request
- That is what makes replicas interchangeable, which is what makes adding a machine behind a [load balancer](./13-load-balancing.md) an actual capacity increase
- Move the state that used to live in process memory — sessions, uploads in progress, rate-limit counters — into a shared store such as Redis or the database
- Sticky sessions are the fallback when state cannot be externalized, at the cost of uneven load and lost state on restart

### Data tier: split reads from writes, then split the data

- **Read/write separation**: Send read-heavy traffic to [replicas](./23-database-replication.md) and keep writes on the primary; budget for replication lag and decide which reads must be read-your-writes consistent
- **Caching**: Put an [application cache](./11-caching.md) in front of expensive reads; plan invalidation and stampede protection before you rely on it
- **Partitioning**: [Shard](./24-database-partitioning.md) by a key that spreads load evenly, and expect to rebalance as traffic shifts

The data tier is usually the real ceiling. Application servers are easy to clone; a single primary is not.

### Background: move slow work off the request path

- Push anything the user does not need to wait for — thumbnails, emails, indexing, analytics — onto a queue with dedicated workers
- Keeps user-facing latency flat while heavy work scales independently
- Trades immediate consistency for a job the user only sees the result of later

## Throughput, latency, and cost

A scalable design does more than survive more load. It also:

- Keeps latency within its SLO as load rises, including [p95/p99 tail latency](./03-latency-and-throughput.md)
- Maintains reliability under load instead of failing all at once
- Adds capacity without a linear (or worse) cost increase

If doubling traffic requires doubling every tier, the design scales but does not scale *economically* — usually a sign that a cache or an async path is missing.

## Capacity planning basics

- Estimate peak RPS/QPS, payload sizes, and growth rate — see [back-of-the-envelope calculations](./33-back-of-the-envelope-calculations.md)
- Set explicit p95/p99 latency targets, not just averages
- Plan headroom (30-50% is a common starting point) so a node loss or a burst does not immediately saturate the rest
- Load test with realistic traffic shapes, including peaks and hot keys, not only average traffic

## Protecting the ceiling

Every system has a load it cannot serve. Scaling decides where that point is; these decide what happens when you reach it:

- **[Rate limiting](./32-rate-limiting.md)**: Bounds what any single caller can consume, so one client cannot spend the whole fleet's capacity
- **Load shedding**: Drops low-priority work once the system is already saturated, protecting the critical paths
- **Autoscaling**: Adds capacity on a metric, but only as fast as instances boot — it does not save you from a sudden spike on its own

Without these, the failure mode at the ceiling is a slow, total collapse instead of a partial, recoverable degradation.

## Scalability anti-patterns

- Splitting into microservices before a scaling or ownership problem exists
- One giant table with no partition strategy and no plan for when it stops fitting
- Tracking only average latency, so tail-latency regressions go unnoticed
- Unlimited retries under overload, which turns a slow dependency into an outage
- Caching without an invalidation plan, which trades a scale problem for a correctness one

## Interview talking points

- Start from bottleneck identification, not from a list of optimizations.
- Walk the scale path by layer: edge, service tier, data tier, background workers.
- Say which dimension is growing (traffic, data, compute, geography) — it determines the technique.
- Name the trade-offs you accept: consistency, operational complexity, and cost.
- Include observability and load testing as part of scale readiness, not as an afterthought.
