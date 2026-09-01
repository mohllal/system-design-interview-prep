---
title: "Database partitioning"
concepts:
  - partition-key-design
  - colocation
  - hash-based-partitioning
  - range-based-partitioning
  - directory-based-partitioning
  - global-secondary-indexes
  - distributed-transactions
  - resharding
related:
  - fundamentals/16-hashing.md
  - fundamentals/22-database-indexes.md
  - fundamentals/23-database-replication.md
  - fundamentals/25-database-concurrency-control.md
  - fundamentals/27-cap-and-pacelc-theorems.md
  - fundamentals/28-leader-election.md
---

# Database partitioning

Partitioning and replication solve different problems. Mixing them up is the usual design-interview miss.

- **Partitioning** (sharding when each partition has its own servers) puts *different* rows on different nodes. That is how you scale **writes, storage, and working set**.
- **[Replication](./23-database-replication.md)** puts the *same* rows on more than one node. That is how you survive a disk death, serve more **reads**, and choose an RPO.

A production database almost always does both:

- Replication without partitioning still funnels every write through one primary.
- Partitioning without replication means losing one machine loses a slice of users.

```plaintext
          shard A                    shard B                    shard C
     ┌──── primary ────┐        ┌──── primary ────┐        ┌──── primary ────┐
     │ replica  replica│        │ replica  replica│        │ replica  replica│
     └─────────────────┘        └─────────────────┘        └─────────────────┘
        copies of A                copies of B                copies of C
```

Read that as: sharding picks the **column**, replication fills the **box**. Each shard is a full replica set with its own primary, its own log, and its own failover, so everything in [replication](./23-database-replication.md) applies once per shard rather than once per cluster.

## What you are trying to keep together

The partition key is the unit of **locality**, not a hashing trivia question.

Anything that must be fast, consistent, or transactional together should share a key: one user's orders, one tenant's rows, one `device_id`'s time series.

That is **colocation** (entity groups in some stores): parent and children hashed on the same ID so a transaction never leaves the shard.

If you cannot name the key, you are not ready to shard. "Shard by user_id" is a claim that almost every query and every transaction is per-user.

**Compound keys.** Often only a prefix is the shard key (`(user_id, order_id)` stored together, hashed on `user_id`) — hashing the whole compound key splits one user's orders across nodes.

**Two different splits people conflate:**

| Split                  | Scope                                   | Wins                                                   | Cost                                                 |
| ---------------------- | --------------------------------------- | ------------------------------------------------------ | ---------------------------------------------------- |
| In-database partitions | One server, one WAL, one failover       | Prune old data, cheaper vacuum, drop a day in one shot | No write or storage scale                            |
| Shards                 | Separate servers, separate replica sets | Write and storage scale                                | Routing, no cross-shard FK, distributed transactions |

In-database partitioning (Postgres `PARTITION BY RANGE (day)`, MySQL partitions) is a **table layout** change: one node, one replica set, one failover. Sharding is a **topology** change: N replica sets, N of everything operational. Saying "we partitioned the table" when you mean "we sharded the fleet" hides all the hard parts.

Vertical split (users DB vs payments DB) is a **service/table** cut, not a row-key cut. It scales teams and blast radius but it does not by itself scale a hot `orders` table.

## Sharding techniques

### Hash

`shard = hash(key) % N` (or a [consistent hash](./16-hashing.md) ring).

You get even load if the key has high cardinality and no celebrity values, but range queries (`WHERE created_at BETWEEN …`) become scatter-gather.

`hash % N` remaps **most** keys when `N` changes. Consistent hashing / virtual nodes remaps about `1/N` of keys. Doubling `N` and splitting each shard in two is a planned special case of "N only grows by split."

Default for OLTP when you need even writes and can live without cross-key range scans.

### Range

Contiguous key ranges on each node (`user_id 1–1M` on shard 1). Range scans stay on one node. Autoincrement IDs and "latest day" time series all land on the **high end** — one hot shard, idle older shards.

Pre-split ranges (Bigtable/HBase/Spanner style) before the load exists, then **split** a hot range and **merge** cold ones. Unbounded "just append to the last range" is how time-series OLTP dies.

Use when the access pattern *is* a range. Do not use sequential keys for OLTP write scale.

### Directory

A lookup table maps key → shard (`tenant_id → cell`). You can move one tenant without hashing everyone.

The directory is a control plane: cache it on the client, replicate it, version it. A wrong cached mapping writes to the wrong shard.

Vitess-style and many multi-tenant "cell" designs are directories plus a proxy.

### Geography

Put EU rows in EU because of latency or law, not because hash was even. Cross-region queries and global uniqueness are the cost.

## Request routing

The app has to **know** which replica set owns the key.

