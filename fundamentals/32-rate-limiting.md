---
title: "Rate limiting"
concepts:
  - token-bucket
  - leaky-bucket
  - fixed-window-counter
  - sliding-window-log
  - sliding-window-counter
  - distributed-rate-limiting
  - retry-backoff-and-jitter
  - fail-open-vs-fail-closed
related:
  - fundamentals/10-scalability.md
  - fundamentals/12-proxies.md
  - fundamentals/13-load-balancing.md
  - fundamentals/14-resilience.md
  - fundamentals/34-cdn.md
---

# Rate limiting

Rate limiting caps how often a client can perform an action, so one user, IP, or integration cannot consume a disproportionate share of capacity.

It is a fairness and protection tool, not a substitute for [load shedding](./14-resilience.md) when the system is already overloaded. Both are part of what makes a [scalable system](./10-scalability.md) degrade gracefully at its ceiling instead of collapsing.

## Why rate limit

- **Protect capacity**: Bound QPS to what origin, DB, or a downstream API can actually serve
- **Fairness**: Stop one noisy client from starving everyone else
- **Abuse**: Slow brute-force login, scraping, and credential stuffing
- **Cost and quotas**: Enforce free vs paid tiers, and cap calls to expensive third parties
- **Product rules**: Inventory grabs, invite spam, SMS/email send limits

Rate limiting answers "is this *caller* allowed to do this *now*?" Load shedding answers "can *the system* take more work?" Use both: identity-based limits at the edge, and overload protection deeper in.

## What you are bounding

These are easy to conflate and should be named separately.

| Mechanism             | Bounds                           | Example                          |
| --------------------- | -------------------------------- | -------------------------------- |
| **Rate limit**        | Events per time window           | 100 requests / minute            |
| **Quota**             | Events over a billing period     | 10,000 API calls / month         |
| **Concurrency limit** | In-flight work                   | 20 simultaneous uploads          |
| **Load shedding**     | Total accepted work under stress | Drop 20% of low-priority traffic |

A client can be inside its per-minute rate, over its monthly quota, and still be rejected by a concurrency cap on a slow endpoint.

Weighted limits charge different endpoints different costs (search = 10 units, `GET /health` = 0). That matches reality better than a single QPS number when some calls are cheap and some run a multi-second query.

## Who and where to enforce limits

### Identity (the key)

- **User / account**: Fair quotas and paid tiers. Requires auth; unauthenticated traffic still needs another key.
- **API key / app**: Partner integrations, service accounts, mobile apps.
- **IP**: Brute-force and pre-auth traffic. Collides users behind NAT; IPv6 and CGNAT make this a blunt instrument.
- **Resource**: Per-object caps (one listing, one inbox) so a hot item cannot be hammered through many identities.

Production systems stack keys: **global IP limit (abuse) + per-user limit (fairness) + per-endpoint limit (expensive paths)**. A request must pass all of them.

### Placement

Limits are enforced at the same points in the request path described in [proxies](./12-proxies.md) — this is one of the cross-cutting concerns a front door exists to own.

- **Edge / gateway**: The [reverse proxy](./12-proxies.md), Layer 7 [load balancer](./13-load-balancing.md), or [CDN](./34-cdn.md) edge. Cheapest place to reject, because it sees all public traffic and spends no backend capacity on a request that will fail. Necessarily coarse: per IP, per API key, per route.
- **Service**: Enforces what the gateway cannot see — per-resource limits, anything that depends on the authenticated identity, or on what a cache hit would have returned.
- **Outbound**: When *you* call Stripe, GitHub, or an SMS provider, you need a limiter on the way out so your own retries do not get the whole fleet banned.
- **Client-side**: A local token bucket improves UX and avoids useless 429s. It is **not** enforcement, since a malicious client simply skips it.

The layers stack rather than compete: coarse and cheap at the edge, precise and expensive close to the resource.

## Algorithms

Pick an algorithm for the *shape* of traffic you want to allow, then worry about how to distribute it.

