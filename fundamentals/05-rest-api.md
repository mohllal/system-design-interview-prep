---
title: "REST API"
concepts:
  - rest-constraints
  - resource-naming
  - http-methods
  - idempotency-keys
  - http-status-codes
  - error-responses
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
  - advanced/01-oauth2.md
---

# REST API

REST (Representational State Transfer) is an architectural style that defines how clients interact with server resources using HTTP semantics in a consistent, predictable way.

This document covers how to design a good REST API. For when to choose REST over RPC, GraphQL, or messaging in the first place, see [Communication Patterns](./06-communication-patterns.md).

## REST constraints

REST is defined by a set of constraints. An API that satisfies them gets consistency, cacheability, and independent evolution largely for free.

- **Uniform interface**: Each resource is identified by a URL, and HTTP methods (GET, POST, PUT, PATCH, DELETE) are used to manipulate it. This makes the interface consistent and predictable.
- **Stateless**: Each request contains all necessary information, and the server stores no state between requests. This simplifies server design and improves scalability and reliability, since any server can handle any request.
- **Cacheable**: Responses are explicitly or implicitly marked as cacheable or non-cacheable. Both intermediaries and the client can reuse a cached response instead of re-requesting it.
- **Client-server**: The client and server are separate entities with distinct responsibilities. This separation allows for independent development and evolution of both components.
- **Layered system**: A client cannot tell whether it is talking to the origin server or to a proxy, cache, or load balancer in front of it. This is what makes CDNs, gateways, and reverse proxies transparent to callers.
- **Code on demand (optional)**: The server may ship executable code to the client. Rarely used in practice.

The uniform interface also includes HATEOAS (hypermedia links in responses that tell the client which transitions are available next). Most production APIs skip it and rely on documented URL templates instead, so treat it as a nice-to-have rather than a requirement.

## Resource and URL naming

Use stable, plural, noun-based paths:

- Collections: `/users`, `/orders`
- Single resource: `/users/{userId}`
- Sub-resource: `/users/{userId}/orders`

Avoid:

- Verb paths: `/getUser`, `/createOrder`
- Inconsistent naming (`/user` vs `/users`)
- Deep nesting beyond 2-3 levels

Some operations are genuinely actions, not resource mutations. Two workable options:

- Model the action as a sub-resource and `POST` to it: `POST /orders/{orderId}/cancel`
- Model the outcome as state and `PATCH` the resource: `PATCH /orders/{orderId}` with `{"status": "cancelled"}`

Pick one convention and apply it consistently across the API.

## HTTP methods and idempotency

| Method   | Purpose                                | Safe | Idempotent            |
| -------- | -------------------------------------- | ---- | --------------------- |
| `GET`    | Read a resource                        | Yes  | Yes                   |
| `POST`   | Create a resource or trigger an action | No   | No                    |
| `PUT`    | Replace the full resource              | No   | Yes                   |
| `PATCH`  | Partially update a resource            | No   | Can be, by contract   |
| `DELETE` | Remove a resource                      | No   | Yes (in API behavior) |

*Safe* means the request does not change server state. *Idempotent* means sending the same request N times leaves the same state as sending it once.

Idempotency matters because clients retry. A network timeout does not tell the client whether the server processed the request, so a retry of a non-idempotent `POST` can create a second order or a second charge.

The standard fix is an **idempotency key**:

1. Client generates a unique key per business action and sends it as a header, for example `Idempotency-Key: 4f9c-...`
2. Server records the key with the result of the operation before responding
3. On a retry with the same key, the server skips the work and replays the stored response
4. Keys expire after a retention window (24 hours is a common choice)

## Status codes

Use the narrowest status code that is accurate. Clients branch on these, so consistency matters more than variety.

| Code                        | Meaning                               | Typical use                                        |
| --------------------------- | ------------------------------------- | -------------------------------------------------- |
| `200 OK`                    | Success with a body                   | Reads, updates that return the resource            |
| `201 Created`               | Resource created                      | `POST` to a collection; return a `Location` header |
| `202 Accepted`              | Accepted for asynchronous processing  | Work handed to a queue or background job           |
| `204 No Content`            | Success with no body                  | `DELETE`, updates that return nothing              |
| `400 Bad Request`           | Malformed request                     | Unparseable body, missing required field           |
| `401 Unauthorized`          | Missing or invalid credentials        | No token, expired token                            |
| `403 Forbidden`             | Authenticated but not permitted       | Caller lacks the required scope or role            |
| `404 Not Found`             | Resource does not exist               | Unknown ID, or hiding a `403` from an outsider     |
| `409 Conflict`              | Request conflicts with current state  | Duplicate unique key, illegal state transition     |
| `412 Precondition Failed`   | Validator did not match               | Stale `If-Match` on an optimistic update           |
| `422 Unprocessable Content` | Syntactically valid, semantically not | Well-formed body that fails business validation    |
| `429 Too Many Requests`     | Rate limited                          | Include `Retry-After`                              |
| `5xx`                       | Server-side failure                   | Bugs, dependency outages, timeouts                 |

The `4xx` versus `5xx` split is the one clients depend on most: `4xx` means "do not retry this request unchanged", `5xx` means "retrying may succeed".

## Error responses

