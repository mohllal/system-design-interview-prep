---
title: "Database indexes"
concepts:
  - b-tree-index
  - clustered-vs-secondary-index
  - covering-index
  - hash-index
  - bitmap-index
  - inverted-index
  - lsm-tree
  - composite-index-left-prefix
related:
  - fundamentals/19-relational-databases.md
  - fundamentals/24-database-partitioning.md
  - fundamentals/17-bloom-filters.md
  - advanced/06-postgresql-internals.md
---

# Database indexes

An index is a **side structure** that finds rows without scanning the whole table.

You pay for it on every write.
You pay for it in disk and memory.
You get cheaper reads when the predicate matches the index.

This note covers the types of indexes, how they're implemented, and when the query planner will refuse to use them.

Related: [Relational Databases](./19-relational-databases.md), [Partitioning](./24-database-partitioning.md), [Bloom Filters](./17-bloom-filters.md), [PostgreSQL Internals](../advanced/06-postgresql-internals.md).

## What problem does an index solve?

Without an index, `WHERE user_id = 42` reads every row (sequential scan).

With an index on `user_id`, the engine jumps to the matching keys and then to the rows.

```plaintext
table (heap or clustered)
  row, row, row, ...     ← sequential scan: O(n)

index
  sorted keys → row pointers
  lookup 42 → a few pages   ← typically O(log n)
```

Indexes do not help if:

- You still need most of the table (`WHERE year = 2024` on a table that is 90% 2024)
- The predicate does not match how keys are stored (`WHERE YEAR(created_at) = 2024` on a btree of `created_at`, unless you have an expression index)
- You index the wrong column for the `WHERE` / `JOIN` / `ORDER BY`

## Clustered vs secondary indexes

**Clustered index:** the table **is** the index.

Rows are stored in key order. InnoDB's primary key is clustered, so a lookup by PK goes through one structure instead of index-then-heap.

**Secondary (non-clustered) index:** a separate tree.

Leaves hold `(indexed columns) → row locator`.

- **Postgres**: the locator is `ctid` (heap file + offset); the table heap is unordered.
- **InnoDB**: the locator is the **primary key**, so a secondary lookup goes index → PK → clustered row (an extra hop).

**Index-only scan:** the index leaf already has every column the query needs.
Postgres can skip the heap if the visibility map says the page is all-visible.

Covering indexes (`INCLUDE` extra columns, or extra columns in a composite key) exist to make this possible, at the cost of a fatter index.

## B-tree / B+tree

The B+tree is the default index type for almost every OLTP workload (`=`, `<`, `>`, `BETWEEN`, `ORDER BY`, or a prefix of a composite key).

### B+tree structure

A **B+tree** keeps all row pointers in the **leaves**. Internal nodes only have separator keys and child pointers.

```plaintext
                    [ 50 | 90 ]
                   /     |     \
            [ 20|40 ]  [ 70 ]  [ 100|120 ]
               /  \       |        /   \
           leaves: 10,20,30,40  50,60,70  90,100,110,120
                   ↔ linked to the next leaf for range scans
```

Leaves are linked, so a range scan walks the leaf chain without going back up to the root for every key.

**B-tree** (without the plus) can store records in internal nodes too. The storage engines that matter in practice are B+tree-like.

### Why B+trees stay shallow

Each node is a **page** (often 8KB or 16KB).

Fanout is hundreds of keys per page, so a few hundred million rows still fit in **3–4 levels** — a handful of page reads if the upper levels are in cache.

### Writes and page splits

Insert finds the leaf and puts the key there.

If the leaf is full, it **splits**: half the keys move to a new page, and the parent gets a new separator key. Splits can cascade to the root, growing the tree by one level.

Random inserts (e.g., UUIDs as the PK in a clustered table) split constantly and fragment pages. Append-ish keys (bigserial, time-ordered ULID) append to the right edge instead, which is cheaper.

Deletes leave holes; engines may compact later (fillfactor, VACUUM, page merge). A bloated index is a real operational issue, not just a theoretical one.

### What queries does a B+tree support?

- **Equality**: `user_id = 42`
- **Range**: `created_at > yesterday`
- **Sort**: `ORDER BY created_at`, if the index order matches
- **Prefix**: the longest left prefix of a composite index (see below)

It does not turn `WHERE LOWER(email) =` into a seek unless you indexed `LOWER(email)`.

## Hash indexes

The key is hashed into a **bucket**. Equality only (`=`).

No order.
No range.
No `ORDER BY` from the index.

