---
title: "PostgreSQL internals"
concepts:
  - slotted-page
  - heap-storage
  - hot-updates
  - buffer-cache
  - write-ahead-log
  - mvcc
  - vacuum
  - index-only-scan
  - query-planner
related:
  - fundamentals/19-relational-databases.md
  - fundamentals/22-database-indexes.md
  - fundamentals/25-database-concurrency-control.md
  - fundamentals/23-database-replication.md
---

# PostgreSQL internals

PostgreSQL is a process-based relational engine with an **unordered heap**, **MVCC on the heap**, a **write-ahead log**, and a **cost-based planner**.

This document treats Postgres as a **case study** of database internals, not as a configuration manual. The point is the set of problems every disk-backed OLTP engine has to solve:

- How bytes sit on disk
- How a page is laid out
- How a row is found
- How readers avoid blocking writers
- How a SQL string becomes a plan

Postgres is a useful example because those layers are relatively explicit and separable. InnoDB clusters the table on the primary key. LSM engines (RocksDB, Cassandra) write immutable files and compact later. The questions stay the same; the answers move. Each section below ends with an **Elsewhere** note naming the equivalent choice in another engine, because that mapping is what an interview is actually testing.

The generic versions of these topics live in the fundamentals notes: this document assumes them and only covers what Postgres does specifically.

Related: [Relational Databases](../fundamentals/19-relational-databases.md), [Indexes](../fundamentals/22-database-indexes.md), [Concurrency Control](../fundamentals/25-database-concurrency-control.md), [Replication](../fundamentals/23-database-replication.md).

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

A query never "touches a table" as a logical object. It pins **pages** in the buffer cache, then reads **tuples** (row versions) from those pages. Every layer below the executor deals in fixed-size pages, not rows.

**Elsewhere:** every disk-backed database has this stack. Names change (buffer pool, redo log, tablespace). The layers do not.

## On disk: the file system view

A cluster lives in one data directory (`PGDATA`).

Important pieces:

- **`base/`**: Per-database directories. Each table and index is one or more files named by **relfilenode** (an OID).
- **`pg_wal/`**: Write-ahead log segments, 16MB files by default.
- **`pg_xact/`**: The **commit log**: two bits per transaction ID recording whether it committed, aborted, or is still running. Visibility checks consult it.
- **`global/`**: Cluster-wide catalogs.
- **`pg_tblspc/`**: Symlinks to tablespaces on other filesystems.

A relation is not one file but a set of **forks**: the main data fork, a free space map (`_fsm`), and a visibility map (`_vm`). Both side maps matter later — inserts use the FSM to find a page with room, and index-only scans use the visibility map to skip the heap.

Each fork grows in **1GB segments** (`12345`, then `12345.1`). That is an OS-friendly file size cap, not a limit on logical table size.

System catalogs (`pg_class`, `pg_attribute`) are ordinary tables. The engine answers "what is `users`?" with the same machinery it uses to read any other row, which is why DDL participates in transactions like everything else.

**Elsewhere:** MySQL InnoDB uses an `.ibd` file per table, or a shared tablespace. SQLite is one file. The pattern is **catalog + data files + log**, not "a spreadsheet on disk."

## Pages

The unit of I/O is a **page** (block), 8KB by default. The size is fixed when the cluster is initialized and cannot be changed afterwards without a dump and reload.

A page is a **slotted page**: a header at the front, a growing array of item pointers behind it, and tuples packed from the end backwards. The free space is whatever is left in the middle.

```plaintext
 0                                                             8KB
 +---------+-------------------+-----------+-------------------+
 | header  | item pointers     | free      | tuples            |
 |         | 1, 2, 3 ...       | space     | ... 3, 2, 1       |
 +---------+-------------------+-----------+-------------------+
                        grows →             ← grows

 slot 3 → tuple 3        the address of that tuple is its ctid: (block, 3)
```

