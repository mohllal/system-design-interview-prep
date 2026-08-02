# CAP and PACELC Theorems

CAP and PACELC describe trade-offs in distributed systems, especially under failure and load.

They help explain why "strong consistency and always-on low latency everywhere" is not realistic.

## CAP Theorem

In a partition, a distributed system must choose between:

- **Consistency (C)**: Reads reflect latest write (or fail)
- **Availability (A)**: Every request gets a non-error response
- **Partition Tolerance (P)**: System continues despite network splits

Important nuance:

- In real distributed systems, partitions happen, so **P is not optional**
- During partition, you effectively choose between **C** and **A**

### CAP Classifications

**CP systems**

- Favor consistency over availability during partition
- May reject/timeout writes/reads to avoid stale responses
- Examples: strongly consistent metadata stores, many RDBMS setups

**AP systems**

- Favor availability during partition
- System keeps responding, possibly with stale data
- Convergence happens later (eventual consistency)
- Examples: Dynamo-style KV stores, Cassandra (default posture)

**CA systems**

- Only practical on single-node/non-partitioned deployments
- Not a realistic distributed partition-tolerant model

## PACELC Extension

PACELC adds normal-operation trade-offs:

- **If Partition (P)**: choose A or C (same CAP idea)
- **Else (E)**: choose Latency (L) or Consistency (C)

So even without partitions, systems often trade consistency for speed.

Examples:

- **AP / EL**: Available under partition, optimized for low latency normally
- **CP / EC**: Consistent under partition, stronger consistency even in normal path

## Practical Interpretation

Do not treat CAP labels as rigid product tags.  
They describe design priorities under failure and load.

Questions to ask:

- Can this read be stale?
- Can this write be rejected during partition?
- Is p99 latency more important than strict freshness?

## Common Misconceptions

- "Pick any 2 of 3 always" -> oversimplified; partition forces C vs A choice
- "AP means no consistency" -> false; usually eventual consistency with convergence
- "Strong consistency is free" -> false; coordination adds latency and fragility

## How to Choose in Interviews

Choose **CP-ish** when:

- Financial balances, inventory invariants, leader metadata
- Incorrect stale reads/writes are unacceptable

Choose **AP-ish** when:

- Feeds, analytics, recommendations, activity streams
- Availability and responsiveness matter more than immediate global consistency

Use explicit mechanisms either way:

- Versioning, quorum reads/writes, conflict resolution, idempotency, repair jobs

## Interview Talking Points

- Explain CAP with a partition scenario, not abstract definitions.
- Use PACELC to discuss latency vs consistency in normal operation.
- Map system components to different guarantees (metadata vs user feed vs analytics).
- Mention convergence/repair strategy for AP designs.

## Reference Materials

- [Brewer's CAP Paper](https://users.ece.cmu.edu/~adrian/731-sp04/readings/GL-cap.pdf)
- [PACELC Theorem (Daniel Abadi)](https://dbmsmusings.blogspot.com/2010/04/problems-with-cap-and-yahoos-little.html)
