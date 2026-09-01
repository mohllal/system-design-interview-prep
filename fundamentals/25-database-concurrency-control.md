---
title: "Database concurrency control"
concepts:
  - transaction-isolation-levels
  - concurrency-anomalies
  - write-skew
  - mvcc
  - snapshot-isolation
  - pessimistic-locking
  - optimistic-concurrency-control
  - lock-contention-and-hotspots
related:
  - fundamentals/19-relational-databases.md
  - fundamentals/23-database-replication.md
  - fundamentals/24-database-partitioning.md
  - fundamentals/26-concurrency-control.md
  - advanced/06-postgresql-internals.md
---

# Database concurrency control

Database concurrency control keeps data correct when many transactions read and write the same records concurrently.

Without it, systems silently violate invariants: wrong balances, double bookings, stale overwrites. Correctness is not free — it is paid for in latency and throughput — so the goal is the **weakest mechanism that still protects the invariant you actually have**.

Two questions decide every case:

1. Which **anomaly** would break this workflow?
2. Is the cheapest defense a **lock**, a **version check**, or a stronger **isolation level**?

Everything up to the last section describes transactions racing on one node: a single primary ordering its own commits. Replica reads and cross-shard transactions change that picture, and are covered at the end.

## Common concurrency anomalies

Ordered roughly from the ones every engine already prevents to the ones that survive most production configurations.

### Dirty write

One uncommitted write overwrites another uncommitted write.

Every mainstream engine prevents this at every isolation level by holding a row's write lock until commit. It mostly matters as the reason those write locks exist at all.

### Dirty read

A transaction reads data written by another transaction that has not committed yet. If the writer then rolls back, the reader acted on a value that never existed.

### Non-repeatable read

Inside one transaction, the same row is read twice and returns different values because another transaction committed an update in between.

1. Transaction A reads record X = 100
2. Transaction B writes X = 90 and commits
3. Transaction A reads X again and gets 90 — a row it already read changed underneath it

### Phantom read

A transaction repeats a **range** query and gets extra or missing rows, because another transaction inserted or deleted rows matching the predicate.

1. Transaction A reads all records with `age > 18`
2. Transaction B inserts a record with `age = 19` and commits
3. Transaction A repeats the query and now sees a row that was not there before

The difference from a non-repeatable read is *what* changed: a row you already read, versus the **set** of rows matching a predicate.

### Lost update

Two transactions read the same value, each computes a new value from it, and both write back. The second write silently erases the first.

1. Balance = 100
2. Transaction A reads 100, Transaction B reads 100
3. A writes 90, B writes 80
4. Final value is 80 — A's deduction is gone

The cheapest fix is to stop round-tripping the value through the application: `UPDATE accounts SET balance = balance - 10 WHERE id = 42` is applied atomically by the engine. When the new value cannot be written as an in-place expression, you need a lock or a version check instead.

### Write skew

Two transactions read overlapping state, then each writes a **different** row. Each write is individually legal, but together they break an invariant that spans rows. Because no single row was concurrently modified, row-level locks and snapshot reads both miss it.

Classic example, "at least one doctor must be on call":

1. Transaction A reads two doctors on call; Transaction B reads two doctors on call
2. A takes doctor 1 off call; B takes doctor 2 off call
3. Both transactions commit
4. Nobody is on call — the invariant is violated

Write skew is the anomaly snapshot isolation does **not** prevent, which is what makes it the interesting one in interviews. Fixes: run `SERIALIZABLE`, lock the rows you *read* (`SELECT ... FOR UPDATE` over the whole set), or materialize the invariant into a single row you can lock (an `on_call_count` row).

## Isolation levels

An isolation level is the database's promise about which of the anomalies above you will not see.

| Isolation level                      | Prevents                                | Still allows                                             | Notes                                                             |
| ------------------------------------ | --------------------------------------- | -------------------------------------------------------- | ----------------------------------------------------------------- |
| Read uncommitted                     | Dirty writes                            | Dirty reads, and everything below                        | Rarely useful; PostgreSQL treats it as read committed             |
| Read committed                       | Dirty reads                             | Non-repeatable reads, phantoms, lost updates, write skew | Default in PostgreSQL, Oracle, and SQL Server                     |
| Repeatable read / snapshot isolation | Non-repeatable reads, most lost updates | Write skew; phantoms depending on the engine             | Default in InnoDB, which also blocks most phantoms with gap locks |
| Serializable                         | Everything above                        | Nothing in this list                                     | Highest cost: retryable aborts under SSI, or blocking under 2PL   |

The names are not portable. PostgreSQL's `REPEATABLE READ` and Oracle's `SERIALIZABLE` are both snapshot isolation; InnoDB's `REPEATABLE READ` adds gap locks that block most phantoms. Ask what an engine prevents, not what it calls the level.

