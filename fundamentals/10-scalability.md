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
  - fundamentals/13-load-balancing.md
  - fundamentals/23-database-replication.md
  - fundamentals/24-database-partitioning.md
  - fundamentals/33-back-of-the-envelope-calculations.md
---

# Scalability

Scalability is the ability of a system to handle growth in traffic, data, and workload without unacceptable performance or cost.

## Types of scalability

### Vertical scaling (scale up)

- Add more CPU/RAM/storage to a single machine
- Simple to start with, but hits hardware limits

### Horizontal scaling (scale out)

- Add more machines and distribute work
- Better long-term growth and fault isolation
- Requires coordination (load balancing, partitioning, consistency)

## Dimensions of growth

- **Traffic growth**: More requests per second
- **Data growth**: More records, bigger indexes, longer history
- **Compute growth**: More expensive business logic/model inference
- **Geographic growth**: More users across regions

## Common bottlenecks

- Single database primary
- Chatty synchronous service-to-service calls
- Hot keys / uneven partitioning
- Shared locks and contention
- CPU-heavy endpoints without caching

## Core scalability patterns

### Stateless service tier

- Keep app servers stateless where possible
- Scale instances behind a load balancer

### Caching

- Use CDN, reverse proxy, and application/data caches
- Reduce repeated expensive reads

### Data partitioning

- Shard by well-chosen key to spread load
- Avoid hotspots and rebalance as traffic shifts

### Async processing

- Move slow work to queues/workers
- Keep user-facing latency low

### Read/write separation

- Replicas for read-heavy traffic
- Careful handling of replication lag and read consistency

## Throughput, latency, and cost

A scalable design does more than handle more load — it also:

- Keeps latency within SLO
- Maintains reliability under load
- Improves capacity without linear cost explosion

## Capacity planning basics

- Estimate peak RPS/QPS, payload sizes, and growth rate
- Identify p95/p99 latency targets
- Plan headroom (for example, 30-50%) for bursts and failures
- Test with realistic load, not only average traffic

## Scalability anti-patterns

- Premature microservices without clear need
- One giant database table with no partition strategy
- Ignoring tail latency while tracking only averages
- Unlimited retries under overload

## Interview talking points

- Start from bottleneck identification, not random optimizations.
- Explain scale path by layer: edge, service, data, background workers.
- Mention trade-offs: consistency, complexity, and cost.
- Include observability and load testing as part of scale readiness.