There are two families. **Window algorithms** (fixed, sliding log, sliding counter) count events inside a time window and ask "how many already?". **Bucket algorithms** (token, leaky) model a continuously refilling or draining reservoir and ask "is there room right now?". Windows are easier to reason about and to explain; buckets handle bursts more gracefully and are what most public APIs actually ship.

### At a glance

| Algorithm              | Accuracy | Memory  | Burst behaviour            | Typical use                 |
| ---------------------- | -------- | ------- | -------------------------- | --------------------------- |
| Fixed window           | Low      | Lowest  | Spike at the boundary      | Simple quotas               |
| Sliding window log     | Highest  | Highest | Smooth                     | Strict limits, low volume   |
| Sliding window counter | High     | Low     | Mostly smooth              | General-purpose HTTP APIs   |
| Token bucket           | High     | Low     | Intentional, bounded burst | Public APIs                 |
| Leaky bucket           | High     | Low     | Queued or dropped          | Smoothing a fragile backend |

The worked examples below use Redis, because a shared store is what you need the moment there is more than one replica.

> **Note — the Redis primitives used below:**
> `INCR <key>` increments an integer and returns the new value; a missing key counts as 0, so the first increment returns 1.
> `GET <key>` reads a value. `EXPIRE <key> <seconds>` sets a TTL, which is how counters clean themselves up.
> `ZADD <key> <score> <member>` adds a member to a sorted set, `ZCARD <key>` counts its members, and
> `ZREMRANGEBYSCORE <key> <min> <max>` removes members whose score falls in that range.
>
> Every multi-step sequence below must run as **one atomic unit** — a Lua script or a `MULTI` block — or two replicas can read the same count and both allow.
> Only single-command `INCR` is safe on its own.

### Fixed window

Count requests in a clock-aligned window (for example, 4 per minute). Reset to zero at the boundary.

```plaintext
Limit = 4 / window

  12:00:00                 12:01:00                 12:02:00
       |------------------------|------------------------|
         ✓  ✓  ✓  ✓  ✗            ✓  ✓
         1  2  3  4  reject       1  2
                       ↺ counter back to 0
```

The counter does not slide. At `12:01:00` the client gets a full new budget, even if they just spent the previous one:

```plaintext
Limit = 4 / window

  12:00:00                          12:01:00
       |----------- window 1 -----------|----------- window 2 -----------|
                            ✓ ✓ ✓ ✓      ✓ ✓ ✓ ✓
                            last ~1s     first ~1s
                            |<—— 8 allows in ~1s = 2× the limit ——>|
```

**Example:** limit = 4 requests per 60 seconds, user `42`.

The window id is `floor(now / 60)`. That id is **part of the key**, so a new minute is a new counter. You do not trim members; the previous key is simply no longer read, and `EXPIRE` deletes it.

- Key: `rl:user:42:{window}` — e.g. `rl:user:42:12:00`, then `rl:user:42:12:01`
- Value: integer count

On every request (atomic `INCR`; set TTL only when the key is born):

1. `window = floor(now / 60)`, key = `rl:user:42:{window}`
2. `count = INCR <key>`
3. If `count == 1` → `EXPIRE <key> 60` so a quiet user does not leave the key forever
4. If `count > 4` → **reject**
5. If `count ≤ 4` → **allow**

`INCR` of a missing key starts at 1, which is how the "reset" happens: you never reset `12:00`, you start incrementing `12:01`.

| now      | window | key                | `INCR` | decision   |
| -------- | ------ | ------------------ | ------ | ---------- |
| 12:00:10 | 12:00  | `rl:user:42:12:00` | 1      | allow      |
| 12:00:20 | 12:00  | `rl:user:42:12:00` | 2      | allow      |
| 12:00:40 | 12:00  | `rl:user:42:12:00` | 3      | allow      |
| 12:00:50 | 12:00  | `rl:user:42:12:00` | 4      | allow      |
| 12:00:55 | 12:00  | `rl:user:42:12:00` | 5      | **reject** |
| 12:01:05 | 12:01  | `rl:user:42:12:01` | 1      | allow      |

