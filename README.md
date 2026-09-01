# System design interview preparation

A curated collection of system design concepts and architecture patterns built as a personal refresher for technical interviews.

## Contents

| Section                                  | What it covers                                                                                                                  |
| ---------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| [Fundamentals](./fundamentals/README.md) | Core building blocks: networking, reliability, caching, databases, distributed systems basics, capacity planning                |
| [Advanced](./advanced/README.md)         | Deep dives and case studies: OAuth 2.0, multi-region replication, MapReduce, peer-to-peer networks, Kafka, PostgreSQL internals |
| [Patterns](./patterns/README.md)         | Proven architectural patterns: structural, event-driven, distributed transaction, and resilience patterns                       |
| [Examples](./examples/README.md)         | Worked system design problems, applied end to end                                                                               |

## Additional resources

### Books

**Software architecture**

- [Patterns of Enterprise Application Architecture](https://martinfowler.com/books/eaa.html) — Martin Fowler
- [Building Microservices](https://samnewman.io/books/building_microservices/) — Sam Newman
- [Software Architecture Patterns](https://www.oreilly.com/library/view/software-architecture-patterns/9781491971437/) — Mark Richards
- [Designing Distributed Systems](https://www.oreilly.com/library/view/designing-distributed-systems/9781098156343/) — Brendan Burns

**Distributed systems**

- [Designing Data-Intensive Applications](https://dataintensive.net/) — Martin Kleppmann. The single best-regarded book for this whole repository's subject matter .. read this one first.
- [Understanding Distributed Systems](https://understandingdistributed.systems/) — Roberto Vitillo. A more concise, interview-oriented complement to Kleppmann's book.
- [Database Internals](https://www.oreilly.com/library/view/database-internals/9781492040330/) — Alex Petrov. Pairs directly with this repo's database-indexing and storage-engine notes.
- [Distributed Systems: Principles and Paradigms](https://www.distributed-systems.net/index.php/books/ds3/) — Andrew S. Tanenbaum, Maarten van Steen. The classic academic textbook treatment.
- [Patterns of Distributed Systems](https://www.oreilly.com/library/view/patterns-of-distributed/9780138222246/) - Unmesh Joshi. A great catalog of patterns used in distributed systems
- [Release It!](https://pragprog.com/titles/mnee2/release-it-second-edition/) — Michael Nygard. The origin of the circuit breaker and bulkhead patterns.
- [Site Reliability Engineering](https://sre.google/sre-book/table-of-contents/) — Google (free online).

### Educational content

**Repositories**

- [System Design Primer](https://github.com/donnemartin/system-design-primer)
- [System Design 101](https://github.com/ByteByteGoHq/system-design-101)
- [Awesome System Design Resources](https://github.com/ashishps1/awesome-system-design-resources)

**Video learning**

- [Hello Interview - System Design Walkthroughs](https://www.youtube.com/playlist?list=PL5q3E8eRUieWtYLmRU3z94-vGRcwKr9tM)

## Foundational papers

**Distributed systems**

- [MapReduce](https://static.googleusercontent.com/media/research.google.com/en//archive/mapreduce-osdi04.pdf) — Simplified data processing on large clusters
- [Bigtable](https://static.googleusercontent.com/media/research.google.com/en//archive/bigtable-osdi06.pdf) — Distributed storage system for structured data
- [Dynamo](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf) — Amazon's highly available key-value store
- [Kafka: a Distributed Messaging System for Log Processing](https://www.microsoft.com/en-us/research/wp-content/uploads/2017/09/Kafka.pdf) — The log-as-broker design
- [F1: A Distributed SQL Database That Scales](https://www.cs.princeton.edu/courses/archive/spring16/cos598F/f1-google.pdf) — Google's globally-distributed relational database, built on Spanner

**Transaction management**

- [Sagas](https://www.cs.cornell.edu/andru/cs711/2002fa/reading/sagas.pdf) — Managing long-lived transactions
- [Life beyond Distributed Transactions](https://ics.uci.edu/~cs223/papers/cidr07p15) — Alternative approaches to ACID

---

## Contributing

This repository reflects personal learning and interview preparation. It is primarily for personal use, but suggestions and improvements are welcome through issues and pull requests.
