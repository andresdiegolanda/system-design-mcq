# System Design Reference Architecture

> A comprehensive reference showing all major components and when to use each alternative.

---

## Complete Reference Architecture

```plantuml
@startuml
skinparam backgroundColor #FEFEFE
skinparam defaultFontSize 14
skinparam rectangleBorderThickness 2

title **Complete System Design Reference Architecture**

' === CLIENT LAYER ===
rectangle "CLIENT LAYER" as clientLayer #E3F2FD {
    actor "Web Browser" as browser
    actor "Mobile App" as mobile
    actor "3rd Party\nService" as thirdparty
}

' === EDGE LAYER ===
rectangle "EDGE LAYER" as edgeLayer #E8F5E9 {
    rectangle "CDN\n(CloudFront, Cloudflare)" as cdn #C8E6C9
    rectangle "DNS\n(Route 53)" as dns #A5D6A7
}

' === GATEWAY LAYER ===
rectangle "GATEWAY LAYER" as gatewayLayer #FFF3E0 {
    rectangle "API Gateway\n/ Load Balancer" as gateway #FFE0B2 {
        card "Rate Limiting" as rate #FFCC80
        card "Auth / JWT" as auth #FFCC80
        card "SSL Termination" as ssl #FFCC80
    }
}

' === SERVICE LAYER ===
rectangle "SERVICE LAYER" as serviceLayer #F3E5F5 {
    rectangle "Stateless\nMicroservices" as services #E1BEE7 {
        rectangle "Service A" as svcA #CE93D8
        rectangle "Service B" as svcB #CE93D8
        rectangle "Service N" as svcN #CE93D8
    }

    rectangle "Real-Time\nServices" as realtime #E1BEE7 {
        rectangle "WebSocket\nServer" as ws #BA68C8
    }
}

' === ASYNC LAYER ===
rectangle "ASYNC LAYER" as asyncLayer #FFEBEE {
    queue "Message Queue\n(Kafka/RabbitMQ/SQS)" as queue #FFCDD2
    rectangle "Workers\n(Consumers)" as workers #EF9A9A
    rectangle "Scheduler\n(Cron Jobs)" as scheduler #E57373
}

' === CACHE LAYER ===
rectangle "CACHE LAYER" as cacheLayer #E0F7FA {
    database "In-Memory Cache\n(Redis Cluster)" as redis #B2EBF2
    database "Local Cache\n(Per Instance)" as localCache #80DEEA
}

' === DATA LAYER ===
rectangle "DATA LAYER" as dataLayer #FBE9E7 {
    database "Primary DB\n(PostgreSQL)" as primaryDb #FFCCBC
    database "Read Replicas" as replicas #FFAB91
    database "NoSQL\n(MongoDB/DynamoDB)" as nosql #FF8A65
    database "Search\n(Elasticsearch)" as search #FF7043
}

' === STORAGE LAYER ===
rectangle "STORAGE LAYER" as storageLayer #F1F8E9 {
    database "Object Storage\n(S3/GCS)" as objectStorage #DCEDC8
    database "File System\n(HDFS)" as fileSystem #C5E1A5
}

' === EXTERNAL LAYER ===
rectangle "EXTERNAL SERVICES" as externalLayer #ECEFF1 {
    rectangle "Payment\n(Stripe/PayPal)" as payment #CFD8DC
    rectangle "Email/SMS\n(SendGrid/Twilio)" as notifications #B0BEC5
    rectangle "Analytics\n(Mixpanel)" as analytics #90A4AE
}

' === MONITORING ===
rectangle "OBSERVABILITY" as obsLayer #FCE4EC {
    rectangle "Metrics\n(Prometheus)" as metrics #F8BBD9
    rectangle "Logging\n(ELK Stack)" as logging #F48FB1
    rectangle "Tracing\n(Jaeger)" as tracing #EC407A
}

' === CONNECTIONS ===

' Client to Edge
browser --> dns
mobile --> dns
dns --> cdn : Static assets
dns --> gateway : API calls

' Edge to Gateway
cdn --> objectStorage : Origin
thirdparty --> gateway : Webhooks

' Gateway to Services
gateway --> services : REST/gRPC
gateway --> ws : Upgrade to WS

' Services to Cache
svcA --> redis
svcB --> redis
svcA --> localCache

' Services to Data
svcA --> primaryDb : Writes
svcB --> replicas : Reads
svcN --> nosql
svcN --> search : Full-text

' Services to Async
svcA --> queue : Publish events
queue --> workers : Consume
scheduler --> queue : Scheduled jobs

' Workers to Data
workers --> primaryDb
workers --> nosql
workers --> notifications

' External integrations
svcB --> payment
payment --> gateway : Webhook callback
workers --> analytics

' Replication
primaryDb --> replicas : Async replication

' Cache sync
primaryDb ..> redis : Cache invalidation

' Monitoring (dotted - passive)
services ..> metrics
services ..> logging
services ..> tracing

@enduml
```

