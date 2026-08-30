---
title: "PostgreSQL internals"
concepts:
  - mvcc
  - write-ahead-log
  - buffer-cache
  - vacuum
  - heap-storage
  - hot-updates
  - query-planner
  - index-only-scan
related:
  - fundamentals/19-relational-databases.md
  - fundamentals/22-database-indexes.md
  - fundamentals/25-database-concurrency-control.md
  - fundamentals/23-database-replication.md
---

# PostgreSQL internals

PostgreSQL is a process-based relational engine with an **unordered heap**, **MVCC on the heap**, a **WAL**, and a **cost-based planner**.

This document treats Postgres as a **case study** of database internals, not as a configuration manual.

The same problems show up in MySQL, SQL Server, and most OLTP engines:

- How bytes sit on disk
- How a page is laid out
- How a row is found
- How readers avoid blocking writers
- How a SQL string becomes a plan

Postgres is a useful example because those layers are relatively explicit.
InnoDB clusters the table on the primary key.
LSM engines (RocksDB, Cassandra) write immutable files and compact later.
The questions stay the same.
The answers move.

Related: [Relational Databases](../fundamentals/19-relational-databases.md), [Indexes](../fundamentals/22-database-indexes.md),
[Concurrency Control](../fundamentals/25-database-concurrency-control.md), [Replication](../fundamentals/23-database-replication.md).

## The stack

```plaintext
SQL
  → parser / rewriter
  → planner / optimizer
  → executor
       → buffer cache  (shared_buffers)
            → OS page cache
                 → files: heap, indexes, WAL
```

A query never "touches a table" as a logical object.
It pins **pages** in the buffer cache, then reads **tuples** from those pages.

**Elsewhere:** every disk-backed database has this stack.
Names change (buffer pool, redo log, tablespace).
The layers do not.

## On disk: the file system view

A cluster lives in one data directory (`PGDATA`).

Important pieces:

- **`base/`**: Per-database directories. Each table and index is one or more files named by **relfilenode** (an OID).
- **`pg_wal/`**: Write-ahead log segments (usually 16MB files).
- **`pg_tblspc/`**: Tablespaces (files on another filesystem).
- **`global/`**: Cluster-wide catalogs.

A relation file grows in **1GB segments** (`12345`, then `12345.1`).
That is an OS-friendly cap, not a logical table size limit.

System catalogs (`pg_class`, `pg_attribute`) are ordinary tables.
The engine looks up "what is `users`?" the same way it looks up a row.

**Elsewhere:** MySQL InnoDB uses `.ibd` per table (or a shared tablespace).
SQLite is one file.
The pattern is: **catalog + data files + log**, not "a spreadsheet on disk."

## Pages

The unit of IO is a **page** (block).
Default size is **8KB**.
You pick this at init.
You do not change it later.

```plaintext
  0                         item pointers                         8KB
  |-- page header --|-- 1, 2, 3, ... --|-- free --|-- tuples --|
                                              growing ←     → growing
```

- **Page header**: LSN, checksum, flags, lower/upper offsets.
- **Item identifiers (line pointers)**: An array from the start of the page. Slot `i` points at a tuple (or is unused or redirected).
- **Tuples**: Packed from the end of the page toward the middle.
- **Free space**: The gap in the middle.

A row version's address is **`ctid`**: `(block_number, offset_number)`.
That is the heap pointer indexes store.

Why line pointers exist:

- You can move a tuple inside the page without breaking every index, if the slot stays
- HOT updates (below) use redirects
- You can mark a slot empty without compacting immediately

The **free space map (FSM)** is a side structure.
Inserts ask it "which page has room?" instead of scanning the heap.

**Elsewhere:** InnoDB pages are typically 16KB.
The header + slot + records-from-the-end layout is a classic slotted page.
Almost every row store uses it.

## The heap

Postgres tables are a **heap**: rows are not stored in primary-key order.

Insert puts a tuple on some page with free space.
`ORDER BY id` without an index is a sort, not a walk of the file.

That is the opposite of InnoDB, where the **primary key is the table** (clustered index).

Trade-off:

- **Heap**: Inserts do not split a PK btree. Secondary indexes point at `ctid`. An update that moves the row must update **every** secondary index (unless HOT).
- **Clustered PK**: PK lookup is one tree. Secondary indexes store the PK, then you look up the clustered row (bookmark lookup).

**TOAST** (The Oversized-Attribute Storage Technique):
values that do not fit on a page (long TEXT/JSONB/BYTEA) go to a side table, maybe compressed.
The heap row keeps a pointer.
**Elsewhere:** MySQL overflow pages, SQL Server LOB pages.
Large cells never live entirely in the main row.

### HOT updates

If an `UPDATE` does not change indexed columns, and the new version fits on the **same page**, Postgres can write a Heap-Only Tuple.
Indexes still point at the old slot.
The slot redirects to the new version.

That avoids index maintenance on the hot path.
If indexed columns change, or the page is full, you get a new heap tuple and index updates.
That is why a widening update storm bloats both heap and indexes.

