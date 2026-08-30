---
title: "REST API"
concepts:
  - rest-principles
  - resource-naming
  - http-methods
  - idempotency
  - http-status-codes
  - cursor-pagination
  - api-versioning
  - http-caching
  - optimistic-concurrency-control
related:
  - fundamentals/04-http-versions.md
  - fundamentals/06-communication-patterns.md
  - fundamentals/11-caching.md
  - fundamentals/26-concurrency-control.md
  - fundamentals/32-rate-limiting.md
---

# REST API

REST (Representational State Transfer) is an architectural style that defines how clients interact with server resources using HTTP semantics in a consistent, predictable way.

## REST principles

- **Uniform interface**: Each resource is identified by a URL, and HTTP methods (GET, POST, PUT, PATCH, DELETE) are used to manipulate it. This makes the interface consistent and predictable.
- **Stateless**: Each request contains all necessary information, and the server stores no state between requests. This simplifies server design and improves scalability and reliability, since any server can handle any request.
- **Cacheable**: Resources should be explicitly or implicitly marked as cacheable or non-cacheable. Both the server and the client can cache the response for a given request.
- **Client-server**: The client and server are separate entities with distinct responsibilities. This separation allows for independent development and evolution of both components.

## Resource and URL naming

Use stable, plural, noun-based paths:

- Collections: `/users`, `/orders`
- Single resource: `/users/{userId}`
- Sub-resource: `/users/{userId}/orders`

Avoid:

- Verb paths: `/getUser`, `/createOrder`
- Inconsistent naming (`/user` vs `/users`)
- Deep nesting beyond 2-3 levels

## HTTP methods and idempotency

| Method   | Purpose                                | Safe | Idempotent            |
| -------- | -------------------------------------- | ---- | --------------------- |
| `GET`    | Read a resource                        | Yes  | Yes                   |
| `POST`   | Create a resource or trigger an action | No   | No                    |
| `PUT`    | Replace the full resource              | No   | Yes                   |
| `PATCH`  | Partially update a resource            | No   | Can be, by contract   |
| `DELETE` | Remove a resource                      | No   | Yes (in API behavior) |

Idempotency matters for retries. For create/payment operations, use idempotency keys to prevent duplicate effects.

## Status codes

Use clear status codes:

- `200`/`201`/`204` for success
- `400` for validation errors
- `401`/`403` for authentication/authorization failures
- `404` for missing resources
- `409` for conflicts
- `429` for rate limiting
- `5xx` for server-side failures

## Filtering, sorting, and pagination

Use query parameters for collection querying:

- **Filtering**: `/orders?status=paid&customerId=123`
- **Sorting**: `/orders?sort=-createdAt`
- **Pagination**: Cursor-based preferred at scale

Cursor pagination is more stable than offset pagination for changing datasets.

## Versioning and compatibility

Prefer clear major versions when introducing breaking changes:

- **Path versioning**: `/v1/orders`

Deprecation guidance:

- Announce deprecation window early
- Return `Deprecation` and `Sunset` headers where appropriate
- Provide migration docs and replacement endpoints

## Caching control

Use caching headers on read endpoints to reduce latency and backend load.

- `Cache-Control`: Defines freshness (for example, `max-age=60`, `public`, `private`, `no-store`)
- `ETag`: Version fingerprint of a representation
- `Last-Modified`: Timestamp-based validator when `ETag` is not used

How it is used:

1. Client requests `GET /users/123`
2. Server responds with `ETag: "u123-v7"` and cache policy
3. Client later sends `If-None-Match: "u123-v7"`
4. If unchanged, server returns `304 Not Modified` (no body)

Result: lower bandwidth usage and faster repeated reads.

## Concurrency control

Use optimistic concurrency to prevent lost updates when multiple clients edit the same resource.

- Client reads the resource and gets the current `ETag`
- Client sends the update with `If-Match: <ETag>`
- Server updates only if the ETag still matches the latest version
- If not matched, server returns `412 Precondition Failed`

How it is used:

1. Client A and client B read `/orders/99` with `ETag: "v10"`
2. A updates first, so the resource becomes `v11`
3. B sends an update with a stale `If-Match: "v10"`
4. Server rejects with `412`, forcing B to re-read and retry safely

Result: prevents accidental overwrite of newer data without using locks.

## Interview talking points

- REST quality comes from consistent resource modeling and HTTP correctness.
- Idempotency and error consistency are critical for production reliability.
- Cursor pagination, ETags, and clear status codes are high-signal design choices.

## Reference materials

- [RFC 9110 - HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [RFC 9111 - HTTP Caching](https://www.rfc-editor.org/rfc/rfc9111)
- [RESTful API Design Best Practices](https://restfulapi.net/)
