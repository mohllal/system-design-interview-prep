# SQL vs NoSQL

The interview question is almost never "Postgres or Mongo."

It is: **what is the access path, what must be atomic, and how will we scale when one node is not enough?**

- "SQL" here means the **relational model**: tables, keys, joins, a planner, transactions with a wide default scope.
- "NoSQL" means **other models** (document, KV, wide-column, graph, time-series) that make one path cheap and usually make joins or global constraints expensive.

The language is a bad axis as:

- Plenty of non-relational stores speak SQL-ish (CQL, PartiQL).
- Plenty of relational engines shard like a distributed KV store (Citus, Vitess, Cockroach, Spanner).

## What You Are Actually Choosing

Four properties:

**1. Shape of a record**

- Relational: columns you declared.
  Extra facts live in other tables and you join.
- Document / KV: a blob (often JSON) you fetch by key.
  Extra facts are embedded or duplicated.

**2. How you query**

- Relational: any `WHERE` / `JOIN` / `GROUP BY` the planner can index.
  A new screen can use a new query without a new collection.
- Non-relational: queries you **designed the key and indexes for**.
  A second access pattern is often a second store or a denormalized copy.

**3. Transaction scope**

- Relational default: several rows, several tables, one `COMMIT`.
- Non-relational default: one document, one key, or one partition.
  Cross-partition "ACID" exists in some products, but it is slow and limited, so do not assume it.

**4. How the vendor distributed it**

- Classic SQL: one primary, then replicas, then shards you add.
- Many NoSQL products: partition key on day one.
  Replication style varies (primary, quorum).

## Myths

**"SQL cannot scale."**

Short answer is "no, it can".

- A well-indexed primary handles a lot and reads scale with replicas.
- Writes and size scale with a partition key, same as Cassandra.

Switching to a document store does not remove the need for a key, indexes, and operational work.

**"NoSQL has no transactions or schema."**

- Transactions:
  - Single-document updates are atomic in MongoDB.
  - DynamoDB has transactions with limits.
  - Redis has `MULTI`.
- Schema:
  - Schema still exists but it is in the app and in implicit document shape.
  - A missed field is a production bug, not "flexibility."

**"NoSQL is eventually consistent / BASE."**

That describes some deployments, not the category.

- MongoDB replica sets have a primary.
- Cassandra lets you pick `R` and `W`.

**"Pick one database for everything."**

Normal designs use **more than one**:

- Relational system of record (orders, money)
- Redis for cache, sessions, locks
- Search engine for text
- Object store for blobs
- Maybe Cassandra or a TSDB for high-ingest telemetry

The interview move is which store owns **which invariant**, not a single database.

## A Decision Sequence

Walk this in order. Stop when the answer is obvious.

**1. What does a write have to keep true?**

Debit and credit together, inventory and order, unique email.
If that spans several facts, start relational (or one carefully designed document that holds all of them).

**2. What does a read look like?**

- Many shapes, joins, ad-hoc filters: relational + indexes.
- Always `GET` by id, nested payload: document or KV.
- Time range for one device: wide-column or time-series.
- Walk N hops of friends: graph.

**3. Is the nested data bounded?**

- User + address: embed.
- User + unbounded order history: do not embed.

That is a table or a child collection keyed by `user_id` in the relational case.

**4. Are you out of one primary, or just imagining it?**

Measure and then:

1. Add a replica for reads.
2. Partition with a key you can defend.

"We'll use NoSQL because we might have a billion users" is not a design.

## Hybrid Is the Default

A typical service:

| Data                      | Store                               | Why                                          |
|---------------------------|-------------------------------------|----------------------------------------------|
| Orders, payments, users   | Postgres                            | Transactions, constraints, many query shapes |
| Session, rate limit, lock | Redis                               | TTL, speed, not the ledger                   |
| Product search            | Elasticsearch / OpenSearch          | Inverted text index                          |
| Images, videos            | Object storage + [CDN](./34-cdn.md) | Size, not rows                               |
| High-rate device events   | Cassandra / TSDB                    | Partition `(device, time)`, cheap append     |

The relational database stays the **source of truth** for money and identity while other stores are derived, cached, or specialized.

Dual-writes between stores fail independently so use an [outbox](../architecture/07-transactional-outbox.md) or CDC when the copy must not diverge silently.

## Changing Your Mind Later

- Relational → document: you give up cheap joins and FKs.
  You must embed or dual-write every new access path.

- Document → relational: you split blobs into tables and backfill.
  Painful, but common when a second access path appears.

- Adding Redis in front of SQL is not a migration.
  It is a cache with an invalidation story.

- Sharding SQL is a migration.
  So is changing a Cassandra primary key. Treat both as expensive.

## Interview Talking Points

- SQL vs NoSQL is **model + transaction scope + default distribution**, not a scale trophy.
- Start from **invariants** and **queries**.
  Then pick a store.
- One primary SQL is the default until it is not.
  Replicas, then partition key.
- NoSQL still needs a partition key and a story for the **second** query.
- Using two stores is normal.
  Say which one is the system of record.
- "We'll use Mongo for scale" without a key and a transaction boundary is a weak answer.