---

## Layer-by-Layer Breakdown

### 1. Edge Layer (CDN + DNS)

```plantuml
@startuml
skinparam backgroundColor #FEFEFE
skinparam defaultFontSize 14

rectangle "EDGE LAYER DECISIONS" as edge #E8F5E9 {

    rectangle "DNS (Route 53, Cloudflare DNS)" as dns #C8E6C9 {
        card "**Purpose:** Domain -> IP resolution" as d1
        card "**Latency-based:** Route to nearest DC" as d2
        card "**Failover:** Health checks, automatic reroute" as d3
        card "**TTL:** Balance freshness vs DNS load" as d4
    }

    rectangle "CDN (CloudFront, Cloudflare, Akamai)" as cdn #A5D6A7 {
        card "**Purpose:** Cache static assets at edge" as c1
        card "**Reduces:** Latency, origin load, bandwidth costs" as c2
        card "**Cache:** Images, JS, CSS, videos" as c3
        card "**Don't cache:** User-specific data, real-time data" as c4
    }
}

note bottom of edge
  **When to use CDN:**
  - Static assets (images, JS, CSS)
  - Public content (landing pages)
  - Video streaming

  **When to skip CDN:**
  - Dynamic per-user content
  - Real-time data
  - Internal APIs
end note

@enduml
```

---

### 2. API Gateway / Load Balancer

```plantuml
@startuml
skinparam backgroundColor #FEFEFE
skinparam defaultFontSize 14

rectangle "GATEWAY RESPONSIBILITIES" as gw #FFF3E0 {

    rectangle "Load Balancing" as lb #FFE0B2 {
        card "**Round Robin:** Equal distribution" as lb1
        card "**Least Connections:** Route to least busy" as lb2
        card "**Hash-based:** Sticky sessions by user_id" as lb3
        card "**Geographic:** Route to nearest region" as lb4
    }

    rectangle "Security" as sec #FFCC80 {
        card "**SSL Termination:** HTTPS -> HTTP internally" as s1
        card "**Authentication:** Validate JWT/OAuth tokens" as s2
        card "**Rate Limiting:** Protect from abuse" as s3
        card "**WAF:** Block malicious requests" as s4
    }

    rectangle "Request Handling" as req #FFB74D {
        card "**Routing:** URL -> Service mapping" as r1
        card "**Protocol Translation:** REST, gRPC, GraphQL" as r2
        card "**Request/Response Transform:** Add headers" as r3
        card "**Circuit Breaker:** Fail fast on outages" as r4
    }
}

note bottom of gw
  **L4 vs L7 Load Balancer:**
  - L4 (Transport): Fast, route by IP/port only
  - L7 (Application): Can inspect headers, URLs, cookies

  **Use L7 when you need:**
  - URL-based routing (/api/v1 vs /api/v2)
  - User-based sharding (hash user_id)
  - A/B testing (route % of traffic)
end note

@enduml
```

---

### 3. Communication Protocols