The interesting rows are **12:00:55** and **12:01:05**:

```plaintext
t=12:00:55  key=rl:user:42:12:00
            INCR  →  5  > 4  →  429
            rl:user:42:12:00 stays at 5 until EXPIRE (still unread after 12:01)

t=12:01:05  key=rl:user:42:12:01     ← different key
            INCR  →  1  (missing key starts at 1)
            EXPIRE rl:user:42:12:01 60
            allow
            old rl:user:42:12:00 is not deleted by hand; TTL finishes it
```

Note that a single `INCR` is the whole decision here, which is why fixed window is the one algorithm that needs no script: the atomicity comes free.

**Pros:**

- Cheap (one counter per key)

**Cons:**

- Boundary burst

**Best fit:** Coarse quotas, dashboards, places where a 2× spike is acceptable.

### Sliding window log

Store a timestamp per allowed request. On each arrival, drop anything older than `now - window`, then count what is left. The window is always "the last N seconds from now," not "this clock minute."

**Example:** limit = 3 requests per 10 seconds, user `42`.

In Redis that log is **one sorted set**, not one key per timestamp:

- Key: `rl:user:42`
- Score and member: the request timestamp (append a uuid to the member if two requests can share the same millisecond)

On every request, in this order, as one atomic script (otherwise two replicas can both see `count = 2` and both allow):

1. `ZREMRANGEBYSCORE rl:user:42 -inf <cutoff>` — remove members with score ≤ `now − 10` (aged-out requests)
2. `ZCARD rl:user:42` — how many timestamps are still in the window
3. If `count ≥ 3` → **reject**, and do not write
4. If `count < 3` → `ZADD rl:user:42 <now> <now>`, then `EXPIRE rl:user:42 10` → **allow**. The TTL is a safety net so a quiet user does not leave the key behind forever.

Each row is one arrival. Cutoff is `now − 10`. Only timestamps **strictly after** the cutoff count.

| now | cutoff (`now−10`) | `ZREMRANGEBYSCORE -inf <cutoff>`         | `rl:user:42` after trim | `ZCARD` | decision   | write                     |
| --- | ----------------- | ---------------------------------------- | ----------------------- | ------- | ---------- | ------------------------- |
| 0s  | −10s              | nothing to drop                          | `{}`                    | 0       | allow      | `ZADD 0` → `{0}`          |
| 2s  | −8s               | nothing to drop                          | `{0}`                   | 1       | allow      | `ZADD 2` → `{0, 2}`       |
| 5s  | −5s               | nothing to drop                          | `{0, 2}`                | 2       | allow      | `ZADD 5` → `{0, 2, 5}`    |
| 8s  | −2s               | nothing to drop (`0` is still in window) | `{0, 2, 5}`             | 3       | **reject** | no `ZADD`                 |
| 11s | 1s                | drops `0`                                | `{2, 5}`                | 2       | allow      | `ZADD 11` → `{2, 5, 11}`  |
| 12s | 2s                | drops `2`                                | `{5, 11}`               | 2       | allow      | `ZADD 12` → `{5, 11, 12}` |
| 13s | 3s                | nothing to drop                          | `{5, 11, 12}`           | 3       | **reject** | no `ZADD`                 |

The interesting rows are **8s** and **11s**:

```plaintext
t=8s   cutoff=-2
       ZREMRANGEBYSCORE rl:user:42 -inf -2   →  {0, 2, 5} unchanged
       ZCARD                                 →  3  ≥ 3  →  429, leave the set alone

t=11s  cutoff=1
       ZREMRANGEBYSCORE rl:user:42 -inf 1    →  removes 0, left {2, 5}
       ZCARD                                 →  2  < 3  →  ZADD rl:user:42 11 11
                                                         EXPIRE rl:user:42 10
                                                         allow
```

```plaintext
  time    0    2    5    8         11   12   13
          ✓    ✓    ✓    ✗          ✓    ✓    ✗
          |------- 3 in 10s -------|
                         at 8s the oldest (0) is still
                         inside the window, so 4th is denied

                    at 11s, cutoff=1 → ZREMRANGEBYSCORE drops 0
                    only 2 and 5 remain → room for one more
```

