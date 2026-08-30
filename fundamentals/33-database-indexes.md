# Database Indexes

An index is a **side structure** that finds rows without scanning the whole table.

You pay for it on every write.
You pay for it in disk and memory.
You get cheaper reads when the predicate matches the index.

This note is the types, how they are implemented, and when the planner will refuse to use them.

Related: [Relational Databases](./19-relational-databases.md), [Partitioning](./22-database-partitioning.md), [Bloom Filters](./17-bloom-filters.md), [PostgreSQL Internals](../advanced/06-postgresql-internals.md).

## What Problem It Solves

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

## Clustered vs Secondary

**Clustered index:** the table **is** the index.

Rows are stored in key order.
InnoDB's primary key is clustered.
A lookup by PK is one structure, not index-then-heap.

**Secondary (non-clustered) index:** a separate tree.

Leaves hold `(indexed columns) → row locator`.

- Postgres: locator is `ctid` (heap file + offset).
  The table heap is unordered.
- InnoDB: locator is the **primary key**.
  A secondary lookup is index → PK → clustered row (extra hop).

**Index-only scan:** the index leaf already has every column the query needs.
Postgres can skip the heap if the visibility map says the page is all-visible.

Covering indexes (`INCLUDE` extra columns, or extra columns in a composite key) exist to make this possible.
They make the index fatter.

## B-Tree / B+Tree

Default for almost every OLTP index (`=` , `<`, `>`, `BETWEEN`, `ORDER BY`, prefix of a composite key).

### Shape

A **B+tree** keeps all row pointers in the **leaves**.
Internal nodes only have separator keys and child pointers.

```plaintext
                    [ 50 | 90 ]
                   /     |     \
            [ 20|40 ]  [ 70 ]  [ 100|120 ]
               /  \       |        /   \
           leaves: 10,20,30,40  50,60,70  90,100,110,120
                   ↔ linked to the next leaf for range scans
```

Leaves are linked.
A range scan walks the leaf chain.
It does not go back up to the root for every key.

**B-tree** (without the plus) can store records in internal nodes too.
Storage engines you care about are B+tree-like.

### Why it is shallow

Each node is a **page** (often 8KB or 16KB).

Fanout is hundreds of keys per page.
A few hundred million rows still fit in **3–4 levels**.
That is a handful of page reads if the upper levels are in cache.

### Writes

Insert finds the leaf and puts the key there.

If the leaf is full, it **splits**:
half the keys move to a new page.
The parent gets a new separator.
Splits can cascade to the root (the tree grows one level).

Random inserts (UUIDs as PK in a clustered table) split constantly and fragment pages.
Append-ish keys (bigserial, time-ordered ULID) append to the right edge.
That is cheaper.

Deletes leave holes.
Engines may compact later (fillfactor, VACUUM, page merge).
A bloated index is a real ops issue, not just a theory.

### What queries it supports

- Equality: `user_id = 42`
- Range: `created_at > yesterday`
- Sort: `ORDER BY created_at` if the index order matches
- Longest **left prefix** of a composite index (see below)

It does not turn `WHERE LOWER(email) =` into a seek unless you indexed `LOWER(email)`.

## Hash Indexes

Key hashed into a **bucket**.
Equality only (`=`).

No order.
No range.
No `ORDER BY` from the index.

Useful for exact-match dictionaries.
Less useful as a general SQL index.
Postgres hash indexes exist.
They are not the usual default.
InnoDB adaptive hash is an in-memory extra on top of btree, not a durable hash index you create.

## Bitmap Indexes

For each distinct value, a **bit vector** over row ids.

`status = 'open' AND region = 'eu'` becomes two bitmaps **ANDed**.

Good when:

- Cardinality is **low** (status, country, boolean)
- Reads are analytical
- Updates are rare (flipping a bit in a huge bitmap is expensive, and OLTP updates are the wrong workload)

Bad as the clustered OLTP PK.
Data warehouses and some Postgres bitmap index **scans** (build a bitmap from a btree, then heap fetch) are related: the btree is still btree.
The bitmap is how it batches heap access.

## Inverted Indexes (GIN, full text, JSON)

One document has **many** tokens (words, array elements, JSON keys).

The index is `token → list of row ids`.

That is a **GIN** (Generalized Inverted Index) in Postgres, or a Lucene segment in Elasticsearch.

Cheap: "which rows contain `error` and `timeout`?"
Expensive: updating a document rewrites many posting lists.

**GiST** is a generalized tree for types btree does not handle well (geometry, some ranges).
Think "plug-in btree," not "inverted."

## LSM Trees (write-optimized stores)

Cassandra, RocksDB, Lucene, many KV internals:

1. Writes go to a **memtable** (sorted in RAM) plus a commit log
2. Flush to an immutable **SSTable** on disk
3. **Compact** overlapping tables in the background

```plaintext
WAL + memtable  →  SST-L0  →  compact  →  SST-L1  →  ...
```

Point reads may check several levels.
**Bloom filters** skip SSTs that cannot contain the key.

Reads are more work than a hot btree.
Writes avoid btree page splits in place.
This is why LSM shows up in high-ingest stores.

Compaction is the hidden cost:
disk IO, write amplification, space until old SSTs die.

## Composite Indexes and the Left Prefix

Index `(user_id, created_at)` is a btree on the pair, in that order.

It can seek:

- `user_id = 42`
- `user_id = 42 AND created_at > t`

It generally **cannot** seek:

- `created_at > t` alone

The left column is the first sort key.
Skipping it is like a phone book with no last name.

`ORDER BY user_id, created_at` can ride this index.
`ORDER BY created_at` cannot.

Put equality columns first, then range columns, matching the query.
Or create a second index.

## Unique, Partial, Expression

- **Unique index**: lookup structure **and** a constraint.
  Two rows with the same key cannot both commit.
- **Partial index**: `WHERE status = 'open'`.
  Smaller.
  Only queries that imply that predicate can use it.
- **Expression / functional index**: `LOWER(email)`, `(data->>'sku')`.
  The query must use the same expression.

## Cost Model

Every `INSERT` / `UPDATE` / `DELETE` of an indexed column maintains every matching index.

Costs:

- Extra WAL and disk
- Slower writes
- More cache pressure (index pages vs table pages)
- Longer autovacuum / rebuild windows

Selectivity matters.
An index on `boolean is_deleted` where 99% are false is often useless for `WHERE is_deleted = false`.
The planner will seq-scan.

Too many indexes is a write-latency bug.
Too few is a read-latency bug.
`EXPLAIN ANALYZE` decides, not a rule of "index every FK."

## Planner Choices (Short)

- **Index seek / range**: few rows, good selectivity
- **Bitmap index scan**: several indexes ANDed, then heap fetch in order
- **Seq scan**: cheap when the table is small or most rows qualify
- **Index-only**: all needed columns in the index, visibility OK

If the estimate of row count is wrong (stale stats), the planner picks the wrong one.
`ANALYZE` is part of index operations.

## Interview Talking Points

- An index is a maintained **copy of keys** for lookup.
  Writes get slower.
- Default is **B+tree**: short tree, range scans via linked leaves, splits on full pages.
- Clustered vs secondary: InnoDB PK *is* the table.
  Postgres heap + secondary indexes.
- Composite indexes: **leftmost prefix**.
- Covering / index-only scans avoid the heap.
- LSM for write-heavy engines.
  Inverted/GIN for many-values-per-row.
- Hash = equality only.
  Bitmap = low-cardinality analytics.