```plantuml
@startuml
skinparam backgroundColor #FEFEFE
skinparam defaultFontSize 14

rectangle "COMMUNICATION PROTOCOL SELECTION" as comm #E3F2FD {

    rectangle "REST / HTTP" as rest #BBDEFB {
        card "**Model:** Request/Response, stateless" as r1
        card "**Format:** JSON over HTTP" as r2
        card "**Caching:** HTTP cache headers" as r3
        card "**Throughput:** ~10K req/s per instance" as r4
    }

    rectangle "WebSockets" as ws #90CAF9 {
        card "**Model:** Bidirectional, persistent connection" as w1
        card "**Server push:** No polling needed" as w2
        card "**Overhead:** Lower after handshake" as w3
        card "**State:** Connection is stateful" as w4
    }

    rectangle "gRPC" as grpc #64B5F6 {
        card "**Model:** RPC with Protocol Buffers" as g1
        card "**Performance:** Binary, faster than JSON" as g2
        card "**Streaming:** Bidirectional streaming" as g3
        card "**Use for:** Internal service-to-service" as g4
    }

    rectangle "Webhooks" as hooks #42A5F5 {
        card "**Model:** Server-to-server callbacks" as h1
        card "**Trigger:** Event-driven (payment complete)" as h2
        card "**Direction:** External -> Your system" as h3
        card "**Idempotency:** Must handle duplicates" as h4
    }
}

@enduml
```

#### When to Use Each Protocol

| Protocol | Use When | Examples |
|----------|----------|----------|
| **REST** | Standard CRUD, public APIs, browser clients | URL Shortener, Coupon system |
| **WebSockets** | Real-time bidirectional, live updates | Auctions, Chat, Live sports |
| **gRPC** | Internal microservices, high throughput | Service mesh, ML inference |
| **Webhooks** | 3rd party integrations, event notifications | Payment confirmation, GitHub events |
| **Polling** | Simple, firewall-friendly, fallback | Payment status check, legacy systems |

---

### 4. Database Selection

```plantuml
@startuml
skinparam backgroundColor #FEFEFE
skinparam defaultFontSize 14

rectangle "DATABASE SELECTION GUIDE" as db #FBE9E7 {

    rectangle "Relational (PostgreSQL, MySQL)" as sql #FFCCBC {
        card "**ACID:** Full transactions" as sq1
        card "**Schema:** Structured, enforced" as sq2
        card "**Queries:** Complex JOINs, aggregations" as sq3
        card "**Scale:** 10-50K writes/s, vertical first" as sq4
    }

    rectangle "Key-Value (Redis, DynamoDB)" as kv #FFAB91 {
        card "**Speed:** 100K+ ops/s" as kv1
        card "**Access:** GET/SET by key only" as kv2
        card "**Schema:** None (schemaless)" as kv3
        card "**Scale:** Horizontal, easy sharding" as kv4
    }

    rectangle "Document (MongoDB)" as doc #FF8A65 {
        card "**Schema:** Flexible, nested JSON" as d1
        card "**Queries:** Rich queries within documents" as d2
        card "**Use for:** Varied structure, rapid iteration" as d3
        card "**Caution:** No JOINs, denormalize" as d4
    }

    rectangle "Wide-Column (Cassandra)" as wide #FF7043 {
        card "**Writes:** Excellent write throughput" as w1
        card "**Scale:** Massive horizontal scale" as w2
        card "**Consistency:** Tunable (eventual default)" as w3
        card "**Use for:** Time-series, logs, IoT" as w4
    }

    rectangle "Search (Elasticsearch)" as search #FF5722 {
        card "**Full-text:** Natural language search" as s1
        card "**Facets:** Aggregations, filters" as s2
        card "**Speed:** Sub-second on millions of docs" as s3
        card "**Use for:** Product search, logs, analytics" as s4
    }
}

@enduml
```

#### Database Decision Tree

