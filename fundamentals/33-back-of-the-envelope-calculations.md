---
title: "Back-of-the-envelope calculations"
concepts:
  - order-of-magnitude-estimation
  - traffic-estimation
  - storage-estimation
  - bandwidth-estimation
  - cache-sizing
  - server-capacity-planning
  - latency-budgeting
related:
  - fundamentals/03-latency-and-throughput.md
  - fundamentals/10-scalability.md
  - fundamentals/11-caching.md
  - fundamentals/23-database-replication.md
  - fundamentals/34-cdn.md
---

# Back-of-the-envelope calculations

Back-of-the-envelope calculations are quick approximations used in system design to estimate scale, cost, and bottlenecks.

The goal is to get within the right order of magnitude quickly, not to achieve exact precision.

## Why it matters

- Validates whether a design can handle expected load
- Exposes bottlenecks early (QPS, storage, bandwidth, servers)
- Helps compare architecture options quickly in interviews

## Stages of an estimate

1. State assumptions clearly and out loud before you use them
    - e.g. 100M users, 10% of users upload at least once a day
    - 500KB average file size
2. Do the math
    - 100M users * 10% = 10M users upload at least once a day
    - 10M writes per day / 100k seconds = 100 writes per second
    - 10M users * 500KB = 5TB of data uploaded per day
    - 5TB / day * 365 days = ~2PB of data uploaded per year
3. Think about the implications
    - Read/write ratio
    - Replication factor
    - Does it fit into memory / cache?

## Rules of thumb

- Round aggressively — precision is not the goal (e.g. 100k seconds = 1 day, not 86,400)
- Do one calculation at a time — don't try to do everything at once
- Say the assumption out loud before you use it (e.g. "I will assume 30 percent of users upload at least once a day")
- Be careful with unit slips — e.g. seconds confused with days, bits confused with bytes

## Useful approximations

### Storage units

| Unit   | Approximately                  | Something That Size                         |
| ------ | ------------------------------ | ------------------------------------------- |
| 1 Byte | 1 character                    | "a" (a single letter)                       |
| 1 KB   | 1,000 bytes                    | 1 small text page, photo thumbnail          |
| 1 MB   | 1 million bytes                | 1 high-res photo, a minute of MP3           |
| 1 GB   | 1 billion bytes                | 1 hour HD video, a movie                    |
| 1 TB   | 1 trillion bytes or 1,000 GB   | Laptop SSD, all books in a library          |
| 1 PB   | 1000 terabytes or 1,000,000 GB | Facebook photos per year, small data center |

**Mental Anchors:**

| Example Item                      | Approximate Size |
| --------------------------------- | ---------------- |
| Short text + metadata             | ~2 KB            |
| Thumbnail image                   | ~10 KB           |
| Typical mobile photo (compressed) | ~2 MB            |
| High-res camera photo (raw)       | ~10 MB           |
| 1 minute of MP3 audio             | ~1 MB            |
| 1 minute of HD streaming video    | ~10 MB           |
| 1 hour of HD movie                | ~1 GB            |
| Laptop SSD                        | ~1 TB            |
| Typical database row (user)       | ~1 KB            |

### Latency units

| Operation                                  | Typical Latency   | Comparison to Memory Access (1x) |
| ------------------------------------------ | ----------------- | -------------------------------- |
| Reading from main memory (RAM)             | ~100 nanoseconds  | 1x (base)                        |
| Reading from SSD disk                      | ~100 microseconds | ~1,000x slower                   |
| Reading from spinning HDD                  | ~10 milliseconds  | ~100,000x slower                 |
| Data center network round trip (same rack) | ~500 microseconds | ~5,000x slower                   |
| Data center round trip (cross-region)      | ~50 milliseconds  | ~500,000x slower                 |
| Internet round trip (US <-> Europe)        | ~100 milliseconds | ~1,000,000x slower               |
| Round trip around the world                | ~200 milliseconds | ~2,000,000x slower               |

**Mental Anchors:**

- 1 day ≈ 100,000 seconds (**drop 5 zeros** as an approximation, e.g. 100 million operations per day = 100k operations per second)
- Memory access is fast. Everything else is many orders of magnitude slower.
- Distance costs more than hardware. A round trip across the world is slower than a disk seek.
  No faster machine on the other side of the planet fixes that, which is why content is served from locations near the user rather than from one central place.

