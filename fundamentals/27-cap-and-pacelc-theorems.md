---
title: "CAP and PACELC theorems"
concepts:
  - cap-theorem
  - pacelc-theorem
  - partition-tolerance
  - cp-vs-ap-tradeoffs
  - linearizability
  - tunable-consistency
  - eventual-consistency
related:
  - fundamentals/08-availability.md
  - fundamentals/20-non-relational-databases.md
  - fundamentals/23-database-replication.md
  - fundamentals/28-leader-election.md
  - fundamentals/29-consensus.md
---

# CAP and PACELC theorems

CAP and PACELC name the trade-offs a distributed system cannot design its way out of. CAP describes what happens during a network partition; PACELC extends the same reasoning to the normal case, when the network is healthy.

Together they explain why "strongly consistent, always available, and fast everywhere" is not a system you can build.

## CAP theorem

CAP defines three properties:

- **Consistency (C)**: Every read returns the most recent committed write, or an error. This is linearizability — it is not the "C" in ACID.
- **Availability (A)**: Every request to a non-failing node gets a non-error response. The response is not required to be current.
- **Partition tolerance (P)**: The system keeps operating when the network drops or arbitrarily delays messages between nodes.

The important nuance is that these are not three equal options:

- In any real network, partitions eventually happen, so **P is not something you choose** — it is a condition you are handed.
- The actual decision is what to do **during** a partition: keep answering (A) or keep answers correct (C).

### The choice only exists during a partition

Take a 5-node cluster split across two racks, 3 nodes in rack A and 2 in rack B. The link between the racks fails, so neither side can reach the other. Each side is still healthy and still receiving client traffic.

- **A CP system** lets only the majority side (rack A) make progress. Clients talking to rack B get errors or timeouts until the partition heals.
- **An AP system** lets both sides keep answering. Rack B serves data that may already be stale and accepts writes that will have to be reconciled with rack A afterwards.

Two things follow that are easy to get wrong in an interview:

- **CP does not mean "the system is down"**. The majority side keeps serving normally. It is the minority side, and the clients pinned to it, that lose availability.
- **AP does not mean "no consistency"**. It means consistency is restored after the fact, through convergence and repair, rather than enforced before the response.

### CAP classifications

| Type | Behavior during a partition                                                                                       | Examples                                                   |
| ---- | ----------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------- |
| CP   | Favors consistency. Rejects or times out requests on the side that cannot confirm it holds current state.         | ZooKeeper, etcd, Consul, primary-replica RDBMS setups      |
| AP   | Favors availability. Keeps responding, possibly with stale data, and converges afterwards (eventual consistency). | Dynamo-style key-value stores, Cassandra (default posture) |
| CA   | Only meaningful for a single node or a network that never partitions. Not a realistic distributed model.          | Single-node PostgreSQL or MySQL                            |

## Example systems

### MongoDB: CP

MongoDB replicates data using **one primary and several secondaries**. Only the primary accepts writes; secondaries copy from it.

Now lose the primary, either because it crashed or because a partition cut it off from the rest of the replica set.

The remaining nodes run a [leader election](./28-leader-election.md) and promote a new primary. During that window, **writes are not accepted** — the system halts them until a new primary exists.

That pause is the point. A primary that can no longer reach a majority of the replica set steps down, so it cannot keep taking writes while a new primary is being chosen. Two primaries accepting conflicting writes would be far worse than a few seconds of write unavailability.

### Cassandra: AP

Cassandra is built the other way around. It has no single primary. **Any node can accept a read or a write at any time**.

When a partition happens, every side simply keeps working. Writes land on whichever nodes are reachable and the copies drift apart. Cassandra reconciles them afterwards using timestamps, hinted handoff, and repair jobs.

The result is a system that would rather hand you older data than fail your request. That is the definition of AP.

Cassandra also offers **tunable consistency**: per query, you choose how many replicas must acknowledge a read or a write. Setting both to a majority of replicas pushes an individual query toward CP behavior — see [read and write quorums](./29-consensus.md) for why the overlap matters.

### ZooKeeper: CP

ZooKeeper is a coordination service. Systems use it for configuration, leader election, and distributed locks — jobs where two different answers would be a disaster.

It runs an atomic broadcast protocol called **Zab** (see [Consensus](./29-consensus.md)), which needs a **majority quorum** to make progress. When a network split leaves one side without a majority, that side stops serving writes rather than serve state it cannot confirm is current.