Stronger is not free: `SERIALIZABLE` costs either aborts you have to retry (PostgreSQL's serializable snapshot isolation) or blocking (two-phase locking). Most engines let you set the level per transaction, so raise it on the few transactions that need it rather than globally.

## How engines implement isolation

### Two-phase locking

Take locks as you go — shared for reads, exclusive for writes — and release none until commit. Correct and easy to reason about, but readers block writers and writers block readers, so throughput collapses under contention. Today it survives mainly as the model behind `SERIALIZABLE` in engines without SSI, and as what `FOR UPDATE` gives you on a single row.

### MVCC

Modern engines (PostgreSQL, InnoDB, Oracle) keep **multiple versions** of each row. A write creates a new version stamped with the writing transaction; a reader gets a **snapshot** — the set of versions committed as of a chosen point in time. The headline consequence is that **readers do not block writers and writers do not block readers**; only writer/writer conflicts on the same row actually wait.

Isolation levels then fall out of *when* the snapshot is taken:

- **Read committed**: a fresh snapshot at the start of every statement, so each statement sees the latest committed data — which is exactly why reads are not repeatable.
- **Repeatable read / snapshot isolation**: one snapshot for the whole transaction, so every read sees the same consistent state — but two transactions reading two snapshots can still write different rows and produce write skew.

What MVCC costs is **version cleanup**. Old versions have to be kept as long as any open snapshot might need them, so a single long-running or idle-in-transaction session holds back cleanup for the whole database: table bloat and vacuum pressure in PostgreSQL, undo log growth in InnoDB. "Keep transactions short" is a storage argument as much as a locking one. See [PostgreSQL internals](../advanced/06-postgresql-internals.md).

### Deadlocks

Two transactions take the same rows in opposite order and wait on each other forever.
The engine detects the cycle and aborts one as the victim, so this surfaces as a runtime error to retry, not as a hang.
Prevent it with:

- A consistent lock ordering (always lock accounts by ascending ID)
- Short transactions
- Bounded retries with jitter

The general treatment of deadlock, livelock, and starvation is in [concurrency control](./26-concurrency-control.md).

## Pessimistic vs optimistic control

The isolation level is what you ask the database for. This is what you write in application code.

### Pessimistic locking

Assume conflicts are likely and take the lock before doing the work.

```sql
BEGIN;
SELECT balance FROM accounts WHERE id = 42 FOR UPDATE;  -- other writers wait here
UPDATE accounts SET balance = balance - 100 WHERE id = 42;
COMMIT;
```

Pros:

- Correct under high contention: the winner does the work once, with no wasted retries
- Locking the rows you *read* is the direct defense against write skew

Cons:

- Wait time grows with contention, and the lock is held for the whole transaction
- Deadlock risk, so lock ordering becomes a rule the whole codebase must follow
- A lock held across a network call (payment gateway, another service) is an outage waiting to happen

### Optimistic concurrency control

Assume conflicts are rare, do the work unlocked, and detect the conflict at write time. Read the row with its version (or `updated_at` / `ETag`), then make that version part of the update predicate:

```sql
UPDATE accounts
SET balance = balance - 100, version = version + 1
WHERE id = 42 AND version = 7;
```

If the affected row count is 0, another writer won the race: re-read, recompute, retry.

Pros:

- No lock is held while a user thinks, or while you wait on another service
- Works across request boundaries — `ETag` plus `If-Match` is this pattern over HTTP

Cons:

- Every conflict is wasted work, so the cost scales with the conflict rate
- Needs a bounded, jittered retry loop, or a contention spike becomes a retry storm

Rule of thumb: optimistic for low-conflict traffic and anything spanning user think time; pessimistic for genuinely hot rows and strict invariants.

## Hotspots and contention

Concurrency control only bites where two transactions want the same row. The usual suspects:

- A single inventory or "seats remaining" counter
- A popular account, wallet, or celebrity user row
- A global sequence or config row that every write touches

Mitigations:

- **Split the row**: shard a counter into `N` sub-counters and sum them on read
- **Serialize deliberately**: route writes for a hot key through a queue with one consumer per key, so contention becomes a queue depth you can measure
- **Shrink the critical section**: short transactions, narrow lock scope, no external calls inside the transaction
- **Move work out**: emails, webhooks, and analytics do not belong inside the transaction boundary

## Concurrency across replicas and shards

Everything above assumes one node ordering its own transactions. Both neighboring notes break that assumption, in different ways.

**Replica reads.** An isolation level is a promise made by the node that serves the read.
A replica seconds behind the primary ([replication](./23-database-replication.md)) serves a perfectly consistent snapshot that is simply **old**, so raising the isolation level on the replica fixes nothing.
The resulting problems are session-scoped rather than transaction-scoped: read-your-writes (a user does not see their own commit) and monotonic reads (successive reads land on different replicas and time appears to run backwards).
The fixes are routing decisions:

- Read from the primary after a write
- Pin the session
- Wait for the replica's apply position

**Cross-shard transactions.** Within one shard the primary can order everything it is asked to serialize.
Across shards ([partitioning](./24-database-partitioning.md)) no single node sees the whole transaction, so atomicity needs [2PC](../patterns/05-two-phase-commit.md) and isolation needs a shared notion of time (Spanner's TrueTime).
Write skew is worse here: the two conflicting reads can land on different shards, so no node ever observes the conflict even when every shard is serializable on its own.
That is the real argument for colocating whatever must commit together under one partition key.

## Practical design guidelines

- Start from the invariant, then choose the mechanism — not the other way round
- Name the specific anomaly you are defending against; "we will use transactions" is not an answer
- Keep transactions short: it cuts lock wait, deadlock risk, and MVCC version bloat at the same time
- Keep retries bounded and jittered
- Track conflict rate, lock wait time, deadlocks, and retry success — contention is invisible until it is measured

## Interview talking points

- Name the exact anomaly the workflow risks, then tie the fix to it: lock, version check, or isolation level
- Read committed is the common default and still allows lost updates and write skew — say so before claiming the database handles it
- MVCC means readers do not block writers; the price is version cleanup, so long transactions are expensive
- Write skew is the anomaly snapshot isolation misses: lock what you read, or go serializable
- A stale replica read is a routing problem, not an isolation problem
- Serializable across shards needs 2PC and synchronized clocks — colocate instead

## Reference materials

- [PostgreSQL transaction isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
- [PostgreSQL concurrency control and MVCC](https://www.postgresql.org/docs/current/mvcc.html)
