---
title: "Database indexes"
concepts:
  - b-tree-index
  - clustered-vs-secondary-index
  - covering-index
  - composite-index-left-prefix
  - hash-index
  - bitmap-index
  - inverted-index
  - lsm-tree
  - partitioned-secondary-indexes
related:
  - fundamentals/19-relational-databases.md
  - fundamentals/20-non-relational-databases.md
  - fundamentals/21-sql-vs-nosql.md
  - fundamentals/24-database-partitioning.md
  - fundamentals/17-bloom-filters.md
  - advanced/06-postgresql-internals.md
---

# Database indexes

An index is a **side structure** that finds rows without scanning the whole table.

You pay for it on every write. You pay for it in disk and memory. You get cheaper reads when the predicate matches the index.

This note covers the index structures, how they are implemented, when the query planner will refuse to use them, and how the same structures show up in non-relational stores. The examples are relational because that is where the vocabulary comes from, but almost none of it is specific to SQL.

Related: [Relational Databases](./19-relational-databases.md), [Non-Relational Databases](./20-non-relational-databases.md), [Partitioning](./24-database-partitioning.md), [Bloom Filters](./17-bloom-filters.md), [PostgreSQL Internals](../advanced/06-postgresql-internals.md).

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
- The predicate does not match how keys are stored (`WHERE YEAR(created_at) = 2024` on a B-tree of `created_at`, unless you have an expression index)
- You index the wrong column for the `WHERE` / `JOIN` / `ORDER BY`

That last case is the common one: an index answers the query it was built for, and no other.

## B-tree / B+tree

The B+tree is the default index type for almost every OLTP workload (`=`, `<`, `>`, `BETWEEN`, `ORDER BY`, or a prefix of a composite key). Everything later in this note is a specialization for a workload the B+tree handles badly, so start here.

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

**B-tree** (without the plus) can store records in internal nodes too. The storage engines that matter in practice are B+tree-like, and "B-tree index" in everyday usage means this structure.

### Why B+trees stay shallow

Each node is a **page** (often 8KB or 16KB).

Fanout is hundreds of keys per page, so a few hundred million rows still fit in **3-4 levels** — a handful of page reads, and fewer if the upper levels are cached.

### Writes and page splits

Insert finds the leaf and puts the key there.

If the leaf is full, it **splits**: half the keys move to a new page, and the parent gets a new separator key. Splits can cascade to the root, growing the tree by one level.

Random inserts (for example UUIDv4 as the primary key of a clustered table) split constantly and fragment pages. Append-ish keys (`bigserial`, a time-ordered ULID or UUIDv7) land on the right edge instead, which is cheaper and keeps the index dense.

Deletes leave holes; engines may compact later (fillfactor, `VACUUM`, page merge). A bloated index is a real operational issue, not just a theoretical one.

### What queries does a B+tree support?

- **Equality**: `user_id = 42`
- **Range**: `created_at > yesterday`
- **Sort**: `ORDER BY created_at`, when the index order matches the requested order
- **Prefix**: the longest left prefix of a composite index (see below)

It does not turn `WHERE LOWER(email) = ...` into a seek unless you indexed `LOWER(email)`.

## Clustered vs secondary indexes

The B+tree above describes the shape. Where the **row data** sits relative to it is a separate decision, and it is the one that decides how many hops a lookup costs.

**Clustered index:** the table **is** the index.

Rows are stored in key order in the leaves. InnoDB's primary key is clustered, so a lookup by primary key goes through one structure instead of index-then-heap.

**Secondary (non-clustered) index:** a separate tree whose leaves hold `(indexed columns) → row locator`.

- **Postgres**: The locator is the `ctid` (heap file and offset); the table heap is unordered.
- **InnoDB**: The locator is the **primary key**, so a secondary lookup goes index → primary key → clustered row, one extra hop.

This is why a wide primary key is expensive in InnoDB: every secondary index stores a copy of it.

**Index-only scan:** the index leaf already holds every column the query needs, so the engine never touches the table. Postgres can skip the heap only if the visibility map says the page is all-visible.

Covering indexes (`INCLUDE`d extra columns, or extra columns in a composite key) exist to make this possible, at the cost of a fatter index and slower writes.

## Composite indexes and the left prefix

Index `(user_id, created_at)` is one B-tree on the pair, sorted by `user_id` first.

It can seek:

- `user_id = 42`
- `user_id = 42 AND created_at > t`

It generally **cannot** seek:

- `created_at > t` alone

The left column is the first sort key, and skipping it is like searching a phone book with no last name.

`ORDER BY user_id, created_at` can ride this index. `ORDER BY created_at` cannot.

Put equality columns first, then the range column, matching the query — or create a second index for the other shape.

## Unique, partial, and expression indexes

- **Unique index**: A lookup structure **and** a constraint. Two rows with the same key cannot both commit, which is how a relational engine enforces "unique email."
- **Partial index**: Indexes only the rows matching a predicate, for example `WHERE status = 'open'`. Much smaller on a table where 1% of rows are open, but only queries that imply that predicate can use it.
- **Expression / functional index**: Built on a computed value, for example `LOWER(email)` or `(data->>'sku')`. The query has to use the same expression verbatim to benefit.

The B+tree family above covers almost every OLTP need. The structures below exist for the workloads it handles badly: pure equality at memory speed, low-cardinality analytics, many values per row, and write-heavy ingest.

## Hash indexes

The key is hashed into a **bucket**. Equality only (`=`).

No order, no range scans, and no `ORDER BY` served from the index.