Concretely, in a 5-node ZooKeeper ensemble split 3/2, the 3-node side elects a leader and keeps working, and the 2-node side rejects writes for as long as the split lasts.

### DynamoDB: AP

DynamoDB replicates data across availability zones and keeps serving even when some replicas or network links fail. Availability is the priority.

**Reads are eventually consistent by default**. A read right after a write may return slightly stale data, and after a partition it may take a moment for everything to line up.

DynamoDB does offer **strongly consistent reads within a region** as an option, but you have to ask for them, they cost more, and they can fail in situations where the eventually consistent path would have answered.

The default tells you the design intent: operations should succeed even when parts of the infrastructure are having a bad day, and any disagreement gets sorted out afterwards.

## PACELC extension

CAP only says something about the partitioned case, which is rare. PACELC covers the other 99% of the time as well:

- **If there is a partition (P)**: Choose availability (A) or consistency (C) — this is exactly the CAP choice.
- **Else (E)**, when the network is healthy: Choose latency (L) or consistency (C).

The second half is the part CAP misses. Even with a perfectly healthy network, keeping replicas in agreement costs round trips. A system that commits a write only after a majority of replicas acknowledge it pays that latency on every write; a system that acknowledges locally and replicates in the background does not, and accepts stale reads in exchange.

Classifications are written as a pair, such as `PA/EL` or `PC/EC`.

| System                              | PACELC | What that means in practice                                                                         |
| ----------------------------------- | ------ | --------------------------------------------------------------------------------------------------- |
| ZooKeeper, etcd, Consul             | PC/EC  | Minority side refuses writes; every write costs a majority round trip even when healthy             |
| MongoDB with majority write concern | PC/EC  | Primary steps down without a majority; writes wait for majority acknowledgement                     |
| Cassandra with default settings     | PA/EL  | Both sides of a partition keep serving; low-consistency-level reads answer from the nearest replica |
| DynamoDB with default reads         | PA/EL  | Keeps serving through failures; eventually consistent reads avoid the cross-replica round trip      |

These labels follow **configuration, not product names**. Cassandra with `QUORUM` reads and writes behaves much closer to PC/EC, and MongoDB with `w: 1` behaves much closer to PA/EL.

## Common misconceptions

| Misconception                            | Reality                                                                                                     |
| ---------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| "Pick any 2 of 3, always"                | Oversimplified — P is forced on you, and the choice between C and A only arises during a partition          |
| "AP means no consistency"                | False — usually eventual consistency, with explicit convergence and repair                                  |
| "A CP system is down during a partition" | False — the majority side keeps serving; only the minority side loses availability                          |
| "CAP's C is the C in ACID"               | False — CAP's C is linearizability across replicas; ACID's C is invariant preservation within a transaction |
| "Strong consistency is free"             | False — coordination costs round trips and reduces write availability                                       |

## Applying this in a design

CAP labels are not product tags. They describe design priorities under failure and load, and different components of the same system will land in different places.

Ask three questions per component:

- Can this read be stale, and for how long?
- Can this write be rejected or delayed during a partition?
- Is p99 latency more important than strict freshness?

Choose **CP-ish** for financial balances, inventory counts, cluster membership, and leader metadata — anywhere a stale read or a conflicting write causes real damage.

Choose **AP-ish** for feeds, analytics, recommendations, and activity streams — anywhere availability and responsiveness matter more than immediate global agreement.

Either way, name the mechanism that backs the choice: majority quorums and consensus for CP, and versioning, conflict resolution, idempotency, and repair jobs for AP.

## Interview talking points

- Explain CAP through a concrete partition scenario (a 5-node cluster split 3/2), not abstract definitions.
- Frame the trade-off as a product question: is it worse for this system to show wrong data, or to show nothing at all?
- Use PACELC to move the discussion to the normal path, where the latency-versus-consistency choice is made on every request.
- Map individual components to different guarantees — cluster metadata CP, user feed AP, analytics AP.
- For any AP design, say how convergence happens: version metadata, conflict resolution, read repair, anti-entropy.

## Reference materials

- [Brewer's CAP paper](https://users.ece.cmu.edu/~adrian/731-sp04/readings/GL-cap.pdf)
- [PACELC theorem (Daniel Abadi)](https://dbmsmusings.blogspot.com/2010/04/problems-with-cap-and-yahoos-little.html)
