# Load Balancing

Load balancing distributes incoming requests across multiple servers to prevent any single server from becoming overwhelmed.

**Core Benefits:**

- Prevents server overload and bottlenecks
- Improves application availability and fault tolerance
- Enables horizontal scaling for growing traffic
- Optimizes resource utilization across infrastructure

## Scaling Strategies

### Vertical Scaling (Scale Up)

**Approach**: Add more compute resources to existing machines

- Upgrade CPU, memory, storage on single server
- ✅ Simple to implement, no architecture changes
- ✅ No load balancing complexity
- ❌ Hardware limits and diminishing returns
- ❌ Expensive at scale

### Horizontal Scaling (Scale Out)

**Approach**: Add more machines to resource pool

- Distribute load across multiple servers
- ✅ Nearly unlimited scaling potential
- ✅ Better fault tolerance and redundancy
- ✅ Cost-effective with commodity hardware
- ❌ Increased complexity and coordination
- ❌ Requires load balancing and state management

## Load Balancing Algorithms

Different algorithms optimize for various factors like simplicity, performance, and session management.

Load balancing algorithms can be categorized into two types:

- **Static algorithms**: The mapping of requests to servers is determined in advance using fixed rules.
- **Dynamic algorithms**: Decisions are made in real-time, based on current server load or performance metrics.

### Static Algorithms

**Round Robin**

```mermaid
sequenceDiagram
    participant LB as Load Balancer
    participant S1 as Server 1
    participant S2 as Server 2
    participant S3 as Server 3
    
    LB->>S1: Request 1
    LB->>S2: Request 2
    LB->>S3: Request 3
    LB->>S1: Request 4 (cycle repeats)
```

- Distributes requests sequentially across servers
- ✅ Simple implementation, predictable distribution
- ✅ Works well with homogeneous servers
- ❌ Doesn't consider server capacity or current load
- **Use case**: Equal-capacity servers, stateless applications

**Weighted Round Robin**

- Assigns weights based on server capacity
- More powerful servers receive proportionally more requests
- ✅ Accounts for different server capacities
- ❌ Static weights don't adapt to real-time conditions
- **Use case**: Heterogeneous server configurations

**IP Hash (Consistent Assignment)**

```mermaid
sequenceDiagram
    participant ClientA as Client A (IP: 192.168.1.10)
    participant ClientB as Client B (IP: 192.168.1.20)
    participant LB as Load Balancer
    participant S1 as Server 1
    participant S2 as Server 2

    ClientA->>LB: Request (IP 192.168.1.10)
    LB->>S1: Hash(IP) → Server 1

    ClientB->>LB: Request (IP 192.168.1.20)
    LB->>S2: Hash(IP) → Server 2

    ClientA->>LB: Next Request (same IP)
    LB->>S1: Hash(IP) → Server 1 (consistent mapping)
```

- Uses client IP to determine target server
- Same client always routes to same server
- ✅ Maintains session affinity without sticky sessions
- ❌ Uneven distribution if clients cluster by IP
- **Use case**: Session-dependent applications, caching benefits

### Dynamic Algorithms

**Least Connections**

- Routes to server with fewest active connections
- ✅ Better load distribution than round robin
- ✅ Adapts to varying request processing times
- ❌ Requires connection tracking overhead
- **Use case**: Long-lived connections, varying request complexity

**Least Response Time**

- Considers both connection count and response time
- Routes to server with fastest response + fewest connections
- ✅ Optimizes for performance
- ❌ Higher computational overhead
- **Use case**: Performance-critical applications

**Resource-Based (Adaptive)**

- Uses real-time server metrics (CPU, memory, custom metrics)
- ✅ Most intelligent load distribution
- ✅ Adapts to server health and capacity
- ❌ Complex implementation and monitoring required
- **Use case**: Critical applications, heterogeneous environments

## Load Balancer Types