A **fixed window** of 10s would have reset at t=10 and allowed the request at t=8's "neighbor" times with a full new budget. The log does not: at t=11 you still carry `2` and `5` from the previous 10 seconds.

Memory is one sorted-set member per request in the window (here at most 3). At high QPS that is the cost of exactness.

**Pros:**

- Accurate; no boundary spike

**Cons:**

- Memory grows with request rate (`limit` members per key, or worse)

**Best fit:** Low-volume, strict accuracy (login, OTP).

### Sliding window counter

Approximate a sliding window with two fixed windows: `previous_count × (overlap fraction) + current_count`.

At 30s into a 60s window, half of the previous window still overlaps the sliding 60s lookback, so it is counted at 50% weight.

```plaintext
                 previous (80 reqs)                  current (20 reqs)
    |<-------------- 60s --------------->|<-------------- 60s --------------->|
    12:00                                12:01              now              12:02
                                                            30s in
                                         |<-- overlap = 30/60 = 0.5 --------->|

    estimate = 80 × 0.5  +  20  =  60
               prev×weight    current
```

Those two integers are the same fixed-window keys as above. At 12:01:30:

```plaintext
  GET rl:user:42:12:00  →  80   (previous, weight 0.5)
  GET rl:user:42:12:01  →  20   (current,  weight 1.0)
  estimate = 80 × 0.5 + 20 = 60
  if 60 < limit → INCR rl:user:42:12:01
```

**Example:** limit = 5 requests per 10 seconds, user `42`. Window id = `floor(now / 10)`.

- Previous key: `rl:user:42:{window - 1}`
- Current key: `rl:user:42:{window}`

You still need the previous key **after that window has ended**, so `EXPIRE` must be `2 × window` (20s), not 10s. A fixed-window TTL of 10s would delete the previous count before you can weight it.

On every request, again as one atomic script so the two `GET`s and the `INCR` cannot race:

1. `curr_id = floor(now / 10)`, `prev_id = curr_id - 1`
2. `elapsed = now % 10`, `weight = 1 − elapsed / 10`
3. `prev = GET rl:user:42:{prev_id}` or 0, `curr = GET rl:user:42:{curr_id}` or 0
4. `estimate = prev × weight + curr`
5. If `estimate ≥ 5` → **reject**, do not `INCR`
6. If `estimate < 5` → `INCR rl:user:42:{curr_id}`, `EXPIRE rl:user:42:{curr_id} 20` → **allow**

Notes:

- `elapsed` is "how many seconds into the current 10s bucket am I?" `now % 10` is the remainder after dividing by the window: at `now=14`, `14 % 10 = 4`, so you are 4 seconds into window `1` (`[10s, 20s)`).
- `weight` is the fraction of the **previous** bucket that is still inside the sliding last-10-seconds. A lookback of 10s from t=14 starts at t=4, so it still covers 6s of window `0`. That is `(10 − elapsed) / 10 = 1 − elapsed / 10 = 0.6`:

```plaintext
window 0 [0, 10)              window 1 [10, 20)
|-----------------------------|-----------------------------|
0                            10         14                  20
                              |<- elapsed=4 ->|
               |<------- sliding last 10s ------->|
               4              10         14

previous overlap = [4, 10) = 6s out of 10s  →  weight = 0.6
count previous at 60%, current at 100%
```

Early in the new window, `elapsed` is small, `weight` is close to 1 (almost all of the previous burst still counts). Late in the window, `elapsed` is large, `weight` goes toward 0 (previous burst has aged out).

Worked requests. Window `0` is `[0s, 10s)`, window `1` is `[10s, 20s)`.

