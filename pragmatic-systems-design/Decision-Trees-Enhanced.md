# System Design Decision Trees — Interview Edition

> Quick decision guides with **trade-offs, numbers, and clarifying questions** for system design interviews.

---

## How to Use This Document

**In an interview, don't jump to solutions.** Use the "Questions to Ask" sections to gather requirements first. The decision trees guide you AFTER you understand the constraints.

**Know the numbers.** Interviewers expect back-of-envelope calculations. Memorize the key figures in each section.

**Articulate trade-offs.** Every choice has downsides. Stating them proactively shows senior-level thinking.

---

## 1. Database Selection

### Questions to Ask First
- What's the read:write ratio?
- Do we need ACID transactions? Which operations?
- What's the expected data volume? (GB/TB/PB)
- What query patterns? Point lookups vs. range scans vs. full-text?
- What's the consistency requirement? Can we tolerate stale reads?
- Is the schema stable or evolving rapidly?

### Key Numbers to Know
| Database | Throughput | Latency | Max Storage |
|----------|-----------|---------|-------------|
| **PostgreSQL** | ~10K TPS | 1-5ms | ~10TB practical |
| **MySQL** | ~10K TPS | 1-5ms | ~10TB practical |
| **Redis** | ~100K ops/sec | <1ms | Limited by RAM |
| **DynamoDB** | ~100K+ ops/sec | <10ms | Unlimited |
| **MongoDB** | ~20K ops/sec | 1-5ms | ~100TB |
| **Cassandra** | ~50K+ writes/sec | 1-10ms | Petabytes |
| **Elasticsearch** | ~10K writes/sec | 10-100ms (search) | Petabytes |

```plantuml
@startuml
skinparam backgroundColor #FEFEFE
skinparam defaultFontSize 14
skinparam ActivityBackgroundColor #E3F2FD
skinparam ActivityBorderColor #1976D2

title **Database Selection**

start
:What's your primary need?;

if (Need ACID transactions?) then (Yes)
    #C8E6C9:Use **PostgreSQL/MySQL**;
    note right
      Banking, Orders
      Inventory, Bookings
    end note
    stop
else (No)
endif

if (Need 100K+ ops/sec?) then (Yes)
    if (Simple key lookups?) then (Yes)
        #C8E6C9:Use **Redis/DynamoDB**;
        note right
          Caching, Sessions
          Counters, Leaderboards
        end note
        stop
    else (Complex queries)
        #C8E6C9:Use **MongoDB** or **Cassandra**;
        stop
    endif
else (No)
endif

if (Need full-text search?) then (Yes)
    #C8E6C9:Use **Elasticsearch**;
    note right
      Product search
      Log analysis
    end note
    stop
else (No)
endif

if (Need flexible schema?) then (Yes)
    #C8E6C9:Use **MongoDB**;
    note right
      CMS, User profiles
      Rapid prototyping
    end note
    stop
else (No)
endif

if (Time-series / Write-heavy?) then (Yes)
    #C8E6C9:Use **Cassandra** or **TimescaleDB**;
    note right
      Metrics, IoT
      Logs, Events
    end note
    stop
else (No)
endif

#C8E6C9:Default to **PostgreSQL**;
note right
  Most versatile
  Start here
end note
stop

@enduml
```

### Trade-offs Deep Dive

| Choice | Strengths | Weaknesses | When NOT to Use |
|--------|-----------|------------|-----------------|
| **PostgreSQL** | ACID, JSON support, mature ecosystem, extensions | Vertical scaling limits, complex sharding | >10TB data, >50K TPS needed |
| **Redis** | Sub-millisecond latency, data structures | RAM-limited, durability concerns | Primary data store, complex queries |
| **DynamoDB** | Unlimited scale, managed, predictable latency | Expensive at scale, limited query flexibility, vendor lock-in | Complex joins, ad-hoc queries |
| **MongoDB** | Flexible schema, horizontal scaling, aggregation pipeline | Weaker consistency by default, memory-hungry | Financial transactions, strict ACID |
| **Cassandra** | Linear write scaling, no single point of failure | Eventually consistent, no joins, operational complexity | Small datasets, need for ACID |
| **Elasticsearch** | Powerful full-text search, aggregations | Not a primary datastore, eventual consistency, resource-intensive | Primary storage, transactions |

### Implementation Notes
- **PostgreSQL sharding**: Use Citus extension or application-level sharding. Built-in partitioning for time-series.
- **Redis persistence**: RDB (snapshots) vs AOF (append-only). AOF safer but slower. Use both in production.
- **DynamoDB capacity**: On-demand vs provisioned. On-demand 5-7x more expensive but auto-scales.
- **MongoDB replica set**: Minimum 3 nodes. Primary handles writes, secondaries for reads.

---

## 2. Communication Protocol

### Questions to Ask First
- Who initiates communication? Client or server?
- Is it request-response or streaming?
- Do we need bidirectional communication?
- What's the latency requirement?
- Are there firewall/proxy constraints?
- Internal services or external/public API?

### Key Numbers to Know
| Protocol | Latency | Throughput | Connection Overhead |
|----------|---------|------------|---------------------|
| **REST/HTTP** | 10-100ms | ~10K req/sec per server | New connection per request (without keep-alive) |
| **gRPC** | 1-10ms | ~50K req/sec per server | Persistent connection, multiplexed |
| **WebSocket** | <10ms | ~100K msgs/sec per server | Single persistent connection |
| **SSE** | <100ms | ~10K clients per server | One-way, auto-reconnect |

