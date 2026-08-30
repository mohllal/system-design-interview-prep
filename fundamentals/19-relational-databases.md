# Relational Databases

A relational database stores data in **tables**.

Each table has a declared schema: named columns, types, and constraints.

Core Components:

- **Tables**: Structured data storage representing entities (Customers, Orders)
- **Rows**: Individual records containing specific data instances
- **Columns**: Attributes defining data types and constraints
- **Primary Keys**: Unique identifiers for each row
- **Foreign Keys**: References establishing relationships between tables
- **Indexes**: Data structures improving query performance

The engine can join tables, enforce constraints, and run **transactions** so several writes succeed or fail together.

## When This Is the Right Tool

Use a relational database when:

- You have **entities and relationships** (customer, order, line item) that you query in more than one shape
- You need **constraints** the app might forget (unique email, FK, CHECK)
- Several rows must change as **one transaction** (payment + ledger + outbox row)
- Ad-hoc SQL and a planner matter more than a single primary-key lookup

It is a poor default when:

- The working set or write QPS no longer fits one primary
- The document *is* the API and you almost never join (a document store may match the access path)
- You only need cache semantics (Redis), graph walks, or append-only metrics

## Tables, Keys, and Joins

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

- **Primary key**: stable identity of a row
- **Foreign key**: a column that must point at an existing PK (or NULL)
- **Join**: the engine matches keys.

Normalize so each fact lives in one place and join at read time when the question needs several facts.

If every screen is "user plus their last 20 posts," you will denormalize or cache. That is an access-path choice, not a reason to abandon the relational model for the system of record.

## SQL (Structured Query Language)

SQL provides a standardized interface for relational database operations, supporting complex queries, transactions, and data integrity constraints.

SQL operations categories:

- Data Query Language (DQL): `SELECT`
- Data Manipulation Language (DML): `INSERT`, `UPDATE`, `DELETE`
- Data Definition Language (DDL): `CREATE`, `ALTER`, `DROP`
- Data Control Language (DCL): `GRANT`, `REVOKE`
- Transaction Control Language (TCL): `COMMIT`, `ROLLBACK`

## Transactions and ACID

A **transaction** is a batch of reads and writes that the database treats as one unit.

### Atomicity

All of the statements commit, or none do.

Example: A transfer that debits A and credits B cannot leave only the debit applied after a crash.

```sql
BEGIN TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT; -- Both updates succeed, or both rollback
```

The database engine uses a log plus rollback (or undo) to make that true.

### Consistency

The database only commits states that pass **declared** constraints:

- Entity integrity: PK unique and not null
- Referential integrity: FK targets exist
- Domain integrity: CHECK / NOT NULL / unique

Example: "Order total equals the sum of lines" is consistency only if you actually declared it (or enforced it in the same transaction in the app).

```sql
CHECK (total_amount = (
    SELECT SUM(quantity * unit_price) 
    FROM order_items 
    WHERE order_id = orders.order_id
))
```

### Isolation

Concurrent transactions should not trample each other.

How much they can see of each other is the **isolation level**.

That is a long topic of its own: dirty reads, phantoms, write skew, locks vs MVCC, see [Database Concurrency Control](./23-database-concurrency-control.md).

### Durability

After `COMMIT` returns, a crash must not lose the transaction.

The usual implementation is **write-ahead logging (WAL)**:

1. Write the change to the log and flush it (or group-commit several txns)
2. Tell the client success
3. Later, checkpoint those changes into data files

If you crash between 2 and 3, recovery replays the log.

```plaintext
client  →  COMMIT
              │
              ▼
         WAL flush   ←  durability happens here
              │
              ▼
         data files (checkpoint, later)
```

## Normalization

Normalization reduces data redundancy and improves data integrity by organizing tables according to normal forms.

### First Normal Form (1NF)

Requirements:

- Atomic Values: Each column contains indivisible values
- Unique Rows: No duplicate rows allowed
- Primary Key: Each table must have a primary key
- No Repeating Groups: No repeating columns (e.g., phone1, phone2, phone3)

Example: Violates atomicity and has repeating groups:

```plaintext
Customers:
| CustomerID | Name     | Phones              | Orders           |
|------------|----------|---------------------|------------------|
| 1          | John Doe | 555-1234, 555-5678  | Laptop, Mouse    |
```

After 1NF:

