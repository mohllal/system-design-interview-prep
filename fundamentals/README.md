# System design fundamentals

Essential building blocks for understanding distributed systems and scalable architectures.

## Contents

### Communication & networking

Core concepts for how systems communicate and exchange data.

| Topic                                                              | Key Concepts                                                                                     |
| ------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------ |
| [Client-Server Model](./01-client-server.md)                       | Client-server roles, stateless vs stateful servers, sticky sessions, three-tier architecture     |
| [Network Protocols](./02-network-protocols.md)                     | TCP/IP stack, TCP vs UDP, DNS resolution, TCP/TLS handshakes, request lifecycle                  |
| [Latency and Throughput](./03-latency-and-throughput.md)           | Latency components, round-trip time, percentiles and tail latency, bandwidth-delay product       |
| [HTTP Versions](./04-http-versions.md)                             | HTTP/1.1 to HTTP/3, multiplexing, header compression, head-of-line blocking, QUIC                |
| [REST API](./05-rest-api.md)                                       | REST constraints, resource naming, HTTP methods and idempotency, pagination, versioning, caching |
| [Communication Patterns](./06-communication-patterns.md)           | REST, RPC, GraphQL, webhooks, server push, sync vs async messaging                               |
| [Real-Time Communication](./07-realtime-communication-patterns.md) | Short/long polling, SSE, WebSockets, heartbeats, reconnect backoff, connection fan-out           |
| [Messaging Patterns](./31-messaging-patterns.md)                   | Queues, pub/sub, delivery semantics, dead-letter queues, transactional outbox                    |

### Reliability & performance

Strategies for building resilient and performant systems.

| Topic                                              | Key Concepts                                                                              |
| -------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| [Availability](./08-availability.md)               | SLIs/SLOs/SLAs, availability composition, redundancy and failover, RTO/RPO, error budgets |
| [Reliability](./09-reliability.md)                 | Correctness over time, MTTF/MTTR/MTBF, idempotency and deduplication, outbox pattern      |
| [Scalability](./10-scalability.md)                 | Vertical vs horizontal scaling, statelessness, data partitioning, hot keys                |
| [Caching](./11-caching.md)                         | Cache-aside, read/write strategies, invalidation and eviction, cache failure modes        |
| [Proxies](./12-proxies.md)                         | Forward vs reverse proxy, sidecar proxies, TLS termination, request routing               |
| [CDN](./34-cdn.md)                                 | Edge caching, push vs pull, cache-control directives, origin shielding, anycast routing   |
| [Load Balancing](./13-load-balancing.md)           | Load-balancing algorithms, layer 4 vs layer 7, health checks, session affinity            |
| [Resilience](./14-resilience.md)                   | Cascading failures, timeouts and retries, circuit breakers, bulkheads, chaos engineering  |
| [Observability](./15-observability.md)             | Metrics, logs, distributed tracing, sampling, RED/USE methods, alerting                   |
| [Concurrency Control](./26-concurrency-control.md) | Mutual exclusion, mutexes/semaphores, atomic operations, race conditions, deadlocks       |

### Data & storage

Database concepts and data management strategies.

| Topic                                                                | Key Concepts                                                                        |
| -------------------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| [Hashing](./16-hashing.md)                                           | Cryptographic vs non-cryptographic hashing, consistent hashing, virtual nodes       |
| [Bloom Filters](./17-bloom-filters.md)                               | Membership testing, false-positive rate, sizing, cache-penetration protection       |
| [Checksums](./18-checksums.md)                                       | CRC and error detection, data-integrity verification, end-to-end integrity          |
| [Relational Databases](./19-relational-databases.md)                 | Tables/keys/joins, ACID transactions, write-ahead logging, normalize vs denormalize |
| [Non-Relational Databases](./20-non-relational-databases.md)         | Document/key-value/wide-column/graph stores, BASE vs ACID, secondary indexes        |
| [SQL vs NoSQL](./21-sql-vs-nosql.md)                                 | Decision framework, transaction scope, access patterns, polyglot persistence        |
| [Database Indexes](./22-database-indexes.md)                         | B-tree, clustered vs secondary, composite/covering indexes, hash/bitmap/LSM         |
| [Database Replication](./23-database-replication.md)                 | Sync vs async replication, quorum reads/writes, failover, replica lag, CDC          |
| [Database Partitioning](./24-database-partitioning.md)               | Partition-key design, hash/range/directory partitioning, resharding                 |
| [Database Concurrency Control](./25-database-concurrency-control.md) | Isolation levels, concurrency anomalies, MVCC, locking strategies                   |

### Distributed systems

Core concepts for coordination and consistency in distributed architectures.

| Topic                                                      | Key Concepts                                                                         |
| ---------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| [CAP and PACELC Theorems](./27-cap-and-pacelc-theorems.md) | CAP theorem, PACELC, CP vs AP trade-offs, tunable/eventual consistency               |
| [Leader Election](./28-leader-election.md)                 | Leader-follower pattern, majority-vote vs lease-based election, split-brain, fencing |
| [Consensus](./29-consensus.md)                             | Quorums, Paxos vs Raft, log replication, state-machine replication                   |
| [Pub/Sub](./30-pub-sub.md)                                 | Topics and subscriptions, fan-out, push vs pull delivery, retention and replay       |
| [Rate Limiting](./32-rate-limiting.md)                     | Token bucket, leaky bucket, fixed/sliding window algorithms, distributed limits      |

### Capacity planning

Estimation and performance analysis techniques.

| Topic                                                                          | Key Concepts                                                                           |
| ------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------- |
| [Back-of-the-Envelope Calculations](./33-back-of-the-envelope-calculations.md) | Order-of-magnitude estimation, traffic/storage/bandwidth estimation, capacity planning |

---

**Tip**: Each topic includes a reference materials section with curated external resources for deeper exploration.
