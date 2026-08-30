---
title: "Kafka architecture"
concepts:
  - distributed-log
  - partitioning
  - in-sync-replicas
  - consumer-groups
  - log-compaction
  - exactly-once-semantics
  - control-plane-data-plane-split
  - kraft
related:
  - fundamentals/16-hashing.md
  - fundamentals/23-database-replication.md
  - fundamentals/28-leader-election.md
  - fundamentals/29-consensus.md
  - fundamentals/31-messaging-patterns.md
  - advanced/02-multi-region-replication.md
---

# Kafka architecture

Kafka is a distributed append-only log used as a high-throughput event stream.

This document treats Kafka as a **case study**, not a product manual.

The goal is to understand the problems it was built to solve, why the architecture looks the way it does, what each decision costs, and which of those ideas transfer to other distributed systems.

## The problem Kafka was built to solve

Traditional message queues are designed to **hand work to a worker and forget it**:

- A message is deleted after it is acked
- Fan-out means copying the message into many queues
- Replay is awkward — the broker, not the consumer, owns progress
- Throughput is limited by random I/O, per-message acks, and broker-side routing logic

At scale, the need can be different: a durable, ordered history of events that many independent systems could read, at their own pace, and re-read when they broke or changed.

## Core abstraction: the distributed log

A Kafka **topic** is a named stream. It is split into **partitions**. Each partition is an ordered, immutable sequence of records, addressed by an **offset**.

```mermaid
graph TD
    T[Topic: orders] --> P0[Partition 0<br/>offsets 0..104]
    T --> P1[Partition 1<br/>offsets 0..98]
    T --> P2[Partition 2<br/>offsets 0..110]
```

Producers **append**. Consumers **read from an offset**. Nothing is removed because a consumer finished — records stay until **retention** expires (or compaction rewrites the log).

Why a log?

- Sequential writes are fast on disk — random writes are not
- Immutability makes replication and caching simpler: a written offset never changes
- Many readers can share one copy of the data, each with its own cursor
- The same structure is a natural WAL, changelog, and event source

**Trade-off:** the broker is no longer a work tracker — consumers must store progress, handle duplicates, and catch up after downtime. Disk and retention policy become first-class capacity planning.

