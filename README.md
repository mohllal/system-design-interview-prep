# System Design Interview Preparation

A curated collection of system design concepts, patterns, and architectures designed as a comprehensive refresher for technical interviews.

It contains my personal collection of notes and resources I consider important for preparing for system design interviews.

## 🎯 Purpose

This repository serves as:

- **Interview Refresher**: Quick review of essential system design concepts
- **Visual Learning Guide**: Mermaid diagrams and clear explanations for better retention

## 🚀 How to Use This Repository

1. **Start with Fundamentals**: Build strong foundational knowledge
2. **Go Deeper with Advanced Topics**: Authentication, multi-region, and specialized paradigms
3. **Explore Architectures**: Understand patterns and their applications
4. **Practice with Systems**: Apply concepts to real-world design problems
5. **Challenge Ideas**: Question assumptions and understand trade-offs

## 🔧 Fundamentals

Essential building blocks for understanding distributed systems and scalable architectures. See the [full index](./fundamentals/README.md).

### Communication & Networking

- [Client-Server Model](./fundamentals/01-client-server.md) - Request-response patterns, DNS, service roles
- [Network Protocols](./fundamentals/02-network-protocols.md) - TCP/IP stack, browser request flow, protocol trade-offs
- [Latency and Throughput](./fundamentals/03-latency-and-throughput.md) - Performance metrics, optimization strategies
- [HTTP Versions](./fundamentals/04-http-versions.md) - HTTP/1.0 to HTTP/3, QUIC, performance trade-offs
- [REST API](./fundamentals/05-rest-api.md) - Resource modeling, HTTP semantics, versioning
- [Communication Patterns](./fundamentals/06-communication-patterns.md) - REST, RPC, GraphQL, messaging
- [Real-Time Communication](./fundamentals/07-realtime-communication-patterns.md) - Short/long polling, SSE, WebSockets
- [Messaging Patterns](./fundamentals/31-messaging-patterns.md) - Queues, pub/sub, request-reply, delivery semantics

### Reliability & Performance

- [Availability](./fundamentals/08-availability.md) - SLOs, uptime targets, failover patterns
- [Reliability](./fundamentals/09-reliability.md) - Correctness over time, failure handling patterns
- [Scalability](./fundamentals/10-scalability.md) - Scale-up/out, bottlenecks, scaling patterns
- [Caching](./fundamentals/11-caching.md) - Cache strategies, levels, invalidation patterns
- [Proxies](./fundamentals/12-proxies.md) - Forward/reverse proxies, load balancers, CDNs
- [CDN](./fundamentals/34-cdn.md) - Edge caches, origin protection, cache keys, global routing
- [Load Balancing](./fundamentals/13-load-balancing.md) - Distribution algorithms, health checks
- [Resilience](./fundamentals/14-resilience.md) - Failure isolation, graceful degradation, recovery
- [Observability](./fundamentals/15-observability.md) - Metrics, logs, traces, alerts, telemetry design
- [Concurrency Control](./fundamentals/26-concurrency-control.md) - Threads, locks, semaphores, deadlocks

### Data & Storage

- [Hashing](./fundamentals/16-hashing.md) - Consistent hashing, partitioning strategies
- [Bloom Filters](./fundamentals/17-bloom-filters.md) - Membership checks, false positives
- [Checksums](./fundamentals/18-checksums.md) - Data integrity, hash verification, error detection
- [Relational Databases](./fundamentals/19-relational-databases.md) - Tables, keys, ACID/WAL, normalize vs denormalize
- [Non-Relational Databases](./fundamentals/20-non-relational-databases.md) - Access paths, store types, embedding, atomicity scope
- [SQL vs NoSQL](./fundamentals/21-sql-vs-nosql.md) - Access path, transaction scope, hybrids, common myths
- [Database Indexes](./fundamentals/22-database-indexes.md) - B+tree, hash, bitmap, LSM, GIN, composite/covering
- [Database Replication](./fundamentals/23-database-replication.md) - Ack policy, failover/fencing, replica lag, quorums
- [Database Partitioning](./fundamentals/24-database-partitioning.md) - Split vs copy, shard keys, replica sets per shard
- [Database Concurrency Control](./fundamentals/25-database-concurrency-control.md) - Isolation levels, anomalies, locking strategies

### Distributed Systems

- [CAP and PACELC Theorems](./fundamentals/27-cap-and-pacelc-theorems.md) - Distributed systems constraints
- [Leader Election](./fundamentals/28-leader-election.md) - Leader-follower pattern, failover, leases
- [Consensus](./fundamentals/29-consensus.md) - Quorum, Raft vs Paxos, consistency guarantees
- [Pub/Sub](./fundamentals/30-pub-sub.md) - Messaging patterns, event-driven architectures
- [Rate Limiting](./fundamentals/32-rate-limiting.md) - Algorithms, distributed limits, client retries

### Capacity Planning

- [Back-of-the-Envelope Calculations](./fundamentals/33-back-of-the-envelope-calculations.md) - Estimation techniques, performance numbers

## 🔬 Advanced Topics