```plantuml
@startuml
skinparam backgroundColor #FEFEFE
skinparam defaultFontSize 14

start
:What's your primary need?;

if (Need ACID transactions?) then (Yes)
    :Use **PostgreSQL/MySQL**;
    note right: Banking, Orders, Inventory
    stop
else (No)
endif

if (Need 100K+ ops/sec?) then (Yes)
    if (Simple key lookups?) then (Yes)
        :Use **Redis/DynamoDB**;
        note right: Caching, Sessions, Counters
        stop
    else (Complex queries)
        :Use **MongoDB** or **Cassandra**;
        stop
    endif
else (No)
endif

if (Need full-text search?) then (Yes)
    :Use **Elasticsearch**;
    note right: Product search, Log analysis
    stop
else (No)
endif

if (Need flexible schema?) then (Yes)
    :Use **MongoDB**;
    note right: CMS, User profiles
    stop
else (No)
endif

if (Time-series / Write-heavy?) then (Yes)
    :Use **Cassandra** or **TimescaleDB**;
    note right: Metrics, IoT, Logs
    stop
else (No)
endif

:Default to **PostgreSQL**;
note right: Most versatile choice
stop

@enduml
```

---

### 5. Caching Strategy

```plantuml
@startuml
skinparam backgroundColor #FEFEFE
skinparam defaultFontSize 14

rectangle "MULTI-TIER CACHING" as cache #E0F7FA {

    rectangle "Tier 1: Local Cache (In-Process)" as local #B2EBF2 {
        card "**Latency:** Nanoseconds" as l1
        card "**Size:** 1-4 GB per instance" as l2
        card "**Scope:** Single instance only" as l3
        card "**Use for:** Hot config, compiled templates" as l4
    }

    rectangle "Tier 2: Distributed Cache (Redis)" as redis #80DEEA {
        card "**Latency:** Microseconds (network)" as r1
        card "**Size:** 10-100+ GB cluster" as r2
        card "**Scope:** Shared across instances" as r3
        card "**Use for:** Sessions, API responses, hot data" as r4
    }

    rectangle "Tier 3: CDN Edge Cache" as cdn #4DD0E1 {
        card "**Latency:** Milliseconds (edge proximity)" as c1
        card "**Size:** Virtually unlimited" as c2
        card "**Scope:** Global edge locations" as c3
        card "**Use for:** Static assets, public pages" as c4
    }
}

note bottom of cache
  **The 1% Rule:**
  Only ~1% of data is frequently accessed
  Cache the hot 1%, not everything

  **Example:**
  3TB total data = 30GB cache needed
end note

@enduml
```

#### Cache Patterns

| Pattern | How It Works | Use When |
|---------|--------------|----------|
| **Cache-Aside** | App checks cache, if miss -> query DB -> populate cache | Most common, flexible |
| **Write-Through** | Write to cache, cache writes to DB synchronously | Need consistency |
| **Write-Behind** | Write to cache, async batch write to DB | High write throughput |
| **Read-Through** | Cache fetches from DB on miss automatically | Simpler app code |

---

### 6. Message Queues & Async Processing

```plantuml
@startuml
skinparam backgroundColor #FEFEFE
skinparam defaultFontSize 14

rectangle "ASYNC PROCESSING PATTERNS" as async #FFEBEE {

    rectangle "Point-to-Point Queue" as p2p #FFCDD2 {
        card "**Delivery:** Each message to ONE consumer" as p1
        card "**Use for:** Task distribution, work queues" as p2
        card "**Example:** Process uploaded images" as p3
        card "**Tech:** SQS, RabbitMQ" as p4
    }

    rectangle "Publish/Subscribe" as pubsub #EF9A9A {
        card "**Delivery:** Each message to ALL subscribers" as ps1
        card "**Use for:** Event broadcasting, fanout" as ps2
        card "**Example:** Order placed -> notify 5 services" as ps3
        card "**Tech:** SNS, Redis Pub/Sub" as ps4
    }

    rectangle "Event Streaming" as stream #E57373 {
        card "**Delivery:** Ordered log, replayable" as s1
        card "**Use for:** Event sourcing, audit trail" as s2
        card "**Example:** All user actions, CDC" as s3
        card "**Tech:** Kafka, Kinesis" as s4
    }
}

@enduml
```

#### Queue Technology Comparison

