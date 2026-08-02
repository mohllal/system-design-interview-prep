# Database Concurrency Control

Database concurrency control keeps data correct when many transactions read and write the same records concurrently.

Without it, systems can silently violate invariants (wrong balances, duplicate bookings, stale overwrites).

## Why It Matters

- Protects data correctness under parallel access
- Prevents race conditions in high-traffic systems
- Balances consistency vs latency/throughput

## Common Concurrency Problems

### Lost Update

Two transactions read the same value and both write back updates based on the old value.
The later write overwrites the earlier one.

Example:

1. Balance = 100
2. Transaction A reads 100, Transaction B reads 100
3. A writes 90, B writes 80
4. Final value = 80 (A's change lost)

### Dirty Read

A transaction reads data written by another transaction that has not committed yet.
If the writer rolls back, the reader used invalid data.

### Non-Repeatable Read

Inside one transaction, the same row is read twice and returns different values because another committed transaction updated it in between.

Example:

1. Transaction A reads record X = 100, Transaction B writes record X = 90
2. Transaction B commits
3. Transaction A reads record X = 90

### Phantom Read

A transaction repeats a range query and gets extra/missing rows because another transaction inserted/deleted rows matching the range.

Example:

1. Transaction A reads all records with age > 18, Transaction B inserts a new record with age = 19
2. Transaction B commits
3. Transaction A reads all records with age > 18 again and gets the new record

### Write Skew

Two transactions read overlapping state, then each writes a different row.
Each write is locally valid, but together they break a higher-level invariant.

Classic example: "at least one doctor must be on-call." Both transactions see two doctors on-call and each turns one off, but together they violate the invariant.

1. Transaction A reads two doctors on-call, Transaction B reads two doctors on-call
2. Transaction A turns one doctor off, Transaction B turns the other doctor off
3. Both transactions commit
4. The invariant is violated: there are now no doctors on-call

### Dirty Write (severe, usually prevented)

One uncommitted write overwrites another uncommitted write.
Most modern databases prevent this by default.

## Main Concurrency Approaches

### Pessimistic Concurrency

Assumes conflicts are likely and uses locks to block conflicting operations.

- `SELECT ... FOR UPDATE` for row-level write intent
- Strong correctness under high contention
- Higher waiting time and deadlock risk

### Optimistic Concurrency

Assumes conflicts are less frequent and detects them at update/commit time.

- Read row with version (or `updated_at`/`ETag`)
- Update with version predicate
- Retry on conflict

```sql
UPDATE accounts
SET balance = balance - 100, version = version + 1
WHERE id = 42 AND version = 7;
```

If affected rows = 0, another writer won the race, client re-reads and retries.

## Isolation Levels

- Read Uncommitted: Highest concurrency, weakest correctness (rarely used in critical systems)
- Read Committed: Prevents dirty reads (common default); still allows non-repeatable reads/phantoms
- Repeatable Read: Prevents non-repeatable reads; phantom behavior depends on DB engine/MVCC rules
- Serializable: Prevents all standard anomalies by enforcing serial equivalence (highest overhead)

Choose isolation based on correctness needs and contention profile.

## Hotspots and Contention

Common high-contention rows:

- Single inventory counter
- Popular account/wallet rows
- Global sequence/config rows

Mitigations:

- Shard counters
- Queue writes for hot keys
- Use short transactions and limited lock scope
- Move non-critical work out of transaction boundary

## Practical Design Guidelines

- Start from invariants, then choose mechanism
- Use optimistic control for low-conflict traffic
- Use pessimistic locking for hot rows and strict invariants
- Keep retries bounded with jitter
- Track conflict rate, lock wait time, deadlocks, retry success

## Interview Talking Points

- Name the exact anomaly risk for the given workflow
- Tie solution to invariant (locks, CAS/version checks, serializable transactions)

## Reference Materials

- [PostgreSQL Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
