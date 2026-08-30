---
title: "Communication patterns"
concepts:
  - rest
  - rpc
  - graphql
  - polling
  - streaming
  - message-queues
  - publish-subscribe
related:
  - fundamentals/05-rest-api.md
  - fundamentals/07-realtime-communication-patterns.md
  - fundamentals/30-pub-sub.md
  - fundamentals/31-messaging-patterns.md
---

# Communication patterns

The goal of this document is not to pick a single "best" pattern, but to choose the right one for latency, scale, and product needs.

## Synchronous request-response patterns

### REST

REST models server capabilities as resources (for example, `/users/123`) and uses standard HTTP semantics.

**Strengths:**

- Simple and widely understood
- Strong caching support with HTTP primitives
- Good interoperability across tools and teams

**Trade-offs:**

- Can be chatty for complex views
- Over-fetching or under-fetching in some use cases

**Best fit:** Public APIs, CRUD-heavy backends, service boundaries where stability matters.

### RPC

RPC models remote operations as function calls (for example, `GetUser`, `CreateOrder`).

**Strengths:**

- Efficient and explicit operation contracts
- Strong typing (especially with gRPC/Protobuf)
- Good for internal low-latency service-to-service calls

**Trade-offs:**

- Tighter coupling between clients and servers
- Weaker native caching model than REST

**Best fit:** Internal microservice communication, performance-sensitive APIs.

### GraphQL

GraphQL lets clients request exactly the fields they need from a typed schema.

**Strengths:**

- Reduces over-fetching for complex UI screens
- Single endpoint can aggregate many backend sources
- Strong schema and introspection support

**Trade-offs:**

- Query cost and complexity must be controlled
- Caching and rate limiting are often more complex than REST

**Best fit:** Client-driven experiences (web/mobile) with diverse data-shape needs.

## Real-time delivery patterns

### Polling

Client asks the server for updates on a fixed interval.

**Strengths:**

- Very simple to implement and operate
- Works through most proxies and network setups

**Trade-offs:**

- Wasted requests when no new data exists
- Update latency depends on polling interval

**Best fit:** Low-frequency updates, simple integrations, admin dashboards.

### Streaming and push

Server pushes updates as events happen (for example, WebSockets, SSE, gRPC streaming).

**Strengths:**

- Low-latency updates
- More efficient than high-frequency polling

**Trade-offs:**

- Connection lifecycle and backpressure are harder to manage
- Operational complexity is higher than polling

**Best fit:** Live dashboards, chat, collaborative editing, market/event feeds.

## Asynchronous messaging patterns

For channel types, delivery semantics, routing, and queue vs stream trade-offs, see [Messaging Patterns](./31-messaging-patterns.md).

### Message queues (point-to-point)

Producer sends work to a queue; one consumer processes each message.

**Strengths:**

- Decouples producer and consumer availability
- Smooths traffic spikes
- Enables retry and dead-letter patterns

**Trade-offs:**

- Increased operational complexity
- Eventual processing instead of immediate response

**Best fit:** Background jobs, order processing, email/notification pipelines.

### Publish-subscribe

Producer publishes an event to a topic; multiple consumers handle it independently.

**Strengths:**

- One event can drive many downstream workflows
- Easy extensibility for new consumers

**Trade-offs:**

- Harder debugging and end-to-end tracing
- Requires idempotent consumers and schema discipline

**Best fit:** Event-driven architectures, analytics fan-out, integrations.

## Pattern selection checklist

When choosing a pattern, answer:

- Do we need immediate response or asynchronous processing?
- Is low latency more important than simplicity?
- Who controls schema evolution: server teams or client teams?
- Do we need fan-out to multiple consumers?
- What failure model do we accept: retries, duplicates, eventual consistency?

## Reference materials

- [What is REST?](https://restfulapi.net/)
- [gRPC Concepts](https://grpc.io/docs/what-is-grpc/core-concepts/)
