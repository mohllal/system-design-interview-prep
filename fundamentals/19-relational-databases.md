---
title: "Relational databases"
concepts:
  - tables-rows-columns
  - primary-and-foreign-keys
  - joins
  - acid-transactions
  - transaction-scope
  - write-ahead-logging
  - normalization
  - denormalization
related:
  - fundamentals/20-non-relational-databases.md
  - fundamentals/21-sql-vs-nosql.md
  - fundamentals/22-database-indexes.md
  - fundamentals/23-database-replication.md
  - fundamentals/24-database-partitioning.md
  - fundamentals/25-database-concurrency-control.md
  - advanced/06-postgresql-internals.md
---

# Relational databases

A relational database stores data in **tables** with a declared schema: named columns, types, and constraints.

The engine can join tables, enforce those constraints, and run **transactions** so several writes succeed or fail together.

Core components:

- **Tables**: Structured storage representing entities (customers, orders)
- **Rows**: Individual records containing specific data instances
- **Columns**: Attributes with a declared type and constraints
- **Primary keys**: Unique identifiers for each row
- **Foreign keys**: References establishing relationships between tables
- **Indexes**: Side structures that make lookups cheap, see [Indexes](./22-database-indexes.md)

This note covers the model, ACID, and normalization. For the other models see [Non-Relational Databases](./20-non-relational-databases.md), and for choosing between them see [SQL vs NoSQL](./21-sql-vs-nosql.md).

## When this is the right tool

Use a relational database when:

- You have **entities and relationships** (customer, order, line item) that you query in more than one shape
- You need **constraints** the app might forget (unique email, FK, CHECK)
- Several rows must change as **one transaction** (payment + ledger + outbox row)
- Ad-hoc SQL and a planner matter more than a single primary-key lookup

It is a poor default when:

- The working set or write QPS no longer fits one primary
- The document *is* the API and you almost never join, so a [document store](./20-non-relational-databases.md) matches the access path better
- You only need cache semantics (Redis), graph walks, or append-only metrics

## Tables, keys, and joins

```plaintext
customers (id PK)
    │
    │ 1:N
    ▼
orders (id PK, customer_id FK)
    │
    │ 1:N
    ▼
order_items (order_id FK, product_id FK)
```

- **Primary key**: Stable identity of a row
- **Foreign key**: A column that must point at an existing PK (or NULL)
- **Join**: The engine matches keys at read time

Normalize so each fact lives in one place, then join when the question needs several facts.

If every screen is "user plus their last 20 posts," you will denormalize or cache. That is an access-path choice, not a reason to abandon the relational model for the system of record.

## SQL (Structured Query Language)

SQL is the declarative interface to the model: you state the result you want, and the planner decides how to get it.

Its operations fall into five categories:

| Category                           | Commands                     |
| ---------------------------------- | ---------------------------- |
| Data Query Language (DQL)          | `SELECT`                     |
| Data Manipulation Language (DML)   | `INSERT`, `UPDATE`, `DELETE` |
| Data Definition Language (DDL)     | `CREATE`, `ALTER`, `DROP`    |
| Data Control Language (DCL)        | `GRANT`, `REVOKE`            |
| Transaction Control Language (TCL) | `COMMIT`, `ROLLBACK`         |

## Transactions and ACID

A **transaction** is a batch of reads and writes that the database treats as one unit.

ACID names the four guarantees the engine gives that unit. The part that matters in an interview is the **scope**: in a relational database the default unit spans many rows across many tables, which is exactly the property most non-relational stores narrow.

### Atomicity

All of the statements commit, or none do.

Example: A transfer that debits A and credits B cannot leave only the debit applied after a crash.

```sql
BEGIN TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT; -- Both updates succeed, or both rollback
```

The engine uses a log plus rollback (undo) to make that true.

### Consistency

The database only commits states that pass **declared** constraints:

- **Entity integrity**: Primary key is unique and not null
- **Referential integrity**: Foreign key targets exist
- **Domain integrity**: Enforced via CHECK / NOT NULL / unique constraints

The word "declared" is load-bearing. A `CHECK` covers rules that fit in one row:

```sql
CHECK (quantity > 0 AND total_amount >= 0)
```

"Order total equals the sum of its lines" spans two tables, and the major engines reject subqueries inside a `CHECK`. That invariant is a guarantee only if you enforce it with a trigger, or in application code running inside the **same** transaction. The engine holds whatever you declared; it does not infer the rule for you.

### Isolation

Concurrent transactions should not trample each other.

How much they can see of each other is the **isolation level**, and that is a long topic of its own: dirty reads, phantoms, write skew, locks vs MVCC. See [Database Concurrency Control](./25-database-concurrency-control.md).

### Durability

After `COMMIT` returns, a crash must not lose the transaction.

The usual implementation is **write-ahead logging (WAL)**:

1. Write the change to the log and flush it (or group-commit several transactions)
2. Tell the client success
3. Later, checkpoint those changes into data files

If you crash between steps 2 and 3, recovery replays the log.

```plaintext
client  →  COMMIT
              │
              ▼
         WAL flush   ←  durability happens here
              │
              ▼
         data files (checkpoint, later)
```

### Where BASE fits

**BASE** (Basically Available, Soft state, Eventual consistency) is the label usually attached to non-relational stores, and it is best read as the same four concerns with two dials turned: the atomic unit shrinks to one key, document, or partition, and consistency becomes something replicas reach *eventually* rather than at `COMMIT`.

