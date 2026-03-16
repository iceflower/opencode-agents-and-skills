# System Design Interview Patterns

Large-scale system design principles and patterns. Use when designing scalable architectures, preparing for system design interviews, or evaluating system architecture.

> **Note**: This document synthesizes system design concepts from various sources including industry best practices, technical blogs, and community knowledge.

## Design Discussion Framework

### General Approach

When approaching a system design problem:

1. **Understand Requirements**
   - Clarify functional requirements
   - Define scope and constraints
   - Identify non-functional requirements (scale, latency)

2. **Create High-Level Design**
   - Sketch core components
   - Show data flow between components
   - Discuss key decisions

3. **Detail Components**
   - Deep dive on critical components
   - Discuss trade-offs
   - Handle edge cases

4. **Review and Improve**
   - Identify bottlenecks
   - Suggest optimizations
   - Consider failure scenarios

---

## 1. Scalability Fundamentals

### Vertical vs Horizontal Scaling

| Aspect     | Vertical (Scale Up)   | Horizontal (Scale Out)  |
|------------|-----------------------|-------------------------|
| Approach   | Bigger server         | More servers            |
| Limit      | Hardware max          | Theoretically unlimited |
| Cost       | Expensive at high end | Linear growth           |
| Complexity | Simple                | Requires load balancing |
| Failure    | Single point          | Graceful degradation    |

### Load Balancer

Distribute traffic across servers.

```text
         ┌──────────────┐
         │Load Balancer │
         │  (Public IP) │
         └──────┬───────┘
          ┌─────┼─────┐
          ▼     ▼     ▼
      ┌─────┐ ┌─────┐ ┌─────┐
      │ Svr1│ │ Svr2│ │ Svr3│
      └─────┘ └─────┘ └─────┘
```

**Algorithms**:

- Round-robin
- Weighted round-robin
- Least connections
- IP hash
- Health-check based

**Layer Selection**:

- Layer 4 (Transport): TCP/UDP routing
- Layer 7 (Application): HTTP path/header routing

---

## 2. Database Architecture

### RDBMS vs NoSQL

| Feature  | RDBMS           | NoSQL                     |
|----------|-----------------|---------------------------|
| Schema   | Fixed           | Flexible                  |
| Scaling  | Vertical        | Horizontal                |
| ACID     | Full            | Varies                    |
| Joins    | Native          | Limited                   |
| Use Case | Structured data | Unstructured, high volume |

### Database Replication

```text
     Writes           Reads
        │                │
        ▼                ▼
┌───────────────┐  ┌───────────────┐
│ Master DB     │──▶ Replica 1    │
│ (Primary)     │  └───────────────┘
└───────────────┘  ┌───────────────┐
                   │ Replica 2    │
                   └───────────────┘
```

**Benefits**:

- Better read performance
- Reliability (data copies)
- Availability (failover)

### Database Sharding

Partition data across multiple servers.

```text
┌─────────────────────────────────────────────┐
│                Sharding Key                 │
│                                             │
│  user_id % 3 = 0  →  Shard 1               │
│  user_id % 3 = 1  →  Shard 2               │
│  user_id % 3 = 2  →  Shard 3               │
└─────────────────────────────────────────────┘
```

**Sharding Strategies**:

- Hash-based: `shard = hash(key) % num_shards`
- Range-based: A-M → Shard 1, N-Z → Shard 2
- Directory-based: Lookup table for mapping

**Challenges**:

- Cross-shard joins
- Rebalancing
- Hot spots

---

## 3. Caching Strategies

### Cache Benefits

- Reduced latency
- Reduced database load
- Improved throughput

### Cache Patterns

#### Cache-Aside (Lazy Loading)

```text
1. Check cache
2. If miss, read from DB
3. Write to cache
4. Return data
```

```java
public Data getData(String key) {
    Data data = cache.get(key);
    if (data == null) {
        data = database.query(key);
        cache.set(key, data, ttl);
    }
    return data;
}
```

#### Write-Through

```text
1. Write to cache
2. Write to database
3. Confirm only after both succeed
```

#### Write-Back (Write-Behind)

```text
1. Write to cache
2. Acknowledge immediately
3. Async write to database
```

### Cache Eviction Policies

| Policy | Description           | Use Case             |
|--------|-----------------------|----------------------|
| LRU    | Least Recently Used   | General purpose      |
| LFU    | Least Frequently Used | Popular items matter |
| FIFO   | First In First Out    | Simple needs         |
| TTL    | Time To Live          | Freshness matters    |

### Cache Considerations

- **Consistency**: Cache may be stale
- **TTL**: Balance freshness vs load
- **Penetration**: Handle missing keys
- **Avalanche**: Stagger expirations
- **Breakdown**: Lock hot keys

---

## 4. Content Delivery Network (CDN)

Distribute static content geographically.