In-depth notes for specialized or deep-dive interview topics. See the [full index](./advanced/README.md).

- [OAuth 2.0](./advanced/01-oauth2.md) - Authorization flows, PKCE, tokens, scopes
- [Multi-Region Replication](./advanced/02-multi-region-replication.md) - Global deployment, replication topologies
- [MapReduce](./advanced/03-mapreduce.md) - Distributed batch processing paradigm
- [Peer-to-Peer Networks](./advanced/04-peer-to-peer-networks.md) - Decentralized architectures, DHT
- [Kafka Architecture](./advanced/05-kafka-architecture.md) - Distributed log, partitioning, replication, consumers as a case study
- [PostgreSQL Internals](./advanced/06-postgresql-internals.md) - Pages, heap, MVCC, WAL, planner as a database internals case study

## 🏗️ Architecture Patterns

Proven architectural approaches for building scalable and maintainable systems.

### Structural Patterns

- [Layered Architecture](./architecture/01-layered-architecture.md) - Traditional n-tier separation of concerns
- [Hexagonal Architecture](./architecture/02-hexagonal-architecture.md) - Ports and adapters pattern
- [Microservices Architecture](./architecture/03-microservices-architecture.md) - Distributed service-oriented design

### Event-Driven Patterns  

- [Event-Driven Architecture](./architecture/04-event-driven-architecture.md) - Asynchronous communication patterns
- [Event Sourcing](./architecture/09-event-sourcing.md) - State as sequence of events
- [CQRS](./architecture/10-cqrs.md) - Command Query Responsibility Segregation

### Resilience Patterns

- [Circuit Breaker](./architecture/08-circuit-breaker.md) - Fault tolerance and graceful degradation
- [Two-Phase Commit](./architecture/05-two-phase-commit.md) - Distributed transaction coordination
- [Saga Pattern](./architecture/06-saga.md) - Managing distributed transactions
- [Transactional Outbox](./architecture/07-transactional-outbox.md) - Reliable event publishing

## 🏢 System Design Examples

Real-world system design problems with comprehensive solutions.

- [URL Shortening System](./examples/01-url-shortening-system.md) - Design bit.ly or tinyurl.com.
- [OCR File Processing System](./examples/02-ocr-file-processing-system.md) - Design a system to process 100,000 files per hour using OCR to extract text from PDF documents.

*More system designs coming soon...*

## 📚 Additional Learning Resources

Curated external resources for deepening system design knowledge.

### 🎓 Educational Content

**Repositories**

- [System Design Primer](https://github.com/donnemartin/system-design-primer) - Comprehensive collection of system design concepts
- [System Design 101](https://github.com/ByteByteGoHq/system-design-101) - Visual explanations by ByteByteGo
- [Awesome System Design Resources](https://github.com/ashishps1/awesome-system-design-resources) - Curated list of resources

**Video Learning**

- [Hello Interview - System Design Walkthroughs](https://www.youtube.com/playlist?list=PL5q3E8eRUieWtYLmRU3z94-vGRcwKr9tM) - Detailed interview walkthroughs
- [codeKarle - System Design Questions](https://www.youtube.com/playlist?list=PLhgw50vUymycJPN6ZbGTpVKAJ0cL4OEH3) - Interview-focused content

**Online Courses**

- [Grokking the System Design Interview](https://www.designgurus.io/course/grokking-the-system-design-interview) - Interview preparation
- [Educative - Modern System Design](https://www.educative.io/courses/grokking-modern-system-design-interview-for-engineers-managers) - Comprehensive course

### 🏢 Industry Insights

**Engineering Blogs**

- [Netflix Tech Blog](http://techblog.netflix.com/) - Streaming and microservices insights
- [Uber Engineering](http://eng.uber.com/) - Real-time systems and data processing
- [Meta Engineering](https://engineering.fb.com/) - Large-scale social systems
- [AWS Architecture Blog](https://aws.amazon.com/blogs/architecture/) - Cloud-native patterns

**System Design Newsletters**

- [ByteByteGo](https://blog.bytebytego.com/) - Weekly system design insights
- [System Design Newsletter](https://newsletter.systemdesign.one/) - Deep dives into architecture
- [Software Architecture Monday](https://www.developertoarchitect.com/lessons/) - Architecture patterns

### 📄 Foundational Papers

**Distributed Systems**

- [MapReduce](https://static.googleusercontent.com/media/research.google.com/en//archive/mapreduce-osdi04.pdf) - Simplified data processing on large clusters
- [Bigtable](https://static.googleusercontent.com/media/research.google.com/en//archive/bigtable-osdi06.pdf) - Distributed storage system for structured data
- [Dynamo](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf) - Amazon's highly available key-value store

**Transaction Management**

- [Sagas](https://www.cs.cornell.edu/andru/cs711/2002fa/reading/sagas.pdf) - Managing long-lived transactions
- [Life beyond Distributed Transactions](https://ics.uci.edu/~cs223/papers/cidr07p15) - Alternative approaches to ACID

---

## 🤝 Contributing

This repository reflects personal learning and interview preparation. While primarily for personal use, suggestions and improvements are welcome through issues and pull requests.