| now | window | elapsed | weight (`1−elapsed/10`) | `GET` prev `rl:user:42:0` | `GET` curr `rl:user:42:1` | estimate          | decision   | write            |
| --- | ------ | ------- | ----------------------- | ------------------------- | ------------------------- | ----------------- | ---------- | ---------------- |
| 1s  | 0      | 1s      | — (no previous)         | —                         | 0                         | 0                 | allow      | `INCR` `…:0` → 1 |
| 3s  | 0      | 3s      | —                       | —                         | 1                         | 1                 | allow      | `INCR` `…:0` → 2 |
| 5s  | 0      | 5s      | —                       | —                         | 2                         | 2                 | allow      | `INCR` `…:0` → 3 |
| 8s  | 0      | 8s      | —                       | —                         | 3                         | 3                 | allow      | `INCR` `…:0` → 4 |
| 11s | 1      | 1s      | 0.9                     | 4                         | 0                         | 4×0.9+0 = **3.6** | allow      | `INCR` `…:1` → 1 |
| 12s | 1      | 2s      | 0.8                     | 4                         | 1                         | 4×0.8+1 = **4.2** | allow      | `INCR` `…:1` → 2 |
| 13s | 1      | 3s      | 0.7                     | 4                         | 2                         | 4×0.7+2 = **4.8** | allow      | `INCR` `…:1` → 3 |
| 14s | 1      | 4s      | 0.6                     | 4                         | 3                         | 4×0.6+3 = **5.4** | **reject** | no `INCR`        |

At 14s, **current is only 3** — a fixed window would still allow. The weighted previous burst (`4 × 0.6`) pushes the estimate over 5, so you reject. That is the boundary spike being smoothed.

```plaintext
t=14s  curr_id=1  prev_id=0  elapsed=4  weight=0.6
       GET rl:user:42:0  →  4
       GET rl:user:42:1  →  3
       estimate = 4 × 0.6 + 3 = 5.4  ≥ 5  →  429, leave both keys alone

       rl:user:42:0 expires on its TTL (20s from last INCR in that window)
       rl:user:42:1 stays at 3 until a later allow increments it
```

Two integers of storage instead of every timestamp. The estimate is slightly high or low vs a true log, but the boundary spike is mostly gone.

**Pros:**

- Near-sliding accuracy, two integers of storage

**Cons:**

- Slightly over- or under-counts compared with a true sliding window log

**Best fit:** Default for HTTP APIs at scale (this is what many Redis implementations actually do, despite the name "sliding window").

### Token bucket

Tokens refill at a steady rate. The bucket has a **capacity** (burst) and a **refill rate** (sustained). A request takes one token (or `cost` tokens). If empty, reject (or wait).

```plaintext
         refill 2 tokens/s
                │
                ▼
           ┌─────────┐
           │ ● ● ● ● │  capacity 4  ← max burst
           │         │
           └────┬────┘
                │ take 1 per request
                ▼
        allow if tokens ≥ 1, else 429
```

A full bucket can be emptied immediately (the burst). After that, allows happen only as fast as the refill rate:

```plaintext
  capacity 4, refill 2 tokens/s

  t=0.0s   burst of 4   tokens 4 -> 0     all 4 allowed
  t=0.1s   request      tokens 0.2        reject (needs 1)
  t=0.5s   request      tokens 1.0        allow  -> 0
  t=1.0s   request      tokens 1.0        allow  -> 0
  t=1.1s   request      tokens 0.2        reject

  a burst of 4 up front, then a sustained 2/s — exactly the configured refill rate
```

**Example:** capacity = 4, refill = 2 tokens/s, user `42`.

There is no background timer refilling anything. The bucket is **two fields**, and refill is computed lazily on read:

- Key: `rl:user:42` (a hash)
- Fields: `tokens` (a float) and `last_refill` (a timestamp)

On every request, as one atomic script:

1. Read `tokens` and `last_refill`. If the key is missing, start at `tokens = capacity` and `last_refill = now`.
2. `tokens = min(capacity, tokens + (now − last_refill) × refill_rate)`, then set `last_refill = now`
3. If `tokens < cost` → **reject**, writing back only the refilled state
4. If `tokens ≥ cost` → `tokens −= cost`, write back → **allow**
5. `EXPIRE rl:user:42` a little past `capacity / refill_rate`, so an idle bucket cleans itself up (a bucket idle that long is full anyway, which is the same as absent)