- **Page header**: LSN of the last WAL record that changed this page, checksum, flags, and the lower/upper offsets that bound the free space.
- **Item identifiers (line pointers)**: A slot array indexed from 1. Slot `i` points at a tuple, or is marked unused, dead, or redirected.
- **Tuples**: Row versions, each with a header (`xmin`, `xmax`, infomask) followed by the column data.
- **Free space**: The gap the two ends grow into.

A row version's address is its **`ctid`**: `(block_number, offset_number)` — page 42, slot 3. That pair is what an index leaf stores. Note that a `ctid` identifies a *row version*, not a row: an update that writes a new version elsewhere gives that row a new `ctid`, so `ctid` is never a stable key for the application to hold onto.

The indirection through line pointers buys three things:

- **Movable tuples**: The page can be compacted internally and tuples shifted, as long as slot numbers keep pointing at the right data — no index touched.
- **Redirects**: A slot can point at another slot instead of a tuple, which is what makes HOT updates possible.
- **Cheap deletes**: A slot is marked dead immediately; reclaiming the space it pointed at is deferred to vacuum.

**Elsewhere:** InnoDB pages are typically 16KB. Header, slot array, and records packed from the end is the classic slotted-page layout that nearly every row store uses.

## The heap

A Postgres table is a **heap**: rows are stored in no particular order. An insert lands on whatever page the free space map offers, so `ORDER BY id` without an index is a sort, not a walk down the file.

That is the opposite of InnoDB, where the **primary key is the table** (a clustered index). The trade-off is a real one in both directions:

- **Heap**: Inserts never split a primary-key B-tree, and every index is symmetric — the primary key is not privileged. The cost is that secondary indexes point at `ctid`, so an update that moves a row to another page must update **every** index on the table, unless it qualifies as a HOT update.
- **Clustered primary key**: A primary-key lookup reads one structure with the row already in it. The cost is an extra hop on every secondary index lookup (index leaf holds the primary key, then look up the clustered row) and a copy of the primary key in every secondary index.

Postgres has no maintained clustered index. `CLUSTER` rewrites the table in index order once, and the ordering decays with the next updates.