An API is only as usable as its errors. Return one consistent error shape from every endpoint so clients can parse failures generically.

A minimal useful error body carries a machine-readable code, a human-readable message, and enough context to act on:

```json
{
  "type": "https://api.example.com/errors/insufficient-funds",
  "title": "Insufficient funds",
  "status": 409,
  "detail": "Account acct_123 has a balance of 4.20 USD but the transfer requires 10.00 USD.",
  "instance": "/transfers/tr_789",
  "traceId": "b7ad6b7169203331"
}
```

That shape follows RFC 9457 (Problem Details for HTTP APIs), which is a reasonable default when you have no reason to invent your own.

Guidelines:

- Use stable, documented error codes; clients should never have to string-match on the message
- Return all validation failures at once, not one per round trip
- Include a trace or correlation ID so a user-reported failure can be found in logs
- Never leak internal details (stack traces, SQL, hostnames) in a public API

## Filtering, sorting, and pagination

Use query parameters for collection querying:

- **Filtering**: `/orders?status=paid&customerId=123`
- **Sorting**: `/orders?sort=-createdAt`
- **Pagination**: `/orders?limit=50&cursor=eyJpZCI6...`

Two pagination styles:

- **Offset pagination** (`?limit=50&offset=100`): easy to implement and allows jumping to an arbitrary page. It degrades on large offsets because the database still scans and discards the skipped rows, and it produces duplicate or missing items when rows are inserted or deleted between page requests.
- **Cursor pagination** (`?limit=50&cursor=...`): the cursor encodes the position of the last item in a stable sort order (for example `createdAt` plus `id` as a tiebreaker). The query becomes `WHERE (createdAt, id) < (:cursor)`, which uses an index and stays fast at any depth. The cost is losing random page access.

Prefer cursor pagination for anything that grows or changes while being read: feeds, event logs, large exports. Always cap `limit` server-side.

## Versioning and compatibility

Version only when you must. Most changes can be made without a new version if they are additive and backward compatible:

- Safe: adding a new endpoint, a new optional request field, or a new response field
- Breaking: removing or renaming a field, changing a type, tightening validation, changing default behavior

Clients must tolerate unknown response fields for this to work, so state that in the API contract.

When a change is genuinely breaking, introduce a new major version:

| Strategy              | Example                                   | Notes                                                                      |
| --------------------- | ----------------------------------------- | -------------------------------------------------------------------------- |
| Path versioning       | `/v1/orders`                              | Most common. Obvious in logs, easy to route at the gateway, easy to cache. |
| Header versioning     | `API-Version: 2026-01-15`                 | Keeps URLs stable, but versions are invisible in logs and harder to debug. |
| Media type versioning | `Accept: application/vnd.example.v2+json` | Purest REST, least ergonomic for callers.                                  |

Path versioning is the usual default. Whatever you pick, run old and new versions side by side rather than migrating everyone at once.

Deprecation guidance:

- Announce the deprecation window early and give a concrete sunset date
- Return `Deprecation` and `Sunset` headers on affected endpoints
- Provide migration docs and replacement endpoints
- Track usage per version so you know when it is safe to remove the old one

## Caching control

Use caching headers on read endpoints to reduce latency and backend load. See [Caching](./11-caching.md) for cache placement and invalidation strategy.

- `Cache-Control`: Defines freshness (for example, `max-age=60`, `public`, `private`, `no-store`)
- `ETag`: Version fingerprint of a representation
- `Last-Modified`: Timestamp-based validator when `ETag` is not used

Conditional request flow:

1. Client requests `GET /users/123`
2. Server responds with `ETag: "u123-v7"` and cache policy
3. Client later sends `If-None-Match: "u123-v7"`
4. If unchanged, server returns `304 Not Modified` (no body)

Result: lower bandwidth usage and faster repeated reads.

## Concurrency control

Use optimistic concurrency to prevent lost updates when multiple clients edit the same resource. The same `ETag` that powers caching doubles as a version validator for writes.

- Client reads the resource and gets the current `ETag`
- Client sends the update with `If-Match: <ETag>`
- Server updates only if the ETag still matches the latest version
- If not matched, server returns `412 Precondition Failed`

Example:

1. Client A and client B read `/orders/99` with `ETag: "v10"`
2. A updates first, so the resource becomes `v11`
3. B sends an update with a stale `If-Match: "v10"`
4. Server rejects with `412`, forcing B to re-read and retry safely

Result: prevents accidental overwrite of newer data without holding locks across requests, which a stateless API cannot do anyway. See [Concurrency Control](./26-concurrency-control.md) for the optimistic versus pessimistic trade-off.

## Interview talking points

- REST quality comes from consistent resource modeling and HTTP correctness, not from the word "REST".
- Idempotency keys and a single consistent error shape are what make an API safe to retry against.
- Cursor pagination, ETags for caching and optimistic concurrency, and precise status codes are high-signal design choices.
- Prefer additive, backward-compatible evolution; reach for a new major version only for genuinely breaking changes.

## Reference materials

- [RFC 9110 - HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [RFC 9111 - HTTP Caching](https://www.rfc-editor.org/rfc/rfc9111)
- [RFC 9457 - Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457)
- [RESTful API Design Best Practices](https://restfulapi.net/)
