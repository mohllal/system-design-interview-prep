---
title: "Latency and throughput"
concepts:
  - latency-components
  - round-trip-time
  - percentiles
  - tail-latency
  - goodput
  - bandwidth-delay-product
  - littles-law
  - bottleneck-analysis
related:
  - fundamentals/02-network-protocols.md
  - fundamentals/04-http-versions.md
  - fundamentals/11-caching.md
  - fundamentals/15-observability.md
  - fundamentals/33-back-of-the-envelope-calculations.md
  - fundamentals/10-scalability.md
---

# Latency and throughput

Latency and throughput are the two performance metrics almost every design discussion runs on, and they answer different questions:

- **Latency**: How long does one operation take?
- **Throughput**: How many operations can the system handle per unit of time?

They are not interchangeable, and optimizing one frequently costs the other: batching raises throughput while adding latency, and running a server far below capacity keeps latency low while wasting throughput. This note defines the vocabulary the rest of the notes use when they talk about performance.

## Latency

Latency is the time between issuing a request and receiving the response. It is the sum of four components, and knowing which one dominates tells you what to fix:

- **Propagation delay**: Time for a signal to cross the distance. Bounded by the speed of light, so it can only be reduced by moving closer to the user
- **Transmission delay**: Time to push the bytes onto the link. Payload size divided by link bandwidth
- **Processing delay**: Time the application, database, and intermediaries spend actually doing work
- **Queuing delay**: Time spent waiting behind other requests. The only component that grows without bound as load rises

**Round-trip time (RTT)** is the latency of one request-and-reply exchange across the network.
It is the unit of account for protocol design: connection setup, TLS negotiation, and every dependent request are measured in round trips, not milliseconds, because the millisecond cost changes with distance while the round-trip count does not.
A page that needs four sequential round trips is four times as sensitive to distance as one that needs one — see the round-trip budget in [network protocols](02-network-protocols.md).

### Latency numbers worth memorizing

Order-of-magnitude figures are enough. The point is to know which operations are 1000x more expensive than others, so you can rule out designs without measuring.

| Operation                              | Approximate latency |
| -------------------------------------- | ------------------- |
| L1 cache reference                     | 1 ns                |
| Main memory reference                  | 100 ns              |
| Read 1 MB sequentially from memory     | 0.05 ms             |
| SSD random read                        | 0.1 ms              |
| Round trip within the same data center | 0.5 ms              |
| Read 1 MB sequentially from SSD        | 1 ms                |
| HDD seek                               | 10 ms               |
| Round trip across a continent          | 50 ms               |
| Round trip between continents          | 150 ms              |

The gaps drive most architectural decisions: memory is roughly 1000x faster than an SSD, an SSD is roughly 100x faster than a disk seek, and one cross-continent round trip costs about as much as a thousand local SSD reads. That is the whole argument for [caching](11-caching.md) and for serving users from a nearby region.

### Why percentiles matter

Averages hide the users having a bad time. One request in a hundred taking 5 seconds barely moves the mean, but it is the request that generates the support ticket.

- **p50 (median)**: The typical experience
- **p95 and p99**: Tail latency, the experience of the unluckiest users
- **max**: Useful for debugging a specific incident, too noisy to build an SLO on

Two things about tails that are worth saying out loud in an interview:

- **Percentiles do not average.** You cannot average p99 across servers or across time buckets and get a meaningful p99. Track them from the underlying distribution (see [observability](15-observability.md))
- **Fan-out amplifies the tail.** If a request calls 10 backends in parallel and waits for all of them, a 1% chance of a slow backend means roughly a 10% chance of a slow request. At high fan-out, the p99 of a dependency becomes the p50 of the caller

## Throughput

Throughput is the amount of work completed per unit of time, measured in requests per second, messages per second, or bytes per second depending on what the system moves. Three related terms get confused:

- **Bandwidth**: The theoretical maximum capacity of a link or system
- **Throughput**: The rate actually achieved, including protocol overhead and retransmissions
- **Goodput**: The rate of useful application data delivered, excluding headers, retries, and duplicated work

