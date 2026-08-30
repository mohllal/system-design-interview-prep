---
title: "SQL vs NoSQL"
concepts:
  - relational-model
  - document-model
  - transaction-scope
  - partition-keys
  - query-access-patterns
  - polyglot-persistence
related:
  - fundamentals/19-relational-databases.md
  - fundamentals/20-non-relational-databases.md
  - fundamentals/22-database-indexes.md
  - fundamentals/23-database-replication.md
  - fundamentals/24-database-partitioning.md
  - fundamentals/25-database-concurrency-control.md
  - fundamentals/27-cap-and-pacelc-theorems.md
  - fundamentals/34-cdn.md
---

# SQL vs NoSQL

The interview question is almost never "Postgres or Mongo."

It is: **what is the access path, what must be atomic, and how will we scale when one node is not enough?**

- "SQL" here means the **relational model**: tables, keys, joins, a planner, and transactions with a wide default scope.
- "NoSQL" means **other models** (document, key-value, wide-column, graph, time-series) that make one access path cheap and usually make joins or global constraints expensive.

The language itself is a poor signal:

- Plenty of non-relational stores speak SQL-ish dialects (CQL, PartiQL).
- Plenty of relational engines shard like a distributed key-value store (Citus, Vitess, CockroachDB, Spanner).

## What you are actually choosing

Four properties:

**1. Shape of a record**

- **Relational**: Columns you declared — extra facts live in other tables and you join them in.
- **Document / KV**: A blob (often JSON) you fetch by key — extra facts are embedded or duplicated.

**2. How you query**

- **Relational**: Any `WHERE` / `JOIN` / `GROUP BY` the planner can index — a new screen can use a new query without a new collection.
- **Non-relational**: Queries you **designed the key and indexes for** — a second access pattern is often a second store or a denormalized copy.

**3. Transaction scope**

- **Relational default**: Several rows, several tables, one `COMMIT`.
- **Non-relational default**: One document, one key, or one partition. Cross-partition "ACID" exists in some products, but it's slow and limited — don't assume it.

**4. How the vendor distributed it**

- **Classic SQL**: One primary, then replicas, then shards you add.
- **Many NoSQL products**: Partition key on day one; replication style varies (primary, quorum).

## Myths

**"SQL cannot scale."**

The short answer: no, it can.

- A well-indexed primary handles a lot, and reads scale with replicas.
- Writes and size scale with a partition key, same as Cassandra.

Switching to a document store does not remove the need for a key, indexes, and operational work.

**"NoSQL has no transactions or schema."**

- **Transactions**:
  - Single-document updates are atomic in MongoDB.
  - DynamoDB supports transactions, with limits.
  - Redis has `MULTI`.
- **Schema**:
  - The schema still exists — it just lives in the application code and the implicit shape of documents.
  - A missed field is a production bug, not "flexibility."

**"NoSQL is eventually consistent / BASE."**

That describes some deployments, not the category.

- MongoDB replica sets have a primary.
- Cassandra lets you pick `R` and `W`.

**"Pick one database for everything."**

Normal designs use **more than one**:

- **Relational**: System of record for orders and money.
- **Redis**: Cache, sessions, and locks.
- **Search engine**: Full-text search.
- **Object storage**: Blobs.
- **Cassandra / TSDB**: High-ingest telemetry, if you need it.

The interview move is deciding which store owns **which invariant**, not picking a single database.

## A decision sequence

Walk this in order. Stop when the answer is obvious.

**1. What does a write have to keep true?**

Examples:

- Debit and credit together.
- Inventory and order.
- Unique email.

If the invariant spans several facts, start relational (or with one carefully designed document that holds all of them).

**2. What does a read look like?**

| Read pattern                       | Best store                 |
| ---------------------------------- | -------------------------- |
| Many shapes, joins, ad-hoc filters | Relational + indexes       |
| Always `GET` by ID, nested payload | Document or KV             |
| Time range for one device          | Wide-column or time-series |
| Walk N hops of friends             | Graph                      |

**3. Is the nested data bounded?**

- **User + address**: Embed.
- **User + unbounded order history**: Do not embed — use a table, or a child collection keyed by `user_id` in the relational case.

**4. Are you out of one primary, or just imagining it?**

Measure first, then:

1. Add a replica for reads.
2. Partition with a key you can defend.

"We'll use NoSQL because we might have a billion users" is not a design.

## Hybrid is the default

A typical service:

| Data                      | Store                               | Why                                          |
| ------------------------- | ----------------------------------- | -------------------------------------------- |
| Orders, payments, users   | Postgres                            | Transactions, constraints, many query shapes |
| Session, rate limit, lock | Redis                               | TTL, speed, not the ledger                   |
| Product search            | Elasticsearch / OpenSearch          | Inverted text index                          |
| Images, videos            | Object storage + [CDN](./34-cdn.md) | Size, not rows                               |
| High-rate device events   | Cassandra / TSDB                    | Partition `(device, time)`, cheap append     |

The relational database stays the **source of truth** for money and identity, while other stores are derived, cached, or specialized.

Dual writes between stores fail independently, so use an [outbox](../architecture/07-transactional-outbox.md) or CDC when the copies must not diverge silently.

## Changing your mind later

- **Relational → document**: You give up cheap joins and FKs — you must embed or dual-write every new access path.
- **Document → relational**: You split blobs into tables and backfill — painful, but common when a second access path appears.
- **Adding Redis in front of SQL**: Not a migration — it's a cache with an invalidation story.
- **Sharding SQL**: A migration. So is changing a Cassandra primary key. Treat both as expensive.

## Interview talking points

- SQL vs NoSQL is **model + transaction scope + default distribution**, not a scale trophy.
- Start from **invariants** and **queries**, then pick a store.
- One primary SQL is the default until it isn't — replicas, then a partition key.
- NoSQL still needs a partition key and a story for the **second** query.
- Using two stores is normal — say which one is the system of record.
- "We'll use Mongo for scale," without a key and a transaction boundary, is a weak answer.