That lazy refill is the whole trick: two numbers and a subtraction give you a continuously refilling bucket without a single scheduled job.

```plaintext
t=0.1s   read  tokens=0    last_refill=0.0
         refill  0 + (0.1 − 0.0) × 2 = 0.2
         0.2 < 1  →  429, write back tokens=0.2 last_refill=0.1

t=0.5s   read  tokens=0.2  last_refill=0.1
         refill  0.2 + (0.5 − 0.1) × 2 = 1.0
         1.0 ≥ 1  →  allow, write back tokens=0.0 last_refill=0.5
```

**Pros:**

- Allows a real burst, then enforces the average — the shape most APIs actually want
- Maps cleanly onto a documented contract: "100/s sustained, burst 500"
- Two numbers per key, regardless of request rate
- Refill is relative, so it does not care about clock alignment across hosts

**Cons:**

- Two knobs to tune instead of one, and the wrong burst size hides a sustained-overload problem
- An empty bucket is a hard reject unless you also queue
- Read-modify-write, so a shared deployment needs a script — unlike fixed window, whose bare `INCR` is atomic on its own

**Best fit:** Public APIs where a short burst is fine but sustained overload is not. This is the common default (AWS and Stripe-style limits).

### Leaky bucket

Requests enter a queue (the bucket). They **leave at a constant rate**. If the queue is full, new requests drop (or wait).

```plaintext
   requests arrive at any rate
        ✓ ✓ ✓ ✓ ✓ ✓
           \     /
            \   /     queue depth ≤ 4
             \ /
              |       leak 2/s  (constant, always)
              ▼
           backend sees a smooth stream

   queue full → drop (or wait). The backend never sees the burst.
```

The implementation is the mirror image of token bucket: two fields again, `level` and `last_leak`, with the leak computed lazily as `level = max(0, level − (now − last_leak) × leak_rate)`. A request is admitted when `level + 1 ≤ capacity`, and admission increments `level`. Token bucket counts what you *may still spend*; leaky bucket counts what is *already in flight*.

**Pros:**

- Smooth, predictable outflow, so a fragile backend sees a constant rate no matter what arrives
- Queue depth is a direct backpressure signal

**Cons:**

- Bursts turn into latency (if queued) or rejects (if not) — never into a fast success
- A queue adds a place for requests to sit and time out

**Best fit:** Protecting a downstream that genuinely cannot absorb spikes, such as a legacy database or a fragile third-party dependency.

Token bucket **shapes the incoming burst** (a short spike of allows, then throttling). Leaky bucket **shapes the outgoing rate** (the backend always sees a drip). They are duals. In an interview, say explicitly which side you are smoothing.

### Choosing one

- **Default to token bucket** for a public HTTP API. It expresses the contract users care about ("N per second, burst M") and handles real client behaviour.
- **Use the sliding window counter** when you want a plain per-window limit without the boundary spike, and two integers per key is the budget you have.
- **Use fixed window** only when a 2x spike at the boundary is genuinely acceptable — internal quotas, dashboards, coarse monthly counters.
- **Use the sliding window log** when exactness matters more than memory: login attempts, OTP sends, anything where "5 means 5".
- **Use leaky bucket** when the constraint is downstream capacity rather than caller fairness.

## HTTP contract

A rejected request should be **machine-readable**, so a client can react correctly instead of guessing.

| Signal                                      | Meaning                                                                                |
| ------------------------------------------- | -------------------------------------------------------------------------------------- |
| `429 Too Many Requests`                     | This identity hit a limit. Retrying immediately will fail again.                       |
| `503 Service Unavailable`                   | The system is overloaded or shedding. Not necessarily *this* client's fault.           |
| `Retry-After`                               | Seconds (or an HTTP date) to wait. The single most important client hint.              |
| `RateLimit-Limit` / `-Remaining` / `-Reset` | Current window budget (IETF `RateLimit` fields; many APIs still send `X-RateLimit-*`). |

