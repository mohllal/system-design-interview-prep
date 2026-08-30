---
title: "Communication patterns"
concepts:
  - request-response
  - rpc
  - graphql
  - webhooks
  - server-push
  - asynchronous-messaging
  - synchronous-vs-asynchronous
related:
  - fundamentals/05-rest-api.md
  - fundamentals/07-realtime-communication-patterns.md
  - fundamentals/30-pub-sub.md
  - fundamentals/31-messaging-patterns.md
  - fundamentals/14-resilience.md
---

# Communication patterns

Every service boundary needs an answer to two questions: who starts the conversation, and does the caller wait for the result. The answers pick the pattern; latency, coupling, and failure behavior follow from there.

This document is the survey: what each pattern is, when it fits, and where to read more. The deep dives live in [REST API](./05-rest-api.md), [Real-Time Communication Patterns](./07-realtime-communication-patterns.md), and [Messaging Patterns](./31-messaging-patterns.md).

## The two axes

- **Initiator**: the client asks (pull), or the server delivers on its own (push).
- **Temporal coupling**: the caller blocks on the reply (synchronous), or hands the work off and moves on (asynchronous).

Synchronous coupling is the one that hurts at scale: if A calls B synchronously, A's availability is capped by B's, and B's latency becomes A's latency. Asynchronous patterns break that link at the cost of eventual, not immediate, results.

| Pattern           | Initiator | Caller waits        | Typical transport       | Deep dive                                            |
| ----------------- | --------- | ------------------- | ----------------------- | ---------------------------------------------------- |
| REST              | Client    | Yes                 | HTTP/JSON               | [REST API](./05-rest-api.md)                         |
| RPC               | Client    | Yes (or streaming)  | gRPC over HTTP/2        | This document                                        |
| GraphQL           | Client    | Yes                 | HTTP/JSON, one endpoint | This document                                        |
| Polling           | Client    | Per request         | HTTP                    | [Real-Time](./07-realtime-communication-patterns.md) |
| Server push       | Server    | No, connection open | SSE, WebSockets, gRPC   | [Real-Time](./07-realtime-communication-patterns.md) |
| Webhook           | Server    | No                  | HTTP callback           | This document                                        |
| Queue             | Producer  | No                  | Broker (SQS, RabbitMQ)  | [Messaging](./31-messaging-patterns.md)              |
| Publish-subscribe | Producer  | No                  | Broker (Kafka, SNS)     | [Pub/Sub](./30-pub-sub.md)                           |

## Synchronous request-response

The caller sends a request and blocks until a response or a timeout. Simple to reason about, and the right default for reads and for anything the user is waiting on. Every synchronous call needs a timeout, a retry policy, and a fallback; see [Resilience](./14-resilience.md).

### REST

REST models server capabilities as resources (`/users/123`) and manipulates them with standard HTTP methods and status codes.

**Pros:**

- Simple, universally understood, debuggable with a browser or `curl`
- HTTP caching, proxies, and CDNs work out of the box
- Loose coupling: clients tolerate additive changes without regeneration

**Cons:**

- Chatty for composite views; a screen may need several round trips
- Over-fetching and under-fetching, since the response shape is fixed by the server
- Text-based encoding costs bandwidth and CPU relative to binary formats

**Best fit:** Public APIs, CRUD-heavy backends, service boundaries where contract stability matters.

For resource modeling, idempotency keys, status codes, pagination, and versioning, see [REST API](./05-rest-api.md).

### RPC

RPC models remote operations as function calls (`GetUser`, `CreateOrder`). gRPC is the common modern implementation: an interface definition file generates client and server stubs, and messages travel as Protobuf over HTTP/2.

**Pros:**

- Compact binary encoding and multiplexed HTTP/2 connections, so lower latency and less bandwidth than JSON over HTTP/1.1
- Generated stubs give a strongly typed contract and remove hand-written client code
- Supports client, server, and bidirectional streaming as first-class call types

**Cons:**

- Tighter coupling: a schema change usually means regenerating and redeploying clients
- No usable HTTP caching layer, because everything is a `POST` to an opaque path
- Harder to debug by hand, and browsers need a proxy layer such as gRPC-Web

**Best fit:** Internal service-to-service calls, latency-sensitive paths, polyglot backends that want one generated contract.

### GraphQL

GraphQL exposes a typed schema behind a single endpoint. The client sends a query describing exactly the fields it wants, and gets back a response with that shape.

**Pros:**

- One round trip for a screen that would need several REST calls
- No over-fetching: clients select fields, so mobile and web can share one backend
- Schema and introspection give strong tooling and a clear contract

**Cons:**

- HTTP caching does not apply directly, so caching moves into the server or a persisted-query layer
- A single expensive query can be a denial-of-service vector, so query depth and cost limits are mandatory
- Naive resolvers cause N+1 database queries; batching (a dataloader) is effectively required
- Rate limiting by request count is meaningless; you have to price queries by cost

**Best fit:** Client-driven experiences with diverse data-shape needs, or an aggregation layer over many backend services.

### Choosing between them

| Question                  | REST            | RPC (gRPC)         | GraphQL                    |
| ------------------------- | --------------- | ------------------ | -------------------------- |
| Who is the caller?        | Anyone          | Internal services  | First-party clients        |
| Response shape decided by | Server          | Server             | Client                     |
| HTTP caching              | Native          | Effectively none   | Needs extra machinery      |
| Payload encoding          | JSON            | Protobuf (binary)  | JSON                       |
| Cost of a breaking change | Low if additive | Regenerate clients | Deprecate fields in schema |