```text
┌──────────────┐
│    User      │
│  (Seoul)     │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  Edge Server │  ← Cached content
│  (Seoul)     │
└──────────────┘
       │ (cache miss)
       ▼
┌──────────────┐
│   Origin     │
│  (US East)   │
└──────────────┘
```

### CDN Workflow

1. User requests static content (image, CSS, JS)
2. CDN edge server checks cache
3. If cached → return immediately
4. If not cached → fetch from origin, cache, return

### CDN Considerations

- **Cost**: Pay for bandwidth
- **TTL**: Content freshness
- **Invalidation**: How to update content
- **Failure**: Fallback to origin

---

## 5. Stateless Architecture

### Problem: Sticky Sessions

```text
User A always → Server 1
User B always → Server 2

If Server 1 fails → User A loses session
```

### Solution: External Session Store

```text
┌─────────────────────────────────────────────────┐
│                  Web Servers                     │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐         │
│  │ Server 1│  │ Server 2│  │ Server 3│         │
│  └────┬────┘  └────┬────┘  └────┬────┘         │
│       │            │            │               │
│       └────────────┼────────────┘               │
│                    │                            │
│                    ▼                            │
│            ┌───────────────┐                   │
│            │ Session Store │  ← Redis/Memcached │
│            └───────────────┘                   │
└─────────────────────────────────────────────────┘
```

### Benefits of Stateless Design

- Any server can handle any request
- Easy horizontal scaling
- Graceful server failure

---

## 6. Data Center Architecture

### Multi-Data Center Setup

```text
┌─────────────────────┐    ┌─────────────────────┐
│   Data Center 1     │    │   Data Center 2     │
│    (US East)        │    │    (US West)        │
│                     │    │                     │
│  ┌───────────────┐  │    │  ┌───────────────┐  │
│  │ Load Balancer │  │    │  │ Load Balancer │  │
│  └───────┬───────┘  │    │  └───────┬───────┘  │
│          │          │    │          │          │
│    ┌─────┴─────┐    │    │    ┌─────┴─────┐    │
│    ▼           ▼    │    │    ▼           ▼    │
│ ┌─────┐    ┌─────┐  │    │ ┌─────┐    ┌─────┐  │
│ │Web 1│    │Web 2│  │    │ │Web 1│    │Web 2│  │
│ └─────┘    └─────┘  │    │ └─────┘    └─────┘  │
└─────────────────────┘    └─────────────────────┘
```

### Geographic Routing

- Users directed to nearest data center
- DNS-based (GeoDNS)
- Anycast routing

### Failure Handling

- **Active-Active**: Both serve traffic
- **Active-Passive**: Standby takes over
- Failover time vs data consistency trade-off

---

## 7. Message Queue

Decouple components with async messaging.

```text
Producer                 Queue                   Consumer
   │                      │                        │
   │─── Message ─────────▶│                        │
   │                      │─── Message ───────────▶│
   │                      │                        │
   │                      │─── Message ───────────▶│
   │─── Message ─────────▶│                        │
```

### Message Queue Benefits

- Decoupling: Producer doesn't need consumer details
- Buffering: Handle traffic spikes
- Scalability: Add more consumers
- Reliability: Persistent messages

### Use Cases

- Async processing
- Background jobs
- Event notification
- Log aggregation

---

## 8. System Design Components

### Latency Reference Numbers

> Source: Based on "Numbers Every Programmer Should Know" by Jeff Dean (Google). Actual values vary by hardware.

| Operation                 | Approximate Latency |
|---------------------------|---------------------|
| L1 cache reference        | ~1 ns               |
| L2 cache reference        | ~4 ns               |
| Mutex lock/unlock         | ~17 ns              |
| Main memory reference     | ~100 ns             |
| SSD random read           | ~16 μs              |
| Read 1MB from SSD         | ~50 μs              |
| Network round-trip same DC| ~500 μs             |
| Disk seek                 | ~3 ms               |

### Capacity Estimation Example

```text
Example calculation (adjust numbers for your use case):

Daily Active Users: 500K
Requests per user: 80/day
QPS = 500K * 80 / 86400 ≈ 460 QPS
Peak QPS = QPS * 2-3 ≈ 1,400 QPS

Storage:
Per user: 500KB/day
Daily: 500K * 500KB = 250GB
With replication (3x): 750GB/day
```

---

## 9. Consistent Hashing

Distribute keys evenly across changing servers.

### Problem: Rehashing

```text
With 4 servers: key → server = hash(key) % 4
Add 1 server:   key → server = hash(key) % 5
Result: Most keys need to move!
```

### Solution: Ring-based Hashing

```text
         0
        / \
   Server1  Server3
      /        \
   270        90
      \        /
   Server2  Server4
        \ /
        180

Key hashes to position on ring,
assigned to next clockwise server.
```

**Benefits**:

- Adding/removing server: Only K/N keys move
- Even distribution with virtual nodes