The unifying idea (Jay Kreps' "The Log"): an ordered sequence of facts is a general building block for messaging, replication, and data integration.

## Anatomy of a cluster

Enough Kafka vocabulary to talk about the decisions:

- **Broker**: a node that stores partition replicas and serves produce/fetch
- **Topic / partition**: the shard of the log — the unit of ordering, parallelism, and replication
- **Record**: key, value, timestamp, headers — the key chooses the partition
- **Replica**: a copy of a partition on a broker. One replica is the **leader** and the others are **followers**
- **Controller**: a single elected broker that assigns leaders and reacts to membership changes
- **Consumer group**: a set of consumers that jointly read a topic, with each partition owned by one member

```mermaid
graph LR
    PR[Producers] --> L0[Broker 1<br/>leader P0]
    PR --> L1[Broker 2<br/>leader P1]
    L0 --> F0[Broker 2<br/>follower P0]
    L1 --> F1[Broker 3<br/>follower P1]
    L0 --> C[Consumers]
    L1 --> C
```

The cluster is a **shared-nothing** collection of partition leaders: scale by adding partitions and brokers, not by making one log bigger.

## Partitioning: parallelism without global order

**Problem:** a single totally ordered log cannot use more than one disk, one leader, or one consumer at a time. Global order is also rarely required.

**Kafka's choice:** hash the record key to a partition (`hash(key) % N`). Order is guaranteed **inside a partition**, not across the topic.

```mermaid
graph LR
    K1["key=order-123"] --> H[Hash]
    K2["key=order-456"] --> H
    H --> P0[Partition 0]
    H --> P1[Partition 1]
    H --> P2[Partition 2]
```

All events for `order-123` land on the same partition, so a consumer sees them in produce order. Events for different orders can be processed in parallel.

Trade-offs:

- Throughput scales with partition count (and with brokers that host them)
- A hot key creates a hot partition — extra consumers cannot split that key's work
- Changing `N` remaps keys under modulo hashing, so partition count is treated as relatively stable
- Max useful consumers in a group equals the number of partitions

Kafka uses simple modulo hashing for record placement, not [consistent hashing](../fundamentals/16-hashing.md). Partition count is an operator decision, not a constantly changing node set.

Consistent hashing would reduce remapping when `N` changes, at the cost of a more complex mapping and less even load for small `N`.

Design rule: pick a partition key with high cardinality and even access. If you need global order, you have chosen a single partition — and a single-threaded bottleneck.

## Replication: leader, followers, and ISR

**Problem:** a partition on one broker is a single point of failure. Replicating every append synchronously to every replica makes writes as slow as the slowest, most distant disk.

**Kafka's choice:** Producers and consumers talk only to the **leader**. Followers pull the log and catch up.
Durability is defined against the **in-sync replica set (ISR)** — replicas that have caught up within a timeout — not against a static majority of all replicas.

```mermaid
sequenceDiagram
    participant P as Producer
    participant L as Leader
    participant F as Follower
    participant C as Consumer

    P->>L: Append records
    F->>L: Fetch replica data
    F-->>L: Caught up (in ISR)
    L-->>P: ACK (acks=all)
    Note over L: High watermark advances
    C->>L: Fetch up to high watermark
```

Two offsets matter:

- **Log end offset**: the latest record the leader has written
- **High watermark**: the latest record fully replicated to the ISR. Consumers may read only up to this point, so they never see data that could disappear in a leader failover

This is the same idea as a consensus log's commit index: **do not expose uncommitted suffix**.

**ISR vs a fixed quorum:**

Classic [consensus](../fundamentals/29-consensus.md) (Raft) commits when a **majority of voting members** persist the entry.
Kafka commits when **all current ISR members** persist it, and ISR membership itself shrinks when a replica lags.

Pros:

- Writes are not blocked by a permanently slow replica — it is removed from ISR
- You can have `replication.factor=3` and still ack after 2 caught-up replicas

Cons:

- If ISR shrinks to one, `acks=all` no longer means "replicated"
- This is why `min.insync.replicas` exists: a floor on how small ISR is allowed to be before the leader rejects writes

Kafka's ISR is a **dynamic durability set**: optimize for throughput by excluding laggards, then re-introduce a minimum-quorum guardrail so availability does not silently eat durability.

## Acknowledgments: latency vs durability

The producer chooses how many replicas must persist the record before the write is considered successful.

- `acks=0`: Fire-and-forget — can lose data on any failure
- `acks=1`: Leader has written locally — lost if leader dies before followers catch up
- `acks=all`: All ISR members have the record — survives leader failure, subject to `min.insync.replicas`

This is the same knob as [sync vs async replication](../fundamentals/23-database-replication.md) and as RPO in [multi-region](./02-multi-region-replication.md): wait for more copies, or return faster and accept a loss window.

**Unclean leader election** is the failure-mode version of the same trade-off.
If every ISR replica is gone, should the controller promote an out-of-sync replica (availability, possible data loss) or keep the partition offline (durability, downtime)? Kafka's default is to refuse unclean election.

## Consumers own the cursor

**Problem:** if the broker deletes a message on ack, only one consumer can see it, and nobody can replay.

**Kafka's choice:** the log stays. Each consumer (or group) stores an **offset** — a checkpoint of "I have processed up to here."

```mermaid
graph LR
    LOG[Partition log<br/>0 1 2 3 4 5 6 7]
    G1[Group billing<br/>offset 5]
    G2[Group analytics<br/>offset 2]
    LOG --> G1
    LOG --> G2
```

Independent groups read the same partition at different speeds. A buggy consumer rewinds its offset and reprocesses. A new service starts at the latest offset, or at the beginning, without asking producers to change.

**Pull, not push:** consumers fetch when they are ready. That is natural [backpressure](../fundamentals/31-messaging-patterns.md).
A slow consumer lags; it does not force the broker to buffer per-consumer in memory or block producers (until disk/retention fills).

**Trade-offs:**

- Lag is a first-class metric; "the queue is empty" is the wrong mental model
- At-least-once is the easy default: crash after processing but before committing the offset, and you will see the record again
- Exactly-once requires extra machinery (see below)
- Retention bounds how far you can rewind; after that, the data is gone

**Elsewhere:** Flink/Spark checkpoints, CDC offsets, replica positions in DB replication, any consumer that stores "high-water mark" rather than relying on the broker to forget work.

## Consumer groups: exclusive ownership, not competing consumers

**Problem:** you want both **fan-out** (many systems see every event) and **parallel processing** (one system's work split across workers) without breaking per-key order.

**Kafka's choice:**

- **Different groups** = fan-out. Each group independently reads the full stream
- **Members of one group** = parallelism. Each partition is assigned to **at most one** consumer in that group

```mermaid
graph TD
    T[Topic partitions P0 P1 P2]
    T --> G1C1[Group payments / consumer A<br/>owns P0 P1]
    T --> G1C2[Group payments / consumer B<br/>owns P2]
    T --> G2C1[Group search / consumer A<br/>owns P0 P1 P2]
```

This is **partition ownership**, not [competing consumers](../fundamentals/31-messaging-patterns.md) on the same message. Two workers in the same group will never process the same offset.
That preserves per-partition order at the cost of a hard parallelism cap: extra consumers in a group sit idle once every partition has an owner.

When membership changes, the group **rebalances** — partitions are reassigned. A rebalance is a correctness event (ownership must be exclusive) and an availability event (processing pauses or stutters).
Cooperative / incremental rebalancing reduces the pause, but the invariant stays: one owner per partition per group.

**Elsewhere:** shard ownership in databases, Kubernetes controller-manager leader per object, "sticky" work assignment vs work stealing.
If you need finer parallelism than your shard count, you must reshard — the same constraint as a sharded database.

## Retention and compaction

**Problem:** a log that grows forever will fill the cluster. A log that deletes on consume cannot serve new readers.

**Kafka's choice:** retention is a **policy** on the log, independent of consumers.

- **Time or size retention**: drop old segments (typical for event streams)
- **Log compaction**: for keyed data, keep the **latest value per key** and discard older ones

Compaction turns a partition into a replicated changelog: replay it and you reconstruct current state. Kafka Streams state stores and compacted "table" topics use this.

```mermaid
graph LR
    subgraph Before
    B1["k1=A"] --> B2["k2=B"] --> B3["k1=C"] --> B4["k2=D"]
    end
    subgraph After compaction
    A1["k1=C"] --> A2["k2=D"]
    end
    Before --> After
```

**Trade-offs:**

- Retention is a product decision (how long can we replay?) and a cost decision (disk)
- Compaction is not a general-purpose database: it is last-write-wins per key, with no secondary indexes
- Consumers that lag beyond retention **skip data**; they do not block producers
- Tombstones (null values) exist so compaction can delete keys

**Elsewhere:** LSM-tree compaction, event-sourced snapshots plus a suffix of events, CDC tables that keep current row state, Redis AOF rewrite. Snapshot + log suffix is the general way to bound replay time.

## Why sequential I/O makes it fast

Kafka's throughput is not mainly a clever in-memory data structure. It is a storage design that stays on the fast path of modern kernels and disks.

- **Append-only segments**: each partition is a sequence of large segment files plus offset/timestamp indexes. Writes are sequential
- **OS page cache**: Kafka treats the kernel cache as its serving layer instead of a large JVM heap. Recently written data is often still in RAM when consumers catch up
- **Zero-copy fetches**: `sendfile` (or equivalent) moves bytes from page cache to the socket without copying into user space
- **Producer batching and compression**: many records become one request and one compressed chunk, amortizing syscalls and network

```mermaid
graph LR
    D[Disk segments] --> PC[Page cache]
    PC -->|sendfile| S[Network socket]
    S --> C[Consumer]
```

**Trade-off:** sequential throughput assumes consumers mostly read recent data (the cache-friendly case). Cold reads from old segments compete for disk.
Huge heaps would fight the page cache; Kafka deliberately keeps the JVM small relative to RAM.

**Elsewhere:** LSM trees (sequential flushes), WAL-first databases, nginx static file serving, any system that asks "can we turn random work into sequential appends and let the OS cache do the rest?"

Batching is the same latency vs throughput trade-off as Nagle's algorithm or Redis pipelining: wait a few milliseconds, send more bytes per round trip.

## Control plane vs data plane

**Problem:** partition placement, leader election, and membership need strong consistency. The data path needs to append millions of records per second. One mechanism is a bad fit for both.

**Kafka's choice:** split the cluster in two.

- **Data plane**: produce/fetch against partition leaders. Replication is leader-follower + ISR, optimized for throughput
- **Control plane**: cluster metadata (topics, assignments, leader identity). Historically ZooKeeper; now **KRaft** (Raft inside Kafka)

A single **controller** is elected via that strongly consistent metadata log. It watches broker liveness and moves partition leadership. Producers do not run Raft for every record.

```mermaid
graph TD
    subgraph Control plane
        M[Metadata log / KRaft] --> CTL[Controller]
    end
    subgraph Data plane
        CTL -->|assign leaders| B1[Broker leaders]
        CTL --> B2[Broker leaders]
        PR[Producers / consumers] --> B1
        PR --> B2
    end
```

**Why this split:** [consensus](../fundamentals/29-consensus.md) is expensive on the latency path. Use it for a small, critical dataset (who is leader, what is the schema of the cluster), and use cheaper replication for bulk data.

**Trade-off:** two failure domains. If metadata is unavailable, you cannot create topics or elect new leaders, even if existing leaders can still serve produce/fetch for a while.
If you put consensus on the data path instead, every append pays majority round-trips.

**Elsewhere:** Kubernetes (etcd vs kubelet), SDN controllers vs switches, Spanner (placement/metadata vs tablet serving), almost every large storage system.
"Consensus for control, simpler replication for data" is one of the most reusable architecture rules in this repository.

[Leader election](../fundamentals/28-leader-election.md) here is not optional decoration: a partition with two leaders splits the log.
Kafka fences stale leaders with **leader epochs** so a revived old leader cannot append into a fork. Fencing tokens after failover are the same idea as in multi-region promotion.

## Exactly-once as a composition

Kafka's default is **at-least-once**: retry produces, retry fetches, commit offsets after processing. Duplicates happen. See [delivery semantics](../fundamentals/31-messaging-patterns.md).

"Exactly-once" is not a broker flag. It is several mechanisms stacked so that a retry cannot create a second visible effect:

1. **Idempotent producer**: each producer gets an ID; each partition tracks a sequence number. The broker drops retries with an old sequence, so network retries do not duplicate records **within a session**
2. **Transactions**: a producer can atomically commit writes to several partitions, and atomically include the consumer offset commit. Downstream readers with `read_committed` skip aborted data
3. **Fencing**: a new transactional producer with the same `transactional.id` fences the old one, so a restarted process cannot commit after its successor has taken over

```mermaid
sequenceDiagram
    participant App
    participant Kafka
    App->>Kafka: Begin txn
    App->>Kafka: Produce to output partitions
    App->>Kafka: Commit offsets in the same txn
    App->>Kafka: Commit txn
    Note over Kafka: Readers with read_committed<br/>see output only after commit
```

**Trade-offs:**

- Higher latency and more coordinator work than fire-and-forget
- Exactly-once is end-to-end only if **every side effect** is in the transaction (or is idempotent). A send-email call outside Kafka is still at-least-once
- Idempotent producer without transactions still does not make a consumer's external DB write exactly-once

**Elsewhere:** database transactions plus outbox ([Transactional Outbox](../architecture/07-transactional-outbox.md)), idempotency keys on APIs, fencing tokens in leader failover,
"effectively once" = at-least-once + idempotent handlers.
Always ask *which* hop is exactly-once, not whether the product brochure says the words.

## What Kafka is not

Using Kafka as a case study also means knowing when the architecture is the wrong template.

- **Not a work queue.** No competing consumers on the same offset, no per-message visibility timeout as the primary API. Use a queue when a *task* must be processed once by any worker
- **Not a request-reply bus.** You can build request-reply on topics; you fight the model. RPC is still the right tool for low-latency, strongly consistent answers
- **Not a database.** Compacted topics are changelogs. They do not give you ad-hoc queries, secondary indexes, or multi-row transactions across arbitrary keys
- **Not cheap to operate at small scale.** Partition count, retention, rebalances, and consumer lag are operational surface area that a managed queue may not earn

## Pattern catalog

These are the transferable decisions, independent of Kafka APIs.

| Pattern                        | What Kafka does                              | Use the same idea when                                       |
| ------------------------------ | -------------------------------------------- | ------------------------------------------------------------ |
| Append-only log                | Partition = immutable sequence               | You need replay, replication, or an audit of facts           |
| Shard by key                   | Hash key to partition                        | You need per-entity order and horizontal scale               |
| Leader-follower + commit index | Leader writes, ISR defines high watermark    | You want one writer per shard and lagging replicas           |
| Dynamic durability set         | ISR shrinks; `min.insync.replicas` floors it | Slow replicas should not stall writes, but RPO still matters |
| Consumer-owned cursor          | Offset commits                               | Multiple readers, replay, independent progress               |
| Exclusive shard ownership      | One consumer per partition per group         | Parallelism without breaking order                           |
| Retention policy               | Time/size/compaction                         | Bound storage without coupling to consumers                  |
| Sequential I/O + page cache    | Segments, sendfile, small heap               | Throughput is the goal and the workload can be append-heavy  |
| Control vs data plane          | KRaft/controller vs produce/fetch            | Metadata must be consistent; bulk data must be fast          |
| Fencing                        | Leader epochs, transactional IDs             | Failover must stop the old primary from writing              |
| Composed exactly-once          | Idempotent produce + txn + offset commit     | Retries exist and side effects must not double-apply         |
| Pull + lag                     | Consumers fetch; monitor lag                 | Backpressure should sit at the consumer, not in broker RAM   |

## Design guidelines

- Start from the workload: event history and fan-out (log) vs one-shot tasks (queue)
- Choose a partition key for even load and the ordering you actually need
- Set `acks` and `min.insync.replicas` as an explicit RPO decision, not as defaults you never looked at
- Size partitions for consumer parallelism, then treat that count as hard to change
- Monitor consumer lag, ISR shrinks, and under-replicated partitions as health, not as trivia
- Design consumers as idempotent even if you enable transactions
- Keep consensus off the record path; use it for membership and leadership
- Bound replay with retention or snapshots; do not assume infinite history

## Interview talking points

- Kafka is a **partitioned, replicated commit log**, not a smart queue. That single choice explains fan-out, replay, and consumer offsets.
- Order is **per partition**. Name the key and the hot-key failure mode.
- Durability is **ISR + acks + min.insync.replicas**, not "replication factor = safe."
- Consumer groups give **ownership-based parallelism**; extra instances do nothing without extra partitions.
- Exactly-once is a **composition** (idempotency, transactions, fencing), and it stops at the boundary of Kafka.
- Call out **control plane vs data plane**: Raft for metadata, leader-follower for the log.
- Contrast with RabbitMQ/SQS: delete-on-ack vs retain-and-cursor; competing consumers vs partition assignment.

## Reference materials

- [The Log: What every software engineer should know about real-time data's unifying abstraction][the-log]
- [Kafka: a Distributed Messaging System for Log Processing Paper](https://www.microsoft.com/en-us/research/wp-content/uploads/2017/09/Kafka.pdf)
- [Apache Kafka Documentation - Design](https://kafka.apache.org/documentation/#design)

[the-log]: https://www.linkedin.com/blog/engineering/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying
