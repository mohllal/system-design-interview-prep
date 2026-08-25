# System Design Fundamentals

Essential building blocks for understanding distributed systems and scalable architectures.

## Contents

### Communication & Networking

Core concepts for how systems communicate and exchange data.

| Topic                                                              | Key Concepts                                                      |
|--------------------------------------------------------------------|-------------------------------------------------------------------|
| [Client-Server Model](./01-client-server.md)                       | Request-response patterns, DNS resolution, service roles          |
| [Network Protocols](./02-network-protocols.md)                     | TCP/IP stack, browser request flow, protocol trade-offs           |
| [Latency and Throughput](./03-latency-and-throughput.md)           | Performance metrics, optimization strategies, bottleneck analysis |
| [HTTP Versions](./04-http-versions.md)                             | HTTP/1.0 to HTTP/3, QUIC, performance trade-offs                  |
| [REST API](./05-rest-api.md)                                       | Resource modeling, HTTP semantics, versioning                     |
| [Communication Patterns](./06-communication-patterns.md)           | REST, RPC, GraphQL, polling, streaming, messaging                 |
| [Real-Time Communication](./07-realtime-communication-patterns.md) | Short/long polling, SSE, WebSockets, live update trade-offs       |
| [Messaging Patterns](./29-messaging-patterns.md)                   | Queues, pub/sub, request-reply, delivery semantics, routing       |

### Reliability & Performance

Strategies for building resilient and performant systems.

| Topic                                                                | Key Concepts                                       |
|----------------------------------------------------------------------|----------------------------------------------------|
| [Availability](./08-availability.md)                                 | SLOs, uptime targets, failover patterns            |
| [Reliability](./09-reliability.md)                                   | Correctness over time, failure handling patterns   |
| [Scalability](./10-scalability.md)                                   | Scale-up/out, bottlenecks, scaling patterns        |
| [Caching](./11-caching.md)                                           | Cache levels and strategies, invalidation patterns |
| [Proxies](./12-proxies.md)                                           | Forward/reverse proxies, CDNs, load balancers      |
| [CDN](./32-cdn.md)                                                   | Edge caches, cache keys, origin protection, routing |
| [Load Balancing](./13-load-balancing.md)                             | Load balancing algorithms, health checks, failover |
| [Resilience](./14-resilience.md)                                     | Failure isolation, degradation, recovery           |
| [Observability](./15-observability.md)                               | Metrics, logs, traces, alerts, telemetry design    |
| [Concurrency Control](./24-concurrency-control.md)                   | Threads, locks, semaphores, deadlocks              |

### Data & Storage

Database concepts and data management strategies.

| Topic                                                                    | Key Concepts                                                 |
|--------------------------------------------------------------------------|--------------------------------------------------------------|
| [Hashing](./16-hashing.md)                                               | Consistent hashing, partitioning strategies                  |
| [Bloom Filters](./17-bloom-filters.md)                                   | Membership checks, false positives                           |
| [Checksums](./18-checksums.md)                                           | Data integrity, hash verification, error detection           |
| [Relational Databases](./19-relational-databases.md)                     | ACID properties, SQL optimization, indexing                  |
| [Non-Relational Databases](./20-non-relational-databases.md)             | NoSQL types, CAP theorem trade-offs                          |
| [Database Replication](./21-database-replication.md)                     | Master-slave, master-master patterns                         |
| [Database Sharding](./22-database-sharding.md)                           | Horizontal partitioning, shard key selection                 |
| [Database Concurrency Control](./23-database-concurrency-control.md)     | Isolation levels, anomalies, locking strategies              |

### Distributed Systems

Core concepts for coordination and consistency in distributed architectures.

| Topic                                                      | Key Concepts                                   |
|------------------------------------------------------------|------------------------------------------------|
| [CAP and PACELC Theorems](./25-cap-and-pacelc-theorems.md) | Distributed systems constraints                |
| [Leader Election](./26-leader-election.md)                 | Leader-follower pattern, failover, leases      |
| [Consensus](./27-consensus.md)                             | Quorum, Raft vs Paxos, consistency guarantees  |
| [Pub/Sub](./28-pub-sub.md)                                 | Messaging patterns, event-driven architectures |
| [Rate Limiting](./30-rate-limiting.md)                     | Algorithms, distributed limits, client retries     |

### Capacity Planning

Estimation and performance analysis techniques.

| Topic                                                                          | Key Concepts                               |
|--------------------------------------------------------------------------------|--------------------------------------------|
| [Back-of-the-Envelope Calculations](./31-back-of-the-envelope-calculations.md) | Estimation techniques, performance numbers |

---

For deeper dives into authentication, multi-region replication, MapReduce, P2P networks, and Kafka as a distributed-systems case study, see [Advanced Topics](../advanced/README.md).
