# System Design Interview Preparation

A curated collection of system design concepts, patterns, and architectures designed as a comprehensive refresher for technical interviews.

It contains my personal collection of notes and resources I consider important for preparing for system design interviews.

## 🎯 Purpose

This repository serves as:

- **Interview Refresher**: Quick review of essential system design concepts
- **Visual Learning Guide**: Mermaid diagrams and clear explanations for better retention

## 🚀 How to Use This Repository

1. **Start with Fundamentals**: Build strong foundational knowledge
2. **Explore Architectures**: Understand patterns and their applications  
3. **Practice with Systems**: Apply concepts to real-world design problems
4. **Challenge Ideas**: Question assumptions and understand trade-offs

## 🔧 Fundamentals

Essential building blocks for understanding distributed systems and scalable architectures.

### Communication & Networking

- [Client-Server Model](./fundamentals/01-client-server.md) - Request-response patterns, DNS, REST vs RPC vs GraphQL
- [Network Protocols](./fundamentals/02-network-protocols.md) - TCP/IP stack, browser request flow, protocol trade-offs  
- [Latency and Throughput](./fundamentals/03-latency-and-throughput.md) - Performance metrics, optimization strategies

### Reliability & Performance  

- [Availability and Reliability](./fundamentals/04-availability-and-reliability.md) - Uptime targets, redundancy patterns
- [Caching](./fundamentals/05-caching.md) - Cache strategies, levels, invalidation patterns
- [Proxies](./fundamentals/06-proxies.md) - Forward/reverse proxies, load balancers, CDNs
- [Load Balancing](./fundamentals/07-load-balancing.md) - Distribution algorithms, health checks

### Data & Storage

- [Hashing](./fundamentals/08-hashing.md) - Consistent hashing, partitioning strategies
- [Relational Databases](./fundamentals/09-relational-databases.md) - ACID properties, SQL optimization, indexing
- [Non-Relational Databases](./fundamentals/10-non-relational-databases.md) - NoSQL types, CAP theorem trade-offs
- [Database Replication](./fundamentals/11-database-replication.md) - Master-slave, master-master patterns
- [Database Sharding](./fundamentals/12-database-sharding.md) - Horizontal partitioning strategies

### Distributed Systems Concepts

- [Leader Election](./fundamentals/13-leader-election.md) - Consensus algorithms, coordination patterns
- [Peer-to-Peer Networks](./fundamentals/14-peer-to-peer-networks.md) - Decentralized architectures
- [Rate Limiting](./fundamentals/15-rate-limiting.md) - Traffic control, algorithms, implementation
- [Pub/Sub](./fundamentals/16-pub-sub.md) - Messaging patterns, event-driven architectures
- [MapReduce](./fundamentals/17-mapreduce.md) - Distributed computing paradigm
- [CAP and PACELC Theorems](./fundamentals/18-cap-and-pacelc-theorems.md) - Distributed systems constraints

### Capacity Planning

- [Back-of-the-Envelope Calculations](./fundamentals/19-back-of-the-envelope-calculations.md) - Estimation techniques, performance numbers

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
