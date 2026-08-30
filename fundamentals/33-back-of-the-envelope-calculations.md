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
  - fundamentals/13-load-balancing.md
  - fundamentals/23-database-replication.md
  - fundamentals/32-rate-limiting.md
  - fundamentals/34-cdn.md
---

# Back-of-the-envelope calculations

Back-of-the-envelope calculations are quick approximations used in system design to estimate scale, cost, and bottlenecks.

The goal is to land in the right order of magnitude quickly, not to be precise. An answer that is off by 20% still leads to the right architecture; an answer that is off by 1,000x leads to the wrong one.

## Why it matters

- Validates whether a design can actually handle the expected load
- Exposes the binding constraint early (traffic, storage, bandwidth, or memory)
- Turns vague design debates into a comparison of two numbers
- Shows the interviewer how you reason about scale, which is usually the point of the question

## How to run an estimate

Three steps, in order, out loud:

1. **State your assumptions before you use them.** Every number downstream depends on them, and a stated assumption the interviewer disagrees with costs you one correction — an unstated one costs you the whole answer.
2. **Do the math one step at a time.** Carry round numbers forward rather than chaining exact figures.
3. **Say what the result implies.** A number nobody interprets is not an estimate, it is arithmetic.

For example:

1. Assumptions: 100M daily active users, 20 requests per user per day, a 100:1 read/write ratio, and 500 KB per uploaded photo.
2. Math: `100M x 20 = 2B requests/day`, and `2B / 100k seconds ≈ 20,000 RPS` average. Writes are ~1% of that, so ~200 uploads per second, or 20M per day.
3. Implication: 20M uploads/day at 500 KB is 10 TB of new data every day — object storage plus a CDN, not a single database.

This example is developed in full through the rest of the document, so every section below builds on the same numbers.

## Rules of thumb

- Round aggressively; precision is not the goal (treat 1 day as 100,000 seconds, not 86,400)
- Do one calculation at a time — do not try to hold the whole chain in your head at once
- Say the assumption out loud before you use it, and pick defensible round numbers over precise-sounding ones
- Watch for unit slips: seconds vs days, bits vs bytes, KB vs MB. Most badly wrong estimates are unit errors, not reasoning errors
- Provision against peak, not average — the peak is the number that decides whether the system stays up

## Useful approximations

### Storage units

| Unit   | Approximately           | Something that size                               |
| ------ | ----------------------- | ------------------------------------------------- |
| 1 byte | 1 character             | A single letter                                   |
| 1 KB   | 1,000 bytes             | A small text page, or a photo thumbnail           |
| 1 MB   | 1 million bytes         | A compressed photo, or a minute of MP3 audio      |
| 1 GB   | 1 billion bytes         | An hour of HD video                               |
| 1 TB   | 1,000 GB                | A laptop SSD, or ~1,000 hours of HD video         |
| 1 PB   | 1,000 TB (1 million GB) | A large video library, or a rack of dense storage |

**Mental anchors:**

| Item                                 | Approximate size |
| ------------------------------------ | ---------------- |
| Short text post plus metadata        | ~2 KB            |
| Thumbnail image                      | ~10 KB           |
| Typical mobile photo (compressed)    | ~500 KB          |
| High-res camera photo (raw)          | ~10 MB           |
| 1 minute of MP3 audio                | ~1 MB            |
| 1 minute of HD streaming video       | ~10 MB           |
| 1 hour of HD video                   | ~1 GB            |
| Typical database row (a user record) | ~1 KB            |

### Latency units

| Operation                              | Typical latency   | Relative to memory access |
| -------------------------------------- | ----------------- | ------------------------- |
| Read from main memory (RAM)            | ~100 nanoseconds  | 1x (baseline)             |
| Read from SSD                          | ~100 microseconds | ~1,000x slower            |
| Read from spinning HDD                 | ~10 milliseconds  | ~100,000x slower          |
| Network round trip (same data center)  | ~500 microseconds | ~5,000x slower            |
| Network round trip (cross-region)      | ~50 milliseconds  | ~500,000x slower          |
| Internet round trip (US to Europe)     | ~100 milliseconds | ~1,000,000x slower        |
| Internet round trip (around the world) | ~200 milliseconds | ~2,000,000x slower        |

**Mental anchors:**