```plantuml
@startuml
skinparam backgroundColor #FEFEFE
skinparam defaultFontSize 14
skinparam ActivityBackgroundColor #E3F2FD
skinparam ActivityBorderColor #1976D2

title **Communication Protocol Selection**

start
:What type of communication?;

if (Server needs to push to client?) then (Yes)
    if (Bidirectional real-time?) then (Yes)
        #C8E6C9:Use **WebSockets**;
        note right
          Chat, Live auctions
          Multiplayer games
          Collaborative editing
        end note
        stop
    else (One-way push)
        if (Browser client?) then (Yes)
            #C8E6C9:Use **Server-Sent Events (SSE)**;
            note right
              Live scores
              Stock tickers
              Notifications
            end note
            stop
        else (Server-to-server)
            #C8E6C9:Use **Webhooks**;
            note right
              Payment callbacks
              GitHub events
              3rd party integrations
            end note
            stop
        endif
    endif
else (Client initiates)
endif

if (Internal service-to-service?) then (Yes)
    if (High performance critical?) then (Yes)
        #C8E6C9:Use **gRPC**;
        note right
          Microservices mesh
          ML inference
          Low latency required
        end note
        stop
    else (Simplicity preferred)
        #C8E6C9:Use **REST over HTTP**;
        stop
    endif
else (External/Public API)
endif

if (Firewall restrictions?) then (Yes)
    #C8E6C9:Use **Long Polling**;
    note right
      Corporate networks
      Fallback option
    end note
    stop
else (No)
endif

#C8E6C9:Use **REST/HTTP**;
note right
  Standard APIs
  CRUD operations
  Browser clients
end note
stop

@enduml
```

### Trade-offs Deep Dive

| Choice | Strengths | Weaknesses | When NOT to Use |
|--------|-----------|------------|-----------------|
| **REST** | Universal, cacheable, simple, great tooling | Chattier, no server push, overfetching | Real-time needs, high-frequency internal calls |
| **gRPC** | Fast (binary/HTTP2), streaming, strong typing | Harder to debug, limited browser support, steeper learning curve | Public APIs, simple CRUD |
| **WebSocket** | True bidirectional, low latency | Stateful (complicates scaling), no built-in retry | One-way communication, RESTful patterns |
| **SSE** | Simple, auto-reconnect, HTTP-based | One-way only, limited browser connections (~6 per domain) | Bidirectional needs, non-browser clients |
| **Webhooks** | Decoupled, async, simple | Delivery not guaranteed, security concerns, no backpressure | Real-time UI updates, synchronous needs |

### Implementation Notes
- **WebSocket scaling**: Use Redis Pub/Sub or Kafka to broadcast across server instances. Sticky sessions OR broadcast to all.
- **gRPC in Java**: Use `grpc-spring-boot-starter`. Define `.proto` files, generate stubs.
- **REST pagination**: Cursor-based > offset-based for large datasets. Include `next_cursor` in response.
- **Webhook reliability**: Implement retry with exponential backoff. Store events, process async. Signature verification (HMAC).

---

## 3. Caching Strategy

### Questions to Ask First
- What's the read:write ratio? (Caching helps when reads >> writes)
- What's the cache hit rate target? (90%? 99%?)
- Can we tolerate stale data? For how long?
- Is the data shared across instances or instance-local?
- What's the data size? Fits in memory?
- What's the eviction policy? LRU? TTL?

### Key Numbers to Know
| Cache Type | Latency | Throughput | Typical Use |
|------------|---------|------------|-------------|
| **L1/L2 CPU Cache** | ~1ns | - | JVM optimizations |
| **In-Process (Caffeine)** | ~100ns | millions/sec | Config, hot data |
| **Redis** | ~1ms | 100K ops/sec | Sessions, distributed cache |
| **Memcached** | ~1ms | 100K ops/sec | Simple K/V, multi-threaded |
| **CDN** | 10-50ms | unlimited | Static assets |

**Cache Math**: If DB query = 50ms, cache hit = 1ms, and hit rate = 95%, average latency = 0.95 × 1ms + 0.05 × 50ms = **3.45ms** (14x improvement)

```plantuml
@startuml
skinparam backgroundColor #FEFEFE
skinparam defaultFontSize 14
skinparam ActivityBackgroundColor #E3F2FD
skinparam ActivityBorderColor #1976D2

title **Caching Strategy Selection**

start
:What are you caching?;

if (Static assets (JS, CSS, images)?) then (Yes)
    #C8E6C9:Use **CDN**;
    note right
      CloudFront, Cloudflare
      Global edge caching
    end note
    stop
else (Dynamic data)
endif

if (Data shared across instances?) then (Yes)
    if (Need data structures (lists, sets)?) then (Yes)
        #C8E6C9:Use **Redis**;
        note right
          Sessions, Leaderboards
          Rate limiting
        end note
        stop
    else (Simple key-value)
        if (Managed service preferred?) then (Yes)
            #C8E6C9:Use **ElastiCache/Memcached**;
            stop
        else (Self-managed)
            #C8E6C9:Use **Redis Cluster**;
            stop
        endif
    endif
else (Instance-local)
endif

if (Very hot data, nanosecond access?) then (Yes)
    #C8E6C9:Use **In-Process Cache**;
    note right
      Guava, Caffeine
      Config, Templates
    end note
    stop
else (No)
endif

#C8E6C9:Use **Redis** (default distributed cache);
stop

@enduml
```

### Cache Pattern Selection

```plantuml
@startuml
skinparam backgroundColor #FEFEFE
skinparam defaultFontSize 14
skinparam ActivityBackgroundColor #E3F2FD
skinparam ActivityBorderColor #1976D2

title **Cache Pattern Selection**

start
:How should cache interact with DB?;

if (Read-heavy workload?) then (Yes)
    if (Can tolerate stale data?) then (Yes)
        #C8E6C9:Use **Cache-Aside**;
        note right
          Most common pattern
          App manages cache
          TTL-based expiry
        end note
        stop
    else (Need freshness)
        #C8E6C9:Use **Read-Through + TTL**;
        note right
          Cache fetches on miss
          Short TTL for freshness
        end note
        stop
    endif
else (Write-heavy)
endif

if (Must guarantee consistency?) then (Yes)
    #C8E6C9:Use **Write-Through**;
    note right
      Cache + DB updated together
      Slower writes, consistent reads
    end note
    stop
else (Eventual consistency OK)
endif

if (Need maximum write throughput?) then (Yes)
    #C8E6C9:Use **Write-Behind**;
    note right
      Async batch writes to DB
      Risk of data loss
      High throughput
    end note
    stop
else (No)
endif

#C8E6C9:Use **Cache-Aside** (default);
stop

@enduml
```