- **Client-side**: hash in the app, connect to the right pool. Fast, embeds topology in every service.
- **Proxy / coordinator** (Vitess, Citus, Spanner's API, a smart load balancer): one SQL endpoint, routing in the middle. Easier for apps but the proxy is now in the availability path and must not become a single-node bottleneck.
- **Forwarding**: node receives a mis-routed query and proxies to the owner. Simple, extra hop.

Queries are **targeted** (key in the predicate → one shard) or **scatter** (no key → all shards, merge). p99 of scatter is the slowest shard plus merge.

```plaintext
targeted:  WHERE user_id = 42        → shard hash(42)
scatter:   WHERE status = 'open'     → all shards, merge, p99 = worst shard
```

## Secondary indexes

Local secondary index: `orders(user_id, created_at)` on the shard that already holds that user. Cheap, partition-local.

Global secondary index: "find user by email" when you've sharded by `user_id`. Email lives on an unknown shard, so you maintain another table/index sharded by `email` (or a dedicated lookup store).

That index is a **second dataset** to write, fail, and repair. If it cannot be wrong, dual-write it with the same rigor as an outbox pattern.

Foreign keys do not span shards. Enforce in the app or denormalize.

## Uniqueness and IDs

Unique constraints (`UNIQUE email`) are local to a shard unless you have a global index or a dedicated allocation service. "Unique in the universe" is a distributed problem.

IDs: per-shard autoincrement collides when you merge or when the client sees IDs from two shards. Use unique IDs that need no central counter (snowflake-style, ULID, UUID).

If you need roughly-increasing IDs for range scans, put time in the high bits and accept some skip, or keep the range on a key that is not the shard key.

## Cross-partition work

A transaction that touches two shards is a distributed transaction ([2PC](../patterns/05-two-phase-commit.md)): extra RTTs, coordinator failure, more ways to be unsure.

What it costs is isolation, not just latency. Each shard can be perfectly serializable on its own and the pair of them still be wrong, because no single node ever sees both halves of the transaction — cross-shard write skew is the standard example. See [concurrency control](./25-database-concurrency-control.md) for the anomalies themselves.

Prefer:

- Same partition key (colocate)
- [Saga](../patterns/06-saga.md) / outbox: accept eventual consistency between aggregates
- Denormalize so the read path does not join
- App-side join only for rare admin paths, with a timeout and a cap on keys

Spanner/F1-style databases make cross-shard SQL *possible* with TrueTime and a lot of engineering. That is not "MySQL + hash user_id + 2PC on Friday."

## Replication per shard

Each shard is a replica set, so the whole of [replication](./23-database-replication.md) — ack policy, RPO, lag, promotion, fencing — applies once per shard. Four consequences are specific to sharding:

- **Failover is per shard**: shard 7 can elect a new primary while shard 3 is untouched. A reported "cluster outage" is often one shard's replica set losing quorum, not every node.
- **Replicas do not fix a hot partition**: ten replicas of a celebrity key still serialize its writes on that key's one primary. Copies buy reads and a promotion candidate, never write capacity.
- **Lag is per shard too**: a scatter query reading replicas of ten shards is only as fresh as the laggiest one, and different shards can be at different points in time.
- **Leaderless stores still partition**: Cassandra tokens and Dynamo partitions split the keyspace. Quorum `R`/`W` is about copies of one key, not about avoiding sharding.

Operationally this is the cost people underestimate: `N` shards × `R` replicas is `N` replica sets to size, back up, page on, and fail over.

## Resharding without losing a shard

Typical live split:

1. Stand up the destination replica set (already replicated)
2. Backfill (dump/restore, logical copy, or catch-up from the log)
3. Dual-write both shards, or pause writes for that key range
4. Compare, cut reads, then drop the old range

Hash `% N` with a new `N` means almost everyone moves. Prefer **split in half** (range or hashed half), consistent hashing, or a directory move of a few tenants.

Keep the cluster **replicated the whole time**. Copying onto a single node "to be quick" is how a shard disappears mid-migration.

## How many shards?

Too few: you are back to one hot primary. Too many: each replica set has little data, scatter queries fan out to hundreds of nodes, connections and schema deploys multiply, and you still have hot keys.

Start from working-set RAM, disk, and write QPS **per shard**, plus headroom for the hottest shard (not the average). Plan to **split**, not to guess the perfect `N` forever.

## Schema and operations

There is no single lock, and no single transaction, across shards. Schema changes are rolling and expand-contract: add the column nullable everywhere, backfill shard by shard, switch the app, then drop the old column. A migration that must be atomic across all shards is a design smell, not a deploy problem.

Measure **per shard**, never as a cluster average: QPS, CPU, disk, replication lag, row count. The average shard is healthy in almost every incident caused by one shard.

## What breaks when you shard

- Hot keys: one partition key cannot be split by adding shards. Cache, queue, split the key (`user_id + bucket`), or use a bigger box / dedicated shard.
- Scatter-gather: an unkeyed list or filter query hits every shard — p99 is the slowest shard plus the merge.
- Rebalancing: hash-modulo resharding reshuffles the world. Plan `N` ahead of time, or use a ring/directory.
- IDs, uniqueness, FKs: distributed-systems problems once you shard (see above).

## Order of moves

1. Indexes, query shape, connection pooling, a bigger primary
2. Replicas for reads, failover, backups — still one write primary
3. Partition *inside* one database if the table is the problem (drop old partitions, prune)
4. Shard when writes or data no longer fit, with a key you can defend against the query list
5. Global indexes / sagas only for the access paths the key cannot serve
6. Multi-region only if users or the law require it — not as a substitute for (4)

Skipping to (4) because "scale" was on the rubric is how you inherit scatter-gather and a bad key.

## Interview talking points

- Name the **axis**: replication is copies, partitioning is split. Replica set per shard.
- Defend the **partition key** with a query and a transaction (colocation), not "hash user_id."
- **Hot keys** vs too many shards.
- Hash modulo vs split/consistent hashing for resharding.
- Failover, lag, and metrics are all **per shard**.
- Resharding must stay replicated.
- Cross-shard work costs **atomicity and isolation**, not just round trips — that is what colocation buys back.

## Reference materials

- [Sharding Pinterest: How we scaled our MySQL fleet](https://medium.com/pinterest-engineering/sharding-pinterest-how-we-scaled-our-mysql-fleet-3f341e96ca6f)
- [F1 / Spanner-style Paper](https://www.cs.princeton.edu/courses/archive/spring16/cos598F/f1-google.pdf)