## Buffer cache

`shared_buffers` is a shared memory cache of pages.

A backend **pins** a page, reads or writes it, then unpins.
Dirty pages flush on checkpoint, on eviction, and via the background writer.

Postgres also relies on the **OS page cache**.
The same bytes may sit in both.
That is a deliberate choice (simpler, uses RAM the engine did not claim).

Some engines use `O_DIRECT` and own the cache (InnoDB buffer pool as the main cache).
Then you size the buffer pool as *the* RAM budget.
In Postgres you size `shared_buffers` as a fraction (often ~25%) and leave the rest for the kernel and `work_mem`.

**Checkpoint:** flush dirty pages so WAL older than that point can be recycled.
Too rare: crash recovery replays a long WAL.
Too often: write storms.

**Elsewhere:** buffer pool, dirty page flushing, checkpoint vs fuzzy checkpoint.
Same control problem.

## WAL

Durability is **write-ahead**: describe the change in the log and flush it **before** the client sees `COMMIT` (with group commit, several transactions share one flush).

```plaintext
change in memory
  → WAL record (and often a full-page image after a checkpoint)
  → fsync WAL
  → COMMIT returns
  → heap/index pages written later
```

After a crash, recovery **replays WAL** from the last checkpoint.
Heap pages may be older than the log.
Redo makes them match.

**Full-page writes:** after a checkpoint, the first change to a page logs the whole page.
That defends against a torn page (4KB OS write, 8KB Postgres page).
You pay extra WAL for safety.

Physical replication is shipping this WAL to another node.
See [replication](../fundamentals/23-database-replication.md).

**Elsewhere:** redo log (InnoDB), binlog (MySQL logical, different job), RocksDB WAL.
The invariant is: **log is the source of truth for crash recovery**.
Data files are a cache of that history.

## MVCC

Postgres implements MVCC **in the heap**.

Each tuple version has:

- **`xmin`**: Transaction that created it.
- **`xmax`**: Transaction that deleted or replaced it (0 if live).

A query takes a **snapshot**: which transaction IDs are in progress, which are committed.

The executor, for each tuple, asks: "is this version visible to my snapshot?"

```plaintext
row id=1, balance=100   xmin=10  xmax=20     ← created by tx 10, replaced by tx 20
row id=1, balance=80    xmin=20  xmax=0      ← created by tx 20, still live
```

A snapshot that started when only tx 10 had committed still sees `100`.
A later snapshot sees `80`.
Readers do not take a lock on the row to get a stable view.

**Elsewhere:** InnoDB keeps old versions in **undo logs**, not extra heap tuples.
The clustered row is updated in place.
Readers reconstruct the old image from undo.
LSM stores keep old versions until compaction.

Same idea: **multiple versions so readers do not block writers**.
Different place the versions live.

Costs in Postgres:

- Updates produce **dead tuples** until VACUUM runs
- Table and index **bloat** if vacuum cannot keep up
- **Transaction ID wraparound**: `xmin` is 32-bit. Very old tuples must be **frozen** so IDs can be reused. If freeze lag is ignored, the cluster stops writes to protect data.

**Hint bits** cache "this xmin committed" on the tuple so later readers skip `pg_xact`.
The first reader after commit may set them.
That is an extra write for a read.

Isolation:

- **Read Committed** (default): New statement, new snapshot.
- **Repeatable Read**: One snapshot for the whole transaction.
- **Serializable**: Snapshot plus SSI (detect write skew, abort one transaction).

Anomalies and when to lock: [concurrency control](../fundamentals/25-database-concurrency-control.md).

## VACUUM

VACUUM is the garbage collector for heap MVCC.

It:

- Reclaims space from tuples no snapshot can see
- Updates the **visibility map** (all-visible pages → index-only scans can skip the heap)
- **Freezes** old `xmin` values
- Removes dead index entries (or leaves work for later)

**Autovacuum** does this in the background.
If it is too weak, tables bloat, queries randomly get slower, and eventually wraparound alarms fire.

`VACUUM FULL` rewrites the table.
It is a lock-heavy compaction, not the normal path.

**Elsewhere:** InnoDB purge thread (undo).
LSM compaction (drop obsolete keys).
Every MVCC or LSM engine has a **reclaim process**.
If you do not run it, you buy space and latency with interest.

## Indexes (Postgres-specific)

Generic btree / hash / bitmap / LSM: [database indexes](../fundamentals/22-database-indexes.md).

Postgres defaults:

- B+tree on the heap. Leaf value is **`ctid`**, not the primary key.
- The table is **not** clustered on the PK unless you `CLUSTER` (a one-shot rewrite, not maintained).

**Index-only scan:** read the btree, skip the heap, if the visibility map says the page is all-visible.
Otherwise it **heap-fetches** anyway.
Vacuum quality decides whether this optimization works.

Other access methods:

| Access method | What it stores                          | Good for                                                                     |
| ------------- | --------------------------------------- | ---------------------------------------------------------------------------- |
| **BRIN**      | Min/max per range of pages (tiny index) | Append-mostly columns physically in order; poor if values are random on disk |
| **GIN**       | Inverted index                          | Arrays, JSONB, full-text search                                              |
| **GiST**      | Generalized tree                        | Geometry, ranges                                                             |
| **Hash**      | Equality only                           | Rare as a user-created index                                                 |

**Elsewhere:** same menu with different names (MySQL covering index, SQL Server columnstore, ES inverted index).
The Postgres twist is **heap + ctid + visibility map**.

## Processes

Postgres is **one process per connection** (plus a postmaster and helpers).

Helpers include checkpointer, WAL writer, autovacuum workers, and the stats collector / cumulative stats.

A connection is a real OS process.
`work_mem` is per sort/hash, per node, per backend.
A hundred connections doing big sorts can blow RAM even if `shared_buffers` looks fine.

**Elsewhere:** MySQL is typically **thread per connection** in one process.
Connection pooling (PgBouncer, ProxySQL) exists because processes and threads are not free.
The internals lesson: **the execution unit has a memory budget**.
Design pools around that, not around "max clients on the slide."

## Query optimizer

Path of a statement:

1. **Parse**: SQL to a tree.
2. **Rewrite**: Views, rules, RLS.
3. **Plan**: Pick an access path.
4. **Execute**: Pull tuples through nodes (Volcano-style iterator).

The planner is **cost-based**.
It estimates rows using **statistics** (`ANALYZE`: histograms, most-common values, null fraction).
It assigns a cost to seq scan, index scan, bitmap scan, nested loop, hash join, and merge join.
It picks a cheap tree.

```plaintext
Seq Scan on orders          — no useful index, or most rows qualify
Index Scan                  — btree seek + heap fetch per row
Bitmap Index Scan + heap    — many matches, sort tids, then heap in order
Hash Join / Nested Loop / Merge Join
```

If stats are stale, the estimate is wrong, and you get a nested loop over millions of rows.

`EXPLAIN` shows the chosen tree.
`EXPLAIN ANALYZE` runs it and shows **actual** rows vs estimate.

`work_mem` bounds in-memory sort and hash.
It spills to disk if the node is bigger.

**Elsewhere:** this is the standard SQL optimizer story (System R, Selinger).
Oracle, SQL Server, MySQL 8, and SQLite all estimate and search a plan space.
The transferable skill is **read the plan**, not memorize `enable_seqscan`.

## Transactions, locks, and WAL together

A `COMMIT`:

1. Transaction status → committed (in WAL, then `pg_xact`).
2. WAL flushed.
3. Client unblocked.
4. Tuples already in the heap with that `xmin` become visible to new snapshots.

Row locks (`FOR UPDATE`) live in the tuple (`xmax` / infomask) and in a lock table for waiting.
MVCC visibility is not the same as an exclusive lock.
You can read a row another transaction is updating (you see the old version).
You cannot take `FOR UPDATE` on it until they commit or abort.

**Elsewhere:** lock vs version is the pessimistic vs MVCC split.
See [concurrency control](../fundamentals/25-database-concurrency-control.md).

## Pattern catalog

| Layer      | Postgres choice                   | Same idea elsewhere                      |
| ---------- | --------------------------------- | ---------------------------------------- |
| IO unit    | 8KB slotted page                  | InnoDB 16KB pages, slotted records       |
| Table      | Unordered heap + `ctid`           | Clustered PK (InnoDB), LSM SST (RocksDB) |
| Cache      | `shared_buffers` + OS cache       | Buffer pool, sometimes O_DIRECT          |
| Durability | WAL, full-page writes, checkpoint | Redo log, group commit                   |
| MVCC       | Heap tuple versions + snapshot    | Undo log versions, LSM old keys          |
| Reclaim    | VACUUM, freeze, visibility map    | Purge, compaction                        |
| Lookup     | B+tree → `ctid`                   | B+tree → PK → row, LSM + bloom           |
| SQL        | Cost-based planner + `EXPLAIN`    | Every serious SQL engine                 |

## What this case study is not

- Not a warehouse column store (no default vectorized execution like DuckDB/Parquet engines).
- Not an LSM-first ingest path (see Cassandra / RocksDB).
- Not automatically clustered on PK (see InnoDB).
- Not a distributed SQL planner (see Spanner, Cockroach). Extensions and forks exist, but vanilla Postgres is one node plus replicas.

## Interview talking points

- Pages are slotted. Row id is `(block, offset)`.
- Postgres heap is **not** PK order. Indexes point at `ctid`. Contrast with InnoDB clustering.
- MVCC = extra tuple versions on the heap. Readers do not block writers. VACUUM is mandatory.
- WAL is the durability and replication stream. Data files lag the log.
- The optimizer is cost + stats. `EXPLAIN ANALYZE` is how you debug, not guesses about indexes.
- Process-per-connection and `work_mem` explain why pooling exists.