### Trade-offs Deep Dive

| Pattern | How It Works | Strengths | Weaknesses |
|---------|--------------|-----------|------------|
| **Cache-Aside** | App checks cache → miss → query DB → populate cache | Simple, resilient to cache failure | Cache miss = slow, stale data possible |
| **Read-Through** | Cache fetches from DB on miss automatically | Simpler app code | Cache becomes critical path |
| **Write-Through** | Write to cache + DB synchronously | Strong consistency | Write latency doubled |
| **Write-Behind** | Write to cache, async batch to DB | Fast writes, batching | Data loss risk if cache fails |
| **Refresh-Ahead** | Proactively refresh before expiry | No cache miss latency | Wasted refreshes for unused keys |

### Cache Invalidation Strategies
1. **TTL-based**: Simple, eventual consistency. Set TTL based on staleness tolerance.
2. **Event-driven**: Publish invalidation events on write. More complex but immediate.
3. **Version-based**: Include version in cache key. New version = new key.

### Implementation Notes (Java/Spring)
```java
// Caffeine in-process cache
@Bean
public CacheManager cacheManager() {
    CaffeineCacheManager manager = new CaffeineCacheManager();
    manager.setCaffeine(Caffeine.newBuilder()
        .maximumSize(10_000)
        .expireAfterWrite(Duration.ofMinutes(5))
        .recordStats());
    return manager;
}

// Spring Cache abstraction
@Cacheable(value = "users", key = "#userId")
public User getUser(String userId) { ... }

@CacheEvict(value = "users", key = "#user.id")
public void updateUser(User user) { ... }
```

---

## 4. Scaling Strategy

### Questions to Ask First
- What's the current bottleneck? (CPU, memory, I/O, network)
- What's the target scale? (users, requests/sec, data volume)
- Is the workload stateless or stateful?
- Can we partition the data/workload?
- What's the budget? (Vertical is simpler, horizontal is cheaper at scale)

### Key Numbers to Know
| Scaling Type | Limit | Cost Pattern | Complexity |
|--------------|-------|--------------|------------|
| **Vertical** | ~128 vCPU, 4TB RAM (cloud) | Exponential | Low |
| **Horizontal (stateless)** | Unlimited | Linear | Medium |
| **Database read replicas** | ~15 replicas (AWS RDS) | Linear | Medium |
| **Sharding** | Unlimited | Linear | High |

**Rule of Thumb**: Vertical scale until you hit $10K/month or cloud limits, then go horizontal.

```plantuml
@startuml
skinparam backgroundColor #FEFEFE
skinparam defaultFontSize 14
skinparam ActivityBackgroundColor #E3F2FD
skinparam ActivityBorderColor #1976D2

title **Scaling Strategy Selection**

start
:What's your bottleneck?;

if (Read throughput?) then (Yes)
    if (Can cache help?) then (Yes)
        #C8E6C9:Add **Caching Layer**;
        note right
          Redis for hot data
          CDN for static
          1% rule applies
        end note
        stop
    else (Must hit DB)
        #C8E6C9:Add **Read Replicas**;
        note right
          Each replica = full copy
          Linear read scaling
          Replication lag OK
        end note
        stop
    endif
else (Not reads)
endif

if (Write throughput?) then (Yes)
    if (Writes are independent?) then (Yes)
        #C8E6C9:Use **Sharding**;
        note right
          Partition by shard key
          Each shard handles subset
          Scales reads AND writes
        end note
        stop
    else (Need coordination)
        #C8E6C9:Use **Queue + Batching**;
        note right
          Buffer writes in queue
          Batch to reduce load
          Accept slight delay
        end note
        stop
    endif
else (Not writes)
endif

if (Compute/CPU bound?) then (Yes)
    if (Tasks are stateless?) then (Yes)
        #C8E6C9:Use **Horizontal Scaling**;
        note right
          More instances
          Behind load balancer
          Auto-scaling
        end note
        stop
    else (Stateful)
        #C8E6C9:Use **Vertical Scaling** first;
        note right
          Bigger machine
          Then refactor to stateless
        end note
        stop
    endif
else (Not compute)
endif

if (Storage capacity?) then (Yes)
    if (Structured data?) then (Yes)
        #C8E6C9:Use **Database Sharding**;
        stop
    else (Files/Blobs)
        #C8E6C9:Use **Object Storage (S3)**;
        note right
          Virtually unlimited
          Cheap, durable
        end note
        stop
    endif
else (Unknown)
endif

#C8E6C9:Profile first, then decide;
stop

@enduml
```

### Trade-offs Deep Dive

| Strategy | Strengths | Weaknesses | Operational Cost |
|----------|-----------|------------|------------------|
| **Vertical** | Simple, no code changes | Hard limits, single point of failure, expensive at top | Low |
| **Horizontal** | Unlimited scale, fault tolerant | Stateless requirement, complexity | Medium |
| **Read Replicas** | Easy to add, read scaling | Replication lag, doesn't help writes | Low-Medium |
| **Sharding** | Scales everything | Cross-shard queries hard, resharding painful | High |
| **Caching** | Huge latency improvement | Invalidation complexity, stale data | Medium |

### Capacity Planning Formula
```
Required instances = (Peak RPS × Avg Response Time) / (1000 × Target Utilization)

Example: 10,000 RPS, 100ms response, 70% target utilization
= (10,000 × 0.1) / (1 × 0.7) = 1,428 → ~15 instances (with buffer)
```

---

## 5. Sharding Strategy

### Questions to Ask First
- What's the shard key? (Must be in every query)
- Do you need cross-shard queries? (If yes, reconsider sharding)
- What's the expected data distribution?
- Will you need to add shards? How often?
- Any geographic or compliance requirements?

### Key Numbers to Know
- **Shard size sweet spot**: 100GB - 1TB per shard
- **Resharding cost**: Plan for 2-4x current capacity
- **Cross-shard query penalty**: 10-100x slower than single-shard