```plaintext
Customers:
| CustomerID | Name     |
|------------|----------|
| 1          | John Doe |

CustomerPhones:
| CustomerID | Phone    |
|------------|----------|
| 1          | 555-1234 |
| 1          | 555-5678 |

Orders:
| OrderID | CustomerID | Product |
|---------|------------|---------|
| 1       | 1          | Laptop  |
| 2       | 1          | Mouse   |
```

### Second Normal Form (2NF)

Requirement: Must be in 1NF + all non-key attributes fully depend on the entire primary key (eliminates partial dependencies).

Example: Composite key (`PlayerID`, `ItemType`):

```plaintext
PlayerItems:
| PlayerID | ItemType | ItemQuantity | PlayerRating |
|----------|----------|--------------|--------------|
| jodge1   | amulets  | 2            | Intermediate |
| jodge1   | rings    | 4            | Intermediate |
| gilal9   | coins    | 20           | Advanced     |
```

Issue: `PlayerRating` depends only on `PlayerID`, not the full key (`PlayerID`, `ItemType`).

After 2NF:

```plaintext
PlayerItems:
| PlayerID | ItemType | ItemQuantity |
|----------|----------|--------------|
| jodge1   | amulets  | 2            |
| jodge1   | rings    | 4            |
| gilal9   | coins    | 20           |

Players:
| PlayerID | PlayerRating |
|----------|--------------|
| jodge1   | Intermediate |
| gilal9   | Advanced     |
```

### Third Normal Form (3NF)

Requirement: Must be in 2NF + no transitive dependencies (non-key attributes depend only on primary key, not on other non-key attributes).

Example:

```plaintext
Players:
| PlayerID | PlayerRating | PlayerSkillLevel |
|----------|--------------|------------------|
| jodge1   | Intermediate | 6                |
| gilal9   | Advanced     | 9                |
```

Issue: `PlayerRating` depends on `PlayerSkillLevel`, creating transitive dependency: PlayerID → PlayerSkillLevel → PlayerRating

After 3NF:

```plaintext
Players:
| PlayerID | PlayerSkillLevel |
|----------|------------------|
| jodge1   | 6                |
| gilal9   | 9                |

RatingLevels:
| SkillLevel | Rating       |
|------------|--------------|
| 0-4        | Beginner     |
| 5-7        | Intermediate |
| 8-10       | Advanced     |
```

### Why You Still Denormalize

Joins cost CPU and IO. Read-heavy paths often keep a copy:

- `orders.customer_email` duplicated so the list page does not join
- A summary table or materialized view for dashboards
- A cache in Redis for the hot document

You then own **refresh**:

- Trigger (e.g. a database trigger)
- App dual-write
- Nightly job

In cost of:

- Reads get faster
- Writes get more places to update
- Staleness becomes a product decision

## How a Query Actually Runs

SQL is declarative so the **planner** picks an access path.

Typical choices:

- Sequential scan of the heap
- Index lookup or range scan, see [Indexes](./33-database-indexes.md)
- Nested loop / hash / merge join
- Sort and aggregate

`EXPLAIN` (and `EXPLAIN ANALYZE`) is how you see that plan.
"Add an index" without looking at the plan is guessing.

On one well-indexed primary, most SQL optimization is:

1. Index the columns you filter and join on
2. Stop selecting columns you do not need
3. Avoid exploding joins (`SELECT *` from 1:N twice)
4. Keep transactions short

## Constraints vs the Application

Put invariants in the database if a missed check is expensive:

- Unique email
- FK so you cannot orphan line items
- `CHECK (quantity > 0)`

The app will have bugs. A constraint is a last line.

Do not expect the database to enforce rules that need other services (inventory in a warehouse API) or external systems. Those stay in the app or in a saga.

## Order of Moves

1. Correct schema and constraints for the invariants
2. Indexes for the real `WHERE` / `JOIN` / `ORDER BY` list
3. Isolation level appropriate to the anomalies you cannot tolerate
4. Replicas for failover and read offload
5. Partition or shard when one primary cannot hold the writes or the data

## Interview Talking Points

- Relational means **tables, keys, joins, transactions**, not "we use SQL."
- ACID: atomic commit, declared constraints, isolation, WAL durability.
- Normalize to avoid update anomalies.
  Denormalize a **named** read path and say how you refresh it.
- Scale reads with replicas and scale writes with partitioning.
  Do not start with "maybe Mongo."
