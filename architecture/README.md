# Architecture Patterns

This section covers proven architectural patterns and approaches for building scalable, highly available systems.

## 📖 Contents

### 🏛️ Architectural Patterns

These patterns define how to organize system components and their relationships.

| Pattern                                                          | Key Concepts                                                                         |
|------------------------------------------------------------------|--------------------------------------------------------------------------------------|
| [Layered Architecture](./01-layered-architecture.md)             | Separation of concerns, N-tier structure (e.g., presentation, business, persistence) |
| [Hexagonal Architecture](./02-hexagonal-architecture.md)         | Ports and adapters, Application core isolation from external dependencies            |
| [Microservices Architecture](./03-microservices-architecture.md) | Independent deployment, Decentralized data management, Single responsibility         |

### ⚡ Event-Driven Patterns

Patterns for building reactive, loosely coupled systems through event communication.

| Pattern                                                        | Key Concepts                                                                    |
|----------------------------------------------------------------|---------------------------------------------------------------------------------|
| [Event-Driven Architecture](./04-event-driven-architecture.md) | Asynchronous communication, Producers/Consumers, Event channel/broker           |
| [Event Sourcing](./09-event-sourcing.md)                       | Separation of read/write models, Optimized data models for queries and commands |
| [CQRS](./10-cqrs.md)                                           | Separation of read/write models, Optimized data models for queries and commands |

### 💪 Resilience Patterns

Patterns for handling failures and maintaining system reliability in distributed environments.

| Pattern                                              | Key Concepts                                                        |
|------------------------------------------------------|---------------------------------------------------------------------|
| [Circuit Breaker](./08-circuit-breaker.md)           | Service failure protection, State machine (Closed, Open, Half-Open) |
| [Two-Phase Commit](./05-two-phase-commit.md)         | Distributed transaction coordination, Atomic commitment             |
| [Saga](./06-saga.md)                                 | Eventual consistency, Compensating transactions to handle failures  |
| [Transactional Outbox](./07-transactional-outbox.md) | Atomic state change and event publishing, Guaranteed event delivery |

## 📝 Thought Process

1. **Understand the problem domain** - Different patterns solve different problems
2. **Consider trade-offs** - Every pattern has benefits and drawbacks
3. **Start simple** - Begin with simpler patterns and evolve as needed
4. **Justify choices** - Explain why you chose a specific pattern

## 🧵 External Materials

### 📕 Books

- [Patterns of Enterprise Application Architecture](https://martinfowler.com/books/eaa.html)
- [Building Microservices](https://samnewman.io/books/building_microservices/)
- [Software Architecture Patterns](https://www.oreilly.com/library/view/software-architecture-patterns/9781491971437/)
- [Designing Distributed Systems](https://www.oreilly.com/library/view/designing-distributed-systems/9781098156343/)

### 🌐 Online Resources

- [Software Architecture Guide](https://martinfowler.com/architecture/)
- [Microservices.io](https://microservices.io/)
- [Enterprise Integration Patterns](https://www.enterpriseintegrationpatterns.com/)
- [DDD Community](https://github.com/ddd-crew)