```plantuml
@startuml
skinparam backgroundColor #FEFEFE
skinparam defaultFontSize 14
skinparam ActivityBackgroundColor #E3F2FD
skinparam ActivityBorderColor #1976D2

title **Sharding Strategy Selection**

start
:What's your access pattern?;

if (Need even distribution?) then (Yes)
    if (Random access by ID?) then (Yes)
        #C8E6C9:Use **Hash-Based Sharding**;
        note right
          hash(key) % N
          Even distribution
          No hotspots
          URL Shortener, Coupons
        end note
        stop
    else (Range queries needed)
        #C8E6C9:Use **Consistent Hashing**;
        note right
          Easier resharding
          Add nodes smoothly
        end note
        stop
    endif
else (Natural partitions exist)
endif

if (Time-based queries?) then (Yes)
    #C8E6C9:Use **Range-Based (Time)**;
    note right
      Shard by date/month
      Easy archival
      Logs, Events, Metrics
    end note
    stop
else (Not time-based)
endif

if (Geographic locality needed?) then (Yes)
    #C8E6C9:Use **Geographic Sharding**;
    note right
      Shard by region
      Data residency compliance
      Low latency
    end note
    stop
else (No geo requirements)
endif

if (Data naturally partitioned?) then (Yes)
    #C8E6C9:Use **Entity-Based Sharding**;
    note right
      By tenant, venue, org
      No cross-shard queries
      Ticketing by venue
    end note
    stop
else (No natural partitions)
endif

if (Need maximum flexibility?) then (Yes)
    #C8E6C9:Use **Directory-Based**;
    note right
      Lookup table
      Any routing logic
      More complex
    end note
    stop
else (No)
endif

#C8E6C9:Default to **Hash-Based**;
stop

@enduml
```

### Trade-offs Deep Dive

| Strategy | Distribution | Range Queries | Resharding | Best For |
|----------|--------------|---------------|------------|----------|
| **Hash-Based** | Even | Impossible | Hard (rehash all) | User data, random access |
| **Consistent Hashing** | Even | Impossible | Easy (minimal movement) | Caches, distributed systems |
| **Range-Based** | Uneven (hotspots) | Efficient | Medium | Time-series, logs |
| **Geographic** | By region | Within region | Easy | Global apps, compliance |
| **Directory-Based** | Configurable | Configurable | Easy | Complex routing needs |

### Consistent Hashing Implementation
```
Hash ring with virtual nodes:
- Each physical node → 100-200 virtual nodes
- hash(key) → find next clockwise node
- Adding node: only affects adjacent keys (~1/N data moved)
- Removing node: data moves to next clockwise node

Virtual nodes solve uneven distribution problem.
```

---

## 6. Concurrency Control

### Questions to Ask First
- What's the contention level? (Low: <1%, High: >10% conflicts)
- What's the cost of a failed operation? (Retry OK? Lost sale?)
- Do operations need strict ordering?
- Single database or distributed system?
- Is it a fixed inventory (tickets) or dynamic (likes)?

### Key Concepts
| Level | Contention | Strategy | Example |
|-------|------------|----------|---------|
| **Low** | <1% conflicts | Optimistic | User profile updates |
| **Medium** | 1-10% conflicts | Optimistic + retry | Shopping cart |
| **High** | >10% conflicts | Pessimistic or Queue | Flash sale, auction |
| **Critical** | Must not fail | Linearization | Ticket assignment |

```plantuml
@startuml
skinparam backgroundColor #FEFEFE
skinparam defaultFontSize 14
skinparam ActivityBackgroundColor #E3F2FD
skinparam ActivityBorderColor #1976D2

title **Concurrency Control Selection**

start
:What's your concurrency scenario?;

if (Fixed inventory (tickets, coupons)?) then (Yes)
    #C8E6C9:Use **Linearization**;
    note right
      Pre-create all rows
      UPDATE WHERE status='free' LIMIT 1
      DB handles concurrency
      Ticketing, Flash sales
    end note
    stop
else (Not fixed inventory)
endif

if (Strict ordering required?) then (Yes)
    #C8E6C9:Use **Queue Serialization**;
    note right
      All writes through queue
      Single consumer
      Auctions (bid order matters)
    end note
    stop
else (Ordering not critical)
endif

if (Low contention expected?) then (Yes)
    #C8E6C9:Use **Optimistic Locking**;
    note right
      UPDATE WHERE version = X
      Retry on conflict
      URL shortener collisions
    end note
    stop
else (High contention)
endif

if (Critical section, must not fail?) then (Yes)
    #C8E6C9:Use **Pessimistic Locking**;
    note right
      SELECT FOR UPDATE
      Lock row during transaction
      Bank transfers
    end note
    stop
else (Can tolerate some failures)
endif

if (Distributed system?) then (Yes)
    if (Need strong consistency?) then (Yes)
        #C8E6C9:Use **Distributed Lock (Redlock)**;
        note right
          Redis-based
          Leader election
          Expensive
        end note
        stop
    else (Eventual OK)
        #C8E6C9:Use **Optimistic + Retry**;
        stop
    endif
else (Single node)
endif

#C8E6C9:Use **Optimistic Locking** (default);
stop

@enduml
```

### Trade-offs Deep Dive

| Strategy | Throughput | Fairness | Complexity | Failure Mode |
|----------|------------|----------|------------|--------------|
| **Optimistic** | High | First to commit wins | Low | Retry storms under high contention |
| **Pessimistic** | Lower | Fair (FIFO) | Medium | Deadlocks, lock timeouts |
| **Linearization** | Highest | DB decides | Low | None (DB handles it) |
| **Queue Serialization** | Limited by consumer | Perfect (FIFO) | Medium | Queue backup |
| **Distributed Lock** | Lower | Depends | High | Split-brain, lock expiry |

### Implementation Patterns