See [clustered vs secondary indexes](../fundamentals/22-database-indexes.md#clustered-vs-secondary-indexes) for the generic version of this trade-off.

**TOAST** (The Oversized-Attribute Storage Technique) handles values too large to sit comfortably in a page.
Once a row exceeds roughly 2KB, Postgres compresses the wide columns, and if that is not enough moves them into a side table in chunks; the heap row keeps a pointer.
A query that never selects the wide column never reads those chunks, which is why `SELECT *` on a JSONB-heavy table is far more expensive than selecting the three columns you need.

**Elsewhere:** MySQL overflow pages, SQL Server LOB pages. Large cells never live entirely in the main row.

### HOT updates

An `UPDATE` in Postgres is not an in-place edit: it writes a **new tuple** and marks the old one as superseded. Normally that means every index on the table gains an entry pointing at the new `ctid`.

The **Heap-Only Tuple** optimization avoids that when two conditions hold: no indexed column changed, and the new version fits on the **same page**. The old line pointer is turned into a redirect, so existing index entries stay correct without being rewritten.

```plaintext
before                            after a HOT update
  slot 1 → tuple v1                 slot 1 → redirect to slot 2
                                    slot 2 → tuple v2
  index entry → (block, 1)          index entry → (block, 1)   unchanged
```

The second condition is why `fillfactor` matters: setting it below 100 leaves room on each page for future versions, trading space for a much higher HOT rate on update-heavy tables. When either condition fails you get a new heap tuple plus an entry in every index, which is how a wide-update workload bloats indexes even though no indexed value changed.

## Buffer cache

`shared_buffers` is a fixed-size shared memory array of pages. A backend **pins** a page while it reads or writes it, then unpins. Pages that were modified are marked dirty and written out later — at a checkpoint, by the background writer, or by whichever backend needs to evict them.

Eviction is a **clock sweep**: each buffer has a usage counter that goes up on access and down as the sweep passes, and the first buffer to reach zero is evicted. It approximates LRU without maintaining a list under a global lock.

Postgres also relies on the **OS page cache**, so the same bytes can sit in both.
That double buffering is deliberate: the engine stays simpler and gets to use RAM it never claimed.
The consequence is sizing guidance — `shared_buffers` is usually a fraction of RAM (around 25%), with the rest left for the kernel cache and for `work_mem`.
Engines that use `O_DIRECT` and own their cache outright (the InnoDB buffer pool) are sized as *the* memory budget instead.

A **checkpoint** flushes all dirty pages so that WAL older than that point can be recycled and recovery has a bounded starting position. The tuning is a trade-off in one dimension: too rare and crash recovery replays a long WAL, too frequent and you get write storms and repeated full-page images.

**Elsewhere:** buffer pool, dirty page flushing, checkpoint versus fuzzy checkpoint. The same control problem with different names.

## WAL

Durability is **write-ahead**: describe the change in the log and flush the log **before** the client sees `COMMIT`. Under load, concurrent commits are batched into one flush (group commit), so throughput does not fall to one fsync per transaction.

```plaintext
change made to a page in shared_buffers
  → WAL record appended (a full-page image if this is the page's first change since the checkpoint)
  → WAL flushed to disk
  → COMMIT returns to the client
  → the heap and index pages themselves are written later
```

Every WAL record has an **LSN** (log sequence number), a monotonic byte position in the log stream, and every page header stores the LSN of the last record that modified it. That single field is what makes recovery work: after a crash, replay starts at the last checkpoint and applies a record to a page only if the page's LSN is older than the record. Data files may lag the log; redo brings them forward without applying anything twice.

**Full-page writes** defend against torn pages. The OS and disk write in units smaller than 8KB, so a crash mid-write can leave a page half old and half new, which no delta record can repair. Logging the entire page the first time it is touched after a checkpoint gives recovery a known-good copy to start from. The cost is WAL volume, and it is the main reason WAL spikes right after each checkpoint.

The same stream is the replication stream: physical replication ships WAL to a standby that replays it. See [replication](../fundamentals/23-database-replication.md).

**Elsewhere:** the InnoDB redo log, RocksDB's WAL. MySQL's binlog is a separate, logical log used for replication rather than crash recovery. The invariant is that **the log is the source of truth for recovery** and the data files are a materialized cache of it.

## MVCC

Postgres implements MVCC **in the heap**: old row versions are ordinary tuples sitting on ordinary pages. The general mechanism — multiple versions, snapshots, and which anomalies each isolation level still allows — is covered in [concurrency control](../fundamentals/25-database-concurrency-control.md#mvcc). This section is about where Postgres puts the versions and what that costs.

Every tuple header carries two transaction IDs:

- **`xmin`**: The transaction that created this version.
- **`xmax`**: The transaction that deleted or superseded it, or 0 while the version is still current.

So an `UPDATE` is really an insert plus a stamp on the old version, and a `DELETE` only sets `xmax`. Nothing is erased at write time.

A statement or transaction takes a **snapshot**: the set of transaction IDs that had committed at that moment, plus the list of those still in flight. For each tuple it reads, the executor asks whether that version is visible under the snapshot — roughly, was `xmin` committed as of the snapshot, and is `xmax` either absent or not yet committed as of the snapshot. Commit status comes from `pg_xact`.

```plaintext
row id=1, balance=100   xmin=10  xmax=20     created by tx 10, superseded by tx 20
row id=1, balance=80    xmin=20  xmax=0      created by tx 20, current version
```

A snapshot taken when only tx 10 had committed still reads `100`. A snapshot taken after tx 20 committed reads `80`. Neither reader took a lock on the row, which is the headline property: **readers do not block writers and writers do not block readers**; only two writers on the same row actually wait for each other.

**Hint bits** are a cache of that commit lookup. The first transaction to read a tuple after its creator committed records "this `xmin` is committed" in the tuple's infomask so later readers skip `pg_xact` entirely. It is also why a plain `SELECT` can dirty pages and generate I/O on a table nobody has written to recently.

Isolation levels are the same snapshot machinery with different timing, matching the table in [concurrency control](../fundamentals/25-database-concurrency-control.md#isolation-levels):

- **Read committed** (the default): A fresh snapshot at the start of every statement.
- **Repeatable read**: One snapshot for the whole transaction. This is snapshot isolation, so it still allows write skew.
- **Serializable**: Snapshot isolation plus SSI (serializable snapshot isolation), which tracks read/write dependencies and aborts a transaction that would create a non-serializable cycle. The cost is retryable serialization failures, not blocking.

Keeping versions in the heap has three specific costs:

- **Dead tuples**: Superseded versions occupy space until vacuum reclaims them.
- **Bloat**: If vacuum cannot keep up, the heap and its indexes keep growing, so scans read more pages for the same live rows and the working set stops fitting in cache.
- **Transaction ID wraparound**: Transaction IDs are 32-bit, so old tuples must be **frozen** (marked visible to everyone) before their IDs can be reused. If freezing falls far enough behind, the cluster refuses new write transactions rather than risk rows becoming invisible.

**Elsewhere:** InnoDB keeps old versions in **undo logs** and updates the clustered row in place, so readers reconstruct the earlier image by walking undo. LSM stores keep superseded keys in older SSTables until compaction. Same idea, different place for the versions — and a different cleanup process as a result.

## VACUUM

VACUUM is the garbage collector for heap MVCC. It:

- Reclaims space from dead tuples and removes the matching index entries
- Updates the **visibility map**, marking pages where every tuple is visible to every snapshot, which is what enables index-only scans
- **Freezes** old `xmin` values so transaction IDs can be reused
- Updates the free space map so inserts can reuse the reclaimed room

The rule that decides what it can remove is the **xmin horizon**: a version is only removable once no snapshot in the cluster could still need it. Any of the following stalls cleanup **database-wide** while writes keep producing dead tuples:

- One long-running query
- One forgotten `idle in transaction` session
- One abandoned replication slot

This is the concrete form of the "keep transactions short" argument in [concurrency control](../fundamentals/25-database-concurrency-control.md#mvcc) — it is a storage argument, not only a locking one.

**Autovacuum** runs this in the background, triggered by how many rows have changed since the last pass. When it is throttled too aggressively for the write rate, the visible symptoms arrive in order: tables bloat, plans that relied on index-only scans get slower, and eventually wraparound warnings appear.

`VACUUM FULL` is a different operation: it rewrites the table into a fresh file under an `ACCESS EXCLUSIVE` lock. It is how you recover space already lost to bloat, not part of the steady state.

**Elsewhere:** InnoDB's purge thread trimming undo, LSM compaction dropping obsolete keys. Every MVCC or LSM engine has a **reclaim process**, and every one of them charges interest if you do not let it run.

## Commit, locks, and visibility

Pulling the previous three sections together, a `COMMIT` is:

1. Write the commit record to WAL and mark the transaction committed in `pg_xact`.
2. Flush the WAL up to that record.
3. Return to the client.
4. New snapshots now consider tuples with that `xmin` visible.

The tuples were already in the heap before the commit; nothing is copied at commit time. That is why commit cost is dominated by one WAL flush and is largely independent of how many rows the transaction touched.

Row locks are stored in the tuple itself (`xmax` plus infomask bits, or a multixact ID when several transactions lock the same row), with a shared lock table used only for waiting.
The distinction worth stating explicitly in an interview: **visibility and locking are separate mechanisms**.
You can freely read a row another transaction is updating, because you read the older version — but `SELECT ... FOR UPDATE` on that row waits until the writer commits or aborts.

Deadlocks between those waits are detected on a timer and resolved by aborting one transaction, which surfaces to the application as an error to retry.

**Elsewhere:** lock versus version is the pessimistic-versus-MVCC split described in [concurrency control](../fundamentals/25-database-concurrency-control.md#pessimistic-vs-optimistic-control).

## Indexes and access methods

The generic material — B+trees, composite keys, covering indexes, hash and bitmap and inverted indexes — is in [database indexes](../fundamentals/22-database-indexes.md). What is Postgres-specific is the relationship between an index and the heap:

- An index leaf stores **`ctid`**, not the primary key, so a lookup is index seek then heap fetch, with no extra tree traversal.
- Index entries carry **no visibility information**. An index scan therefore has to fetch the heap tuple to find out whether the row version is visible to its snapshot, even when the index already contains every column the query selects.

That second point is why the visibility map exists. An **index-only scan** reads the index and checks the visibility map: if the page is marked all-visible it skips the heap fetch, otherwise it falls back to fetching. So the same query and the same index can behave very differently depending on how recently the table was vacuumed — an index-only scan is a vacuum outcome as much as an index design outcome.

Beyond the default B-tree, the access methods that come up:

| Access method | What it stores                                        | Good for                                                                                       |
| ------------- | ----------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| **B-tree**    | Sorted keys, each pointing at a `ctid`                | The default: equality, ranges, sort order, uniqueness                                          |
| **BRIN**      | Min and max per range of pages, so the index is tiny  | Large append-mostly tables whose physical order tracks the column; near useless if it does not |
| **GIN**       | Inverted index: token to a list of `ctid`s            | Arrays, JSONB, full-text search — many entries per row                                         |
| **GiST**      | Balanced tree over user-defined containment operators | Geometry, ranges, nearest-neighbour queries                                                    |
| **Hash**      | Buckets of hashed keys, equality only                 | Rarely worth choosing; a B-tree answers equality too                                           |

GIN is the inverted index from [database indexes](../fundamentals/22-database-indexes.md#inverted-indexes-gin-full-text-json), and GiST is the "pluggable B-tree" framework it describes: the tree shape is fixed, the comparison semantics are supplied by the data type. BRIN has no real equivalent in that note because it is a *summary*, not a lookup structure — it can only tell the executor which page ranges cannot contain a match.

**Elsewhere:** the same menu appears everywhere under different names (MySQL covering indexes, SQL Server columnstore, an Elasticsearch inverted index). The Postgres twist is the combination of **heap + `ctid` + visibility map**.

## Processes and memory

Postgres runs **one OS process per connection**, forked by the postmaster, alongside background processes: the checkpointer, WAL writer, background writer, autovacuum launcher and workers, and the cumulative statistics subsystem (a separate collector process in older versions, shared memory since PostgreSQL 15).

Memory therefore has two very different budgets. `shared_buffers` is shared once for the whole cluster. `work_mem` is per sort or hash node, per backend — a single query with three hash joins can use several multiples of it, and a hundred such connections can exhaust RAM while `shared_buffers` still looks conservatively sized.

A connection also costs a process: a fork, its own memory, and a share of every snapshot computation. That is the whole reason connection poolers (PgBouncer, ProxySQL) exist — the right answer to "we need 5,000 clients" is a pool in front of a few dozen backends, not 5,000 backends.

**Elsewhere:** MySQL is typically thread-per-connection inside one process, which is cheaper per connection but shares a fate across threads. The transferable lesson is that **the unit of execution has a memory budget**, and pool sizing should follow from that budget rather than from a client count.

## Query planner

A statement goes through four stages:

1. **Parse**: SQL text to a parse tree.
2. **Rewrite**: Expand views, rules, and row-level security predicates.
3. **Plan**: Choose access paths and join order.
4. **Execute**: Pull tuples through a tree of nodes, Volcano-style — each node asks its children for the next row.

The planner is **cost-based**, not rule-based. `ANALYZE` collects statistics per column (histograms, most common values, null fraction, distinct estimate), the planner estimates how many rows each node produces, and it assigns an I/O-plus-CPU cost to each candidate plan and picks the cheapest.

```plaintext
Seq Scan on orders          no useful index, or most rows qualify anyway
Index Scan                  B-tree seek plus one heap fetch per matching row
Index Only Scan             as above, heap fetch skipped where the page is all-visible
Bitmap Index Scan + heap    many matches: collect ctids, sort by page, read the heap in physical order
Nested Loop / Hash Join / Merge Join
```

The bitmap scan is worth understanding because it is where the heap's disorder shows up: for a few hundred matching rows scattered across the file, fetching them in physical page order beats fetching them one at a time in index order. It also lets two indexes be combined by ANDing or ORing their bitmaps. See [bitmap indexes](../fundamentals/22-database-indexes.md#bitmap-indexes) for how that differs from a stored bitmap index.

Everything rests on the row estimates, so the failure mode is always the same shape: a bad estimate picks a plan that is catastrophic at the real cardinality — a nested loop chosen for 10 expected rows, executed against 10 million.
The usual causes are stale statistics after a bulk load, and correlated columns the planner assumes are independent (`city` and `postcode`), which `CREATE STATISTICS` exists to fix.
With more than about a dozen joined relations the planner stops searching exhaustively and switches to a genetic search, so very wide joins can also produce unstable plans.

The tooling follows from that: `EXPLAIN` shows the chosen plan, `EXPLAIN ANALYZE` runs it and prints **actual** rows next to the estimate, and `EXPLAIN (ANALYZE, BUFFERS)` adds how many pages came from the buffer cache versus disk. The first place to look is always the node where estimated and actual row counts diverge by orders of magnitude.

`work_mem` bounds each sort and hash node; exceeding it spills to a temporary file on disk, which `EXPLAIN ANALYZE` reports as an external merge or a batched hash join.

**Elsewhere:** this is the standard System R / Selinger optimizer story. Oracle, SQL Server, MySQL 8, and SQLite all estimate cardinality and search a plan space. The transferable skill is **reading a plan**, not memorizing knobs like `enable_seqscan`.

## Pattern catalog

| Layer         | Postgres choice                          | Same idea elsewhere                       |
| ------------- | ---------------------------------------- | ----------------------------------------- |
| I/O unit      | 8KB slotted page                         | InnoDB 16KB pages, slotted records        |
| Table         | Unordered heap addressed by `ctid`       | Clustered primary key (InnoDB), LSM SSTs  |
| Cache         | `shared_buffers` plus the OS page cache  | Buffer pool, sometimes with `O_DIRECT`    |
| Durability    | WAL, full-page writes, checkpoints       | Redo log, group commit                    |
| MVCC          | Extra heap tuple versions plus snapshots | Undo log versions, superseded LSM keys    |
| Reclaim       | VACUUM, freezing, visibility map         | InnoDB purge, LSM compaction              |
| Lookup        | B+tree to `ctid`, then the heap          | B+tree to primary key to row, LSM + bloom |
| Process model | Process per connection, per-node memory  | Thread per connection, session buffers    |
| SQL           | Cost-based planner, `EXPLAIN ANALYZE`    | Every serious SQL engine                  |

## What this case study is not

- Not a column store: there is no default vectorized, compressed analytical execution engine (compare DuckDB or a Parquet-based engine).
- Not an LSM ingest path: writes go to a B-tree and a heap, not to a memtable (compare Cassandra or RocksDB).
- Not clustered on the primary key by default (compare InnoDB).
- Not a distributed SQL planner: forks and extensions exist, but vanilla Postgres is one primary plus replicas (compare Spanner or CockroachDB).

## Interview talking points

- Pages are slotted, and a row version's address is `(block, offset)` — a `ctid`, which changes when the row moves.
- The Postgres heap is **not** in primary-key order. Indexes point at `ctid`; contrast with InnoDB, where the primary key is the table.
- MVCC keeps old versions as heap tuples stamped with `xmin` and `xmax`, so readers never block writers — and vacuum is therefore mandatory, not optional.
- A long-running transaction holds back the xmin horizon and stalls cleanup cluster-wide. Bloat is a concurrency symptom.
- WAL is both the durability mechanism and the replication stream; data files legitimately lag the log, and the page LSN is what reconciles them.
- Index-only scans depend on the visibility map, so they depend on vacuum keeping up.
- Planning is cost plus statistics: debug with `EXPLAIN ANALYZE` and look for estimate-versus-actual divergence, not by adding indexes speculatively.
- Process-per-connection plus per-node `work_mem` is the reason connection pooling is not optional at scale.