- 1 day ≈ 100,000 seconds. **Drop five zeros** to convert per-day into per-second: 100 million operations per day ≈ 1,000 operations per second.
- Memory is fast; everything else is orders of magnitude slower. If a step touches disk or the network, it dominates the budget.
- Distance costs more than hardware. A round trip across the world is slower than a disk seek, and no faster machine on the far side fixes it. That is the entire argument for serving content from a [CDN](./34-cdn.md) near the user rather than from one central region.

## The three core questions

Almost every estimation question is one of three calculations, or a combination of them:

- How much traffic arrives (QPS/RPS)
- How much data you keep (storage)
- How much data you move (bandwidth)

Each starts from the same place: **a number of users, and an assumption about what each user does per day**. Cache size, server count, and latency budget are then derived from those three.

The running example below is a photo-sharing service with **100M DAU**, **20 requests per user per day**, a **100:1 read/write ratio**, and **500 KB uploads**.

## 1. Traffic (QPS/RPS)

Terminology:

- **RPS**: Incoming HTTP or API requests hitting the service per second
- **QPS**: Operations or queries sent to a backing data store per second
- **DAU**: Daily active users
- **Requests per user per day**: How many requests each active user makes in a day
- **Peak factor**: How much busier the peak hour is than the daily average (2-3x is a reasonable default)

> **Tip:**
> RPS and QPS are the same kind of measurement. RPS is the conventional term for web traffic, QPS for database workload.
> They are not interchangeable *values*, though — one request often fans out into several queries.

Framework:

1. Assume DAU and requests per user per day.
2. Multiply, then divide by 100,000 to get the average per second.
3. Split that total into reads and writes using the read/write ratio.
4. Multiply by the peak factor to get the number you provision against.

Formulas:

- `Average RPS = (DAU x requests/user/day) / 100,000`
- `Write RPS = Average RPS / (read:write ratio + 1)`
- `Read RPS = Average RPS - Write RPS`
- `Peak RPS = Average RPS x peak factor`

> **Tip:**
> If the product gives you a write volume directly ("users upload 20M photos a day"), start there and multiply up by the read/write ratio instead.
> Writes are usually the better-understood number.

Example:

- 100M DAU x 20 requests/user/day = 2B requests/day
- Average RPS ≈ `2B / 100k` ≈ **20,000 RPS**
- At a 100:1 read/write ratio: ≈ 200 write RPS and ≈ 19,800 (round to 20,000) read RPS
- Peak at 3x: ≈ **60,000 RPS**, of which ≈ 600 RPS are writes

Implication: 60,000 peak RPS is far past one machine, so this is a horizontally scaled tier behind a [load balancer](./13-load-balancing.md), and the peak/average gap is exactly what [rate limiting](./32-rate-limiting.md) and headroom exist to absorb.

## 2. Storage

Storage is how much new data you write each day, multiplied by how long you keep it and how many copies you hold.

Formulas:

- `Daily storage = writes/day x average object size`
- `Total storage = daily storage x retention days x replication factor`

Example:

- 200 write RPS x 100k seconds = **20M uploads/day**
- 20M x 500 KB = **10 TB/day**
- 3 years of retention ≈ 1,100 days, so ≈ **11 PB** before replication
- At 3x replication ≈ **33 PB**

Do not forget the overhead that is not the payload: metadata rows, secondary indexes, thumbnails and other derived formats, and backups. For media workloads the derived formats alone can add 20-50% on top of the originals.

Implication: 33 PB is object storage (S3-class) with a metadata database beside it, not blobs in a relational database.

## 3. Bandwidth

Formulas:

- `Outbound = read RPS x average response size`
- `Inbound = write RPS x average request size`

Example, using the peak numbers:

- Outbound: 60,000 read RPS x 50 KB average response ≈ **3 GB/s** at the edge
- Inbound: 600 write RPS x 500 KB ≈ **300 MB/s**

The read response is 50 KB rather than 500 KB because most reads are feed metadata and thumbnails, not full-size originals. Naming that assumption matters: assuming every read returns a full photo would inflate the estimate tenfold.

Implication: 3 GB/s is CDN territory. At a 95% [CDN](./34-cdn.md) hit ratio the origin only serves the remaining 5% — about 150 MB/s and 3,000 read RPS — which is the difference between a normal origin fleet and an impossible one. Always estimate bandwidth at peak, never at average.