**Optimistic Locking (JPA/Hibernate)**
```java
@Entity
public class Account {
    @Version
    private Long version;
    
    private BigDecimal balance;
}

// Throws OptimisticLockException on conflict
// Caller must catch and retry
```

**Pessimistic Locking**
```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT a FROM Account a WHERE a.id = :id")
Account findByIdForUpdate(@Param("id") Long id);

// Blocks other transactions until commit/rollback
// Set timeout: @QueryHint(name = "javax.persistence.lock.timeout", value = "3000")
```

**Linearization for Tickets**
```sql
-- Pre-create all tickets with status='available'
-- Atomic claim:
UPDATE tickets 
SET status = 'claimed', user_id = ?, claimed_at = NOW()
WHERE event_id = ? AND status = 'available'
LIMIT 1;

-- If affected_rows = 1: success
-- If affected_rows = 0: sold out
```

---

## 7. Load Balancing

### Questions to Ask First
- Do you need to inspect HTTP content (headers, cookies, URLs)?
- Is session affinity required?
- Are servers homogeneous or different capacities?
- Single region or global distribution?
- What's the health check strategy?

### Key Numbers to Know
| Algorithm | When to Use | Overhead |
|-----------|-------------|----------|
| **Round Robin** | Homogeneous servers, stateless | Lowest |
| **Weighted Round Robin** | Mixed capacity servers | Low |
| **Least Connections** | Variable request durations | Medium |
| **IP Hash** | Session affinity needed | Low |
| **Consistent Hash** | Cache servers | Medium |

```plantuml
@startuml
skinparam backgroundColor #FEFEFE
skinparam defaultFontSize 14
skinparam ActivityBackgroundColor #E3F2FD
skinparam ActivityBorderColor #1976D2

title **Load Balancing Strategy Selection**

start
:What are your requirements?;

if (Need to inspect HTTP headers/URLs?) then (Yes)
    #C8E6C9:Use **Layer 7 (Application)**;
    note right
      Can route by URL path
      Can route by header
      Can do A/B testing
    end note
else (Just need traffic distribution)
    #C8E6C9:Use **Layer 4 (Transport)**;
    note right
      Faster (no inspection)
      Route by IP/port only
    end note
endif

:Choose algorithm:;

if (Need session affinity?) then (Yes)
    #C8E6C9:Use **Hash-Based**;
    note right
      hash(user_id) % N
      Same user -> same server
      Good for caching
    end note
    stop
else (No stickiness needed)
endif

if (Servers have different capacity?) then (Yes)
    #C8E6C9:Use **Weighted Round Robin**;
    note right
      Assign weights by capacity
      Powerful servers get more
    end note
    stop
else (Homogeneous servers)
endif

if (Request times vary significantly?) then (Yes)
    #C8E6C9:Use **Least Connections**;
    note right
      Route to least busy
      Adapts to slow requests
    end note
    stop
else (Uniform request times)
endif

if (Global users?) then (Yes)
    #C8E6C9:Use **Geographic / Latency-Based**;
    note right
      Route to nearest DC
      Lowest latency
    end note
    stop
else (Single region)
endif

#C8E6C9:Use **Round Robin** (default);
note right
  Simple, effective
  Equal distribution
end note
stop

@enduml
```

### Trade-offs Deep Dive

| Type | Latency | Features | Use Case |
|------|---------|----------|----------|
| **L4 (TCP/UDP)** | Lower (~1ms) | IP/port routing only | High throughput, simple routing |
| **L7 (HTTP)** | Higher (~5ms) | URL routing, header inspection, SSL termination | A/B testing, API routing |

### Health Check Strategies
1. **TCP Check**: Port open? (Fast but shallow)
2. **HTTP Check**: Returns 200? (Better but still shallow)
3. **Deep Health Check**: `/health` endpoint checks DB, cache, dependencies (Best but slower)

**Tip**: Use shallow checks frequently (5s), deep checks less often (30s).

---

## 8. Message Queue Selection

### Questions to Ask First
- Do you need message replay/audit trail?
- What's the throughput requirement?
- Is ordering important? (FIFO, per-partition, none)
- What's the delivery guarantee? (At-least-once, exactly-once)
- Complex routing needed?
- Managed service or self-hosted?

### Key Numbers to Know
| Queue | Throughput | Latency | Retention | Ordering |
|-------|------------|---------|-----------|----------|
| **Kafka** | 1M+ msgs/sec | 2-10ms | Days-forever | Per-partition |
| **RabbitMQ** | 20K msgs/sec | <1ms | Until consumed | Per-queue |
| **SQS** | Unlimited (managed) | 10-20ms | 14 days max | FIFO optional |
| **Redis Pub/Sub** | 100K+ msgs/sec | <1ms | None | None |

```plantuml
@startuml
skinparam backgroundColor #FEFEFE
skinparam defaultFontSize 14
skinparam ActivityBackgroundColor #E3F2FD
skinparam ActivityBorderColor #1976D2

title **Message Queue Selection**

start
:What's your messaging need?;

if (Need event replay/audit trail?) then (Yes)
    #C8E6C9:Use **Kafka**;
    note right
      Persistent log
      Replay from any point
      100K+ msgs/sec
      Event sourcing, CDC
    end note
    stop
else (No replay needed)
endif

if (Need complex routing?) then (Yes)
    #C8E6C9:Use **RabbitMQ**;
    note right
      Exchanges, bindings
      Topic routing
      Request-reply pattern
    end note
    stop
else (Simple routing)
endif

if (Want managed service?) then (Yes)
    if (AWS?) then (Yes)
        #C8E6C9:Use **SQS + SNS**;
        note right
          SQS for queues
          SNS for pub/sub
          Fully managed
        end note
        stop
    else (Multi-cloud)
        #C8E6C9:Use **Managed Kafka (Confluent)**;
        stop
    endif
else (Self-managed OK)
endif

if (Real-time, fire-and-forget?) then (Yes)
    #C8E6C9:Use **Redis Pub/Sub**;
    note right
      Very fast
      No persistence
      Live notifications
    end note
    stop
else (Need durability)
endif

if (High throughput critical?) then (Yes)
    #C8E6C9:Use **Kafka**;
    stop
else (Moderate throughput)
    #C8E6C9:Use **RabbitMQ** or **SQS**;
    stop
endif

@enduml
```

