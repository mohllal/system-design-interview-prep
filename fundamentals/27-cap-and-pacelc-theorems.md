---
title: "CAP and PACELC theorems"
concepts:
  - cap-theorem
  - pacelc-theorem
  - partition-tolerance
  - cp-vs-ap-tradeoffs
  - tunable-consistency
  - eventual-consistency
  - quorum-based-systems
related:
  - fundamentals/08-availability.md
  - fundamentals/20-non-relational-databases.md
  - fundamentals/23-database-replication.md
  - fundamentals/28-leader-election.md
  - fundamentals/29-consensus.md
---

# CAP and PACELC theorems

CAP and PACELC describe trade-offs in distributed systems, especially under failure and load.

They help explain why "strong consistency and always-on low latency everywhere" is not realistic.

## CAP theorem

In a partition, a distributed system must choose between:

- **Consistency (C)**: Reads reflect the latest write (or fail)
- **Availability (A)**: Every request gets a non-error response
- **Partition tolerance (P)**: System continues despite network splits

Important nuance:

- In real distributed systems, partitions eventually happen, so **P is not optional**.
- During a partition, you effectively choose between **C** and **A**.

### CAP classifications

| Type | Behavior during a partition                                                                                       | Examples                                               |
| ---- | ----------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------ |
| CP   | Favors consistency over availability. May reject or time out writes/reads to avoid stale responses.               | Strongly consistent metadata stores, many RDBMS setups |
| AP   | Favors availability. Keeps responding, possibly with stale data, and converges later (eventual consistency).      | Dynamo-style KV stores, Cassandra (default posture)    |
| CA   | Only practical on single-node/non-partitioned deployments; not a realistic distributed, partition-tolerant model. | —                                                      |

## Example systems

### MongoDB: CP

MongoDB replicates data using **one primary and several secondaries**. Only the primary accepts writes. The secondaries copy from it.

Now lose the primary, either because it crashed or because a partition cut it off from the rest.

The remaining nodes hold an election and promote a new primary. During that window, **writes are not accepted** — the system halts them until a new primary exists.

That pause is the point. MongoDB will not let a second node start accepting writes while the old primary might still be alive, because then two primaries would be taking conflicting writes.

### Cassandra: AP

Cassandra is built the other way around. It has no single primary. **Any node can accept a read or a write at any time**.

When a partition happens, every side simply keeps working. Writes land on whichever nodes are reachable, and the copies drift apart. Cassandra then reconciles them afterwards, using timestamps, hinted handoff, and repair jobs.

The result is a system that would rather hand you older data than fail your request. That is the definition of AP.

Cassandra also offers **tunable consistency**: you can specify, per query, how many replicas must acknowledge a read or a write.

### ZooKeeper: CP

ZooKeeper is a coordination service. Systems use it for configuration, leader election, and distributed locks — jobs where two different answers would be a disaster.

It runs on an algorithm called **Zab**, which needs a **majority quorum to make progress**. When a network split leaves one side without a majority, that side stops functioning.

It refuses to answer rather than serve a state it cannot confirm is current.

### DynamoDB: AP

DynamoDB replicates data across availability zones and keeps serving even when some replicas or network links fail. Availability is the priority.

**Reads are eventually consistent by default**. A read right after a write may return slightly stale data, and after a partition it may take a moment for everything to line up.

DynamoDB does offer **strongly consistent reads within a region** as an option, but you have to ask for them, and they cost more and can fail when the eventually consistent path would have answered.

The default tells you the design intent: **operations should succeed even when parts of the infrastructure are having a bad day**, and any disagreement gets sorted out afterwards.

## PACELC extension

PACELC adds normal-operation (when there is no partition and the system is healthy) trade-offs:

- **If partition (P)**: Choose A or C (same CAP idea)
- **Else (E)**: Choose latency (L) or consistency (C)

So even without partitions, systems often trade consistency for speed.

Examples:

- **AP/EL**: Available under a partition, optimized for low latency normally
- **CP/EC**: Consistent under a partition, stronger consistency even in the normal path

## Practical interpretation

Do not treat CAP labels as rigid product tags — they describe design priorities under failure and load.

Questions to ask:

- Can this read be stale?
- Can this write be rejected during a partition?
- Is p99 latency more important than strict freshness?

## Common misconceptions

| Misconception                | Reality                                                      |
| ---------------------------- | ------------------------------------------------------------ |
| "Pick any 2 of 3, always"    | Oversimplified — a partition forces a choice between C and A |
| "AP means no consistency"    | False — usually eventual consistency with convergence        |
| "Strong consistency is free" | False — coordination adds latency and fragility              |

## How to choose in interviews

Choose **CP-ish** when:

- Financial balances, inventory invariants, leader metadata
- Incorrect stale reads/writes are unacceptable

Choose **AP-ish** when:

- Feeds, analytics, recommendations, activity streams
- Availability and responsiveness matter more than immediate global consistency

Use explicit mechanisms either way:

- Versioning, quorum reads/writes, conflict resolution, idempotency, repair jobs

## Interview talking points

- Explain CAP with a partition scenario, not abstract definitions.
- Is it worse for this system to show wrong data, or to show nothing at all?
- Use PACELC to discuss latency vs consistency in normal operation.
- Map system components to different guarantees (metadata vs user feed vs analytics).
- Mention convergence/repair strategy for AP designs.

## Reference materials

- [Brewer's CAP paper](https://users.ece.cmu.edu/~adrian/731-sp04/readings/GL-cap.pdf)
- [PACELC theorem (Daniel Abadi)](https://dbmsmusings.blogspot.com/2010/04/problems-with-cap-and-yahoos-little.html)
