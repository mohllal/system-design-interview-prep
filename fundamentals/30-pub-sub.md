---
title: "Pub/Sub (Publish/Subscribe)"
concepts:
  - publish-subscribe-model
  - delivery-semantics
  - idempotency
  - message-ordering
  - backpressure
  - schema-evolution
  - dead-letter-queues
related:
  - fundamentals/31-messaging-patterns.md
  - fundamentals/06-communication-patterns.md
  - fundamentals/07-realtime-communication-patterns.md
  - fundamentals/09-reliability.md
  - advanced/05-kafka-architecture.md
---

# Pub/Sub (Publish/Subscribe)

Pub/Sub decouples message producers (publishers) from consumers (subscribers) through a broker/topic layer.

Publishers send events to topics without knowing the subscribers. Subscribers receive only the topics they care about.

For queues, request-reply, routing, and reliability patterns, see [Messaging Patterns](./31-messaging-patterns.md).

```mermaid
graph TD
    P1[Publisher] --> T[Topic: orders]
    P2[Publisher] --> T
    T --> S1[Order Service]
    T --> S2[Analytics]
    T --> S3[Notification Service]
```

## Why use pub/sub

- Loose coupling between services
- Easy fan-out to multiple consumers
- Asynchronous processing and better burst handling
- Simple extension (add a new subscriber without changing the publisher)

## Core components

- **Publisher**: Sends messages/events to a topic
- **Topic**: Named channel for routing
- **Subscriber**: Consumes messages from subscribed topics
- **Broker**: Routes, buffers, and delivers messages (Kafka, SNS/SQS, RabbitMQ exchanges, etc.)

## Typical use cases

- Event-driven microservices (order created, payment completed)
- Real-time analytics and audit pipelines
- Notifications and activity feeds
- Data replication/sync fan-out

## Delivery semantics

### At-most-once

- Message may be lost, no duplicates
- Fastest, least reliable
- Use for metrics/non-critical telemetry

### At-least-once

- Message delivered one or more times
- Requires ack + retry
- Most common in production systems
- Consumers must be idempotent

### Exactly-once

- Delivered once with no loss/duplication
- Hard and expensive end-to-end
- Usually achieved by combining at-least-once + idempotency + dedup keys

## Idempotency (critical)

Because duplicates happen, handlers should be safe to rerun.

Common patterns:

- Idempotency keys per business action
- Conditional updates (`WHERE version = ...`)
- State checks before side effects

For example, a payment event `pay_123` processed twice should result in only one charge.

## Pub/Sub vs message queue

- **Pub/Sub**: One event, many subscribers (broadcast/fan-out)
- **Queue**: One message, one consumer (work distribution)

Use pub/sub when multiple systems must react to the same event.
Use queues when tasks should be processed once by one worker.

## Common challenges

- **Ordering**: Hard to guarantee global order across partitions
- **Backpressure**: Slow consumers can lag or overload the broker
- **Debugging**: Asynchronous flows need strong tracing/logging
- **Schema evolution**: Event contracts must be managed carefully

## Design guidelines

- Define clear event schemas and versioning
- Include correlation IDs for traceability
- Set retention and replay policies intentionally
- Monitor consumer lag and dead-letter queues
- Design consumers for retries and poison-message handling

## Interview talking points

- Pub/sub is for decoupling and fan-out, not point-to-point task routing.
- State delivery semantics explicitly (usually at-least-once + idempotency).
- Mention ordering, lag, replay, and failure handling.
- Distinguish pub/sub from simple job queues.

## Reference materials

- [Enterprise Integration Patterns - Publish-Subscribe Channel](https://www.enterpriseintegrationpatterns.com/patterns/messaging/PublishSubscribeChannel.html)