### Trade-offs Deep Dive

| Queue | Strengths | Weaknesses | Best For |
|-------|-----------|------------|----------|
| **Kafka** | Throughput, replay, scaling | Operational complexity, overkill for simple cases | Event streaming, CDC, analytics |
| **RabbitMQ** | Flexible routing, mature, low latency | Lower throughput, no replay | Task queues, RPC |
| **SQS** | Fully managed, scales infinitely | 256KB limit, no replay, AWS lock-in | Simple async jobs |
| **Redis Pub/Sub** | Blazing fast, simple | No persistence, no acknowledgment | Real-time notifications |

### Delivery Guarantees
| Guarantee | How | Trade-off |
|-----------|-----|-----------|
| **At-most-once** | Fire and forget | May lose messages |
| **At-least-once** | Ack after processing | May duplicate (idempotency required) |
| **Exactly-once** | Transactional producer + consumer | Complex, lower throughput |

**Rule**: Design for at-least-once + idempotent consumers. Exactly-once is rarely worth the cost.

---

## 9. Fan-Out Strategy (Social/Feed Systems)

### Questions to Ask First
- What's the follower distribution? (All similar vs. celebrities)
- What's the read:write ratio?
- How many followers does the average user have?
- What's the latency requirement for feed reads?
- Can feeds be slightly stale?

### Key Numbers to Know
| Strategy | Write Cost | Read Cost | Storage |
|----------|------------|-----------|---------|
| **Fan-Out on Write (Push)** | O(followers) | O(1) | High (timeline per user) |
| **Fan-Out on Read (Pull)** | O(1) | O(followees) | Low |
| **Hybrid** | O(regular followers) | O(celebrity followees) | Medium |

**Twitter Stats** (for reference):
- Average user: 200-500 followers
- Celebrities: 1M-100M followers
- Read:write ratio: ~300:1

```plantuml
@startuml
skinparam backgroundColor #FEFEFE
skinparam defaultFontSize 14
skinparam ActivityBackgroundColor #E3F2FD
skinparam ActivityBorderColor #1976D2

title **Fan-Out Strategy Selection**

start
:What's your user distribution?;

if (All users have similar follower counts?) then (Yes)
    if (< 1000 followers average?) then (Yes)
        #C8E6C9:Use **Fan-Out on Write (Push)**;
        note right
          Pre-compute timelines
          O(1) reads
          Write amplification OK
        end note
        stop
    else (Many followers)
        #C8E6C9:Use **Fan-Out on Read (Pull)**;
        note right
          Compute at read time
          O(N) reads
          No write amplification
        end note
        stop
    endif
else (Mix of regular and VIP users)
endif

if (Have celebrities (1M+ followers)?) then (Yes)
    #C8E6C9:Use **Hybrid Fan-Out**;
    note right
      Push for regular users
      Pull for VIPs
      Best of both worlds
      Twitter/Instagram approach
    end note
    stop
else (No celebrities)
endif

if (Read-heavy (30:1 ratio)?) then (Yes)
    #C8E6C9:Use **Fan-Out on Write**;
    note right
      Optimize for reads
      Accept write cost
    end note
    stop
else (Write-heavy)
    #C8E6C9:Use **Fan-Out on Read**;
    note right
      Minimize write cost
      Accept read latency
    end note
    stop
endif

@enduml
```

### Hybrid Fan-Out Implementation
```
On post:
1. Check if author is VIP (>10K followers)
2. If regular user:
   - Fan-out to all followers' timelines (async, via queue)
3. If VIP:
   - Store post in author's posts table only
   - Do NOT fan-out

On timeline read:
1. Fetch user's pre-computed timeline (from fan-out)
2. Fetch posts from followed VIPs (pull)
3. Merge and sort by timestamp
4. Cache the merged result (short TTL)
```

---

## 10. ID Generation

### Questions to Ask First
- Do IDs need to be sortable by time?
- Is coordination between services possible?
- What's the required uniqueness scope? (Per table, global, universal)
- Are there size constraints? (URL length, storage)
- Do you need to embed metadata (datacenter, type)?

### Key Numbers to Know
| Strategy | Size | Sortable | Coordination | Collisions |
|----------|------|----------|--------------|------------|
| **UUID v4** | 128-bit (36 chars) | No | None | ~10^-37 |
| **ULID** | 128-bit (26 chars) | Yes | None | Per-ms: 2^80 |
| **Snowflake** | 64-bit (19 digits) | Yes | Clock sync | None if configured right |
| **Auto-increment** | 64-bit | Yes | Single DB | None |

```plantuml
@startuml
skinparam backgroundColor #FEFEFE
skinparam defaultFontSize 14
skinparam ActivityBackgroundColor #E3F2FD
skinparam ActivityBorderColor #1976D2

title **ID Generation Strategy Selection**

start
:What are your ID requirements?;

if (Need globally unique, no coordination?) then (Yes)
    #C8E6C9:Use **UUID v4**;
    note right
      128-bit random
      No central service
      Large storage (36 chars)
    end note
    stop
else (Can coordinate)
endif

if (Need sortable by time?) then (Yes)
    if (Need custom metadata in ID?) then (Yes)
        #C8E6C9:Use **Snowflake ID**;
        note right
          64-bit: timestamp + machine + sequence
          Time-sortable
          Twitter, Discord style
        end note
        stop
    else (Just time-sortable)
        #C8E6C9:Use **ULID** or **UUID v7**;
        note right
          Lexicographically sortable
          Timestamp prefix
        end note
        stop
    endif
else (No time requirements)
endif

if (Need short, URL-friendly?) then (Yes)
    #C8E6C9:Use **Base62/Base58 encoded**;
    note right
      Short codes: abc123
      URL shorteners
      Human-readable
    end note
    stop
else (Length doesn't matter)
endif

if (Need sequential for DB performance?) then (Yes)
    #C8E6C9:Use **Auto-Increment** or **Snowflake**;
    note right
      B-tree friendly
      Better insert performance
      Single point for auto-inc
    end note
    stop
else (No sequence needed)
endif

#C8E6C9:Use **UUID v4** (default);
note right
  Simple, universal
  No coordination
end note
stop

@enduml
```