Load balancers can be implemented at different layers and using various technologies.

### By Decision Layer

**Layer 4 Load Balancers (Transport Layer)**

- Route traffic using network information like IP, port, and protocol (TCP/UDP)
- Fast and efficient because they do not inspect full application payloads
- ✅ Best fit for high-throughput services, raw TCP services, and low-latency routing
- ❌ Limited visibility into HTTP paths, headers, or cookies

**Layer 7 Load Balancers (Application Layer)**

- Route HTTP/HTTPS requests using URL path, host header, cookies, or request metadata
- ✅ Enable advanced behaviors like path-based routing, API version routing, and canary releases
- ✅ Commonly terminate TLS and integrate with WAF, auth checks, and observability tools
- Higher flexibility with more processing overhead than Layer 4

### By Deployment Model

**Hardware Load Balancers**

- Dedicated physical appliances (F5, Citrix NetScaler)
- ✅ High performance and throughput
- ✅ Advanced features and optimization
- ❌ Expensive and vendor lock-in
- ❌ Limited scalability and flexibility

**Software Load Balancers**

- Applications running on standard servers (Nginx, HAProxy)
- ✅ Cost-effective and flexible
- ✅ Easy to scale and customize
- ❌ Limited by server hardware capacity
- ❌ Requires more management overhead

**Cloud Load Balancers**

- Managed services (AWS ALB/ELB, Google Cloud Load Balancer)
- ✅ Fully managed, auto-scaling
- ✅ Integrated with cloud ecosystem
- ❌ Vendor lock-in and pricing concerns
- ❌ Less control over configuration

### By Traffic Visibility

**External (Public) Load Balancers**

- Internet-facing entry point for web or mobile clients
- Commonly handle TLS termination, DDoS protection, and global DNS integration
- Expose only required public endpoints while hiding backend node details

**Internal (Private) Load Balancers**

- Route traffic inside private networks (service-to-service or east-west traffic)
- Not directly reachable from the public internet
- Useful for microservices, internal APIs, and tier-to-tier communication

### By Geographic Scope

**Regional Load Balancers**

- Route traffic within one region to nearby availability zones or instances
- Simpler to operate with lower cross-region complexity and cost
- If the region fails, failover depends on separate disaster recovery setup

**Global Load Balancers**

- Distribute traffic across multiple regions using geo/latency/policy-based routing
- Improve resilience with automatic multi-region failover
- Better global user experience, with higher operational complexity

## Load Balancer Redundancy Topologies

Redundancy topology defines how multiple load balancers are arranged to avoid a single point of failure.

```mermaid
graph TD
    subgraph "Active-Active"
        A1[Client] --> LB1[Load Balancer 1]
        A1 --> LB2[Load Balancer 2]
        LB1 --> S1[Server 1]
        LB1 --> S2[Server 2]
        LB2 --> S1
        LB2 --> S2
    end
    
    subgraph "Active-Passive"
        A2[Client] --> LB3[Active LB]
        LB4[Passive LB] -.-> LB3
        LB3 --> S3[Server 1]
        LB3 --> S4[Server 2]
    end
```

### Active-Active

**Model**: Two or more load balancers serve traffic at the same time.

- Traffic is distributed across all active nodes (for example, via DNS or anycast/BGP)
- Delivers strong availability and better aggregate throughput
- Uses infrastructure more efficiently because no node sits idle
- Requires careful health checks, config consistency, and traffic steering
- Can increase operational complexity during partial failures

### Active-Passive

**Model**: One load balancer is active while a secondary node stays on standby.

- Normal traffic flows through the primary node only
- On failure, a failover mechanism promotes the standby node
- Easier to reason about and operate than active-active
- Standby capacity is idle during normal operation
- Recovery depends on failover detection and promotion time

## Reference Materials

- [DNS Support for Load Balancing (RFC 1794)](https://datatracker.ietf.org/doc/html/rfc1794)
