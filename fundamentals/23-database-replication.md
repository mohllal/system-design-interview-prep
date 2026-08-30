---
title: "Database replication"
concepts:
  - synchronous-replication
  - multi-primary-replication
  - leaderless-replication
  - quorum-reads-writes
  - failover
  - replica-lag
  - conflict-resolution
  - change-data-capture
related:
  - fundamentals/24-database-partitioning.md
  - fundamentals/27-cap-and-pacelc-theorems.md
  - fundamentals/28-leader-election.md
  - fundamentals/29-consensus.md
  - advanced/02-multi-region-replication.md
  - advanced/05-kafka-architecture.md
  - advanced/06-postgresql-internals.md
---

# Database replication

Replication keeps **copies of the same data** on more than one node.

What it buys is a durability choice, a failover candidate, extra read capacity with lag, and a place to hang backups, analytics, and CDC.

In production each **shard** is usually its own replica set. This note is about the replica set: how the log moves, who you wait for, who may write, and what happens when a node lies or dies.

## What you are choosing

Three knobs:

1. How many copies must exist before the client gets an ack (RPO vs write latency)
2. How many nodes may accept writes (conflicts vs write availability)
3. What a replica read is allowed to return (lag vs load on the primary)

If you cannot name those for a design, "we will add replicas" is not a design.

## How the bytes move

Most single-primary systems ship a **log of changes**, not "the database file" on a timer.

- **Physical / WAL shipping** (Postgres streaming, MySQL binlog in row format as a close cousin): the replica replays the same bytes the primary wrote.
- **Logical / row-level**: the replica applies row-level changes (or SQL statements). This lets you subscribe to a subset of the data, cross versions more easily, and feed CDC — the app-level version of a durable log of facts.

Statement-based replication (sending the raw `UPDATE` text) breaks on non-deterministic SQL (`NOW()`, `RAND()`, triggers), so it is rare in production.

If a replica disconnects, the primary must **retain** WAL until that replica consumes it (replication slots, binlog retention).
A dead replica with a slot held open fills the primary disk and takes the cluster down. TTL the slot or drop the replica — do not assume infinite retention.

Useful variants of "a replica":

- **Streaming hot standby**: can be promoted and often can serve reads.
- **Delayed replica** (hours behind): useful for recovery from a `DROP TABLE` or bad deploy that has already replicated everywhere else.
- **Cascading replica**: primary → hub replica → many tails, so the primary is not fan-out bound.

## Durability: who you wait for

The product decision is when the **client** is told the write is done.

```plaintext
async          client ← ack          replica gets the bytes later
                                     primary dies  →  lose the unshipped tail (RPO > 0)

wait for one   client waits until ≥1 replica has flushed it
                                     one extra copy, not "every replica in the AZ"

sync to all    client waits for every replica
                                     write latency = slowest replica; a dead replica stalls writes
```

Semi-sync ("wait for one") is the usual compromise: the ack is not local-only, and a single slow/far replica does not freeze the primary. Fully sync to all replicas is for a small, critical set of writes, not the whole OLTP path.

Sync replication: if you wait for a replica that is partitioned away, you either stall writes (choose C) or ack without that copy (choose A, RPO > 0).

## Who accepts writes

### One primary

The default option. It works as follows:

1. All writes go to one primary node
2. Replicas pull or receive the log
3. You get a single order of commits and no merge math

```plaintext
           writes
             │
             ▼
          primary  ──────log────►  replica
             ▲                         │
             └──────── reads ──────────┘   (lag if you read the replica)
```

Writes serialize here. Replicas scale **reads** and give you a promotion candidate, but they do not scale writes.

This is MySQL/Postgres streaming replication, MongoDB replica sets, Kafka's partition leader.

Clients must find the current primary after failover: a proxy (ProxySQL, PgBouncer with a pause), a VIP, or a driver that watches the replica set.

### More than one primary

Two nodes (often two regions) both take writes so users are not shipped across an ocean for every `UPDATE`. Concurrent updates to the same row **conflict**.

Merge rules, in increasing honesty:

- **Last-write-wins**: last timestamp wins. Clock skew silently drops a write. Fine for "last profile save," bad for inventory.
- **Version vectors / causality**: detect concurrent writes, but you still need a rule or a human for the concurrent case.
- **Application merge**: union of tags, max(`last_login`), newest shipping address — you state the rule per entity.
- **Conflict-Free Replicated Data Types (CRDTs)**: counters, sets, some text — types designed to commute. Not your order table.

Add-only logs (append events, never update in place) conflict less than "update the same row from two regions." That is why event logs and CRDT-ish counters show up in multi-primary designs.