It is not a hard property of the category — plenty of non-relational products keep a single primary and a narrow but strict transaction. See [ACID and BASE](./20-non-relational-databases.md#acid-and-base) for the row-by-row contrast and for what the popular products actually default to.

## Normalization

Normalization reduces redundancy and prevents update anomalies by organizing tables according to normal forms.

### First normal form (1NF)

Requirements:

- **Atomic values**: Each column contains indivisible values
- **Unique rows**: No duplicate rows allowed
- **Primary key**: Each table must have a primary key
- **No repeating groups**: No repeating columns (e.g., phone1, phone2, phone3)

Violation — `Phones` and `Orders` each pack a list into one cell:

| CustomerID | Name     | Phones             | Orders        |
| ---------- | -------- | ------------------ | ------------- |
| 1          | John Doe | 555-1234, 555-5678 | Laptop, Mouse |

After 1NF, each repeating group becomes its own table:

**Customers**

| CustomerID | Name     |
| ---------- | -------- |
| 1          | John Doe |

**CustomerPhones**

| CustomerID | Phone    |
| ---------- | -------- |
| 1          | 555-1234 |
| 1          | 555-5678 |

**Orders**

| OrderID | CustomerID | Product |
| ------- | ---------- | ------- |
| 1       | 1          | Laptop  |
| 2       | 1          | Mouse   |

### Second normal form (2NF)

Requirement: 1NF, plus every non-key attribute depends on the **entire** primary key (no partial dependencies).

Violation — the key is the pair (`PlayerID`, `ItemType`), but `PlayerRating` depends on `PlayerID` alone, so it is duplicated on every row for that player:

| PlayerID | ItemType | ItemQuantity | PlayerRating |
| -------- | -------- | ------------ | ------------ |
| jodge1   | amulets  | 2            | Intermediate |
| jodge1   | rings    | 4            | Intermediate |
| gilal9   | coins    | 20           | Advanced     |

After 2NF, the partially-dependent attribute moves to a table keyed by what it actually depends on:

**PlayerItems**

| PlayerID | ItemType | ItemQuantity |
| -------- | -------- | ------------ |
| jodge1   | amulets  | 2            |
| jodge1   | rings    | 4            |
| gilal9   | coins    | 20           |

**Players**

| PlayerID | PlayerRating |
| -------- | ------------ |
| jodge1   | Intermediate |
| gilal9   | Advanced     |

### Third normal form (3NF)

Requirement: 2NF, plus no transitive dependencies — non-key attributes depend only on the key, not on each other.

Violation — `PlayerID` determines `PlayerSkillLevel`, which in turn determines `PlayerRating`:

| PlayerID | PlayerRating | PlayerSkillLevel |
| -------- | ------------ | ---------------- |
| jodge1   | Intermediate | 6                |
| gilal9   | Advanced     | 9                |

After 3NF, the derived attribute moves to the table that owns the rule:

**Players**

| PlayerID | PlayerSkillLevel |
| -------- | ---------------- |
| jodge1   | 6                |
| gilal9   | 9                |

**RatingLevels**

| SkillLevel | Rating       |
| ---------- | ------------ |
| 0-4        | Beginner     |
| 5-7        | Intermediate |
| 8-10       | Advanced     |

Higher forms exist (BCNF, 4NF), but 3NF is where practical OLTP schemas stop. Past that point you are usually trading readability for edge cases that a constraint handles better.

### Why you still denormalize

Joins cost CPU and I/O. Read-heavy paths often keep a copy:

- `orders.customer_email` duplicated so the list page does not join
- A summary table or materialized view for dashboards
- A cache in Redis for the hot document

You then own **refresh**, via a database trigger, an application dual-write, or a nightly job.

The trade-off:

- Reads get faster
- Writes get more places to update
- Staleness becomes a product decision

Non-relational stores make the same trade, but they usually **start** denormalized because there is no cheap join planner to fall back on.

## How a query actually runs

SQL is declarative, so the **planner** picks an access path.

Typical choices:

- Sequential scan of the heap
- Index lookup or range scan, see [Indexes](./22-database-indexes.md)
- Nested loop / hash / merge join
- Sort and aggregate

`EXPLAIN` (and `EXPLAIN ANALYZE`) is how you see that plan. "Add an index" without looking at the plan is guessing.

On one well-indexed primary, most SQL optimization is:

1. Index the columns you filter and join on
2. Stop selecting columns you do not need
3. Avoid exploding joins (`SELECT *` from 1:N twice)
4. Keep transactions short

## Constraints vs the application

Put invariants in the database if a missed check is expensive:

- Unique email
- FK so you cannot orphan line items
- `CHECK (quantity > 0)`

The app will have bugs. A constraint is the last line of defense.

Do not expect the database to enforce rules that need other services (inventory in a warehouse API) or external systems. Those stay in the app or in a saga.

## Order of moves

1. Correct schema and constraints for the invariants
2. Indexes for the real `WHERE` / `JOIN` / `ORDER BY` list
3. Isolation level appropriate to the anomalies you cannot tolerate
4. Replicas for failover and read offload
5. Partition or shard when one primary cannot hold the writes or the data

## Interview talking points

- Relational means **tables, keys, joins, transactions**, not "we use SQL."
- ACID: atomic commit, declared constraints, isolation, WAL durability — over a unit that spans **many rows in many tables**. That scope is what BASE narrows.
- Normalize to avoid update anomalies. Denormalize a **named** read path and say how you refresh it.
- Scale reads with replicas and scale writes with partitioning. Do not start with "maybe Mongo."