## Server-initiated delivery

When the server has news the client did not ask for, something has to bridge the gap: the client asks repeatedly, or a channel stays open, or the server calls the client back.

### Polling

The client re-requests on an interval (short polling), or the server holds the request open until data arrives (long polling).

**Pros:**

- Trivial to implement, and works through any proxy or firewall that allows HTTP
- No connection state to manage on the server

**Cons:**

- Wasted requests when nothing changed; update latency is bounded by the interval
- Cost grows with clients multiplied by frequency, whether or not there is data

**Best fit:** Low-frequency updates, admin dashboards, integrations where seconds of delay are fine.

### Server push

The server holds a connection open and writes updates as they happen: Server-Sent Events for one-way streams, WebSockets for full-duplex, gRPC streaming for internal services.

**Pros:**

- Latency measured in milliseconds, with no polling overhead
- Efficient at high update frequency; one connection carries many messages

**Cons:**

- Connection lifecycle (reconnect, heartbeat, backpressure) becomes your problem
- Stateful connections complicate load balancing, deploys, and horizontal scaling

**Best fit:** Chat, collaborative editing, live dashboards, market and event feeds.

Polling and push are compared in detail, with scaling and operational guidance, in [Real-Time Communication Patterns](./07-realtime-communication-patterns.md).

### Webhooks

A server-to-server callback: the receiver registers a URL, and the sender makes an HTTP request to it when an event occurs. This is how most third-party integrations deliver events (payment succeeded, build finished, repository pushed).

**Pros:**

- Push semantics with no persistent connection and no new protocol
- The receiver only runs an ordinary HTTP endpoint

**Cons:**

- The receiver must be publicly reachable and highly available, which rules out browsers and most mobile clients
- The sender owns retries, so receivers get duplicates and out-of-order deliveries and must be idempotent
- Payloads need signing (an HMAC header) so the receiver can verify the sender

**Best fit:** Third-party event delivery across organizational boundaries.

A webhook is essentially a single-subscriber pub/sub delivery over HTTP. When both sides are yours, an internal broker is usually a better fit than a callback URL.

## Asynchronous messaging

A broker sits between producer and consumer. The producer writes a message and returns immediately; the consumer processes it later. This decouples availability (the consumer can be down), latency (a slow consumer does not slow the producer), and scale (the queue absorbs bursts).

### Work queues

Producer sends a unit of work to a queue; exactly one consumer processes each message.

**Pros:**

- Absorbs traffic spikes instead of dropping or blocking work
- Scale throughput by adding competing consumers
- Retries, delays, and dead-letter queues come with the model

**Cons:**

- The caller gets no immediate result, so the API contract has to change (often to `202 Accepted` plus a status endpoint)
- One more stateful system to operate and monitor

**Best fit:** Background jobs, order processing, email and notification pipelines, media transcoding.

### Publish-subscribe

Producer publishes an event to a topic; every interested subscriber gets its own copy and processes it independently.

**Pros:**

- One event drives many workflows, and new consumers are added without touching the producer
- Natural fit for event-driven architectures and analytics fan-out

**Cons:**

- End-to-end tracing is harder, since no single call stack spans the flow
- Requires idempotent consumers and disciplined event schema versioning

**Best fit:** Event-driven microservices, analytics fan-out, integrations.

For channel types, delivery semantics, routing, reliability patterns, and the queue versus stream trade-off, see [Messaging Patterns](./31-messaging-patterns.md). For fan-out mechanics specifically, see [Pub/Sub](./30-pub-sub.md).

## Patterns combine

Real systems use several patterns at once, and the interesting design work is at the seams. A checkout flow might look like this:

1. The browser `POST`s the order over REST and gets `202 Accepted` back
2. The order service publishes `order.placed` to a topic
3. Payment, inventory, and email services consume it independently
4. The order service calls the fraud service over gRPC because it needs a synchronous answer
5. The browser watches an SSE stream for the order status to flip to confirmed
6. The payment provider notifies the platform of settlement through a webhook

The rule of thumb: synchronous where the user is waiting on the answer, asynchronous everywhere the work can finish later.

## Pattern selection checklist

- Does the caller need the result to continue, or can the work finish later?
- Who initiates: does the client know when to ask, or does only the server know when there is news?
- Is the consumer one worker, or many independent systems that each need a copy?
- Who controls schema evolution: the server team, or the client teams?
- Is the boundary internal (optimize for latency and typing) or public (optimize for stability and reach)?
- What failure model is acceptable: retries, duplicates, reordering, eventual consistency?

## Interview talking points

- Lead with the two axes (who initiates, does the caller wait) rather than listing technologies.
- Synchronous chains multiply failure: each added hop lowers availability and adds latency to the critical path.
- REST versus gRPC is mostly public-versus-internal; GraphQL is a client-ergonomics choice that moves caching and rate limiting into the server.
- Any asynchronous pattern implies at-least-once delivery, which implies idempotent consumers. Say it before you are asked.
- Most designs are a mix; be explicit about which boundary uses which pattern and why.

## Reference materials

- [gRPC core concepts](https://grpc.io/docs/what-is-grpc/core-concepts/)
- [GraphQL - Best practices](https://graphql.org/learn/best-practices/)
- [Enterprise Integration Patterns](https://www.enterpriseintegrationpatterns.com/patterns/messaging/index.html)
