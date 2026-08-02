# Back-of-the-Envelope Calculations

Back-of-the-envelope calculations are quick approximations used in system design to estimate scale, cost, and bottlenecks.

Goal: get within the right order of magnitude fast, not exact precision.

## Why It Matters

- Validates whether a design can handle expected load
- Exposes bottlenecks early (QPS, storage, bandwidth, servers)
- Helps compare architecture options quickly in interviews

## Useful Approximations

- 1 day ≈ 100,000 seconds (86400 exactly, round for speed)
- 1 month ≈ 2.5 million seconds
- 1 year ≈ 31.5 million seconds
- 1 KB ≈ 10^3 bytes, 1 MB ≈ 10^6 bytes, 1 GB ≈ 10^9 bytes

## Estimation Framework

1. State assumptions clearly
2. Compute average load
3. Apply peak factor (often 2-3x)
4. Estimate storage/bandwidth/server needs
5. Add headroom (20-50%)
6. Sanity-check result

## 1) Traffic (QPS/RPS)

Formula:

- `Average RPS = (DAU x requests_per_user_per_day) / 86400`
- `Peak RPS = Average RPS x peak_factor`

Example:

- 100M DAU, 20 requests/user/day
- Average RPS ≈ `(100M x 20) / 100k` ≈ 20,000 RPS
- Peak (3x) ≈ 60,000 RPS

Also split read/write when relevant (for example, 80/20).

## 2) Storage

Formula:

- `Daily storage = events_per_day x average_object_size`
- `Total storage = daily_storage x retention_days x replication_factor`

Example:

- 10M uploads/day x 100 MB ≈ 1 TB/day
- 3-year retention ≈ ~1 PB (before replication)

Include metadata overhead and indexes, not just raw payload.

## 3) Bandwidth

Formula:

- `Outbound = read_qps x avg_response_size`
- `Inbound = write_qps x avg_request_size`

Example:

- 8,000 read QPS x 50 KB ≈ 400 MB/s outbound
- 2,000 write QPS x 2 MB ≈ 4 GB/s inbound

Use peak QPS for capacity planning, not average only.

## 4) Server Count

Formula:

- `Servers = peak_load / effective_capacity_per_server`
- Apply redundancy and headroom separately

Example:

- Peak 60,000 RPS
- One server safe capacity 700 RPS (70% of 1000)
- Required ≈ `60,000 / 700` ≈ 86 servers
- With 2x redundancy ≈ 172 servers

## 5) Latency Budget

Break request path into components and sum:

- Client-network round trip
- Gateway/proxy
- App compute
- Cache/DB/storage
- Serialization/response build

Example:

- 80 ms network + 10 ms app + 5 ms DB + 5 ms overhead ≈ 100 ms total

Use this to identify dominant term quickly.

## Quick Sanity Checks

- Does storage growth fit retention policy?
- Is peak traffic handled with realistic server counts?
- Does bandwidth exceed NIC/cloud limits?
- Are DB QPS and connection counts feasible?

## Common Pitfalls

- Using average traffic only (ignoring peaks)
- Forgetting replication/backups/indices overhead
- Ignoring hot keys and uneven traffic distribution
- No headroom for failures or deploy spikes

## Interview Talking Points

- Say assumptions out loud before math.
- Round aggressively; precision is not the goal.
- Show one complete estimate (traffic -> servers or storage).
- Mention what you would validate next with real metrics/load tests.
