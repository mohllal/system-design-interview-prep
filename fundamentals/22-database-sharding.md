# Database Sharding

Database sharding horizontally partitions data across multiple database instances to improve scalability, performance, and availability.

**Core Concepts**:

- **Logical Shards**: Data partitions based on business logic
- **Physical Shards**: Database instances hosting logical shards
- **Shard Key**: Column(s) used to determine data placement
- **Shard Router**: Component directing queries to appropriate shards

## Partitioning vs Sharding

```mermaid
graph TD
    A[Data Distribution] --> B[Vertical Partitioning<br/>Split by columns]
    A --> C[Horizontal Partitioning]
    
    C --> D[Local Partitioning<br/>Same server, multiple tables]
    C --> E[Sharding<br/>Multiple servers, distributed]
```

### Partitioning (Single Database)

- **Vertical**: Split tables by columns (normalization)
- **Horizontal**: Split tables by rows within same database
- **Benefits**: Query optimization, maintenance efficiency
- **Limitations**: Single server resource constraints

### Sharding (Distributed Database)

- **Scope**: Multiple independent database instances
- **Benefits**: Linear scalability, fault isolation
- **Complexity**: Cross-shard queries, distributed transactions

## Shared-Nothing Architecture

Sharding implements a shared-nothing approach where each shard operates completely independently.

```mermaid
graph TD
    A[Query Router] --> B[Shard 1<br/>Independent CPU, Memory, Storage]
    A --> C[Shard 2<br/>Independent CPU, Memory, Storage]
    A --> D[Shard 3<br/>Independent CPU, Memory, Storage]
    
    B --> E[Local Processing<br/>No inter-shard communication]
    C --> F[Local Processing<br/>No inter-shard communication]
    D --> G[Local Processing<br/>No inter-shard communication]
```

**Characteristics**:

- Each shard has dedicated resources (CPU, memory, storage)
- No shared resources between shards
- Parallel processing without contention
- Fault isolation - shard failures don't affect others

## Benefits and Trade-offs

### Benefits

**Scalability**

- ✅ **Linear Scale**: Add shards to increase capacity proportionally
- ✅ **Load Distribution**: Spread traffic across multiple servers
- ✅ **Parallel Processing**: Simultaneous query execution

**Performance**

- ✅ **Reduced Contention**: Fewer concurrent users per shard
- ✅ **Optimized Resources**: Tailored hardware per shard needs
- ✅ **Faster Queries**: Smaller datasets per shard

**Availability**

- ✅ **Fault Isolation**: Failure affects only one shard
- ✅ **Independent Maintenance**: Update shards without downtime
- ✅ **Geographic Distribution**: Place shards near users

### Trade-offs

**Complexity**

- ❌ **Application Logic**: Complex routing and query planning
- ❌ **Operations**: Backup, monitoring, and maintenance complexity
- ❌ **Cross-Shard Operations**: Joins and transactions across shards

**Consistency**

- ❌ **Distributed Transactions**: ACID properties harder to maintain
- ❌ **Data Integrity**: Foreign key constraints across shards
- ❌ **Eventual Consistency**: Some operations may require eventual consistency

**Performance**

- ❌ **Cross-Shard Queries**: Network overhead for multi-shard operations
- ❌ **Hotspots**: Uneven data distribution can create bottlenecks
- ❌ **Rebalancing**: Moving data between shards is expensive

## Sharding Patterns

Different strategies for distributing data across shards, each with specific use cases and trade-offs.

### Range-Based

Partition data based on ranges of the shard key values.

```mermaid
graph TD
    A[User ID 1-1000000] --> B{Range Assignment}
    B --> C[Shard 1<br/>IDs 1-250000]
    B --> D[Shard 2<br/>IDs 250001-500000]
    B --> E[Shard 3<br/>IDs 500001-750000]
    B --> F[Shard 4<br/>IDs 750001-1000000]
```

**Characteristics**:

- Data divided into contiguous ranges
- Range boundaries define shard assignment
- Simple routing logic