```text
Virtual Nodes:
Server1: v1, v2, v3, v4, ...
Server2: v1, v2, v3, v4, ...
```

---

## 10. Rate Limiting

Control request rate per client.

### Algorithms

#### Token Bucket

```text
- Bucket holds tokens (max: capacity)
- Tokens added at fixed rate
- Each request consumes 1 token
- No tokens = reject
```

#### Sliding Window

```text
- Track requests in time window
- New request: count requests in last N seconds
- If count > limit: reject
```

### Implementation

```java
class RateLimiter {
    private final int maxRequests;
    private final Duration window;
    private final Map<String, LinkedList<Long>> requests = new ConcurrentHashMap<>();
    
    public boolean allowRequest(String clientId) {
        long now = System.currentTimeMillis();
        LinkedList<Long> windowRequests = requests.computeIfAbsent(clientId, k -> new LinkedList<>());
        
        synchronized (windowRequests) {
            // Remove expired timestamps
            while (!windowRequests.isEmpty() && 
                   now - windowRequests.getFirst() > window.toMillis()) {
                windowRequests.removeFirst();
            }
            
            if (windowRequests.size() < maxRequests) {
                windowRequests.addLast(now);
                return true;
            }
            return false;
        }
    }
}
```

---

## 11. Key-Value Store Design

### Key-Value Store Requirements

- High availability
- Low latency
- Scalability
- Configurable consistency

### Architecture

```text
┌─────────────────────────────────────────────────┐
│                  Client                          │
└───────────────────────┬─────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────┐
│              Coordinator Service                 │
│  (Membership, Failure Detection, Routing)       │
└───────────────────────┬─────────────────────────┘
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
┌───────────────┐ ┌───────────────┐ ┌───────────────┐
│   Node 1      │ │   Node 2      │ │   Node 3      │
│ (Partition 1) │ │ (Partition 2) │ │ (Partition 3) │
└───────────────┘ └───────────────┘ └───────────────┘
```

### Components

1. **Data Partitioning**: Consistent hashing
2. **Replication**: Leader-follower or quorum
3. **Consistency**: Tunable (R + W > N for strong)
4. **Failure Detection**: Heartbeats, gossip

---

## 12. Unique ID Generation

### Unique ID Requirements

- Uniqueness (global)
- Ordered (approximately)
- Distributed generation

### Approaches

#### UUID

```text
128-bit: 32 hex chars
Pros: No coordination needed
Cons: Not ordered, large (36 chars)
```

#### Snowflake (Twitter)

```text
64-bit ID structure:

Timestamp: milliseconds since epoch
Machine ID: up to 1024 machines
Sequence: up to 4096 IDs per ms per machine
```

```java
class SnowflakeIdGenerator {
    private final long machineId;
    private long lastTimestamp = -1L;
    private long sequence = 0L;
    
    public synchronized long nextId() {
        long timestamp = System.currentTimeMillis();
        
        if (timestamp == lastTimestamp) {
            sequence = (sequence + 1) & 0xFFF; // 12 bits
            if (sequence == 0) {
                timestamp = waitNextMillis(lastTimestamp);
            }
        } else {
            sequence = 0;
        }
        
        lastTimestamp = timestamp;
        return (timestamp << 22) | (machineId << 12) | sequence;
    }
}
```

---

## 13. System Design Patterns Summary

| Problem                  | Pattern                               |
|--------------------------|---------------------------------------|
| Single server bottleneck | Load balancer + horizontal scaling    |
| Database overload        | Caching, read replicas                |
| Large dataset            | Sharding, partitioning                |
| Geographic latency       | CDN, multi-DC                         |
| Session management       | External session store                |
| Service coupling         | Message queue                         |
| Hot partitions           | Consistent hashing with virtual nodes |
| Traffic spikes           | Rate limiting, circuit breaker        |

---

## 14. Design Discussion Checklist

### Clarify Requirements

- [ ] Functional requirements
- [ ] Non-functional requirements (scale, latency)
- [ ] Out of scope items

### Define Constraints

- [ ] Traffic estimates (QPS)
- [ ] Storage estimates
- [ ] Bandwidth estimates

### Design Components

- [ ] Client → API layer
- [ ] API layer → Service layer
- [ ] Service layer → Data layer
- [ ] Cache strategy
- [ ] Async processing (if needed)

### Deep Dive Topics

- [ ] Database schema
- [ ] API design
- [ ] Scalability approach
- [ ] Failure handling
- [ ] Monitoring/logging

---

## Related Skills

- **api-design**: REST API design principles
- **distributed-systems**: Distributed patterns
- **messaging**: Message broker patterns
- **caching**: Cache implementation patterns

---

## References

- Designing Data-Intensive Applications by Martin Kleppmann
- Building Secure and Reliable Systems by Google
- Various technical blogs and community resources
