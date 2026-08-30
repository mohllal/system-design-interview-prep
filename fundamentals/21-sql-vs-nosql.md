---
title: "SQL vs NoSQL"
concepts:
  - transaction-scope
  - query-access-patterns
  - system-of-record
  - polyglot-persistence
  - partition-keys
  - migration-cost
related:
  - fundamentals/19-relational-databases.md
  - fundamentals/20-non-relational-databases.md
  - fundamentals/22-database-indexes.md
  - fundamentals/23-database-replication.md
  - fundamentals/24-database-partitioning.md
  - fundamentals/25-database-concurrency-control.md
  - fundamentals/27-cap-and-pacelc-theorems.md
  - fundamentals/34-cdn.md
  - architecture/07-transactional-outbox.md
---

# SQL vs NoSQL

The interview question is almost never "Postgres or Mongo."

It is: **what is the access path, what must be atomic, and how will we scale when one node is not enough?**

This note is the decision framework. It assumes you already know what each side is — [Relational Databases](./19-relational-databases.md) covers tables, joins, and ACID, and [Non-Relational Databases](./20-non-relational-databases.md) covers the document, key-value, wide-column, graph, and time-series models and what BASE actually means.

One warning before the comparison: the **query language is a poor signal**.

- Plenty of non-relational stores speak SQL-ish dialects (CQL, PartiQL).
- Plenty of relational engines partition like a distributed key-value store (Citus, Vitess, CockroachDB, Spanner).

So "SQL" below means the **relational model** and "NoSQL" means everything else, regardless of what the client library looks like.

## What you are actually choosing

Six axes. Read down a column for a summary of each model; read across a row for the trade-off you have to defend out loud.

| Axis                         | Relational                                                                  | Non-relational                                                        |
| ---------------------------- | --------------------------------------------------------------------------- | --------------------------------------------------------------------- |
| Shape of a record            | Columns you declared; other facts live in other tables and are joined in    | A blob fetched by key; other facts are embedded or duplicated into it |
| Query flexibility            | Any `WHERE` / `JOIN` / `GROUP BY` the planner can index, anticipated or not | Only the queries you designed a key or an index for                   |
| Default transaction scope    | Many rows across many tables, one `COMMIT`                                  | One key, one document, or one partition                               |
| Where the schema lives       | In the database, enforced at write time                                     | In the application, enforced by whatever wrote the document last      |
| Default distribution         | One primary, then replicas, then partitions you add deliberately            | A partition key from day one; replication style varies by product     |
| Cost of a second access path | Another index on the same table                                             | Another index, a second collection, or a second store to keep in sync |

The last two rows are where most designs actually get decided. Everything else can be worked around; a wrong partition key and an unowned second copy cannot.

## Myths

**"SQL cannot scale."**

It can — it just scales in the same two steps as anything else.

- A well-indexed primary handles far more than people assume, and reads scale with [replicas](./23-database-replication.md).
- Writes and storage scale with a [partition key](./24-database-partitioning.md), exactly like Cassandra.

Switching to a document store does not remove the need for a key, indexes, and operational work. It moves that work into your application.

**"NoSQL has no transactions or schema."**

Both halves are wrong, in a way that matters:

- **Transactions**: Single-document updates are atomic in MongoDB, DynamoDB supports transactions with size and rate limits, and Redis has `MULTI`. What varies is the **scope**, not the existence — see [transactions and constraints](./20-non-relational-databases.md#transactions-and-constraints).
- **Schema**: The schema still exists. It lives in the application code and in the implicit shape of documents already written, which means an old document with a missing field is a production bug rather than "flexibility." Schemaless means *unvalidated*, not *absent*.

**"NoSQL means eventually consistent."**

That describes some deployments, not the category — MongoDB replica sets have a primary, and Cassandra lets you pick `R` and `W` per request. The [ACID and BASE contrast](./20-non-relational-databases.md#acid-and-base) is the useful version of this comparison.

**"Pick one database for everything."**

Normal designs use more than one, and the interview move is deciding which store owns **which invariant**, not picking a single database. See [Hybrid is the default](#hybrid-is-the-default).

## A decision sequence

Walk this in order and stop when the answer is obvious.

**1. What does a write have to keep true?**

Examples: debit and credit together, inventory and order together, an email that is unique across all users.

If an invariant spans several facts, start relational — or with one carefully designed document that holds *all* of the facts the invariant touches. If every invariant fits inside one record, that constraint is not pushing you either way.

**2. What does a read look like?**

Ask whether the queries are **enumerable in advance**.

- A short, stable list of lookups, all naming the same key, points at a non-relational model — [picking a model](./20-non-relational-databases.md#picking-a-model) maps each read shape (single-key fetch, time range, graph walk) to the store built for it.
- Ad-hoc filters, joins, reporting, and "the next screen needs a different cut of the same data" point at a relational primary plus [indexes](./22-database-indexes.md).

**3. Is the nested data bounded?**

- **User plus address**: Bounded. Embed it, or keep it in a `1:1` table.
- **User plus order history**: Unbounded. Do not embed it — use a separate table, or a child collection keyed by `user_id`.

Unbounded nesting is the failure mode that forces a migration later, in either model.

**4. Are you out of one primary, or just imagining it?**

Measure first. Then: add a replica for reads, and only after that partition with a key you can defend.

"We'll use NoSQL because we might have a billion users" is not a design. Neither is "we'll shard later" with no candidate key named.

### Worked example: an order and delivery service

Three datasets, one sequence, two different answers.

1. **Invariants**: A charge and an order row are created together, and the last unit in stock is never sold twice. Both span several facts, so orders, payments, and inventory go in a relational primary.
2. **Reads**: Order history is `WHERE customer_id = ? ORDER BY created_at DESC` — one composite index. But the support console filters by status, date, and region in combinations nobody enumerated up front, which is a planner's job, not a designed key.
3. **Bounded?**: Line items per order are bounded and always read with the order. Order history per customer is not — it is a table, never an embedded array.
4. **Scale**: Orders run at thousands per minute, which one primary plus a read replica handles. Courier location pings are a **different** workload: one write per second per courier across tens of thousands of couriers, always read as `(courier_id, time range)`, never joined, discarded after a few days.

The result is two stores for two genuinely different access paths: Postgres owns money and identity, a time-series or wide-column store owns location history, and Redis holds the "which courier is on this order right now" lookup. That is polyglot persistence for a reason you can state, not for scale as a slogan.

## Hybrid is the default

A typical service:

| Data                      | Store                               | Why                                          |
| ------------------------- | ----------------------------------- | -------------------------------------------- |
| Orders, payments, users   | Postgres                            | Transactions, constraints, many query shapes |
| Session, rate limit, lock | Redis                               | TTL, speed, not the ledger                   |
| Product search            | Elasticsearch / OpenSearch          | Inverted text index                          |
| Images, videos            | Object storage + [CDN](./34-cdn.md) | Size, not rows                               |
| High-rate device events   | Cassandra / TSDB                    | Partition `(device, time)`, cheap append     |

The relational database stays the **system of record** for money and identity. Every other store is derived, cached, or specialized — and should be rebuildable from the system of record if it is lost.

Dual writes to two stores fail independently, so use an [outbox](../architecture/07-transactional-outbox.md) or change data capture when the copies must not diverge silently.

## Changing your mind later

Migration cost is part of the choice, so price it while you still have the option:

- **Relational to document**: You give up cheap joins and foreign keys, so you must embed or dual-write every access path you had for free.
- **Document to relational**: You split blobs into tables and backfill. Painful, but common once a second access path shows up.
- **Adding Redis in front of SQL**: Not a migration — a cache, with an invalidation story you now own.
- **Partitioning SQL**: A real migration. So is changing a Cassandra primary key. Treat both as expensive, and choose the key with that in mind.

The asymmetry is worth saying out loud: a relational schema that turns out to be wrong is usually a migration, while a partition key that turns out to be wrong is usually a rewrite.

## Interview talking points

- SQL vs NoSQL is **model plus transaction scope plus default distribution**, not a scale trophy.
- Start from **invariants** and **queries**, then pick a store — in that order.
- One primary SQL is the default until it is not: replicas first, then a partition key.
- NoSQL still needs a partition key and a story for the **second** query.
- Using two stores is normal. Say which one is the system of record and how the others are rebuilt.
- "We'll use Mongo for scale," with no key and no transaction boundary, is a weak answer.