Use this when local writes are a product requirement and you can state the merge rule. Do not use it as "HA, so no primary." You traded a failover problem for a **conflict** problem.

### Leaderless (quorum)

The client (or coordinator) writes to several nodes and waits for **W** acks while reads wait for **R**.

If `R + W > N`, a read quorum overlaps the last write quorum, so you can see the latest value *if you also pick the newest version among the responses*. Overlap alone is not enough — see [consensus](./29-consensus.md).

```plaintext
N = 3 replicas of the same key

W = 2, R = 2     write hits 2, read hits 2 → they intersect
W = 1, R = 1     write and read can miss each other → stale reads are allowed on purpose
```

You bought write availability without a single leader. You paid for the repair path:

- **Read repair**: a read that sees mixed versions writes the newest value back to the stale nodes.
- **Hinted handoff**: a node that missed a write is told later (the hint).
- **Anti-entropy / repair jobs**: background Merkle-tree comparisons so silent drift does not last forever.
- **Sloppy quorum**: during a partition you write to whoever is reachable, not the "home" replicas, then hand off later. Availability goes up; more repair happens later.

The consistency story is a pair `(R, W)`, not "the database is strongly consistent" — this is the Cassandra/Dynamo style. An odd `N` (3, 5) is about how many node failures you tolerate.

Replica-set **elections** (MongoDB, etcd, Kafka KRaft) are a different quorum: majority to pick a **leader**, then that leader orders writes. Dynamo `R/W` is per-request agreement on a value. Do not mix them in one sentence.

## Failover is the hard part

A replica existing is not failover. Failover is: decide the primary is gone, pick a successor that has the **needed log**, **stop the old primary from writing**, and point clients at the new one.

Promote the replica that is **most caught up**, not a random one. An async replica promoted to primary **loses** whatever was not replicated — the RPO you already chose.
Promoting a far-behind replica because it is the only one left is unclean leader election under another name.

After promotion, replicas that were ahead of the new primary on some fork must **rewind** or be rebuilt.

## Reading from replicas

Replica lag is seconds **and** bytes (or LSN delta), not a boolean.

- **Read-your-writes**: a user saves a profile; the next GET hits a replica 2s behind, so they see the old row.
  Fix: read from the primary after a write, stick the session to the primary, or wait until the replica's apply LSN is at least the write's LSN.
- **Monotonic reads**: do not bounce a user between a caught-up replica and a stale one — time would appear to go backwards. Pin the session to one replica or to the primary.
- **Linearizable reads**: must see the latest committed write cluster-wide. That means the primary, a sync replica, or a quorum read — not a random async replica.

Analytics, backups, and CDC are what replicas are *for* more often than "2× OLTP read QPS."

## Backups are not replicas

A replica is live: the same `DROP`, ransomware, or schema bug replicates. Backups and PITR (WAL archive) are a **point in time** you can restore that is not following the primary.

CDC consumers (Kafka, Debezium) are replicas of the log in another system. They have the same lag and "what is committed" questions. Use [outbox](../architecture/07-transactional-outbox.md) when the app must not dual-write.

## What replication does not do

- Fix a hot key. Copies of the same primary still serialize those writes.
- Replace backups — see above.
- Give you multi-region for free — 100–400ms sync is a product decision.
- Remove the primary as a write bottleneck unless you change **who accepts writes**.

## Order of moves

1. One primary, streaming replicas, WAL retention/slots bounded
2. State the ack policy (async / wait-for-one) as an RPO number, and put the waited-on replica in another AZ
3. Replica reads only where lag is acceptable — pin the rest to the primary
4. Delayed replica or PITR before you need to undo a replicated `DROP`
5. Rehearse failover, including fencing and client routing, before you need it
6. Second writable region or leaderless quorums only with an explicit conflict or `(R, W)` story
7. Shard when the primary cannot take the writes — copies will not save you

## Interview talking points

- Replication is **copies**, partitioning is **split**. Replica set per shard.
- Ack policy is **RPO vs write latency**. Async means you can lose the tail.
- Single primary is the default. Extra primaries mean conflicts. Leaderless means `(R, W)` and repair. Election quorum ≠ Dynamo quorum.
- Failover without **fencing** is split brain. Promote the most caught-up replica.
- Replica reads need a **lag** sentence (read-your-writes), not "reads scale."
- Slots/retention: a dead replica can fill the primary.

## Reference materials

- [Master-replica replication](https://arpitbhayani.me/blogs/master-replica-replication)
- [Multi-master replication](https://arpitbhayani.me/blogs/multi-master-replication/)
- [Leaderless replication](https://arpitbhayani.me/blogs/leaderless-replication)