## Estimation framework

Almost every estimation question in an interview is one of three calculations, or a combination of them:

- How much traffic (QPS/RPS)
- How much data kept (storage)
- How much data moved (bandwidth)

Each one starts from the same place: **a number of users, and an assumption about what each user does per day**.

## 1. Traffic (QPS/RPS)

Terminology:

- **RPS**: Total number of incoming HTTP or API requests hitting a web server per second
- **QPS**: Total number of operations or queries sent to a backend data store or database per second
- **DAU**: Daily active users
- **Requests per user per day**: How many requests each user makes per day (e.g. 20 requests per user per day)
- **Peak factor**: How much more traffic you expect during peak hours (2-3x is a good starting point)

> **Tip:**
> RPS and QPS are the same thing. RPS is more common in web traffic estimation, while QPS is more common in database workload estimation.

Framework:

1. Assume DAU and requests per user per day
2. Divide by 100,000 to get the average per second.
3. Multiply by the read to write ratio to get read traffic from write traffic.
4. Multiply by two or three for peak.

> **Tip:**
> Start with writes, because writes are usually the smaller and better understood number, then derive reads from them.

Formula:

- `Average RPS (or QPS) = (DAU x request/user/day) / 100,000`
- `Read RPS = Average RPS x read/write ratio`
- `Write RPS = Average RPS x write/read ratio`
- `Peak RPS = Average RPS x peak factor`

Example:

- 100M DAU, 20 requests/user/day
- Average RPS ≈ `(100M x 20) / 100k` ≈ 20,000 RPS
- Peak (3x) ≈ 60,000 RPS
- Assuming 1/100 write:read ratio
  - 600 RPS of write traffic
  - 60,000 RPS of read traffic (100x more)

## 2. Storage

Storage is the amount of new data written each day, multiplied by how long you keep it.

Formula:

- `Daily storage = writes/day x average object size`
- `Total storage = daily storage x retention days x replication factor`

Example:

- 10M uploads/day x 100 MB ≈ 1 PB/day
- 3 year retention ≈ ~1000 PB (before replication)
- With 3x replication ≈ 3000 PB

Include metadata overhead and indexes, not just raw payload.

## 3. Bandwidth

Formula:

- `Outbound = read QPS x avg response size`
- `Inbound = write QPS x avg request size`

Example:

- 8,000 read QPS x 50 KB ≈ 400 MB/s outbound
- 2,000 write QPS x 2 MB ≈ 4 GB/s inbound

Use peak QPS for capacity planning, not average only.

## 4. Cache size

How much memory a cache needs.

The usual assumption is that a small share of the content gets most of the requests, often stated as 20% of the objects (working set) serving 80% of the reads.

> **Note:**
> Working set is the data that needs to be "hot" in the cache for the application to perform well.

The result answers a real design question: whether the working set fits in the memory of a few machines, or whether the cache itself has to be distributed across many.

Formula:

- `Cache size = total data x working set percentage`
- `Cache size = unique hot objects x average object size`

Example:

- 100 TB total data, assume 10% working set, then cache size = 10 TB
- 10M uploads/day, 100 MB/upload, 1% of uploads are frequently accessed, then cache size = 100 MB

## 5. Server count

Formula:

- `Servers = peak load / effective capacity per server`
- Apply redundancy and headroom separately

Example:

- Peak 60,000 RPS
- One server safe capacity 700 RPS (70% of 1000)
- Required ≈ `60,000 / 700` ≈ 86 servers
- With 2x redundancy ≈ 172 servers

## 6. Latency budget

Break request path into components and sum:

- Client-network round trip
- Gateway/proxy
- App compute
- Cache/DB/storage
- Serialization/response build

Example:

- 80 ms network + 10 ms app + 5 ms DB + 5 ms overhead ≈ 100 ms total

Use this to identify dominant term quickly.

## Common pitfalls

- Using average traffic only (ignoring peaks)
- Forgetting replication/backups/indices overhead
- Ignoring hot keys/paths and uneven traffic distribution

## Interview talking points

- Say assumptions out loud before math.
- Round aggressively, precision is not the goal.
- Show one complete estimate (traffic -> servers or storage).
- Mention what you would validate next with real metrics/load tests.