Useful for exact-match dictionaries; less useful as a general-purpose SQL index. Postgres hash indexes exist but aren't the usual default.
InnoDB's adaptive hash is an in-memory extra on top of the B-tree, not a durable hash index you create.

## Bitmap indexes

For each distinct value, the index keeps a **bit vector** over row IDs.

`status = 'open' AND region = 'eu'` becomes two bitmaps **ANDed**.

Good when:

- Cardinality is **low** (status, country, boolean)
- Reads are analytical
- Updates are rare (flipping a bit in a huge bitmap is expensive, and OLTP updates are the wrong workload)

Bad choice for a clustered OLTP primary key. Data warehouses and Postgres's bitmap index **scans** are related but different:
a bitmap scan builds a bitmap from a B-tree and then does a heap fetch — the underlying index is still a B-tree; the bitmap is just how it batches heap access.

## Inverted indexes (GIN, full-text, JSON)

One document has **many** tokens (words, array elements, JSON keys).

The index is `token → list of row ids`.

That is a **GIN** (Generalized Inverted Index) in Postgres, or a Lucene segment in Elasticsearch.

- **Cheap**: answering "which rows contain `error` and `timeout`?"
- **Expensive**: updating a document, since it rewrites many posting lists

**GiST** is a generalized tree for types that a B-tree doesn't handle well (geometry, some range types). Think "pluggable B-tree," not "inverted."

## LSM trees (write-optimized stores)

Used by Cassandra, RocksDB, Lucene, and many other KV store internals:

1. Writes go to a **memtable** (sorted in RAM) plus a commit log
2. Flush to an immutable **SSTable** on disk
3. **Compact** overlapping tables in the background

```plaintext
WAL + memtable  →  SST-L0  →  compact  →  SST-L1  →  ...
```

Point reads may check several levels. **Bloom filters** skip SSTs that cannot contain the key.

Reads are more work than with a hot B-tree, but writes avoid in-place B-tree page splits — which is why LSM trees show up in high-ingest stores.

Compaction is the hidden cost: disk I/O, write amplification, and space held until old SSTables are reclaimed.

## Composite indexes and the left prefix

Index `(user_id, created_at)` is a btree on the pair, in that order.

It can seek:

- `user_id = 42`
- `user_id = 42 AND created_at > t`

It generally **cannot** seek:

- `created_at > t` alone

The left column is the first sort key; skipping it is like searching a phone book with no last name.

`ORDER BY user_id, created_at` can ride this index.
`ORDER BY created_at` cannot.

Put equality columns first, then range columns, matching the query — or create a second index.

## Unique, partial, and expression indexes

- **Unique index**: A lookup structure **and** a constraint. Two rows with the same key cannot both commit.
- **Partial index**: Indexes only rows matching a predicate, e.g. `WHERE status = 'open'`. Smaller, but only queries that imply that predicate can use it.
- **Expression / functional index**: Built on a computed value, e.g. `LOWER(email)` or `(data->>'sku')`. The query must use the same expression to benefit.

## Cost model

Every `INSERT` / `UPDATE` / `DELETE` of an indexed column maintains every matching index.

Costs:

- Extra WAL and disk
- Slower writes
- More cache pressure (index pages vs table pages)
- Longer autovacuum / rebuild windows

Selectivity matters: an index on a boolean `is_deleted` column where 99% of rows are false is often useless for `WHERE is_deleted = false`, and the planner will fall back to a sequential scan.

Too many indexes is a write-latency bug. Too few is a read-latency bug. `EXPLAIN ANALYZE` decides, not a rule of thumb like "index every FK."

## Planner choices

- **Index seek / range**: few rows, good selectivity
- **Bitmap index scan**: several indexes ANDed, then heap fetch in order
- **Seq scan**: cheap when the table is small or most rows qualify
- **Index-only**: all needed columns in the index, visibility OK

If the row-count estimate is wrong (stale statistics), the planner picks the wrong strategy — `ANALYZE` is part of index operations, not an afterthought.

## Interview talking points

- **Index**: A maintained copy of keys for lookups; writes get slower as a result.
- **Default (B+tree)**: Short tree, range scans via linked leaves, splits on full pages.
- **Clustered vs secondary**: InnoDB's primary key *is* the table; Postgres uses a heap plus secondary indexes.
- **Composite indexes**: Follow the leftmost prefix rule.
- **Covering / index-only scans**: Avoid the heap entirely.
- **LSM vs inverted**: LSM trees suit write-heavy engines; inverted/GIN indexes suit many-values-per-row data.
- **Hash vs bitmap**: Hash indexes support equality only; bitmap indexes suit low-cardinality analytics.
