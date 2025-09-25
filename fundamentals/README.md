# System Design Fundamentals

Essential building blocks for understanding distributed systems and scalable architectures.

## 📖 Contents

### 🌐 Communication & Networking

Core concepts for how systems communicate and exchange data.

| Topic                                                    | Key Concepts                                                      |
|----------------------------------------------------------|-------------------------------------------------------------------|
| [Client-Server Model](./01-client-server.md)             | Request-response patterns, DNS resolution, REST vs RPC vs GraphQL |
| [Network Protocols](./02-network-protocols.md)           | TCP/IP stack, browser request flow, protocol trade-offs           |
| [Latency and Throughput](./03-latency-and-throughput.md) | Performance metrics, optimization strategies, bottleneck analysis |

### 🛡️ Reliability & Performance

Strategies for building resilient and performant systems.

| Topic                                                                | Key Concepts                                     |
|----------------------------------------------------------------------|--------------------------------------------------|
| [Availability and Reliability](./04-availability-and-reliability.md) | Uptime targets, redundancy patterns              |
| [Caching](./05-caching.md)                                           | Cache levels, strategies, invalidation patterns  |
| [Proxies](./06-proxies.md)                                           | Forward/reverse proxies, CDNs, load balancers    |
| [Load Balancing](./07-load-balancing.md)                             | Distribution algorithms, health checks, failover |

### 💾 Data & Storage

Database concepts and data management strategies.

| Topic                                                        | Key Concepts                                 |
|--------------------------------------------------------------|----------------------------------------------|
| [Hashing](./08-hashing.md)                                   | Consistent hashing, partitioning strategies  |
| [Relational Databases](./09-relational-databases.md)         | ACID properties, SQL optimization, indexing  |
| [Non-Relational Databases](./10-non-relational-databases.md) | NoSQL types, CAP theorem trade-offs          |
| [Database Replication](./11-database-replication.md)         | Master-slave, master-master patterns         |
| [Database Sharding](./12-database-sharding.md)               | Horizontal partitioning, shard key selection |

### 🔗 Distributed Systems

Advanced concepts for distributed and decentralized architectures.

| Topic                                                      | Key Concepts                                   |
|------------------------------------------------------------|------------------------------------------------|
| [Leader Election](./13-leader-election.md)                 | Consensus algorithms, coordination patterns    |
| [Peer-to-Peer Networks](./14-peer-to-peer-networks.md)     | Decentralized architectures, DHT               |
| [Rate Limiting](./15-rate-limiting.md)                     | Traffic control, algorithms, implementation    |
| [Pub/Sub](./16-pub-sub.md)                                 | Messaging patterns, event-driven architectures |
| [MapReduce](./17-mapreduce.md)                             | Distributed computing paradigm                 |
| [CAP and PACELC Theorems](./18-cap-and-pacelc-theorems.md) | Distributed systems constraints                |

### 📊 Capacity Planning

Estimation and performance analysis techniques.

| Topic                                                                          | Key Concepts                               |
|--------------------------------------------------------------------------------|--------------------------------------------|
| [Back-of-the-Envelope Calculations](./19-back-of-the-envelope-calculations.md) | Estimation techniques, performance numbers |

---

💡 **Tip**: Each topic includes "Reference Materials" sections with curated external resources for deeper exploration.