**Benefits**:

- ✅ Easy to implement and understand
- ✅ Range queries are efficient
- ✅ Simple to add new ranges

**Drawbacks**:

- ❌ Hot spots from uneven access patterns
- ❌ Sequential key generation creates bottlenecks
- ❌ Difficult to rebalance existing ranges

**Use Cases**: Time-series data, ordered datasets, analytical workloads

### Hash-Based

Use hash function on shard key to determine shard placement.

```mermaid
graph TD
    A[User ID: 12345] --> B[Hash Function<br/>hash user_id % 4]
    B --> C{Hash Result}
    C -->|0| D[Shard 1]
    C -->|1| E[Shard 2] 
    C -->|2| F[Shard 3]
    C -->|3| G[Shard 4]
```

**Characteristics**:

- Deterministic hash function placement
- Even distribution across shards
- No range scan capability

**Benefits**:

- ✅ Even data distribution
- ✅ Prevents hot spots
- ✅ Simple and predictable

**Drawbacks**:

- ❌ Range queries require multiple shards
- ❌ Rebalancing requires data migration
- ❌ Adding/removing shards is complex

**Use Cases**: User data, session storage, OLTP workloads

### Directory-Based

Use lookup service to map shard keys to physical shards.

```mermaid
graph TD
    A[Query: user_id=12345] --> B[Directory Service]
    B --> C[Lookup Table<br/>user_id -> shard_id]
    C --> D[Route to Shard 2]
    
    E[Shard Mapping Table] --> F[user 12345 -> Shard 2<br/>user 67890 -> Shard 1<br/>user 54321 -> Shard 3]
```

**Characteristics**:

- Centralized routing decisions
- Flexible mapping logic
- Dynamic shard assignment

**Benefits**:

- ✅ Maximum flexibility in data placement
- ✅ Can optimize for access patterns
- ✅ Easy to rebalance data

**Drawbacks**:

- ❌ Directory service becomes bottleneck
- ❌ Additional complexity and latency
- ❌ Single point of failure

**Use Cases**: Multi-tenant applications, custom data placement requirements

### Geographic

Distribute data based on geographic location or user region.

```mermaid
graph LR
    A[Global Users] --> B{Geographic Location}
    B -->|North America| C[US Data Center<br/>Shard 1]
    B -->|Europe| D[EU Data Center<br/>Shard 2]
    B -->|Asia Pacific| E[APAC Data Center<br/>Shard 3]
```

**Characteristics**:

- Data locality based on geography
- Compliance with data sovereignty laws
- Optimized for regional access patterns

**Benefits**:

- ✅ Reduced latency for users
- ✅ Data sovereignty compliance
- ✅ Natural fault isolation

**Drawbacks**:

- ❌ Uneven regional data distribution
- ❌ Cross-region queries are expensive
- ❌ Complex global operations

**Use Cases**: Global applications, compliance requirements, CDN-like services

## Pattern Comparison

| Pattern             | Distribution  | Complexity | Range Queries | Rebalancing    | Best For                   |
|---------------------|---------------|------------|---------------|----------------|----------------------------|
| **Range-Based**     | May be uneven | Low        | Excellent     | Difficult      | Time-series, analytics     |
| **Hash-Based**      | Even          | Low        | Poor          | Very difficult | OLTP, user data            |
| **Directory-Based** | Flexible      | High       | Good          | Easy           | Multi-tenant, custom needs |
| **Geographic**      | Geographic    | Medium     | Regional only | Medium         | Global apps, compliance    |

## Reference Materials

- [What is Database Sharding?](https://aws.amazon.com/what-is/database-sharding/)
- [Understanding Database Sharding](https://www.digitalocean.com/community/tutorials/understanding-database-sharding)
- [Shared-Nothing Architecture](https://en.wikipedia.org/wiki/Shared-nothing_architecture)
- [Sharding Pinterest](https://medium.com/pinterest-engineering/sharding-pinterest-how-we-scaled-our-mysql-fleet-3f341e96ca6f)
