# System Design Decision Trees

> Quick decision guides for common system design choices.

---

## 1. Database Selection

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

---

## 2. Communication Protocol

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

---

## 3. Caching Strategy

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

---

## 4. Scaling Strategy

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

---

## 5. Sharding Strategy

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

---

## 6. Concurrency Control

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

---

## 7. Load Balancing

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

---

## 8. Message Queue Selection

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

---

## 9. Fan-Out Strategy (Social/Feed Systems)

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

---

## 10. ID Generation

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

---

## 11. Consistency Model

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

---

## 12. Authentication & Authorization

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

---

*Based on patterns from URL Shortener, News Feed, Ticketing, Auction, Coupon, and Web Crawler system designs.*