## 4. Cache size

Cache sizing answers a specific design question: does the hot data fit in the memory of a few machines, or does the cache itself have to be distributed?

The usual assumption is that a small share of the content serves most of the requests — often stated as 20% of objects serving 80% of reads.

> **Note:**
> The **working set** is the data that must be resident in cache for the application to perform acceptably. It is not the total dataset,
> and for time-ordered content it is usually "the recent stuff plus a long tail of popular older items".

Formulas:

- `Cache size = corpus size x working set percentage`
- `Cache size = number of hot objects x average object size`

Example:

- Most reads target recent uploads, so take the last 7 days as the corpus: 7 x 10 TB = 70 TB
- Apply the 20% working-set assumption: **~14 TB**

Implication: 14 TB does not fit on one machine, so this is a distributed cache — roughly 30 nodes at 512 GB, plus replication for the hot shards. That in turn brings [hot keys and stampede protection](./11-caching.md) into scope, which a single-node cache would let you ignore.

## 5. Server count

Formulas:

- `Servers = peak load / effective capacity per server`
- Apply redundancy and headroom separately, after the base number

Example:

- Peak 60,000 RPS
- One server benchmarks at 1,000 RPS; use 700 RPS as safe capacity (70%, leaving 30% headroom)
- Base requirement ≈ `60,000 / 700` ≈ **86 servers**
- Doubled for redundancy and zone loss ≈ **172 servers**

The headroom is not optional padding. A fleet running at 100% of benchmark capacity has no room for a deploy, a garbage-collection pause, or the traffic redistributed from a failed node — see [scalability](./10-scalability.md).

## 6. Latency budget

Break the request path into components and add them up. The point is to find the dominant term, not to get the sum exactly right.

- Client network round trip
- Gateway or proxy
- Application compute
- Cache, database, or object storage
- Serialization and response construction

Example:

- 80 ms network + 10 ms app + 5 ms database + 5 ms overhead ≈ **100 ms total**

Implication: network dominates at 80% of the budget, so optimizing the 10 ms of application code is nearly pointless while moving the response closer to the user is worth 50 ms or more. That is the estimate that justifies a CDN or a multi-region deployment.

## Putting it together

The full running example, end to end:

| Quantity                  | Value      | Where it came from                                  |
| ------------------------- | ---------- | --------------------------------------------------- |
| Average traffic           | 20,000 RPS | 100M DAU x 20 requests/day / 100k                   |
| Peak traffic              | 60,000 RPS | 3x peak factor                                      |
| Peak write traffic        | 600 RPS    | 1% of peak, from the 100:1 read/write ratio         |
| Uploads per day           | 20M        | 200 average write RPS x 100k seconds                |
| New storage per day       | 10 TB      | 20M uploads x 500 KB                                |
| Stored data after 3 years | ~33 PB     | 10 TB/day x ~1,100 days x 3 replicas                |
| Peak egress at the edge   | ~3 GB/s    | 60,000 read RPS x 50 KB average response            |
| Peak egress at the origin | ~150 MB/s  | 5% of edge traffic, at a 95% cache hit ratio        |
| Cache working set         | ~14 TB     | 20% of the last 7 days of uploads                   |
| Application servers       | ~170       | 60,000 / 700 RPS per server, doubled for redundancy |
| Latency budget            | ~100 ms    | 80 network + 10 app + 5 data + 5 overhead           |

Every row traces back to four stated assumptions. Change the read/write ratio or the object size and the whole table moves with it — which is exactly what makes it useful when the interviewer pushes back on one of them.

## Common pitfalls

- Sizing against average traffic and ignoring the peak
- Forgetting replication, backups, indexes, and derived formats in the storage number
- Assuming traffic is evenly spread when hot keys and hot partitions concentrate it
- Mixing units: bits vs bytes for bandwidth, or per-day vs per-second for rates
- Producing a number and never saying what it implies for the design

## Interview talking points

- State assumptions out loud before doing any math.
- Round aggressively and say that you are rounding; precision is not the goal.
- Carry one complete estimate through to a design conclusion (traffic to servers, or writes to storage tier).
- Point at the dominant term — the constraint that actually decides the architecture — rather than reciting every number.
- Say what you would validate first with real metrics or a load test.