Return `429` for per-client limits and reserve `503` for global overload. The distinction matters to the caller: a `429` means "you, slow down", a `503` means "everyone, come back later", and a well-written client backs off differently for each.

## How clients should retry

A limiter that returns 429 without a retry policy just moves the stampede to `t + 0.1s`.

```mermaid
sequenceDiagram
    participant C as Client
    participant API as API

    C->>API: POST /orders
    API-->>C: 429 Retry-After: 30
    Note over C: wait 30s + jitter
    C->>API: retry (same idempotency key)
    API-->>C: 201
```

**Rules that belong in every client and SDK:**

1. **Honor `Retry-After`.** If it is absent, use exponential backoff (for example 1s, 2s, 4s, …) with a cap.
2. **Add jitter** (random delay). Without it, every client that hit the window boundary retries in lockstep and recreates the burst the limiter just stopped. See [timeouts, retries, and jitter](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/).
3. **Cap attempts and total wait.** Infinite retries turn a 429 into a self-inflicted outage.
4. **Retry 429 and 503 and do not blindly retry 400/401/403/404.** Those will not get better with time.
5. **Idempotency for unsafe methods.** A retried `POST` can double-charge if the first request succeeded and the 429/timeout was on the response. Send an idempotency key — rate limiting alone does not make retries safe.
6. **Treat client-side limiting as a courtesy.** Locally wait when `Remaining` is 0 instead of firing and eating 429s.
7. **Backoff on outbound calls too.** If 50 service replicas each retry a third-party at the full quota, the vendor bans the company IP, not the replica.

**Retry storm (what to avoid):**

```mermaid
sequenceDiagram
    participant C1 as Clients
    participant API as API

    C1->>API: burst → all 429
    Note over C1: wait exactly 60s (no jitter)
    C1->>API: synchronized retry burst
    API-->>C1: 429 again
```

Server-side, prefer token bucket or sliding windows so the reset is not a cliff. Client-side, jitter is mandatory.

Distinguish **user-visible retry** (show "try again in 30s") from **autonomous retry** (jobs, webhooks, meshes). Autonomous retries need the same policy or they amplify load.

## Distributed rate limiting

Local counters on each replica are wrong once you have more than one process: `N` instances each allowing `L` requests yield `N × L`. The [load balancer](./13-load-balancing.md) offers no guarantee about which replica a given client lands on, so each one sees only a fraction of that client's traffic and independently concludes the client is well behaved.

```mermaid
graph TD
    C[Clients] --> LB[Load balancer]
    LB --> A["Replica A<br/>local count 100/100"]
    LB --> B["Replica B<br/>local count 100/100"]
    A --> E["Effective limit = 200<br/>intended limit = 100"]
    B --> E
```

### Approaches

#### Shared counter (usual production choice)

All replicas talk to a fast store (Redis, Memcached, DynamoDB). For a fixed window:

- `INCR key` then `EXPIRE` on first increment
- Reject if the value exceeds `L`

That is atomic enough for a counter. The naive `GET` → check → `SET` (read-modify-write) path from a single-node tutorial **races**: two replicas can both read 99 and both allow.

Any algorithm that reads state, computes, and writes it back — the token bucket refill-and-consume above, or the two `GET`s of a sliding window counter — needs a Lua script or `MULTI` instead. The extra round trip is the price of a genuinely global limit.

#### Sticky identity

Route each user or IP to a fixed replica and limit locally. Simple, and no Redis on the request path. This is [session affinity](./13-load-balancing.md) applied to limiter state, and it inherits all of affinity's problems: it breaks on rebalance and deploys, it distributes load unevenly, and connection-level changes (anycast, HTTP/2 reconnects) can silently move a client to a fresh, empty counter.

#### Local + async reconciliation

Each replica allows a share (`L / N` or a bit more) and periodically syncs. Low latency, **over-allows** during sync lag or when `N` is wrong (deploys, autoscaling).