Example: a 1 Gbps link might sustain 850 Mbps of throughput once congestion control and contention are accounted for, and deliver around 700 Mbps of goodput once TCP/IP headers, TLS framing, and the occasional retransmission are subtracted. Capacity planning against the label on the link overestimates what you actually get.

### Bandwidth-delay product

The **bandwidth-delay product (BDP)** is the amount of data that can be in flight (sent but not yet acknowledged) on a link:

`Data in flight = Bandwidth x RTT`

To use a link fully, the sender's TCP window must be at least as large as the BDP. If it is smaller, the sender stops and waits for acknowledgments before it can send more, and the link sits idle.

Example:

- Bandwidth = 100 Mbps
- RTT = 100 ms (0.1 s)
- BDP = `100 Mbps x 0.1 s = 10 Mb = 1.25 MB`

The sender needs 1.25 MB of unacknowledged data in flight to saturate the link. With a 64 KB window it will spend most of its time waiting for ACKs and achieve a small fraction of 100 Mbps, no matter how much bandwidth is available. This is why bulk transfers over long-distance links need window scaling, and why "we bought a bigger pipe and nothing got faster" is a common story.

## How latency and throughput relate

The link between the two is concurrency, and Little's law makes it precise:

`Average concurrency = Arrival rate x Average response time`

Example:

- Arrival rate = 1,000 requests/sec
- Average response time = 0.2 sec
- Average concurrency = `1,000 x 0.2 = 200`

On average 200 requests are in flight. If the server can only process 100 concurrently, the rest queue, response time rises, and by the same formula the required concurrency rises with it. That feedback loop is why overloaded systems degrade so sharply rather than gracefully.

The formula is the standard tool for sizing thread pools, connection pools, and queue depths, and it is worth rearranging: given a target latency and an expected arrival rate, it tells you the concurrency you must provision.

Practical consequences:

- Latency rises long before throughput saturates. Queuing delay starts climbing well below 100% utilization, so a system that looks like it has headroom on a throughput graph may already be missing its latency SLO
- High throughput with deep queues still means bad latency. Measure both, and cap queue depth so overload sheds load instead of accumulating it
- Low per-request latency does not imply high throughput. A fast but single-threaded stage caps the whole pipeline
- **Rule of thumb**: Optimize latency on user-facing paths, throughput on batch and background pipelines

## Common bottlenecks

### Latency bottlenecks

- Long network paths, plus DNS, TCP, and TLS setup on cold connections (see [network protocols](02-network-protocols.md))
- Slow database queries and N+1 access patterns
- Synchronous dependency chains, where each hop adds its full latency
- Lock contention and hot keys

### Throughput bottlenecks

- CPU or memory saturation
- Database connection pool limits
- Disk and network I/O limits
- Single-threaded or otherwise serialized stages

## Optimization strategies

### Reduce latency

- Move data and compute closer to users (CDN, regional deployments)
- Reuse connections and use modern HTTP so setup costs are not paid per request (see [HTTP versions](04-http-versions.md))
- Cache hot reads to avoid the expensive tier entirely
- Parallelize independent work so its latencies overlap instead of adding up
- Remove unnecessary remote calls; the cheapest round trip is the one you do not make

### Increase throughput

- Scale out stateless services horizontally
- Batch reads and writes where the added latency is acceptable
- Move non-critical work to asynchronous processing
- Compress payloads when CPU is cheaper than bandwidth
- Tune pool sizes and queue concurrency using Little's law rather than by guessing

## Interview talking points

- Clarify whether the requirement is low latency, high throughput, or both, and at which percentile.
- Quote percentiles (p95/p99), not averages, and mention tail amplification if the design fans out.
- Name the bottleneck layer (network, app, database, queue) before proposing a fix.
- Use Little's law to justify pool and queue sizing instead of picking round numbers.
- State the correctness cost of the optimization: caching means staleness, async means eventual consistency, batching means delay.
