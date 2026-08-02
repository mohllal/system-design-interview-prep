# Scalability

Scalability is the ability of a system to handle growth in traffic, data, and workload without unacceptable performance or cost.

## Types of Scalability

### Vertical Scaling (Scale Up)

- Add more CPU/RAM/storage to a single machine
- Simple to start with, but hits hardware limits

### Horizontal Scaling (Scale Out)

- Add more machines and distribute work
- Better long-term growth and fault isolation
- Requires coordination (load balancing, partitioning, consistency)

## Dimensions of Growth

- **Traffic growth**: More requests per second
- **Data growth**: More records, bigger indexes, longer history
- **Compute growth**: More expensive business logic/model inference
- **Geographic growth**: More users across regions

## Common Bottlenecks

- Single database primary
- Chatty synchronous service-to-service calls
- Hot keys / uneven partitioning
- Shared locks and contention
- CPU-heavy endpoints without caching

## Core Scalability Patterns

### Stateless Service Tier

- Keep app servers stateless where possible
- Scale instances behind a load balancer

### Caching

- Use CDN, reverse proxy, and application/data caches
- Reduce repeated expensive reads

### Data Partitioning

- Shard by well-chosen key to spread load
- Avoid hotspots and rebalance as traffic shifts

### Async Processing

- Move slow work to queues/workers
- Keep user-facing latency low

### Read/Write Separation

- Replicas for read-heavy traffic
- Careful handling of replication lag and read consistency

## Throughput, Latency, and Cost

A scalable design is not only "handles more load," but also:

- Keeps latency within SLO
- Maintains reliability under load
- Improves capacity without linear cost explosion

## Capacity Planning Basics

- Estimate peak RPS/QPS, payload sizes, and growth rate
- Identify p95/p99 latency targets
- Plan headroom (for example, 30-50%) for bursts and failures
- Test with realistic load, not only average traffic

## Scalability Anti-Patterns

- Premature microservices without clear need
- One giant database table with no partition strategy
- Ignoring tail latency while tracking only averages
- Unlimited retries under overload

## Interview Talking Points

- Start from bottleneck identification, not random optimizations.
- Explain scale path by layer: edge, service, data, background workers.
- Mention trade-offs: consistency, complexity, and cost.
- Include observability and load testing as part of scale readiness.