| Technology | Throughput | Ordering | Durability | Best For |
|------------|------------|----------|------------|----------|
| **Kafka** | 100K+ msg/s | Per-partition | Persistent log | Event streaming, logs |
| **RabbitMQ** | 20K msg/s | Per-queue | Configurable | Complex routing, RPC |
| **SQS** | 3K msg/s | Best-effort | Managed | Simple async tasks |
| **Redis Pub/Sub** | 100K+ msg/s | None | None (fire-forget) | Real-time notifications |

---

### 7. Scaling Patterns

```plantuml
@startuml
skinparam backgroundColor #FEFEFE
skinparam defaultFontSize 14

rectangle "SCALING STRATEGIES" as scale #F3E5F5 {

    rectangle "Vertical Scaling (Scale Up)" as vertical #E1BEE7 {
        card "**How:** Bigger machine (more CPU, RAM)" as v1
        card "**Pros:** Simple, no code changes" as v2
        card "**Cons:** Hardware limits, expensive, downtime" as v3
        card "**When:** Quick fix, small scale" as v4
    }

    rectangle "Horizontal Scaling (Scale Out)" as horizontal #CE93D8 {
        card "**How:** More machines behind load balancer" as h1
        card "**Pros:** Unlimited scale, fault tolerant" as h2
        card "**Cons:** Complexity, stateless requirement" as h3
        card "**When:** Production systems, >10K RPS" as h4
    }

    rectangle "Read Replicas" as replicas #BA68C8 {
        card "**How:** Copy data to read-only replicas" as r1
        card "**Pros:** Simple, linear read scaling" as r2
        card "**Cons:** Replication lag, writes don't scale" as r3
        card "**When:** Read-heavy workloads (30:1 ratio)" as r4
    }

    rectangle "Sharding" as sharding #AB47BC {
        card "**How:** Partition data by shard key" as s1
        card "**Pros:** Scales reads AND writes" as s2
        card "**Cons:** Complex, cross-shard queries hard" as s3
        card "**When:** >50K writes/s, multi-TB data" as s4
    }
}

@enduml
```

#### Sharding Strategies

| Strategy | How It Works | Best For |
|----------|--------------|----------|
| **Hash-based** | `hash(key) % N` shards | Even distribution, no hotspots |
| **Range-based** | Key ranges per shard | Time-series, range scans |
| **Geographic** | By region/country | Low latency, data residency |
| **Directory** | Lookup table for routing | Maximum flexibility |

---

### 8. Concurrency Control

```plantuml
@startuml
skinparam backgroundColor #FEFEFE
skinparam defaultFontSize 14

rectangle "CONCURRENCY SOLUTIONS" as conc #FFEBEE {

    rectangle "Optimistic Locking" as opt #FFCDD2 {
        card "**How:** UPDATE WHERE version = expected" as o1
        card "**On conflict:** Retry with new version" as o2
        card "**Pros:** No locks, good for low contention" as o3
        card "**Use for:** URL shortener collisions" as o4
    }

    rectangle "Pessimistic Locking" as pess #EF9A9A {
        card "**How:** SELECT FOR UPDATE (lock row)" as p1
        card "**Blocking:** Other transactions wait" as p2
        card "**Pros:** Guarantees exclusive access" as p3
        card "**Use for:** Bank transfers, critical sections" as p4
    }

    rectangle "Queue Serialization" as queue #E57373 {
        card "**How:** Route all writes through queue" as q1
        card "**Single consumer:** Processes one at a time" as q2
        card "**Pros:** Strict ordering guaranteed" as q3
        card "**Use for:** Auction bids (order matters)" as q4
    }

    rectangle "Linearization" as linear #EF5350 {
        card "**How:** Pre-create rows, atomic UPDATE LIMIT 1" as l1
        card "**No read-then-write:** Database handles it" as l2
        card "**Pros:** Simple, horizontally scalable" as l3
        card "**Use for:** Ticketing (fixed inventory)" as l4
    }
}

@enduml
```

---

### 9. External Integrations