Use when slightly exceeding the limit (overshoot) is cheaper than a Redis hop on every request.

#### Two-layer

A cheap in-process limiter (token bucket per replica) plus a slower global quota.

Catches bursts without a network round trip, then settles to the real cap. Same idea as cache-aside: local is the fast path, shared store is the source of truth.

### Design issues specific to distributed rate limiting

- **Fail-open vs fail-closed**: If Redis is down, do you allow (availability, risk of overload) or reject (safety, outage looks like a mass `429`)? Public read APIs often fail open with a tight local cap while login and payments often fail closed to avoid a mass `429`.
- **Hot keys**: One celebrity account is one Redis key. Shard the counter (`user:123:{0..k}` and sum) or keep a local admission cache in front.
- **Multi-region**: A *global* monthly quota needs a global store or a split budget per region (`L / regions`, with a small reserve). Per-region rate limits are simpler and usually what you want for abuse; billing quotas are the hard global ones.
- **Clock skew**: Fixed windows aligned to wall clock disagree across hosts. Prefer relative refill (token bucket) or accept small inaccuracy.
- **Cardinality**: A key per IP per minute can explode memory. TTL everything, and cap the number of tracked keys (LRU / approximate).
- **Accuracy vs latency**: Slightly over-allowing is often better than adding `2ms` to every request.

## Operational considerations

- **Shadow mode**: Log "would have 429'd" before enforcing. You will discover bot traffic, NAT collisions, and mis-sized limits without a launch incident.
- **Headers on success, not only on 429**: `Remaining` lets well-behaved clients slow down *before* they fail.
- **Auth order**: If you rate-limit by user, you must authenticate first — but you still need an IP limit *before* auth so login cannot be brute-forced for free.
- **Cache and 429**: Do not cache 429s the same way as 200s at a shared [CDN](./34-cdn.md) unless the cache key includes the identity. A cached 429 for user A must not be served to user B.
- **Idempotent reads vs costly writes**: Tighter limits on `POST /search` or `POST /export` than on `GET /item/:id`.
- **Human vs machine**: Interactive UIs want small bursts (token bucket). Batch jobs should be given a separate key and a leaky-bucket or queued path, not compete with the UI budget.
- **Observability**: Alert on 429 ratio, limiter store latency, and "limit too low" tickets. A spike in 429s can be an attack, a bad client deploy, or an undersized limit after a launch.
- **Legal / privacy**: Storing every request timestamp (the sliding window log) per IP is a data-retention choice, not only a data-structure choice.

## Design guidelines

- Name the key, the window, the burst, and what happens when the store is down
- Stack coarse edge limits with precise service limits; do not rely on one layer
- Prefer token bucket or the sliding window counter for APIs; use fixed window only when a boundary spike is acceptable
- Return `429` + `Retry-After` + remaining budget; document it for SDK authors
- Assume clients will retry; design jitter and idempotency into the contract
- Use a shared atomic counter (or accept over-allow) once you have more than one replica
- Shadow-enforce new limits, then tighten

## Interview talking points

- Start from **who** (IP vs user vs key) and **what** (rate vs quota vs concurrency vs shedding).
- Explain the **fixed-window boundary burst**, then say why you would pick token bucket or the sliding window counter instead.
- For multiple replicas, say **local limits multiply** and describe Redis `INCR` (atomic) vs get-then-set (racy).
- Mention **fail-open vs fail-closed** when the limiter store dies.
- Close the loop with **clients**: `Retry-After`, jitter, retry caps, idempotency keys — otherwise `429`s become a synchronized retry storm.
- **Placement**: Gateway for coarse public traffic, service for per-resource, outbound for third parties.

## Reference materials

- [RFC 6585 - 429 Too Many Requests](https://datatracker.ietf.org/doc/html/rfc6585)
- [IETF RateLimit header fields](https://datatracker.ietf.org/doc/draft-ietf-httpapi-ratelimit-headers/)
- [Cloudflare - Counting things a lot of different things](https://blog.cloudflare.com/counting-things-a-lot-of-different-things/)