### Snowflake ID Structure
```
64 bits total:
┌────────────────────────────────────────────────────────────────────────────────┐
│ 0 │ 41 bits: timestamp (ms since epoch) │ 10 bits: machine │ 12 bits: sequence │
└────────────────────────────────────────────────────────────────────────────────┘

- Timestamp: ~69 years from custom epoch
- Machine ID: 1024 machines max (split: 5 datacenter + 5 machine)
- Sequence: 4096 IDs per machine per millisecond
- Total: 4M IDs/sec per datacenter

Implementation considerations:
- Clock skew: Reject if clock goes backward
- Machine ID assignment: Zookeeper, config, or derive from IP
```

### Trade-offs Deep Dive

| Strategy | Strengths | Weaknesses | When NOT to Use |
|----------|-----------|------------|-----------------|
| **UUID v4** | No coordination, universally unique | Large, not sortable, poor index locality | Need sorting, space-constrained |
| **Snowflake** | Sortable, compact, high throughput | Clock dependency, machine ID management | Simple systems, no sorting need |
| **ULID** | Sortable, no coordination, URL-safe | Slightly larger than Snowflake | Need custom metadata in ID |
| **Auto-increment** | Simple, compact, perfect sorting | Single point of failure, predictable | Distributed systems, security-sensitive |

---

## 11. Consistency Model

### Questions to Ask First
- What happens if a user sees stale data? (Annoying? Costly? Dangerous?)
- Is this financial/transactional data?
- Single region or multi-region deployment?
- What's more important: availability or consistency?
- Can the application handle conflicts?

### CAP Theorem Refresher
```
You can only have 2 of 3:
- Consistency: All nodes see the same data at the same time
- Availability: Every request gets a response
- Partition tolerance: System works despite network failures

In distributed systems, partitions WILL happen, so you choose:
- CP: Consistency + Partition tolerance (refuse requests during partition)
- AP: Availability + Partition tolerance (serve stale data during partition)
```

```plantuml
@startuml
skinparam backgroundColor #FEFEFE
skinparam defaultFontSize 14
skinparam ActivityBackgroundColor #E3F2FD
skinparam ActivityBorderColor #1976D2

title **Consistency Model Selection**

start
:What's your consistency need?;

if (Financial transactions?) then (Yes)
    #C8E6C9:Use **Strong Consistency**;
    note right
      Every read sees latest write
      ACID transactions
      Bank transfers, Inventory
    end note
    stop
else (Not financial)
endif

if (Can tolerate stale reads?) then (Yes)
    if (How stale is OK?;) then (Seconds)
        #C8E6C9:Use **Eventual Consistency**;
        note right
          Replicas sync async
          Higher availability
          Social feeds, Analytics
        end note
        stop
    else (Milliseconds)
        #C8E6C9:Use **Read-Your-Writes**;
        note right
          User sees their own writes
          Others may see stale
          Profile updates
        end note
        stop
    endif
else (Need fresh reads)
endif

if (Need causal ordering?) then (Yes)
    #C8E6C9:Use **Causal Consistency**;
    note right
      Related events in order
      Comments after posts
      Chat messages
    end note
    stop
else (No ordering needs)
endif

if (Distributed system, need availability?) then (Yes)
    #C8E6C9:Use **Eventual Consistency**;
    note right
      CAP theorem: pick AP
      Sacrifice consistency
      DNS, CDN caches
    end note
    stop
else (Single region)
endif

#C8E6C9:Use **Strong Consistency** (default);
note right
  Start strict
  Relax if needed
end note
stop

@enduml
```

### Consistency Levels Explained

| Level | Guarantee | Latency | Use Case |
|-------|-----------|---------|----------|
| **Strong** | Latest write always visible | Higher | Banking, inventory |
| **Read-Your-Writes** | User sees their own writes | Medium | Profile updates |
| **Monotonic Reads** | Never see older data than before | Medium | Dashboard |
| **Causal** | Causally related ops in order | Medium | Comments, chat |
| **Eventual** | Will converge eventually | Lowest | Social feeds, analytics |

### Implementation Patterns

**Read-Your-Writes in Practice**:
```
Option 1: Sticky sessions
- Route user to same replica
- Simple but limits scaling

Option 2: Read from primary after write
- Write → set flag → read from primary for X seconds
- Falls back to replica after

Option 3: Version vectors
- Track version user has seen
- Read from replica only if version >= seen version
```

---

## 12. Authentication & Authorization

### Questions to Ask First
- First-party auth or delegating to third party (Google, etc.)?
- Stateless API or traditional web app?
- Do you need token revocation?
- Machine-to-machine or user-facing?
- What's the session duration?

### Key Concepts
| Term | Meaning |
|------|---------|
| **Authentication (AuthN)** | Who are you? (Identity verification) |
| **Authorization (AuthZ)** | What can you do? (Permissions) |
| **OAuth 2.0** | Authorization framework (delegation) |
| **OIDC** | OAuth 2.0 + identity layer |
| **JWT** | Self-contained token format |

