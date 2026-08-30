---
title: "Non-relational databases (NoSQL)"
concepts:
  - document-stores
  - key-value-stores
  - wide-column-stores
  - graph-databases
  - base-consistency-model
  - embedding-vs-referencing
  - partition-key-design
  - local-vs-global-secondary-index
  - read-your-writes-consistency
related:
  - fundamentals/19-relational-databases.md
  - fundamentals/21-sql-vs-nosql.md
  - fundamentals/22-database-indexes.md
  - fundamentals/23-database-replication.md
  - fundamentals/24-database-partitioning.md
  - fundamentals/27-cap-and-pacelc-theorems.md
---

# Non-relational databases (NoSQL)

"NoSQL" is not one product.

It is a set of stores that **drop or reshape** the relational contract (joins, global constraints, multi-row transactions) so a **specific access path** is cheap and easy to spread across machines.

Many of these systems now offer transactions **inside one partition or document**, but that is not the same as "ACID for any `JOIN` across the cluster."

This note covers what each model is good at and what it costs. For the model it is defined against see [Relational Databases](./19-relational-databases.md), and for choosing between the two see [SQL vs NoSQL](./21-sql-vs-nosql.md).

## Start from the access path

Every model below is a different answer to the same four questions:

1. What is the **lookup key**? (user id, session id, `(device_id, time)`)
2. Do you need **joins**, or is one blob enough?
3. Is a transaction **one document / one partition**, or many keys?
4. Do writes need to be **visible everywhere immediately**, or is lag acceptable?

If you mostly read by primary key and nest the rest, a document or key-value store matches the path.

If you join five tables and enforce foreign keys, a relational primary is still the default, and you can partition it later to scale writes.

## ACID and BASE

A relational engine gives ACID over a unit that spans **many rows in many tables** — see [Relational Databases](./19-relational-databases.md) for what each letter means.
**BASE** (Basically Available, Soft state, Eventual consistency) is the label usually attached to the stores here.
It is best read not as a different set of concerns but as the **same** concerns with two dials turned: the atomic unit shrinks, and consistency moves from `COMMIT` time to *eventually*.

| Concern                | Relational default (ACID)                             | Common non-relational default (BASE)                       |
| ---------------------- | ----------------------------------------------------- | ---------------------------------------------------------- |
| Unit of atomicity      | Many rows, many tables, one `COMMIT`                  | One key, one document, or one partition                    |
| Constraints            | Declared and enforced by the engine (PK, FK, `CHECK`) | Local at best; cross-partition uniqueness is the app's job |
| Read after write       | A committed write is immediately visible              | Replicas converge, on no promised schedule                 |
| During a partition     | Refuse the write rather than diverge                  | Stay writable and reconcile afterwards                     |
| Failure you design for | A transaction that rolled back                        | A stale read and two writes that have to be merged         |

### BASE is a slogan, not a law

The right-hand column is a **common default**, not a property of the category. Plenty of non-relational products keep a single primary and a narrow but strict transaction:

- **MongoDB**: Replica sets elect a single primary, so a failover looks more like CP than "always writable"
- **Redis**: Often one node, or a primary plus replicas
- **Cassandra**: Lets you choose `R` and `W` per request, so the same cluster can be strict or loose
- **Neo4j**: Typically a strongly consistent cluster with one write leader

Do not write "NoSQL = BASE = AP" on a whiteboard. Name the product's **default write path** and say what happens to it during a partition — see [CAP and PACELC](./27-cap-and-pacelc-theorems.md) for the framing and [Replication](./23-database-replication.md) for the quorum mechanics.

Eventual consistency means replicas **converge**. It does not say when. If a user must see their own write, you still need read-your-writes: route them to the primary, pin a session token, or size `R` and `W` so the quorums overlap.

## Document stores

**Unit:** a JSON-like document in a collection (MongoDB, CouchDB, DocumentDB).

Cheap:

- Fetch or update **one document** by `_id`
- Nested fields that you always load together (profile plus settings)

Painful:

- "All orders in the last hour across users" without a designed index and partition key
- Changing a username copied into a thousand comment documents
- Multi-document transactions across partitions (possible in some products, slow and limited)

**Embedding vs referencing:**

