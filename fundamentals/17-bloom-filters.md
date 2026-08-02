# Bloom Filters

A Bloom filter is a space-efficient probabilistic data structure for membership tests: "might be in set" or "definitely not in set."

It is widely used in distributed systems to avoid expensive lookups (e.g. database or cache) when the answer is likely "not present."

## Core Properties

- **No false negatives**: If filter says "not present," the item is definitely absent
- **Possible false positives**: If filter says "present," item may still be absent
- **Fixed memory footprint**: Memory is allocated up front
- **No deletes in standard form**: Basic Bloom filters support insert + query only

## How It Works

```mermaid
graph TD
    A[Key: user_123] --> H1[Hash 1]
    A --> H2[Hash 2]
    A --> H3[Hash 3]
    H1 --> B1[Bit array index 5 = 1]
    H2 --> B2[Bit array index 17 = 1]
    H3 --> B3[Bit array index 42 = 1]
```

Insert:

1. Hash item with `k` hash functions
2. Set corresponding bit positions to `1`

Query:

1. Hash item with same `k` functions
2. If any bit is `0` -> definitely not present
3. If all bits are `1` -> maybe present

## Why Bloom Filters Are Useful

- Fast negative checks before hitting DB/cache/disk
- Significant memory savings versus storing full keys
- Great fit for high-throughput systems with many misses

## Common System Design Use Cases

### Cache Penetration Protection

Cache penetration happens when requests repeatedly ask for keys that do not exist.

Without protection, every miss can bypass cache and hit the database.

- Put a Bloom filter in front of the DB lookup path
- On request for key `K`, check Bloom filter first
- If filter says "definitely not present," return miss immediately
- If filter says "maybe present," continue to cache/DB for confirmation

Result: DB is protected from high-miss traffic (including malicious random-key scans), while valid keys still follow normal cache + DB flow.

### Storage Engine Read Path

In LSM storage engines, data is spread across many immutable files (often called SSTables).

A lookup might otherwise check many files before finding the key or confirming it is absent.

- Each file keeps a Bloom filter for keys that might exist in that file
- On read for key `K`, engine checks Bloom filters first, then opens only candidate files
- If filter says "definitely not in this file," that file is skipped without disk read

Result: far fewer disk I/O operations and faster tail latency, especially for missing keys.

### Distributed Data Sync

When two nodes reconcile data, sending all keys is expensive.  

Bloom filters are used as a quick first pass to find obvious differences.

1. A has 10 million keys, B has 10.1 million keys
2. A sends Bloom summary (small) not all keys (huge)
3. B marks "definitely not in A" keys as clear diffs/missing candidates
4. Final sync transfers only confirmed differences
5. Remaining uncertain keys are handled by a second pass (range sync, Merkle tree, or periodic anti-entropy)

Result: big bandwidth savings, while a lightweight second pass covers keys hidden by false positives.

## Sizing and False Positive Rate

Given:

- `n` = expected number of inserted items
- `m` = bit array size
- `k` = number of hash functions
- `p` = false positive probability

Useful formulas:

- `m = -(n * ln p) / (ln 2)^2`
- `k = (m / n) * ln 2`
- Approximate optimal bits per item: `m / n ~= 1.44 * log2(1/p)`

Rule of thumb:

- About **10 bits/item** gives roughly **~1%** false positive rate
- As filter fills up beyond planned `n`, false positives rise quickly

## Trade-offs

**Pros:**

- ✅ Very memory efficient
- ✅ O(k) insert/query with small constants
- ✅ Excellent for reducing expensive negative lookups

**Cons:**

- ❌ False positives require downstream confirmation
- ❌ Standard Bloom filter cannot delete safely
- ❌ Requires capacity planning (`n`, `p`) up front

## Variants

- **Counting Bloom Filter**: Supports deletes using counters instead of bits
- **Scalable Bloom Filter**: Adds new filters as dataset grows
- **Partitioned Bloom Filter**: Splits bit array for better cache locality

## Interview Talking Points

- Bloom filters are for **pre-check optimization**, not authoritative existence checks.
- They are most valuable when negative lookups are frequent and expensive.
- Correctness comes from the downstream store; Bloom filter only reduces work.
- Always mention false positives and capacity planning trade-offs.

## Reference Materials

- [Bloom Filter (Wikipedia)](https://en.wikipedia.org/wiki/Bloom_filter)
- [Apache Cassandra Bloom Filters](https://cassandra.apache.org/doc/4.0/cassandra/operating/bloom_filters.html)