Useful for exact-match dictionaries, and less useful as a general-purpose index, since a B-tree also answers equality and answers everything else too. Postgres hash indexes exist and have been crash-safe since they became WAL-logged, but they are still not the usual default. InnoDB's adaptive hash index is an in-memory accelerator on top of the B-tree, not a durable index you create.

## Bitmap indexes

For each distinct value, the index keeps a **bit vector** over row IDs.

`status = 'open' AND region = 'eu'` becomes two bitmaps **ANDed** together, which is very cheap.

Good when:

- Cardinality is **low** (status, country, boolean)
- Reads are analytical
- Updates are rare, because flipping a bit inside a huge bitmap locks and rewrites a large chunk of it

That last point rules them out for an OLTP primary key. Postgres's bitmap index **scans** are a related but different thing: a bitmap scan builds a bitmap from a B-tree at query time and then fetches heap pages in physical order — the stored index is still a B-tree, and the bitmap is just how the planner batches heap access.

## Inverted indexes (GIN, full-text, JSON)

One row has **many** index entries: words in a document, elements of an array, keys in a JSON blob.

The index is `token → list of row ids`.

That is a **GIN** (Generalized Inverted Index) in Postgres, or a Lucene segment in Elasticsearch.

- **Cheap**: Answering "which rows contain `error` and `timeout`?" by intersecting two posting lists
- **Expensive**: Updating a document, since one row touches many posting lists

**GiST** is a generalized tree for types a B-tree does not order well (geometry, ranges, nearest-neighbour searches). Think "pluggable B-tree," not "inverted."

## LSM trees (write-optimized stores)

Used by Cassandra, RocksDB, Lucene, and many other storage engine internals:

1. Writes go to a **memtable** (sorted in RAM) plus a commit log
2. The memtable is flushed to an immutable **SSTable** on disk
3. Overlapping SSTables are **compacted** in the background

```plaintext
WAL + memtable  →  SST-L0  →  compact  →  SST-L1  →  ...
```

A point read may have to check several levels. [Bloom filters](./17-bloom-filters.md) let it skip SSTables that cannot contain the key.

Reads are more work than with a hot B-tree, but writes never do in-place page splits — which is why LSM trees show up in high-ingest stores.

Compaction is the hidden cost: background disk I/O, write amplification, and space held until old SSTables are reclaimed.

## Indexes in non-relational stores

None of the structures above are specific to SQL. What changes in a [non-relational store](./20-non-relational-databases.md) is who maintains the index and how far a lookup has to travel:

- **Document stores**: A secondary index on `status` or `email` is a B-tree over the collection, exactly as it would be over a table.
- **Key-value stores**: The value is opaque, so the key is the only thing indexed. Any other lookup means lifting the field into the key or maintaining a separate index collection yourself.
- **Wide-column stores**: The partition key and clustering columns **are** the primary index, and they fix placement and sort order at design time. An index on any other column is partition-local, so the coordinator has to fan out to every node.
- **Search engines**: The inverted index is the entire product rather than an option — Elasticsearch is the GIN section above, run as its own cluster.
- **LSM-backed engines**: Secondary indexes live in the same SSTable machinery, so index writes are cheap and index reads pay the same multi-level lookup.

The one decision with no single-node relational equivalent is **local vs global** secondary indexes, which a partitioned store forces on you: a local index is cheap to write and expensive to query without the partition key, a global index is the reverse.
See [secondary indexes and the second access path](./20-non-relational-databases.md#secondary-indexes-and-the-second-access-path) for that trade-off and [Partitioning](./24-database-partitioning.md) for how index partitions are placed and resharded.

## Cost model

Every `INSERT` / `UPDATE` / `DELETE` of an indexed column maintains every matching index.

Costs:

- Extra WAL and disk
- Slower writes
- More cache pressure (index pages competing with table pages)
- Longer autovacuum and rebuild windows

**Selectivity** decides whether the index is used at all. An index on a boolean `is_deleted` column where 99% of rows are `false` is useless for `WHERE is_deleted = false` — the planner will read the table instead, correctly. The same index is valuable for `WHERE is_deleted = true`, which is what a partial index exists for.

Too many indexes is a write-latency bug. Too few is a read-latency bug. `EXPLAIN ANALYZE` decides which one you have, not a rule of thumb like "index every foreign key."

## Planner choices

- **Index seek / range scan**: Few rows, good selectivity
- **Bitmap index scan**: Several indexes combined, then heap fetch in physical order
- **Sequential scan**: Cheap when the table is small or most rows qualify
- **Index-only scan**: Every needed column is in the index and visibility allows it

If the row-count estimate is wrong (stale statistics), the planner picks the wrong strategy even with the right index available. `ANALYZE` is part of index operations, not an afterthought.

## Interview talking points

- **Index**: A maintained copy of keys for lookups; every write pays for it.
- **Default (B+tree)**: Short tree, range scans via linked leaves, splits on full pages, and a preference for append-ish keys.
- **Clustered vs secondary**: InnoDB's primary key *is* the table; Postgres uses a heap plus secondary indexes that point into it.
- **Composite indexes**: Follow the leftmost prefix rule — equality columns first, then the range column.
- **Covering / index-only scans**: Avoid the heap entirely, at the cost of a fatter index.
- **Hash vs bitmap**: Hash indexes serve equality only; bitmap indexes suit low-cardinality analytics, not OLTP.
- **LSM vs inverted**: LSM trees suit write-heavy engines; inverted and GIN indexes suit many-values-per-row data.
- **Non-relational stores index too**: The same structures, plus a local vs global choice once the data is partitioned.
