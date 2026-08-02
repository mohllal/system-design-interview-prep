# REST API

REST API defines how clients interact with server resources using HTTP semantics in a consistent, predictable way.

## REST Principles

- **Uniform Interface:** Each resource is identified by a URL, and the four HTTP methods (GET, POST, PUT, DELETE) are used to manipulate the resource. This makes the interface consistent and predictable
- **Stateless:** Each request contains all necessary information, and the server does not store any state between requests. This simplifies server design, improves scalability, and reliability, as any server can handle any request
- **Cacheable:** Resources should be explicitly or implicitly marked as cacheable or non-cacheable. The server can cache the response for a given request, and the client can cache the response for a given request
- **Client-Server:** The client and server are separate entities with distinct responsibilities. This separation allows for independent development and evolution of both components

## Resource and URL Naming

Use stable, plural, noun-based paths:

- Collections: `/users`, `/orders`
- Single resource: `/users/{userId}`
- Sub-resource: `/users/{userId}/orders`

Avoid:

- Verb paths: `/getUser`, `/createOrder`
- Inconsistent naming (`/user` vs `/users`)
- Deep nesting beyond 2-3 levels

## HTTP Methods and Idempotency

- `GET`: Read resource (safe, idempotent)
- `POST`: Create or trigger non-idempotent action
- `PUT`: Replace full resource (idempotent)
- `PATCH`: Partial update (can be idempotent by contract)
- `DELETE`: Remove resource (idempotent in API behavior)

Idempotency matters for retries. For create/payment operations, use idempotency keys to prevent duplicate effects.

## Status Codes

Use clear status codes:

- `200`/`201`/`204` for success
- `400` for validation errors
- `401`/`403` for auth/authz failures
- `404` for missing resources
- `409` for conflicts
- `429` for rate limiting
- `5xx` for server-side failures

## Filtering, Sorting, and Pagination

Use query parameters for collection querying:

- Filtering: `/orders?status=paid&customerId=123`
- Sorting: `/orders?sort=-createdAt`
- Pagination: cursor-based preferred at scale

Cursor pagination is more stable than offset pagination for changing datasets.

## Versioning and Compatibility

Prefer clear major versions when introducing breaking changes:

- Path versioning: `/v1/orders`

Deprecation guidance:

- Announce deprecation window early
- Return `Deprecation` and `Sunset` headers where appropriate
- Provide migration docs and replacement endpoints

## Caching Control

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

## Concurrency Control

Use optimistic concurrency to prevent lost updates when multiple clients edit the same resource.

- Client reads resource and gets current `ETag`
- Client sends update with `If-Match: <ETag>`
- Server updates only if ETag still matches latest version
- If not matched, return `412 Precondition Failed`

How it is used:

1. Client A and B read `/orders/99` with `ETag: "v10"`
2. A updates first -> resource becomes `v11`
3. B sends update with stale `If-Match: "v10"`
4. Server rejects with `412`, forcing B to re-read and retry safely

Result: prevents accidental overwrite of newer data without using locks.

## Interview Talking Points

- REST quality comes from consistent resource modeling and HTTP correctness.
- Idempotency and error consistency are critical for production reliability.
- Cursor pagination, ETags, and clear status codes are high-signal design choices.

## Reference Materials

- [RFC 9110 - HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [RFC 9111 - HTTP Caching](https://www.rfc-editor.org/rfc/rfc9111)
- [RESTful API Design Best Practices](https://restfulapi.net/)
