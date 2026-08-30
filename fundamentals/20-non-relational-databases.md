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
  - read-your-writes-consistency
related:
  - fundamentals/19-relational-databases.md
  - fundamentals/21-sql-vs-nosql.md
  - fundamentals/23-database-replication.md
  - fundamentals/24-database-partitioning.md
  - fundamentals/27-cap-and-pacelc-theorems.md
---

# Non-relational databases (NoSQL)

"NoSQL" is not one product.

It is a set of stores that **drop or reshape** the relational contract (joins, global constraints, multi-row transactions) so a **specific access path** is cheap and easy to spread across machines.

Many of these systems now offer transactions **inside one partition or document**, but that is not the same as "ACID for any `JOIN` across the cluster."

## Start from the access path

Ask:

1. What is the **lookup key**? (user id, session id, `(device_id, time)`)
2. Do you need **joins**, or is one blob enough?
3. Is a transaction **one document / one partition**, or many keys?
4. Do writes need to be **visible everywhere immediately**, or is lag OK?

If you mostly `SELECT` by primary key and nest the rest, a document or KV store matches the path.

If you join five tables and enforce FKs, a relational primary is still the default, and you can shard it later to scale writes.

The old "SQL does not scale, NoSQL does" line is false.

- Plenty of SQL systems shard.
- Plenty of document clusters become painful once you need a second access pattern.

## BASE is a slogan, not a law

**BASE** (Basically Available, Soft state, Eventual consistency) is a mnemonic for "we prioritized staying up during a partition."

It is not a property of every non-relational database.

- MongoDB replica sets default to a **single primary** (more CP during failover than "always write")
- Redis is often one node, or a primary plus replicas
- Cassandra lets you choose `R` and `W` per request
- Neo4j is often a strongly consistent single cluster primary

Do not write "NoSQL = BASE = AP" on a whiteboard.
Name the product's **default write path** and what happens in a partition.

Eventual consistency means replicas **converge**:

- It does not say **when**.
- If the user must see their own write, you still need read-your-writes (same primary, session token, or `W` + `R` overlap).

## Document stores

**Unit:** a JSON-like document in a collection (MongoDB, CouchDB, DocumentDB).

Cheap:

- Fetch or update **one document** by `_id`
- Nested fields that you always load together (profile + settings)

Painful:

- "All orders in the last hour across users" without a designed index and shard key
- Changing a username copied into a thousand comment documents
- Multi-document transactions across shards (possible in some products, slow and limited)

**Embedding vs referencing:**

- Embed when the nested data is bounded and read with the parent (address on the user).
- Reference (`userId`) when the nested set is large or shared (all of a user's orders).

A 16MB document full of unbounded arrays is a trap, and you have reinvented a table with worse querying.

Transactions: start as **single-document atomic** and treat multi-document as the same problem as cross-row SQL, plus your shard key.

## Key-value stores

**Unit:** `GET` / `PUT` / `DEL` by key (Redis, Memcached, DynamoDB as a KV/document hybrid).

Cheap:

- Session, cache, lock, leader election, rate-limit counter
- Huge QPS on a hot key if the value is small

Painful:

- Query by value ("all keys whose JSON contains `city=Paris`")
- Secondary indexes (unless the product adds them, and then you are not in pure KV anymore)

The value is opaque to the store. If you need to filter fields, you either denormalize into the key (`user:42:session`) or pick a different store.

Durability varies:

- **Memcached**: Memory, gone on restart
- **Redis**: Optional AOF/RDB
- **DynamoDB**: Replicated disk

Do not use "Redis" and "durable system of record" as the same sentence unless you have configured persistence and HA on purpose.

## Wide-column (column-family)

**Unit:** a row key, then sparse columns grouped in families (Cassandra, HBase).

Cheap:

- Write a lot of **cells** for one key (`(user_id, timestamp)` → event)
- Scan a slice of columns that sort with the key (time series, inbox)

Painful:

- Ad-hoc `WHERE` on a column that is not part of the key or a designed index
- Changing the primary key later
- Joins

You model **query first**: the partition key is how you shard and the clustering columns are how you sort inside the partition.

## Graph databases

**Unit:** nodes and edges (Neo4j, AWS Neptune).

Cheap:

- Variable-length walks ("friends of friends," fraud rings)
- Recursion that would be painful in SQL

Painful:

- Bulk analytics over all nodes as if it were a warehouse
- Sharding a densely connected graph (cuts are expensive)

Use when the **relationship is the query**, not when you have a users table that happens to have a `follows` join you run twice.

## Time-series databases

**Unit:** `(metric, tags, timestamp) → value` (Influx, Prometheus, Timescale on Postgres).

Cheap:

- Append samples
- Aggregate `cpu` for `host=web1` over an hour
- Retention and downsampling

Painful:

- Updating a sample in the past as if it were a row in `users`
- High-cardinality tags (`user_id` on every metric) exploding series count

If the data is "events over time" **and** the query is always time-bounded, use this shape even if the engine is Postgres + hypertables.

## Modeling: copy on purpose

These stores often **denormalize** so one read is one fetch.

Patterns:

- **Embed** related data in the document
- **Duplicate** a field into another collection keyed for that lookup (user name on every comment)
- **Pre-aggregate** counters and rollups

You pay at **write** time:

- One user rename becomes N document updates
- Concurrent updates to two copies can diverge

That is acceptable when reads dominate and you have a story for refresh, but it is a bad deal when the duplicated field changes constantly.

Relational systems denormalize too. The difference is that NoSQL often **starts** denormalized because there is no cheap join planner.

## Transactions and constraints

Do not assume "NoSQL has no transactions" — assume:

- **Atomicity** is defined for a documented scope (one key, one document, one partition)
- **Unique** and **FK** are either local, an extra index table, or your problem in the app

If the interview invariant is "never double-charge," you need a real transaction or an idempotent ledger.
A document store does not magically provide that across two collections and three shards.

## How to choose (interview)

```plaintext
Lookup by id, nest the rest          → document or KV
Cache / session / lock               → KV (often Redis)
Time-ordered append + range          → wide-column or TSDB
Walk relationships                   → graph
Joins, constraints, multi-row txns   → relational (then replicate / partition)
```

If two access patterns both matter (lookup by id **and** list by email **and** join payments), either:

- Two stores (record in SQL, cache in Redis), or
- A global index / second collection, or
- Stay relational and index both paths

A single clever document schema rarely serves three incompatible queries well.

## Interview talking points

- Name the **access path** and the **unit of atomicity**, not "we use NoSQL for scale."
- BASE is optional.
  Say what the default primary / quorum does in a partition.
- Embedding vs reference: bound the nested data.
- Denormalize for reads.
  Say how writes stay consistent enough.
- Horizontal scale still needs a **partition key**.
