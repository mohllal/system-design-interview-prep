# Architecture patterns

This section covers proven architectural patterns and approaches for building scalable, highly available systems.

## Contents

### Architectural patterns

These patterns define how to organize system components and their relationships.

| Pattern                                                                    | Key concepts                                                                                 |
| -------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| [Layered Architecture](./01-layered-architecture.md)                       | Separation of concerns, dependency direction and inversion, anemic domain model              |
| [Hexagonal Architecture](./02-hexagonal-architecture.md)                   | Ports and adapters, application core isolation, testability without infrastructure           |
| [Microservices Architecture](./03-microservices-architecture.md)           | Business capability alignment, decentralized data management, service granularity            |
| [Service Infrastructure Patterns](./11-service-infrastructure-patterns.md) | API gateway, service discovery, service mesh and sidecar proxies, config/secret distribution |
| [Backend for Frontend](./13-backend-for-frontend.md)                       | One gateway per client type, ownership boundaries, shared edge plus per-client BFFs          |

### Event-driven patterns

Patterns for building reactive, loosely coupled systems through event communication.

| Pattern                                                        | Key concepts                                                                          |
| -------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| [Event-Driven Architecture](./04-event-driven-architecture.md) | Event vs command, event notification vs state transfer, choreography vs orchestration |
| [Event Sourcing](./09-event-sourcing.md)                       | Event store and streams, aggregates, projections, snapshotting, schema evolution      |
| [CQRS](./10-cqrs.md)                                           | Command query separation, write model vs read model, eventual consistency             |

### Distributed transaction and resilience patterns

Patterns for keeping services consistent and available when a single local transaction is not an option.

| Pattern                                              | Key concepts                                                                      |
| ---------------------------------------------------- | --------------------------------------------------------------------------------- |
| [Two-Phase Commit](./05-two-phase-commit.md)         | Prepare and commit phases, coordinator and participants, in-doubt transactions    |
| [Saga](./06-saga.md)                                 | Compensating transactions, choreography vs orchestration, eventual consistency    |
| [Transactional Outbox](./07-transactional-outbox.md) | Dual-write problem, outbox table, relay/polling publisher, change data capture    |
| [Circuit Breaker](./08-circuit-breaker.md)           | Closed/open/half-open state machine, failure-rate thresholds, fail-fast           |
| [Change Data Capture](./12-change-data-capture.md)   | Log tailing, snapshot plus stream bootstrapping, schema evolution, slot retention |

---

**Tip**: Each architectural pattern includes a "Reference materials" section with curated external resources for deeper exploration.