```plantuml
@startuml
skinparam backgroundColor #FEFEFE
skinparam defaultFontSize 14
skinparam ActivityBackgroundColor #E3F2FD
skinparam ActivityBorderColor #1976D2

title **Authentication Strategy Selection**

start
:What's your auth scenario?;

if (3rd party login (Google, GitHub)?) then (Yes)
    #C8E6C9:Use **OAuth 2.0 / OIDC**;
    note right
      Delegate auth to provider
      Get user info via tokens
      Social login
    end note
    stop
else (Own auth system)
endif

if (Stateless API?) then (Yes)
    if (Need token revocation?) then (Yes)
        #C8E6C9:Use **JWT + Token Blacklist**;
        note right
          JWT for stateless
          Redis blacklist for revoke
        end note
        stop
    else (No revocation)
        #C8E6C9:Use **JWT only**;
        note right
          Self-contained tokens
          No session storage
          Set short expiry
        end note
        stop
    endif
else (Stateful OK)
endif

if (Traditional web app?) then (Yes)
    #C8E6C9:Use **Session Cookies**;
    note right
      Server-side sessions
      Redis for scaling
      Simpler security model
    end note
    stop
else (API-first)
endif

if (Machine-to-machine?) then (Yes)
    #C8E6C9:Use **API Keys** or **mTLS**;
    note right
      API keys for simple
      mTLS for high security
    end note
    stop
else (User-facing)
endif

#C8E6C9:Use **JWT with refresh tokens**;
note right
  Short-lived access token
  Long-lived refresh token
  Modern standard
end note
stop

@enduml
```

### JWT Structure
```
Header.Payload.Signature

Header: {"alg": "RS256", "typ": "JWT"}
Payload: {
  "sub": "user123",        // Subject (user ID)
  "iat": 1699900000,       // Issued at
  "exp": 1699903600,       // Expiration (1 hour)
  "roles": ["user", "admin"]
}
Signature: HMAC-SHA256(base64(header) + "." + base64(payload), secret)

Typical lifetimes:
- Access token: 15 min - 1 hour
- Refresh token: 7 - 30 days
```

### Trade-offs Deep Dive

| Strategy | Strengths | Weaknesses | Best For |
|----------|-----------|------------|----------|
| **Session Cookies** | Simple revocation, smaller payload | Server storage, scaling complexity | Traditional web apps |
| **JWT only** | Stateless, scalable | Can't revoke until expiry | Short-lived tokens, microservices |
| **JWT + Blacklist** | Revocable + stateless benefits | Blacklist check on every request | APIs needing revocation |
| **OAuth 2.0/OIDC** | Delegate to experts, social login | Complexity, dependency on provider | Consumer apps |
| **API Keys** | Simple, no expiry | Can't scope permissions easily | Internal services, simple cases |

---

## 13. Rate Limiting (BONUS)

### Questions to Ask First
- What resource are you protecting? (API, DB, downstream service)
- Per user, per IP, or global?
- Hard limit or soft (allow burst)?
- What happens when limit exceeded? (429, queue, degrade)

### Key Algorithms

| Algorithm | Behavior | Memory | Burst Handling |
|-----------|----------|--------|----------------|
| **Token Bucket** | Smooth + allows burst | O(1) per key | Good |
| **Leaky Bucket** | Perfectly smooth | O(1) per key | None |
| **Fixed Window** | Simple, boundary spikes | O(1) per key | Poor |
| **Sliding Window Log** | Accurate | O(N) per key | Good |
| **Sliding Window Counter** | Accurate, efficient | O(1) per key | Good |

### Token Bucket Implementation
```
Each user has a bucket:
- capacity: max tokens (burst size)
- tokens: current tokens
- refill_rate: tokens added per second
- last_refill: timestamp

On request:
1. Refill tokens based on time elapsed
2. If tokens >= 1: allow, decrement
3. Else: reject (429)

Redis implementation:
EVAL "
  local tokens = redis.call('get', KEYS[1])
  local last = redis.call('get', KEYS[2])
  -- refill logic
  -- check and decrement
" 2 bucket:user123:tokens bucket:user123:last
```

---

## Quick Reference Table

| Decision | Default Choice | When to Change |
|----------|---------------|----------------|
| **Database** | PostgreSQL | 100K+ ops/s → Redis; Flexible schema → MongoDB |
| **Protocol** | REST | Real-time → WebSockets; Internal → gRPC |
| **Cache** | Redis | Static assets → CDN; Hot config → Local |
| **Scaling** | Horizontal | Write bottleneck → Sharding |
| **Sharding** | Hash-based | Natural partitions → Entity-based |
| **Concurrency** | Optimistic | Fixed inventory → Linearization; Ordering → Queue |
| **Load Balancer** | Round Robin | Session affinity → Hash-based |
| **Queue** | SQS/RabbitMQ | Event sourcing → Kafka |
| **Fan-Out** | Push | VIPs present → Hybrid |
| **ID Generation** | UUID v4 | Time-sortable → Snowflake; Short → Base62 |
| **Consistency** | Strong | High availability → Eventual |
| **Auth** | JWT | Revocation needed → JWT + Blacklist |
| **Rate Limiting** | Token Bucket | Smooth output → Leaky Bucket |

---

## Back-of-Envelope Cheat Sheet

### Data Size Estimates
| Type | Size |
|------|------|
| UUID | 36 bytes (string) / 16 bytes (binary) |
| Timestamp | 8 bytes |
| Integer ID | 8 bytes |
| Short URL (7 chars) | 7 bytes |
| Tweet-like text | ~200 bytes |
| User profile (JSON) | ~1 KB |
| Image metadata | ~500 bytes |
| Image file | ~200 KB - 2 MB |

### Throughput Rules of Thumb
| Component | Capacity |
|-----------|----------|
| Single web server | 1K-10K RPS |
| PostgreSQL | 10K TPS |
| Redis | 100K ops/sec |
| Kafka partition | 10K msgs/sec |
| SSD random read | 100K IOPS |
| 1 Gbps network | 100 MB/s |

### Time Conversions
| Unit | Seconds |
|------|---------|
| 1 day | 86,400 |
| 1 week | 604,800 |
| 1 month | 2.6M |
| 1 year | 31.5M |

### Scale Estimates
| Metric | Value |
|--------|-------|
| 1M users, 10% DAU | 100K DAU |
| 100K DAU, 10 requests each | 1M requests/day |
| 1M requests/day | ~12 RPS average, ~100 RPS peak |
| 1 billion requests/month | ~400 RPS average |

---

*Enhanced for interview preparation. Based on patterns from URL Shortener, News Feed, Ticketing, Auction, Coupon, and Web Crawler system designs.*
