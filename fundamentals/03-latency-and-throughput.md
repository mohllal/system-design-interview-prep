# Latency and Throughput

Latency and throughput are core performance metrics that answer two different questions:

- **Latency**: How long does one operation take?
- **Throughput**: How many operations can the system handle per unit time?

Both matter in system design, but optimizing one does not automatically optimize the other.

## Latency

Latency is the time between sending a request and receiving a response.

Common components:

- **Propagation delay**: Distance and network path
- **Transmission delay**: Packet size vs link speed
- **Processing delay**: App/DB compute time
- **Queuing delay**: Waiting in overloaded queues

### Why Percentiles Matter

Average latency can hide bad user experience.

- **p50**: Typical request experience
- **p95/p99**: Tail latency (slowest users)
- **max**: Useful for debugging, not SLO design

In interviews, mention tail latency explicitly.

## Throughput

Throughput is the amount of work completed /transmitted per second (for example, requests/sec, MB/sec).

Key terms:

- **Bandwidth**: Theoretical capacity of a link/system
- **Throughput**: Actual data transfer/processing rate achieved
- **Goodput**: Useful data transfer/processing rate (excluding retries, protocol overhead, etc.)

Example:

- Link bandwidth: 1 Gbps
- Effective throughput: 850 Mbps
- Goodput after protocol overhead: lower still (e.g. 700 Mbps)

## Latency vs Throughput Relationship

They are related but not identical.

- High throughput with high queuing can still mean high latency
- Low latency per request does not guarantee high total throughput
- Under load, latency often rises before throughput saturates

Practical rule:

- Optimize latency for user-facing paths
- Optimize throughput for batch/background pipelines

## Common Bottlenecks

**Latency bottlenecks:**

- Long network paths and DNS/TLS setup
- Slow DB queries and N+1 access patterns
- Synchronous dependency chains
- Lock/contention hotspots

**Throughput bottlenecks:**

- CPU/memory saturation
- DB connection pool limits
- Disk I/O limits
- Single-threaded or serial stages

## Optimization Strategies

### Reduce Latency

- Place data/services closer to users (CDN, regional deployment)
- Use connection reuse and modern HTTP (HTTP/2/3)
- Cache hot reads
- Parallelize independent work
- Remove unnecessary remote calls

### Increase Throughput

- Scale out stateless services
- Batch writes/reads where safe
- Use async processing for non-critical work
- Compress payloads when CPU cost is acceptable
- Tune pool sizes and queue concurrency

## Useful Formulas

### Bandwidth-Delay Product

The **Bandwidth-Delay Product (BDP)** is the amount of data that can be "in flight" (sent but not yet acknowledged) on a network link.

`Data in flight = Bandwidth x Round-Trip Time (RTT)`

**Why it matters:** To fully utilize the available link bandwidth, the sender's TCP window (or buffer) must be at least as large as the BDP.

If it's smaller, the sender has to stop and wait for acknowledgments before sending more data, leaving the link underutilized and reducing throughput.

**Example:**

- Bandwidth = **100 Mbps**
- RTT = **100 ms (0.1 s)**

BDP = `100 Mbps × 0.1 s = 10 Mb = 1.25 MB`

This means the sender needs to have **1.25 MB of unacknowledged data in flight** to keep the link fully utilized.

If the TCP window is only **64 KB**, the sender will frequently pause waiting for ACKs, achieving only a small fraction of the available 100 Mbps bandwidth.

### Little's Law

`Average concurrency = Arrival rate x Average response time`

The **Little's Law** relates the number of requests being processed simultaneously (concurrency) to the request arrival rate and the average time each request spends in the system.

**Why it matters:** It helps estimate how many requests your application must handle concurrently, making it useful for sizing thread pools, connection pools, queues, and other limited resources.

**Example:**

- Arrival rate = **1,000 requests/sec**
- Average response time = **0.2 sec**

Average concurrency = `1,000 × 0.2 = 200`

This means that, on average, **200 requests are in flight** at any given time. If your server can only handle **100 concurrent requests**, requests will start queueing, increasing latency and potentially causing timeouts.

## Interview Talking Points

- Clarify whether the requirement is low latency, high throughput, or both.
- Always discuss percentiles (p95/p99), not only averages.
- Identify the bottleneck layer (network, app, DB, queue).
- Explain trade-offs of caching/async/parallelism on correctness.