- **Embed** when the nested data is bounded and read with the parent (an address on a user).
- **Reference** (`userId`) when the nested set is large or shared (all of a user's orders).

A 16MB document full of unbounded arrays is a trap: you have reinvented a table, with worse querying and a rewrite of the whole document on every append.

Transactions start as **single-document atomic**. Treat anything wider as the same problem as a cross-row SQL transaction, plus the cost of coordinating partitions.

## Key-value stores

**Unit:** `GET` / `PUT` / `DEL` by key (Redis, Memcached, DynamoDB as a key-value and document hybrid).

Cheap:

- Session, cache, lock, leader election, rate-limit counter
- Very high QPS on a hot key when the value is small

Painful:

- Querying by value ("all keys whose JSON contains `city=Paris`")
- Secondary indexes, unless the product adds them — and then you are not in pure key-value territory anymore

The value is **opaque** to the store. If you need to filter on fields inside it, you either lift them into the key (`user:42:session`) or pick a different store.

Durability varies, and the difference matters more than the API:

- **Memcached**: Memory only, gone on restart
- **Redis**: Optional AOF or RDB persistence, plus replicas
- **DynamoDB**: Replicated disk, durable by default

Do not use "Redis" and "durable system of record" in the same sentence unless you have configured persistence and HA on purpose.

## Wide-column (column-family)

**Unit:** a partition key, then sparse columns grouped in families (Cassandra, HBase).

Cheap:

- Writing a lot of **cells** under one key (`(user_id, timestamp)` to event)
- Scanning a slice of columns that sort with the key (time series, inbox)

Painful:

- Ad-hoc `WHERE` on a column that is not part of the key or a designed index
- Changing the primary key later, which is a full migration
- Joins

You model **query first**: the partition key decides how data is spread, and the clustering columns decide the sort order inside a partition. Both are chosen before you write a row, and neither is cheap to change.

## Graph databases

**Unit:** nodes and edges (Neo4j, AWS Neptune).

Cheap:

- Variable-length walks ("friends of friends," fraud rings)
- Recursion that is painful to express and slow to run in SQL

Painful:

- Bulk analytics over every node, as if it were a warehouse
- Partitioning a densely connected graph, because every cut adds cross-node hops

Use one when the **relationship is the query**, not when you have a users table that happens to have a `follows` join you run twice.

## Time-series databases

**Unit:** `(metric, tags, timestamp)` to a value (InfluxDB, Prometheus, TimescaleDB on Postgres).

Cheap:

- Appending samples
- Aggregating `cpu` for `host=web1` over an hour
- Retention and downsampling as built-in policies rather than cron jobs

Painful:

- Updating a sample in the past as if it were a row in `users`
- High-cardinality tags (`user_id` on every metric), which multiply the number of stored series

If the data is "events over time" **and** the query is always time-bounded, use this shape even when the engine underneath is Postgres with hypertables.

## Secondary indexes and the second access path

The primary key is effectively free, because it decides where the data lives. **Every other query needs an index**, and in a partitioned store the index is partitioned too. The structures are the same ones a relational engine uses — see [Indexes](./22-database-indexes.md) — but partitioning gives them two shapes:

- **Local secondary index**: Each partition indexes only its own rows. Writes stay inside one partition, so they are cheap. A query that does not name the partition key has to ask **every** partition and merge the results (scatter-gather), which gets worse as the cluster grows.
- **Global secondary index**: The index is partitioned by the indexed column, so a lookup reaches one partition. The write now touches two partitions, so it is either slower or applied asynchronously — which is why DynamoDB's global secondary indexes are eventually consistent with the base table.

See [Partitioning](./24-database-partitioning.md) for how those indexes are spread and resharded.

When two access patterns both matter (fetch by id **and** list by email **and** join to payments), the realistic options are:

- Two stores, with one clearly the system of record (orders in Postgres, sessions in Redis)
- A global index, or a second collection written for that lookup and kept in sync
- Stay relational and index both paths

A single clever document schema rarely serves three incompatible queries well.

## Modeling: copy on purpose

These stores **denormalize** so that one read is one fetch:

- **Embed** related data in the document
- **Duplicate** a field into another collection keyed for that lookup (a user's name on every comment)
- **Pre-aggregate** counters and rollups at write time

You pay for it at write time:

- One user rename becomes N document updates
- Concurrent updates to two copies can diverge, and nothing will tell you

That is a good deal when reads dominate and you have a story for refresh, and a bad one when the duplicated field changes constantly.

Relational systems denormalize for the same reason ([why you still denormalize](./19-relational-databases.md#why-you-still-denormalize)); the difference is that these stores usually **start** denormalized, because there is no cheap join planner to fall back on.

## Transactions and constraints

Do not assume "NoSQL has no transactions." Assume instead:

- **Atomicity** is defined for a documented scope — one key, one document, one partition — and you have to know which
- **Uniqueness** and **referential integrity** are either partition-local, an extra index table you maintain, or your problem in the application

If the invariant is "never double-charge," you need a real transaction or an idempotent ledger keyed by a request id. A document store does not provide that across two collections and three partitions just because it is fast.

## Picking a model

| Model       | Unit                              | What it is built for               | Where it hurts                              |
| ----------- | --------------------------------- | ---------------------------------- | ------------------------------------------- |
| Document    | JSON document in a collection     | Read or write one document by id   | A second access path, cross-document writes |
| Key-value   | Opaque value at a key             | Cache, session, lock, counter      | Anything that filters on the value          |
| Wide-column | Partition key plus sparse columns | Time-ordered append and range scan | Ad-hoc filters, changing the key            |
| Graph       | Nodes and edges                   | Multi-hop traversal                | Bulk analytics, partitioning a dense graph  |
| Time-series | `(metric, tags, timestamp)` value | Time-bounded aggregation           | Editing the past, high-cardinality tags     |

If the answer to any row is "but I also need joins, declared constraints, and multi-row transactions," the honest answer is a relational primary that you replicate and then partition. [SQL vs NoSQL](./21-sql-vs-nosql.md) walks that decision in order.

## Interview talking points

- Name the **access path** and the **unit of atomicity**, not "we use NoSQL for scale."
- BASE is a default, not a definition. Say what the primary or quorum does during a partition.
- Embedding vs referencing is a question about whether the nested data is **bounded**.
- Denormalize for reads, then say how the copies stay consistent enough.
- Horizontal scale still needs a **partition key**, and the second query still needs an index.
