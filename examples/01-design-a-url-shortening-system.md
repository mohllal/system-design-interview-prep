# Design a URL Shortening System

Design a URL shortening system like [TinyURL](https://tinyurl.com/).

## 1. Clarify Requirements

### Functional Requirements

- **Core Features**
  - Generate a shorter alias for any given URL
  - Redirect users to the original URL when they access the short URL
  - Ensure no collisions in URL hashes
  - Support custom expiration times (default: 3 years, max: 5 years)
  - Delete expired short URLs from the system

- **Use Cases**
  - Users submit long URLs and receive short URLs
  - Users click short URLs and get redirected to original URLs
  - Users can delete their short URLs
  - System automatically expires URLs after specified time

- **Data Operations**
  - Store URL mappings (short URL → original URL)
  - Retrieve original URL by short URL
  - Delete URL mappings for expired URLs

### Non-Functional Requirements

- **Scalability**: Support 100 million daily active users
- **Performance**:
  - URL generation: < 100ms response time
  - URL redirection: < 50ms response time
- **Availability**: 99.9% uptime
- **Consistency**: Strong consistency for URL creation, eventual consistency acceptable for reads

### Constraints & Assumptions

- **Technical Constraints**
  - Short URLs should be human-readable (6-8 characters)
  - System should handle 100:1 read-to-write ratio
  - URLs expire after maximum 5 years

- **Assumptions**
  - 1 million new short URLs created daily
  - Average URL entry size: 500 bytes
  - Peak traffic is 3x average traffic

### Out of Scope

- **Features not included**
  - Analytics (click tracking, unique visitors)
  - User authentication and custom short URLs
  - Bulk URL operations
  - URL preview functionality

## 2. Resource Estimation

### Scale Assumptions

- **Daily Active Users (DAU)**: 100 million users
- **Peak Traffic Factor**: Peak traffic is 3x average traffic
- **Geographic Distribution**: Global user base
- **Data Retention**: 5 years maximum

### Traffic Estimation

- **Read-to-Write Ratio**: 100:1 (read-heavy system)
- **Queries Per Second (QPS)**
  - Write QPS: `1,000,000 URLs/day ÷ 86,400 seconds = ~12 writes/second`
  - Read QPS: `12 writes/second × 100 = 1,200 reads/second`
- **Peak Traffic**: `1,200 × 3 = 3,600 reads/second`

### Storage Estimation

- **Data Size per Entity**: 500 bytes per URL entry
- **Daily Storage Growth**: `1,000,000 × 500 bytes = 500 MB/day`
- **Total Storage Requirements**: `500 MB/day × 365 days × 5 years = ~900 GB`

### Bandwidth Estimation

- **Write Bandwidth**: `12 writes/second × 500 bytes = 6 KB/second`
- **Read Bandwidth**: `1,200 reads/second × 500 bytes = 600 KB/second`
- **Total Bandwidth**: `6 KB/s + 600 KB/s = 606 KB/second`

### Cache Estimation

- **Cache Hit Ratio**: 80% (Pareto principle, a.k.a. 80/20 rule: 20% of URLs generate 80% of traffic)
- **Cacheable Data**: Frequently accessed URL mappings
- **Cache Size**: `20% × 1,200 QPS × 500 bytes × 1 hour = ~4.3 MB/hour`

### Server Estimation

- **Requests per Server**: 1,000 RPS capacity per server
- **Number of Servers**: `3,600 peak RPS ÷ 1,000 RPS/server = 4 servers` (with `2x` redundancy: `8 servers`)
- **Resource Requirements**: 4 CPU cores, 8GB RAM, 100GB SSD per server

## 3. Core System Components

### System Components

- **Core Services**
  - URL Shortening Service: Generates short URLs and handles CRUD operations
  - URL Redirection Service: Handles redirects
  - URL Expiration Service: Handles URL expiration and cleanup

- **Data Layer**
  - Primary Database: Stores URL mappings
  - Cache Layer: Redis for frequently accessed URLs

- **Infrastructure**
  - Load Balancer: Distributes traffic
  - CDN: Caches static content globally
  - Rate Limiter: Prevents abuse

### API Design

- **RESTful Endpoints**
  - `POST /api/v1/urls` - Create short URL
  - `GET /{shortUrl}` - Redirect to original URL
  - `DELETE /api/v1/urls/{shortUrl}` - Delete short URL
  - `GET /api/v1/urls/{shortUrl}/info` - Get URL information

- **Request/Response Format**: JSON for API responses

- **Rate Limiting**
  - 100 requests per minute per IP
  - 1000 requests per hour per user

#### API Endpoints Schemas

```plaintext
POST /api/v1/urls

Content-Type: application/json
Status: 201 Created

Request:
{
  "originalUrl": "https://www.example.com/very/long/url/path",
  "expiresAt": "2027-12-31T23:59:59Z"
}

Response:
{
  "shortUrl": "https://short.ly/abc123",
  "originalUrl": "https://www.example.com/very/long/url/path",
  "createdAt": "2024-01-01T00:00:00Z",
  "expiresAt": "2027-12-31T23:59:59Z"
}
```

```plaintext
GET /{shortUrl}

Status: 301 Moved Permanently
Redirects to: `https://www.example.com/very/long/url/path`
```

```plaintext
DELETE /api/v1/urls/{shortUrl}

Content-Type: application/json
Status: 200 OK

Response:
{
  "message": "URL deleted successfully"
}
```

```plaintext
GET /api/v1/urls/{shortUrl}/info

Content-Type: application/json
Status: 200 OK

Response:
{
  "shortUrl": "https://short.ly/abc123",
  "originalUrl": "https://www.example.com/very/long/url/path",
  "createdAt": "2024-01-01T00:00:00Z",
  "expiresAt": "2027-12-31T23:59:59Z"
}
```

## 4. High-Level Design

### Component Diagram

```mermaid
graph TD
    subgraph "Client"
        A[Client]
    end
    subgraph "Load Balancer Layer"
        B[Load Balancer & Reverse Proxy]
    end
    subgraph "Rate Limiter"
        C[Rate Limiter]
        D[(Rate Limiter Redis Cluster)]
    end
    subgraph "Services"
        E[URL Shortening Service]
        F[URL Redirection Service]
        L[URL Expiry Service]
    end
    subgraph "Cache"
        G[(URL Cache Redis Cluster)]
    end
    subgraph "Database"
        H[(Primary Database)]
        I[(Read Replica 1)]
        J[(Read Replica 2)]
    end
    
    A --"POST /api/v1/urls"----> B
    A --"GET /{shortUrl}"----> B
    A --"DELETE /api/v1/urls/{shortUrl}"----> B
    B --"forward request"----> C
    C --"check request limits using cache"----> D
    C --"allow/deny decision"----> B
    B --"forward allowed request"-----> E
    B --"forward allowed request"----> F
    B --"rate limit exceeded response"----> A
    E --"send response to Load Balancer"----> B
    F --"send response to Load Balancer"----> B
    E --"check for URL in cache"----> G
    F --"check for URL in cache"---> G
    G --"return cached URL (cache hit)"----> E
    G --"return cached URL (cache hit)"---> F
    E --"fetch from database (cache miss)"----> H
    F --"fetch from database (cache miss)"----> H
    E --"store new URL in database"----> H
    H --"replicate data"---> I
    H --"replicate data"---> J
    L --"run cron job to check expired URLs (e.g., every 24 hours)"----> H
    L --"delete expired URLs"----> H
    B --"forward response to Client"----> A
```

### System Flow

Here is the flow of how the system works:

```mermaid
sequenceDiagram
    actor Client
    participant LB as Load Balancer & Reverse Proxy
    participant RL as Rate Limiter
    participant RLC as Rate Limiter Cache
    participant US as URL Shortening Service
    participant UR as URL Redirection Service
    participant Cache as URL Cache Redis Cluster
    participant DB as Primary Database
    participant RR as Read Replica
    participant Expiry as URL Expiry Service

    Client->>LB: POST /api/v1/urls
    Client->>LB: GET /{shortUrl}
    Client->>LB: DELETE /api/v1/urls/{shortUrl}
    LB->>RL: forward request
    RL->>RLC: check request limits
    alt Request Allowed
        RLC->>RL: allow request
        RL->>LB: allow request
        LB->>US: forward to URL Shortening Service
        LB->>UR: forward to URL Redirection Service
    else Request Denied
        RLC->>RL: deny request (rate limit exceeded)
        RL->>LB: deny request (rate limit exceeded)
        LB->>Client: rate limit exceeded response
    end
    US->>Cache: check for URL in cache
    UR->>Cache: check for URL in cache
    alt Cache Hit
        Cache->>US: return cached URL
        Cache->>UR: return cached URL
    else Cache Miss
        US->>DB: fetch URL from database
        UR->>DB: fetch URL from database
        DB->>US: return fetched URL
        DB->>UR: return fetched URL
        US->>Cache: store URL in cache
        UR->>Cache: store URL in cache
    end
    US->>LB: send response to Load Balancer
    UR->>LB: send response to Load Balancer
    LB->>Client: forward response to Client
    US->>DB: store new URL in database
    DB->>RR: replicate data to read replica
    Expiry-->>DB: periodically check expired URLs (cron job)
    Expiry->>DB: delete expired URLs
```

### Data Flow

- **Request Flow**
  1. Client sends request to load balancer
  2. Rate limiter checks request limits using Redis cluster
  3. Request routed to appropriate service
  4. Service checks cache first, then database
  5. Response sent back through load balancer

- **Response Flow**
  1. Service generates response
  2. Cache updated if needed (write-through)
  3. Response sent to client
  4. CDN caches static responses

- **Background Processing**
  1. Expiration service runs periodically (every 24 hours)
  2. Expired URLs marked for deletion
  3. Cleanup operations performed asynchronously

### Scalability Strategy

- **Horizontal Scaling**: Add more application servers
- **Vertical Scaling**: Increase server resources
- **Database Scaling**: Read replicas, sharding by short URL hash

## 5. Detailed Design

### Database Design

- **Database Choice**: NoSQL (MongoDB) for horizontal scaling and flexible schema
- **Schema Design**: Single collection for URL mappings
- **Indexing Strategy**:
  - Primary index on `shortUrl` (unique)
  - Secondary index on `expiresAt` for cleanup
  - Compound index on `createdAt` and `expiresAt`
- **Data Sharding**: Shard by hash of short URL

#### Example Schema

| Field       | Type     | Description          | Example                 |
|-------------|----------|----------------------|-------------------------|
| shortUrl    | String   | Primary key          | "abc123"                |
| originalUrl | String   | Original URL         | "<https://example.com>" |
| createdAt   | DateTime | Creation timestamp   | "2024-01-01T00:00:00Z"  |
| expiresAt   | DateTime | Expiration timestamp | "2024-12-31T23:59:59Z"  |
| userId      | String   | User identifier      | "user123"               |
| isActive    | Boolean  | URL status           | true                    |

### Core Algorithms

- **URL Shortening Algorithm**
  1. Generate unique ID (using counter or hash)
  2. Encode ID to base62 for human-readable format
  3. Check for collisions (retry if needed)
  4. Store mapping in database

- **Simple Hash-based Approach (with Collision Handling)**

```python
import hashlib
import time
import random
import string

def generate_short_url(original_url):
    """Main function to generate short URL with collision handling"""
    max_retries = 5
    
    for attempt in range(max_retries):
        # Generate unique input string
        unique_string = create_unique_string(original_url, attempt)
        
        # Generate short ID from hash
        short_id = generate_short_id(unique_string)
        
        # Check for collision
        if not url_exists(short_id):
            return f"https://short.ly/{short_id}"
    
    # Fall back to counter-based if all retries failed
    return generate_short_url_counter(original_url)

def create_unique_string(original_url, attempt):
    """Create unique input string for hashing"""
    timestamp = time.time()
    
    if attempt == 0:
        return f"{original_url}{timestamp}"
    else:
        # Add random suffix for retry attempts
        random_suffix = generate_random_suffix()
        return f"{original_url}{timestamp}{random_suffix}"

def generate_random_suffix():
    """Generate 4-character random suffix"""
    chars = string.ascii_letters + string.digits
    return ''.join(random.choices(chars, k=4))

def generate_short_id(unique_string):
    """Generate 7-character short ID from input string"""
    hash_hex = hashlib.md5(unique_string.encode()).hexdigest()
    return hash_hex[:7]

def url_exists(short_id):
    """Check if short_id already exists in database"""
    # Database query to check existence
    # Return True if exists, False otherwise
    pass
```

- **Counter-based Approach**

```python
def generate_short_url_counter(original_url):
    # Get next counter from database
    counter = get_next_counter()
    
    # Convert to base62 (simple approach)
    short_id = base62_encode(counter)
    
    return f"https://short.ly/{short_id}"

def base62_encode(num):
    chars = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789"
    if num == 0:
        return "a"
    
    result = ""
    while num > 0:
        result = chars[num % 62] + result
        num //= 62
    
    # Pad to 7 characters if needed
    return result.zfill(7)
```

### Approaches to Generate 7-Character Short URLs

**Goal**: Generate 7-character short URLs that are human-readable and unique.

**Three Simple Approaches:**

1. **Hash-based**: Take first 7 characters of MD5 hash with collision retry
   - **Pros**: Fast, distributed, good randomness
   - **Cons**: Requires database check, retry logic needed
   - **Example**: `a1b2c3d` from MD5 hash, retry with random suffix if collision

2. **Counter-based**: Use database counter converted to 7 characters base62
   - **Pros**: Guaranteed unique, sequential
   - **Cons**: Requires database call (or maybe we can use a counter in memory)
   - **Example**: Counter 1000 → `g8` (padded to 7 chars)

**Simple Math:**

- **7 characters**: Can hold 62^7 = ~3.5 trillion combinations for base62 encoding
- **With 1M URLs/day**: Takes ~9,500 years to exhaust
- **Collision probability**: Very low for practical purposes

## 6. Scalability & Reliability

### Scaling Strategies

- **Horizontal Scaling**: Add more application servers behind load balancer
- **Database Scaling**:
  - Read replicas for read-heavy workload
  - Sharding by short URL hash for write scaling
- **Load Balancing**: Round-robin with health checks

### Reliability & Fault Tolerance

- **Data Replication**: 3 replicas for each database shard
- **Failover Mechanisms**:
  - Automatic failover to read replicas
  - Circuit breakers for external service calls
- **Consistency vs Availability**: Strong consistency for writes, eventual consistency for reads

### Performance Optimization

- **Caching**:
  - Cache frequently accessed URLs in Redis
  - TTL-based cache expiration
  - Cache-aside pattern for consistency
- **Bottlenecks**:
  - Database write capacity
  - Cache memory usage
  - Network bandwidth
- **Mitigation Strategies**:
  - Database sharding for write scaling
  - Cache eviction policies (LRU)
  - CDN for global content delivery
- **Monitoring**:
  - QPS metrics
  - Response time percentiles
  - Cache hit rates
  - Database connection pools utilization

## 7. Trade-offs & Discussion

### Design Decisions

- **Technology Choices**
  - **NoSQL over SQL**: Better horizontal scaling, flexible schema
  - **Redis for caching**: Fast in-memory storage, built-in expiration

- **Architecture Trade-offs**
  - **Monolithic vs Microservices**: Started monolithic, can evolve to microservices
  - **Synchronous vs Asynchronous**: Synchronous for user requests, asynchronous for cleanup

### Alternative Approaches

- **Other Solutions**
  - **UUID-based**: More collision-resistant but longer URLs
  - **Database auto-increment**: Simpler but harder to scale
  - **Pre-generated short codes**: Faster but requires upfront storage

- **Why Not Chosen**
  - UUIDs create longer URLs (not user-friendly)
  - Auto-increment creates scaling bottlenecks
  - Pre-generated codes waste storage for unused codes

### Future Considerations

- **Scalability Limits**: System can handle 10x growth with horizontal scaling
- **Feature Extensions**:
  - Add analytics and click tracking
  - Support custom short URLs
  - Add bulk operations

## 8. Implementation Considerations

### Database Sharding Strategy

```python
def get_shard_key(short_url):
    # Use consistent hashing to determine shard
    hash_value = hashlib.md5(short_url.encode()).hexdigest()
    shard_id = int(hash_value[:2], 16) % NUM_SHARDS
    return f"shard_{shard_id}"
```

### Caching

```python
import redis

class URLCache:
    def __init__(self):
        self.redis_client = redis.Redis(host='localhost', port=6379, db=0)
    
    def get_url(self, short_url):
        # Try cache first
        cached_url = self.redis_client.get(short_url)
        if cached_url:
            return cached_url.decode()
        
        # Cache miss - get from database
        original_url = self.get_from_database(short_url)
        if original_url:
            # Cache for 1 hour
            self.redis_client.setex(short_url, 3600, original_url)
        
        return original_url
```

### Rate Limiting

```python
from collections import defaultdict, deque
import time

class RateLimiter:
    def __init__(self, max_requests=100, window_seconds=60):
        self.max_requests = max_requests
        self.window_seconds = window_seconds
        self.requests = defaultdict(deque)
    
    def is_allowed(self, client_id):
        now = time.time()
        client_requests = self.requests[client_id]
        
        # Remove old requests outside window
        while client_requests and client_requests[0] <= now - self.window_seconds:
            client_requests.popleft()
        
        # Check if under limit
        if len(client_requests) < self.max_requests:
            client_requests.append(now)
            return True
        
        return False
```

### URL Expiration Service

```python
import schedule
import time
from datetime import datetime, timedelta

class URLExpirationService:
    def __init__(self, database):
        self.db = database
    
    def check_expired_urls(self):
        """Run every 24 hours to check for expired URLs"""
        current_time = datetime.utcnow()
        
        # Find URLs that have expired
        expired_urls = self.db.find_urls_where({
            'expiresAt': {'$lt': current_time},
            'isActive': True
        })
        
        for url in expired_urls:
            # Mark as inactive
            self.db.update_url(url['shortUrl'], {'isActive': False})
    
    def cleanup_expired_urls(self):
        """Actually delete expired URLs (run less frequently)"""
        cutoff_date = datetime.utcnow() - timedelta(days=30)  # Keep for 30 days after expiration
        
        deleted_count = self.db.delete_urls_where({
            'expiresAt': {'$lt': cutoff_date},
            'isActive': False
        })
        
        print(f"Cleaned up {deleted_count} expired URLs")
    
    def start_scheduler(self):
        """Start the background scheduler"""
        # Check for expired URLs every 24 hours
        schedule.every(24).hours.do(self.check_expired_urls)
        
        # Clean up old expired URLs every week
        schedule.every().week.do(self.cleanup_expired_urls)
        
        while True:
            schedule.run_pending()
            time.sleep(3600)  # Check every hour
```

## Summary

- **High Performance**: Sub-100ms response times with caching
- **Scalability**: Handles 100M DAU with horizontal scaling
- **Reliability**: 99.9% availability with redundancy
- **Cost Efficiency**: Optimized resource usage with caching and CDN

## Reference Materials

- [Design Bit.ly](https://www.hellointerview.com/learn/system-design/problem-breakdowns/bitly)
- [Design a URL Shortener (Bitly) - System Design Interview](https://www.youtube.com/watch?v=qSJAvd5Mgio)
- [Beginner System Design Interview: Design Bitly](https://www.youtube.com/watch?v=iUU4O1sWtJA)
