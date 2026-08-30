---
title: "Leader election"
concepts:
  - leader-follower-pattern
  - quorum-based-election
  - lease-based-election
  - failover
  - split-brain
  - fencing
related:
  - fundamentals/29-consensus.md
  - fundamentals/23-database-replication.md
  - fundamentals/27-cap-and-pacelc-theorems.md
  - fundamentals/08-availability.md
---

# Leader election

The leader-follower pattern assigns one node as coordinator (leader) and the others as replicas or workers (followers).

Leader election is the mechanism that selects a new leader when the current one fails or becomes unreachable.

**Why this pattern is used:**

- **Single write coordinator**: Reduces conflicting updates
- **Fast failover**: A new leader can be promoted automatically
- **Scalability**: Followers can handle reads or parallel work

## Leader-follower pattern

```mermaid
graph TD
    C[Clients] --> L[Leader]
    L --> F1[Follower 1]
    L --> F2[Follower 2]
    L --> F3[Follower 3]
```

### Typical responsibilities

**Leader**

- Accepts writes or control-plane operations
- Coordinates replication and ordering
- Sends heartbeats to show liveness

**Followers**

- Replicate leader updates
- Serve reads in some architectures
- Participate in election when the leader is unavailable

## Election triggers

Leader election usually starts when:

- Heartbeats from the leader stop for a timeout window
- Health checks detect leader failure
- A network partition isolates the current leader
- Planned maintenance requires a leadership transfer

## Common election approaches

### Quorum voting (majority-based)

- A candidate requests votes from peer nodes
- The node with a majority of votes becomes leader
- Prevents multiple leaders under normal conditions
- Widely used in production systems

### Lease-based election

- Leadership is tied to a renewable lease with a TTL
- If the lease expires, other nodes can acquire leadership
- Simple mental model with automatic expiration
- Needs careful timeout and clock-skew handling

```mermaid
sequenceDiagram
    participant N1 as Node 1
    participant N2 as Node 2
    participant S as Lease Store

    N1->>S: Acquire lease (TTL=10s)
    S-->>N1: Granted
    N2->>S: Acquire lease
    S-->>N2: Denied
    N1->>S: Renew lease
```

## Failure modes and trade-offs

**Pros:**

- Clear coordination boundary
- Easier conflict handling for writes
- Predictable failover path

**Cons:**

- The leader can become a bottleneck under heavy write load
- Election periods can cause short unavailability
- Split-brain risk if failure detection is weak

## Design guidelines

- Keep election timeouts randomized to avoid vote collisions
- Use quorum rules; avoid leadership decisions by a single node
- Fence old leaders after failover to prevent double-writes
- Tune heartbeat and timeout values for your network's latency profile
- Test failover regularly with chaos/fault-injection drills

## Real-world examples

- **Kubernetes control plane**: Controllers use leader election to select the active coordinator
- **etcd/Consul-backed services**: Use leases and quorum-based elections
- **Primary-replica databases**: Promote a replica when the primary fails

## Leader election vs consensus

Leader election decides *who leads*. Consensus decides *what value or state the cluster accepts*.

See [Consensus](./29-consensus.md) for the agreement protocol side.

## Reference materials

- [Implementing Lease-Based Leader Election Using Azure Blob Storage](https://learn.microsoft.com/en-us/azure/architecture/patterns/leader-election)
