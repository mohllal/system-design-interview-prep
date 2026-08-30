---
title: "Database concurrency control"
concepts:
  - transaction-isolation-levels
  - pessimistic-locking
  - optimistic-concurrency-control
  - concurrency-anomalies
  - write-skew
  - lock-contention-and-hotspots
related:
  - fundamentals/26-concurrency-control.md
  - fundamentals/19-relational-databases.md
  - advanced/06-postgresql-internals.md
---

# Database concurrency control

Database concurrency control keeps data correct when many transactions read and write the same records concurrently.

Without it, systems can silently violate invariants (wrong balances, duplicate bookings, stale overwrites).

## Why it matters

- Protects data correctness under parallel access
- Prevents race conditions in high-traffic systems
- Balances consistency against latency and throughput

## Common concurrency problems

### Lost update

Two transactions read the same value and both write back updates based on the old value. The later write overwrites the earlier one.

Example:

1. Balance = 100
2. Transaction A reads 100, Transaction B reads 100
3. A writes 90, B writes 80
4. Final value = 80 (A's change lost)

### Dirty read

A transaction reads data written by another transaction that has not committed yet. If the writer rolls back, the reader used invalid data.

### Non-repeatable read

Inside one transaction, the same row is read twice and returns different values because another committed transaction updated it in between.

Example:

1. Transaction A reads record X = 100, Transaction B writes record X = 90
2. Transaction B commits
3. Transaction A reads record X = 90

### Phantom read

A transaction repeats a range query and gets extra or missing rows because another transaction inserted or deleted rows matching the range.

Example:

1. Transaction A reads all records with age > 18, Transaction B inserts a new record with age = 19
2. Transaction B commits
3. Transaction A reads all records with age > 18 again and gets the new record

### Write skew

Two transactions read overlapping state, then each writes a different row. Each write is locally valid, but together they break a higher-level invariant.

Classic example: "at least one doctor must be on-call." Both transactions see two doctors on-call and each turns one off, but together they violate the invariant.

1. Transaction A reads two doctors on-call, Transaction B reads two doctors on-call
2. Transaction A turns one doctor off, Transaction B turns the other doctor off
3. Both transactions commit
4. The invariant is violated: there are now no doctors on-call

### Dirty write (severe, usually prevented)

One uncommitted write overwrites another uncommitted write. Most modern databases prevent this by default.

## Main concurrency approaches

### Pessimistic concurrency

Assumes conflicts are likely and uses locks to block conflicting operations.

- `SELECT ... FOR UPDATE` for row-level write intent
- Strong correctness under high contention
- Higher waiting time and deadlock risk

### Optimistic concurrency

Assumes conflicts are less frequent and detects them at update/commit time.

- Read the row along with its version (or `updated_at`/`ETag`)
- Update with a version predicate
- Retry on conflict

```sql
UPDATE accounts
SET balance = balance - 100, version = version + 1
WHERE id = 42 AND version = 7;
```

If the affected row count is 0, another writer won the race; the client re-reads and retries.

## Isolation levels

| Isolation level  | Prevents                                                | Trade-off                                                                 |
| ---------------- | ------------------------------------------------------- | ------------------------------------------------------------------------- |
| Read uncommitted | Nothing (dirty reads allowed)                           | Highest concurrency, weakest correctness; rarely used in critical systems |
| Read committed   | Dirty reads                                             | Common default; still allows non-repeatable reads and phantoms            |
| Repeatable read  | Non-repeatable reads                                    | Phantom behavior depends on the DB engine's MVCC rules                    |
| Serializable     | All standard anomalies, by enforcing serial equivalence | Highest overhead                                                          |

Choose isolation based on correctness needs and contention profile.

## Hotspots and contention

Common high-contention rows:

- Single inventory counter
- Popular account/wallet rows
- Global sequence/config rows

Mitigations:

- Shard counters
- Queue writes for hot keys
- Use short transactions and limited lock scope
- Move non-critical work out of the transaction boundary

## Practical design guidelines

- Start from invariants, then choose a mechanism
- Use optimistic control for low-conflict traffic
- Use pessimistic locking for hot rows and strict invariants
- Keep retries bounded with jitter
- Track conflict rate, lock wait time, deadlocks, and retry success

## Interview talking points

- Name the exact anomaly risk for the given workflow
- Tie the solution to the invariant (locks, CAS/version checks, serializable transactions)

## Reference materials

- [PostgreSQL Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