```plantuml
@startuml
skinparam backgroundColor #FEFEFE
skinparam defaultFontSize 14

rectangle "EXTERNAL SERVICE PATTERNS" as ext #ECEFF1 {

    rectangle "Payment Integration" as payment #CFD8DC {
        card "**Pattern:** Redirect -> Webhook callback" as pay1
        card "**Idempotency:** Use idempotency keys" as pay2
        card "**Retry:** Handle duplicate webhooks" as pay3
        card "**Providers:** Stripe, PayPal, Adyen" as pay4
    }

    rectangle "Notification Services" as notif #B0BEC5 {
        card "**Email:** SendGrid, SES, Mailgun" as n1
        card "**SMS:** Twilio, SNS" as n2
        card "**Push:** Firebase, APNs" as n3
        card "**Pattern:** Queue -> Worker -> Provider" as n4
    }

    rectangle "3rd Party APIs" as api #90A4AE {
        card "**Circuit Breaker:** Fail fast when down" as a1
        card "**Retry with backoff:** Exponential delays" as a2
        card "**Timeout:** Don't wait forever" as a3
        card "**Fallback:** Graceful degradation" as a4
    }
}

@enduml
```

---

### 10. Observability Stack

```plantuml
@startuml
skinparam backgroundColor #FEFEFE
skinparam defaultFontSize 14

rectangle "THREE PILLARS OF OBSERVABILITY" as obs #FCE4EC {

    rectangle "Metrics (Prometheus + Grafana)" as metrics #F8BBD9 {
        card "**What:** Numeric measurements over time" as m1
        card "**Examples:** RPS, latency p99, error rate" as m2
        card "**Alerts:** Threshold-based notifications" as m3
        card "**Dashboard:** Real-time visualization" as m4
    }

    rectangle "Logging (ELK Stack)" as logs #F48FB1 {
        card "**What:** Structured event records" as l1
        card "**Examples:** Errors, audit trail, debug" as l2
        card "**Search:** Full-text across all services" as l3
        card "**Correlation:** Request ID across services" as l4
    }

    rectangle "Tracing (Jaeger, Zipkin)" as trace #EC407A {
        card "**What:** Request flow across services" as t1
        card "**Spans:** Time spent in each service" as t2
        card "**Bottlenecks:** Identify slow dependencies" as t3
        card "**Debug:** Visualize distributed call chain" as t4
    }
}

note bottom of obs
  **Golden Signals (Google SRE):**
  1. Latency - How long requests take
  2. Traffic - How many requests
  3. Errors - Rate of failures
  4. Saturation - How "full" the system is
end note

@enduml
```

---

## Quick Reference: When to Use What

### Communication

| Need | Solution |
|------|----------|
| Standard API | REST |
| Real-time updates | WebSockets |
| High-perf internal | gRPC |
| 3rd party events | Webhooks |
| Simple fallback | Polling |

### Database

| Need | Solution |
|------|----------|
| Transactions (ACID) | PostgreSQL |
| 100K+ ops/sec | Redis / DynamoDB |
| Flexible schema | MongoDB |
| Time-series at scale | Cassandra |
| Full-text search | Elasticsearch |

### Scaling

| Bottleneck | Solution |
|------------|----------|
| Read-heavy | Read replicas + Cache |
| Write-heavy | Sharding |
| Compute | Horizontal scaling |
| Storage | Object storage (S3) |

### Concurrency

| Scenario | Solution |
|----------|----------|
| Low contention | Optimistic locking |
| High contention | Pessimistic locking |
| Ordering required | Queue serialization |
| Fixed inventory | Linearization |

---

## System Benchmarks (Memorize These)

| Component | Throughput |
|-----------|------------|
| PostgreSQL | 10-50K writes/sec |
| Redis | 100K+ ops/sec |
| Kafka | 100K+ msgs/sec per broker |
| Service instance | 1-10K RPS |
| SSD read | 200K+ IOPS |
| Network (1Gbps) | 100K+ small packets/sec |

---

*Based on patterns from: URL Shortener, News Feed, Ticketing, Auction, Coupon, and Web Crawler system designs.*
