# System Design Reference Architecture Poster

> **Format**: A2 Landscape (594mm x 420mm) | **Pages**: 10 | **Print**: Color recommended

---

# PAGE 1: CLIENT COMMUNICATION

```plantuml
@startuml
skinparam backgroundColor #FAFAFA
skinparam defaultFontSize 18
skinparam titleFontSize 28
skinparam rectangleBorderThickness 2

title **CLIENT LAYER - Communication Protocols**

rectangle "HTTP/REST" as http #E3F2FD {
    card "**Request/Response** model" as http1 #BBDEFB
    card "**Stateless** - no server-side session" as http2 #90CAF9
    card "**Cacheable** - responses can be cached" as http3 #64B5F6
    card "**Best for:** CRUD operations, APIs" as http4 #42A5F5
}

rectangle "WebSockets" as ws #E8F5E9 {
    card "**Bidirectional** - server can push to client" as ws1 #C8E6C9
    card "**Persistent** connection maintained" as ws2 #A5D6A7
    card "**Real-time** updates without polling" as ws3 #81C784
    card "**Best for:** Chat, Auctions, Live updates" as ws4 #66BB6A
}

rectangle "Polling" as poll #FFF3E0 {
    card "**Client asks** server repeatedly" as poll1 #FFE0B2
    card "**Simple** to implement" as poll2 #FFCC80
    card "**Firewall-friendly** - just HTTP requests" as poll3 #FFB74D
    card "**Best for:** Payment status, Fallback" as poll4 #FFA726
}

note bottom of http
  **Used In:**
  URL Shortener
  Coupon System
  Web Crawler
end note

note bottom of ws
  **Used In:**
  Auction System
  Ticketing (status)
  News Feed
end note

note bottom of poll
  **Used In:**
  Payment confirmation
  Background job status
end note

@enduml
```

```plantuml
@startuml
skinparam backgroundColor #FAFAFA
skinparam defaultFontSize 18
skinparam titleFontSize 28
skinparam rectangleBorderThickness 2

title **WEBHOOKS - Server-to-Server Communication**

rectangle "HOW WEBHOOKS WORK" as how #E3F2FD {
    card "**1.** Your system registers a callback URL with 3rd party" as h1 #BBDEFB
    card "**2.** Event happens (e.g., payment completed)" as h2 #90CAF9
    card "**3.** 3rd party POSTs to your callback URL" as h3 #64B5F6
    card "**4.** Your system processes the event" as h4 #42A5F5
}

rectangle "WEBHOOK BEST PRACTICES" as best #E8F5E9 {
    card "**Idempotency:** Handle duplicate deliveries" as b1 #C8E6C9
    card "**Verification:** Validate webhook signatures" as b2 #A5D6A7
    card "**Quick response:** Return 200 immediately, process async" as b3 #81C784
    card "**Retry handling:** Expect retries on failure" as b4 #66BB6A
}

rectangle "TICKETING EXAMPLE" as example #FFF3E0 {
    card "**1.** User reserves ticket, gets reservation UUID" as e1 #FFE0B2
    card "**2.** User pays via Stripe/PayPal with UUID" as e2 #FFCC80
    card "**3.** Payment provider sends webhook to your server" as e3 #FFB74D
    card "**4.** Your server updates ticket status to 'paid'" as e4 #FFA726
}

@enduml
```

```plantuml
@startuml
skinparam backgroundColor #FAFAFA
skinparam defaultFontSize 18
skinparam titleFontSize 28
skinparam rectangleBorderThickness 2

title **PROTOCOL COMPARISON**

rectangle "WHEN TO USE EACH" as comparison #F3E5F5 {
    card "**HTTP/REST:** Standard APIs, CRUD, stateless operations" as c1 #E1BEE7
    card "**WebSockets:** Real-time bidirectional (chat, live bids)" as c2 #CE93D8
    card "**Polling:** Simple status checks, firewall restrictions" as c3 #BA68C8
    card "**Webhooks:** 3rd party integrations (payments, notifications)" as c4 #AB47BC
}

rectangle "TRADE-OFFS" as tradeoffs #FFEBEE {
    card "**HTTP:** Simple but client must initiate all communication" as t1 #FFCDD2
    card "**WebSockets:** Real-time but complex connection management" as t2 #EF9A9A
    card "**Polling:** Wasteful but works everywhere" as t3 #E57373
    card "**Webhooks:** Efficient but requires public endpoint" as t4 #EF5350
}

@enduml
```

---

# PAGE 2: LOAD BALANCING

```plantuml
@startuml
skinparam backgroundColor #FAFAFA
skinparam defaultFontSize 18
skinparam titleFontSize 28
skinparam rectangleBorderThickness 2

title **LOAD BALANCING STRATEGIES**

rectangle "ROUND ROBIN" as rr #E3F2FD {
    card "**How:** Rotate through servers sequentially" as rr1 #BBDEFB
    card "**Pros:** Simple, equal distribution" as rr2 #90CAF9
    card "**Cons:** Ignores server load/capacity" as rr3 #64B5F6
    card "**Best for:** Homogeneous servers, stateless services" as rr4 #42A5F5
}

rectangle "LEAST CONNECTIONS" as lc #E8F5E9 {
    card "**How:** Route to server with fewest active connections" as lc1 #C8E6C9
    card "**Pros:** Adapts to varying request times" as lc2 #A5D6A7
    card "**Cons:** Requires connection tracking" as lc3 #81C784
    card "**Best for:** Long-running requests, varied workloads" as lc4 #66BB6A
}

rectangle "HASH-BASED (L7)" as hash #FFF3E0 {
    card "**How:** hash(user_id) % num_servers" as hash1 #FFE0B2
    card "**Pros:** Same user always hits same server" as hash2 #FFCC80
    card "**Cons:** Uneven if hash distribution is poor" as hash3 #FFB74D
    card "**Best for:** Session affinity, cache locality" as hash4 #FFA726
}

@enduml
```

```plantuml
@startuml
skinparam backgroundColor #FAFAFA
skinparam defaultFontSize 18
skinparam titleFontSize 28
skinparam rectangleBorderThickness 2

title **ADVANCED LOAD BALANCING**

rectangle "GEOGRAPHIC / LATENCY-BASED" as geo #E3F2FD {
    card "**How:** Route to nearest/fastest data center" as geo1 #BBDEFB
    card "**Pros:** Lowest latency for users" as geo2 #90CAF9
    card "**Cons:** Complex setup, data sync challenges" as geo3 #64B5F6
    card "**Best for:** Global applications, CDNs" as geo4 #42A5F5
}

rectangle "WEIGHTED" as weight #E8F5E9 {
    card "**How:** Assign weights based on server capacity" as w1 #C8E6C9
    card "**Pros:** Better utilization of powerful servers" as w2 #A5D6A7
    card "**Cons:** Must manually configure weights" as w3 #81C784
    card "**Best for:** Mixed hardware, gradual rollouts" as w4 #66BB6A
}

rectangle "LAYER 4 vs LAYER 7" as layers #FFF3E0 {
    card "**L4 (Transport):** Fast, routes by IP/port only" as l1 #FFE0B2
    card "**L7 (Application):** Can inspect HTTP headers, URLs" as l2 #FFCC80
    card "**L7 enables:** URL routing, header-based routing" as l3 #FFB74D
    card "**Use L7 for:** User ID sharding, A/B testing" as l4 #FFA726
}

@enduml
```

```plantuml
@startuml
skinparam backgroundColor #FAFAFA
skinparam defaultFontSize 18
skinparam titleFontSize 28
skinparam rectangleBorderThickness 2

title **LOAD BALANCING IN PRACTICE**

rectangle "COUPON SYSTEM EXAMPLE" as coupon #E3F2FD {
    card "**Problem:** 1.2M requests/second during flash sale" as c1 #BBDEFB
    card "**Solution:** L7 load balancer with hash(user_id)" as c2 #90CAF9
    card "**Result:** Each shard handles 120K rps" as c3 #64B5F6
    card "**Benefit:** User always hits same shard = cache hit" as c4 #42A5F5
}

rectangle "API GATEWAY RESPONSIBILITIES" as api #E8F5E9 {
    card "**Authentication:** Verify tokens before routing" as a1 #C8E6C9
    card "**Rate Limiting:** Protect backend from abuse" as a2 #A5D6A7
    card "**SSL Termination:** Handle HTTPS at edge" as a3 #81C784
    card "**Request Transform:** Add headers, modify paths" as a4 #66BB6A
}

@enduml
```

---

# PAGE 3: CACHING STRATEGIES

```plantuml
@startuml
skinparam backgroundColor #FAFAFA
skinparam defaultFontSize 18
skinparam titleFontSize 28
skinparam rectangleBorderThickness 2

title **CACHE-ASIDE (LAZY LOADING)**

rectangle "HOW IT WORKS" as how #E3F2FD {
    card "**1.** Application checks cache first" as h1 #BBDEFB
    card "**2.** If MISS: Query database" as h2 #90CAF9
    card "**3.** Store result in cache" as h3 #64B5F6
    card "**4.** Return data to client" as h4 #42A5F5
}

rectangle "PROS" as pros #E8F5E9 {
    card "Only requested data is cached (no waste)" as p1 #C8E6C9
    card "Cache failure doesn't break the system" as p2 #A5D6A7
    card "Simple to implement and understand" as p3 #81C784
}

rectangle "CONS" as cons #FFEBEE {
    card "First request is always slow (cache miss)" as c1 #FFCDD2
    card "Data can become stale" as c2 #EF9A9A
    card "Cache stampede on popular items" as c3 #E57373
}

note bottom of how
  **Most Common Pattern**
  Used in: URL Shortener
  News Feed, Ticketing
end note

@enduml
```

```plantuml
@startuml
skinparam backgroundColor #FAFAFA
skinparam defaultFontSize 18
skinparam titleFontSize 28
skinparam rectangleBorderThickness 2

title **WRITE-THROUGH & WRITE-BEHIND**

rectangle "WRITE-THROUGH" as through #E3F2FD {
    card "**1.** Application writes to cache" as t1 #BBDEFB
    card "**2.** Cache immediately writes to DB" as t2 #90CAF9
    card "**3.** Return success when both complete" as t3 #64B5F6
    card "**Trade-off:** Slower writes, but always consistent" as t4 #42A5F5
}

rectangle "WRITE-BEHIND (WRITE-BACK)" as behind #E8F5E9 {
    card "**1.** Application writes to cache" as b1 #C8E6C9
    card "**2.** Return success immediately" as b2 #A5D6A7
    card "**3.** Cache writes to DB asynchronously" as b3 #81C784
    card "**Trade-off:** Fast writes, but risk of data loss" as b4 #66BB6A
}

rectangle "WHEN TO USE" as when #FFF3E0 {
    card "**Write-Through:** Data integrity critical (banking)" as w1 #FFE0B2
    card "**Write-Behind:** High write throughput needed" as w2 #FFCC80
    card "**Cache-Aside:** Most common, simple, flexible" as w3 #FFB74D
}

@enduml
```

```plantuml
@startuml
skinparam backgroundColor #FAFAFA
skinparam defaultFontSize 18
skinparam titleFontSize 28
skinparam rectangleBorderThickness 2

title **THE 1% RULE & CACHE SIZING**

rectangle "THE 1% HOT DATA RULE" as rule #E3F2FD {
    card "**Observation:** Only ~1% of data is frequently accessed" as r1 #BBDEFB
    card "**Implication:** Cache the hot 1%, not everything" as r2 #90CAF9
    card "**Example:** 3TB total data = 30GB cache needed" as r3 #64B5F6
    card "**Result:** Massive cost savings, still fast" as r4 #42A5F5
}

rectangle "CACHE TECHNOLOGIES" as tech #E8F5E9 {
    card "**In-Memory (Local):** ns latency, per-instance, 1-4GB" as te1 #C8E6C9
    card "**Redis:** us latency, shared, 100K+ ops/s" as te2 #A5D6A7
    card "**Memcached:** us latency, simple key-value" as te3 #81C784
    card "**CDN:** ms latency, static assets, global" as te4 #66BB6A
}

rectangle "MULTI-TIER CACHING" as multi #FFF3E0 {
    card "**Tier 1:** In-memory cache (per service instance)" as m1 #FFE0B2
    card "**Tier 2:** Redis cluster (shared across services)" as m2 #FFCC80
    card "**Tier 3:** Database (source of truth)" as m3 #FFB74D
    card "**URL Shortener uses all 3 tiers**" as m4 #FFA726
}

@enduml
```

---

# PAGE 4: DATABASE SELECTION

```plantuml
@startuml
skinparam backgroundColor #FAFAFA
skinparam defaultFontSize 18
skinparam titleFontSize 28
skinparam rectangleBorderThickness 2

title **RELATIONAL DATABASES (RDBMS)**

rectangle "PostgreSQL / MySQL" as rdbms #E3F2FD {
    card "**ACID Transactions:** Atomicity, Consistency, Isolation, Durability" as r1 #BBDEFB
    card "**Complex Queries:** JOINs, aggregations, subqueries" as r2 #90CAF9
    card "**Strong Consistency:** Every read sees latest write" as r3 #64B5F6
    card "**Throughput:** 10,000 - 50,000 writes/second" as r4 #42A5F5
}

rectangle "WHEN TO USE RDBMS" as when_rdbms #E8F5E9 {
    card "**Transactions required:** Money transfers, inventory" as w1 #C8E6C9
    card "**Complex relationships:** Users, orders, products" as w2 #A5D6A7
    card "**Data integrity critical:** Can't afford inconsistency" as w3 #81C784
    card "**Need complex queries:** Reporting, analytics" as w4 #66BB6A
}

rectangle "SYSTEMS USING RDBMS" as systems #FFF3E0 {
    card "**Ticketing:** Need ACID for seat reservations" as s1 #FFE0B2
    card "**Auction:** Bid history, user accounts" as s2 #FFCC80
    card "**URL Shortener:** URL mappings (with Redis cache)" as s3 #FFB74D
}

@enduml
```

```plantuml
@startuml
skinparam backgroundColor #FAFAFA
skinparam defaultFontSize 18
skinparam titleFontSize 28
skinparam rectangleBorderThickness 2

title **NoSQL: KEY-VALUE STORES**

rectangle "Redis / DynamoDB" as kv #E3F2FD {
    card "**Simple Lookups:** GET key, SET key value" as k1 #BBDEFB
    card "**Ultra-Fast:** 100,000+ operations/second" as k2 #90CAF9
    card "**Horizontal Scaling:** Easy to shard by key" as k3 #64B5F6
    card "**Schema-less:** No fixed structure" as k4 #42A5F5
}

rectangle "WHEN TO USE KEY-VALUE" as when_kv #E8F5E9 {
    card "**High throughput:** 100K+ ops/s needed" as w1 #C8E6C9
    card "**Simple access patterns:** Get by ID only" as w2 #A5D6A7
    card "**Caching layer:** Hot data storage" as w3 #81C784
    card "**Session storage:** User sessions, tokens" as w4 #66BB6A
}

rectangle "SYSTEMS USING KEY-VALUE" as systems #FFF3E0 {
    card "**URL Shortener:** short_code -> long_url mapping" as s1 #FFE0B2
    card "**Web Crawler:** URL deduplication (seen URLs)" as s2 #FFCC80
    card "**News Feed:** Pre-computed timeline cache" as s3 #FFB74D
}

@enduml
```

```plantuml
@startuml
skinparam backgroundColor #FAFAFA
skinparam defaultFontSize 18
skinparam titleFontSize 28
skinparam rectangleBorderThickness 2

title **NoSQL: DOCUMENT & WIDE-COLUMN**

rectangle "DOCUMENT STORES (MongoDB)" as doc #E3F2FD {
    card "**Flexible Schema:** No fixed columns" as d1 #BBDEFB
    card "**Nested Documents:** JSON-like structure" as d2 #90CAF9
    card "**Good for:** Varied data, rapid iteration" as d3 #64B5F6
    card "**Caution:** No JOINs, denormalize carefully" as d4 #42A5F5
}

rectangle "WIDE-COLUMN (Cassandra)" as wide #E8F5E9 {
    card "**Write-Optimized:** Excellent write throughput" as w1 #C8E6C9
    card "**Time-Series:** Natural fit for timestamped data" as w2 #A5D6A7
    card "**Eventual Consistency:** Tunable consistency levels" as w3 #81C784
    card "**Good for:** Logs, metrics, activity feeds" as w4 #66BB6A
}

rectangle "DATABASE SELECTION SUMMARY" as summary #FFEBEE {
    card "**Need ACID?** -> PostgreSQL, MySQL" as su1 #FFCDD2
    card "**Need 100K+ ops/s?** -> Redis, DynamoDB" as su2 #EF9A9A
    card "**Need flexible schema?** -> MongoDB" as su3 #E57373
    card "**Need time-series at scale?** -> Cassandra" as su4 #EF5350
}

@enduml
```

---

# PAGE 5: SCALING PATTERNS

```plantuml
@startuml
skinparam backgroundColor #FAFAFA
skinparam defaultFontSize 18
skinparam titleFontSize 28
skinparam rectangleBorderThickness 2

title **READ REPLICAS**

rectangle "HOW READ REPLICAS WORK" as how #E3F2FD {
    card "**Primary:** Handles all writes" as h1 #BBDEFB
    card "**Replicas:** Copy all data from primary" as h2 #90CAF9
    card "**Reads:** Distributed across replicas" as h3 #64B5F6
    card "**Each replica holds FULL dataset**" as h4 #42A5F5
}

rectangle "PROS" as pros #E8F5E9 {
    card "**Simple:** No application changes for reads" as p1 #C8E6C9
    card "**Linear scaling:** Add replicas for more read capacity" as p2 #A5D6A7
    card "**Failover:** Promote replica if primary fails" as p3 #81C784
}

rectangle "CONS" as cons #FFEBEE {
    card "**Expensive:** Each replica = full data copy" as c1 #FFCDD2
    card "**Replication lag:** Replicas may be slightly behind" as c2 #EF9A9A
    card "**Writes don't scale:** Still single primary" as c3 #E57373
}

note bottom of how
  **Formula:**
  Replicas = Peak Read RPS / RPS per Replica
  Example: 300K / 50K = 6 replicas
end note

@enduml
```

```plantuml
@startuml
skinparam backgroundColor #FAFAFA
skinparam defaultFontSize 18
skinparam titleFontSize 28
skinparam rectangleBorderThickness 2

title **DATABASE SHARDING**

rectangle "HOW SHARDING WORKS" as how #E3F2FD {
    card "**Partition data** by a shard key" as h1 #BBDEFB
    card "**Each shard** holds subset of data" as h2 #90CAF9
    card "**Scales both** reads AND writes" as h3 #64B5F6
    card "**Route requests** based on shard key" as h4 #42A5F5
}

rectangle "SHARDING STRATEGIES" as strat #E8F5E9 {
    card "**Hash-based:** hash(key) % N shards - even distribution" as s1 #C8E6C9
    card "**Range-based:** Key ranges per shard - good for scans" as s2 #A5D6A7
    card "**Geographic:** By region - low latency" as s3 #81C784
    card "**Directory:** Lookup table - most flexible" as s4 #66BB6A
}

rectangle "CHALLENGES" as challenges #FFEBEE {
    card "**Cross-shard queries:** JOINs across shards are hard" as ch1 #FFCDD2
    card "**Resharding:** Adding shards requires data migration" as ch2 #EF9A9A
    card "**Hot spots:** Some shards may get more traffic" as ch3 #E57373
}

note bottom of how
  **Formula:**
  Shards = Peak RPS / RPS per Shard
  Example: 1.2M / 120K = 10 shards
end note

@enduml
```

```plantuml
@startuml
skinparam backgroundColor #FAFAFA
skinparam defaultFontSize 18
skinparam titleFontSize 28
skinparam rectangleBorderThickness 2

title **SCALING EXAMPLES**

rectangle "COUPON SYSTEM" as coupon #E3F2FD {
    card "**Challenge:** 1.2M requests/second peak (flash sale)" as c1 #BBDEFB
    card "**Solution:** 10 shards by last digit of user_id" as c2 #90CAF9
    card "**Result:** 120K rps per shard (manageable!)" as c3 #64B5F6
    card "**Bonus:** User always hits same shard = cache hits" as c4 #42A5F5
}

rectangle "TICKETING SYSTEM" as ticket #E8F5E9 {
    card "**Challenge:** 60K requests/second, 200B rows over 5 years" as t1 #C8E6C9
    card "**Solution:** Shard by venue_id" as t2 #A5D6A7
    card "**Why venue?** Events are venue-specific, no cross-shard" as t3 #81C784
    card "**Result:** Natural data locality" as t4 #66BB6A
}

rectangle "URL SHORTENER" as url #FFF3E0 {
    card "**Challenge:** 300K reads/second, 3TB data" as u1 #FFE0B2
    card "**Solution:** Read replicas + multi-tier cache" as u2 #FFCC80
    card "**Why not shard?** Reads dominate, replicas cheaper" as u3 #FFB74D
    card "**Cache:** 30GB Redis handles most traffic" as u4 #FFA726
}

@enduml
```

---

# PAGE 6: MESSAGE QUEUES

```plantuml
@startuml
skinparam backgroundColor #FAFAFA
skinparam defaultFontSize 18
skinparam titleFontSize 28
skinparam rectangleBorderThickness 2

title **MESSAGE QUEUE PATTERNS**

rectangle "POINT-TO-POINT" as p2p #E3F2FD {
    card "**Each message** consumed by ONE consumer" as p1 #BBDEFB
    card "**Use for:** Task distribution, work queues" as p2 #90CAF9
    card "**Example:** Process uploaded images" as p3 #64B5F6
    card "**Scaling:** Add more consumers for parallelism" as p4 #42A5F5
}

rectangle "PUBLISH/SUBSCRIBE" as pubsub #E8F5E9 {
    card "**Each message** delivered to ALL subscribers" as ps1 #C8E6C9
    card "**Use for:** Event broadcasting, notifications" as ps2 #A5D6A7
    card "**Example:** New order -> notify inventory, shipping, billing" as ps3 #81C784
    card "**Benefit:** Services are decoupled" as ps4 #66BB6A
}

rectangle "PARTITIONED (KAFKA)" as part #FFF3E0 {
    card "**Messages partitioned** by key" as pa1 #FFE0B2
    card "**Ordering guaranteed** within partition" as pa2 #FFCC80
    card "**Use for:** Event streaming, logs, ordered processing" as pa3 #FFB74D
    card "**Scaling:** Add partitions for parallelism" as pa4 #FFA726
}

@enduml
```

```plantuml
@startuml
skinparam backgroundColor #FAFAFA
skinparam defaultFontSize 18
skinparam titleFontSize 28
skinparam rectangleBorderThickness 2

title **QUEUE TECHNOLOGIES**

rectangle "KAFKA" as kafka #E3F2FD {
    card "**Throughput:** 100,000+ messages/second per broker" as k1 #BBDEFB
    card "**Ordering:** Per-partition ordering guaranteed" as k2 #90CAF9
    card "**Durability:** Persistent log, replayable" as k3 #64B5F6
    card "**Best for:** Event streaming, logs, audit trails" as k4 #42A5F5
}

rectangle "RABBITMQ" as rabbit #E8F5E9 {
    card "**Throughput:** ~20,000 messages/second" as r1 #C8E6C9
    card "**Routing:** Flexible routing with exchanges" as r2 #A5D6A7
    card "**Protocol:** AMQP standard" as r3 #81C784
    card "**Best for:** Task queues, complex routing, RPC" as r4 #66BB6A
}

rectangle "SQS / REDIS" as sqs #FFF3E0 {
    card "**SQS:** Managed, 3K msgs/s, at-least-once delivery" as s1 #FFE0B2
    card "**Redis Pub/Sub:** 100K+ msgs/s, fire-and-forget" as s2 #FFCC80
    card "**Best for:** Simple async tasks, real-time notifications" as s3 #FFB74D
}

@enduml
```

```plantuml
@startuml
skinparam backgroundColor #FAFAFA
skinparam defaultFontSize 18
skinparam titleFontSize 28
skinparam rectangleBorderThickness 2

title **WHY USE MESSAGE QUEUES**

rectangle "DECOUPLE SERVICES" as decouple #E3F2FD {
    card "**Problem:** Service A calls Service B synchronously" as d1 #BBDEFB
    card "**Issue:** If B is slow/down, A is affected" as d2 #90CAF9
    card "**Solution:** A publishes to queue, B consumes async" as d3 #64B5F6
    card "**Used in:** Auction bid processing" as d4 #42A5F5
}

rectangle "HANDLE TRAFFIC SPIKES" as spikes #E8F5E9 {
    card "**Problem:** Flash sale causes 10x traffic spike" as sp1 #C8E6C9
    card "**Issue:** Database can't handle burst" as sp2 #A5D6A7
    card "**Solution:** Queue absorbs spike, process at steady rate" as sp3 #81C784
    card "**Used in:** Coupon system flash sales" as sp4 #66BB6A
}

rectangle "GUARANTEE ORDERING" as order #FFF3E0 {
    card "**Problem:** Bids must be processed in arrival order" as o1 #FFE0B2
    card "**Issue:** Parallel processing breaks ordering" as o2 #FFCC80
    card "**Solution:** Single consumer per partition" as o3 #FFB74D
    card "**Used in:** Auction bid serialization" as o4 #FFA726
}

@enduml
```

---

# PAGE 7: CONCURRENCY CONTROL

```plantuml
@startuml
skinparam backgroundColor #FAFAFA
skinparam defaultFontSize 18
skinparam titleFontSize 28
skinparam rectangleBorderThickness 2

title **THE RACE CONDITION PROBLEM**

rectangle "WHAT GOES WRONG" as problem #FFEBEE {
    card "**User A reads:** 1 ticket available" as p1 #FFCDD2
    card "**User B reads:** 1 ticket available (same time!)" as p2 #EF9A9A
    card "**User A reserves:** Success!" as p3 #E57373
    card "**User B reserves:** Also success! (WRONG!)" as p4 #EF5350
    card "**Result:** 2 reservations for 1 ticket = OVERSOLD" as p5 #F44336
}

rectangle "WHY IT HAPPENS" as why #E3F2FD {
    card "**Read-then-write** is not atomic" as w1 #BBDEFB
    card "**Time gap** between check and update" as w2 #90CAF9
    card "**Concurrent requests** see same initial state" as w3 #64B5F6
    card "**Both succeed** because condition was true for both" as w4 #42A5F5
}

rectangle "AFFECTED SYSTEMS" as affected #E8F5E9 {
    card "**Ticketing:** Last seats being oversold" as a1 #C8E6C9
    card "**Auctions:** Higher bid losing to lower bid" as a2 #A5D6A7
    card "**Coupons:** More coupons issued than available" as a3 #81C784
    card "**Inventory:** Stock going negative" as a4 #66BB6A
}

@enduml
```

```plantuml
@startuml
skinparam backgroundColor #FAFAFA
skinparam defaultFontSize 18
skinparam titleFontSize 28
skinparam rectangleBorderThickness 2

title **OPTIMISTIC LOCKING**

rectangle "HOW IT WORKS" as how #E3F2FD {
    card "**1.** Read current state (e.g., price = $2)" as h1 #BBDEFB
    card "**2.** Do your logic (new bid = $5)" as h2 #90CAF9
    card "**3.** UPDATE WHERE current_state = expected" as h3 #64B5F6
    card "**4.** Check rows affected: 1 = success, 0 = retry" as h4 #42A5F5
}

rectangle "SQL EXAMPLE" as sql #E8F5E9 {
    card "UPDATE bids SET price = 5" as s1 #C8E6C9
    card "WHERE item_id = 123 AND price = 2" as s2 #A5D6A7
    card "-- Only succeeds if price is still $2" as s3 #81C784
    card "-- Returns 0 rows if someone else updated first" as s4 #66BB6A
}

rectangle "TRADE-OFFS" as tradeoffs #FFF3E0 {
    card "**Pros:** No locks, good for low contention" as t1 #FFE0B2
    card "**Cons:** Must retry on conflict" as t2 #FFCC80
    card "**Best for:** URL shortener (collision retry)" as t3 #FFB74D
    card "**Not ideal:** High contention scenarios" as t4 #FFA726
}

@enduml
```

```plantuml
@startuml
skinparam backgroundColor #FAFAFA
skinparam defaultFontSize 18
skinparam titleFontSize 28
skinparam rectangleBorderThickness 2

title **QUEUE SERIALIZATION**

rectangle "HOW IT WORKS" as how #E3F2FD {
    card "**1.** All writes go to a message queue" as h1 #BBDEFB
    card "**2.** Single consumer processes sequentially" as h2 #90CAF9
    card "**3.** No race conditions - one at a time" as h3 #64B5F6
    card "**4.** Results published to response queue" as h4 #42A5F5
}

rectangle "AUCTION EXAMPLE" as auction #E8F5E9 {
    card "**Rick bids $5**, **Morty bids $3** (concurrent)" as a1 #C8E6C9
    card "**Queue:** Rick's bid enters first" as a2 #A5D6A7
    card "**Process:** Rick's $5 > current $2 -> Accept" as a3 #81C784
    card "**Process:** Morty's $3 < current $5 -> Reject" as a4 #66BB6A
}

rectangle "TRADE-OFFS" as tradeoffs #FFF3E0 {
    card "**Pros:** Strict ordering, fair processing" as t1 #FFE0B2
    card "**Cons:** Single point of failure, latency" as t2 #FFCC80
    card "**Best for:** Auctions (bid ordering matters)" as t3 #FFB74D
    card "**Scaling:** Multiple queues per auction item" as t4 #FFA726
}

@enduml
```

---

# PAGE 8: LINEARIZATION (BEST FOR TICKETING)

```plantuml
@startuml
skinparam backgroundColor #FAFAFA
skinparam defaultFontSize 18
skinparam titleFontSize 28
skinparam rectangleBorderThickness 2

title **LINEARIZATION - THE BEST SOLUTION FOR TICKETING**

rectangle "THE KEY INSIGHT" as insight #E3F2FD {
    card "**Don't count** available tickets" as i1 #BBDEFB
    card "**Pre-create** all ticket records" as i2 #90CAF9
    card "**Atomic claim:** UPDATE WHERE status='free' LIMIT 1" as i3 #64B5F6
    card "**Database handles** concurrency automatically" as i4 #42A5F5
}

rectangle "STEP 1: PRE-CREATE TICKETS" as step1 #E8F5E9 {
    card "INSERT INTO tickets (id, status) VALUES" as s1a #C8E6C9
    card "(1, 'free'), (2, 'free'), (3, 'free')..." as s1b #A5D6A7
    card "Create ALL tickets upfront with status='free'" as s1c #81C784
}

rectangle "STEP 2: ATOMIC CLAIM" as step2 #FFF3E0 {
    card "UPDATE tickets" as s2a #FFE0B2
    card "SET user_id = ?, status = 'reserved'" as s2b #FFCC80
    card "WHERE status = 'free' LIMIT 1" as s2c #FFB74D
    card "**Only ONE user can claim each ticket!**" as s2d #FFA726
}

@enduml
```

```plantuml
@startuml
skinparam backgroundColor #FAFAFA
skinparam defaultFontSize 18
skinparam titleFontSize 28
skinparam rectangleBorderThickness 2

title **WHY LINEARIZATION IS BEST**

rectangle "NO RACE CONDITIONS" as norace #E3F2FD {
    card "**UPDATE ... WHERE status='free'** is atomic" as n1 #BBDEFB
    card "Database locks the row during update" as n2 #90CAF9
    card "Only ONE transaction sees 'free' and updates" as n3 #64B5F6
    card "Others get 0 rows affected = ticket taken" as n4 #42A5F5
}

rectangle "HORIZONTAL SCALING" as scale #E8F5E9 {
    card "**Shard by venue_id** - each venue is independent" as s1 #C8E6C9
    card "No coordination needed between shards" as s2 #A5D6A7
    card "Add shards as venues grow" as s3 #81C784
    card "No single point of failure" as s4 #66BB6A
}

rectangle "SIMPLER ARCHITECTURE" as simple #FFF3E0 {
    card "**No message queue** for serialization needed" as si1 #FFE0B2
    card "**No distributed locks** required" as si2 #FFCC80
    card "**No retry logic** - just check rows affected" as si3 #FFB74D
    card "**Database does the hard work**" as si4 #FFA726
}

@enduml
```

```plantuml
@startuml
skinparam backgroundColor #FAFAFA
skinparam defaultFontSize 18
skinparam titleFontSize 28
skinparam rectangleBorderThickness 2

title **TICKET STATUS STATE MACHINE**

rectangle "STATUS FLOW" as flow #E3F2FD {
    card "**FREE** -> Initial state, available for reservation" as f1 #BBDEFB
    card "**RESERVED** -> User claimed, awaiting payment" as f2 #90CAF9
    card "**PAID** -> Payment confirmed, ticket issued" as f3 #64B5F6
    card "**REVERSED** -> Payment failed or timeout" as f4 #42A5F5
}

rectangle "STATE TRANSITIONS" as trans #E8F5E9 {
    card "**free -> reserved:** Atomic UPDATE claim" as t1 #C8E6C9
    card "**reserved -> paid:** Payment webhook received" as t2 #A5D6A7
    card "**reserved -> reversed:** Timeout or payment failed" as t3 #81C784
    card "**reversed -> free:** Ticket released back to pool" as t4 #66BB6A
}

rectangle "CONCURRENCY PATTERN SUMMARY" as summary #FFEBEE {
    card "**Linearization:** Fixed resources (tickets, coupons)" as su1 #FFCDD2
    card "**Queue Serialization:** Strict ordering (bids)" as su2 #EF9A9A
    card "**Optimistic Locking:** Low contention (URLs)" as su3 #E57373
}

@enduml
```

---

# PAGE 9: FAN-OUT STRATEGIES

```plantuml
@startuml
skinparam backgroundColor #FAFAFA
skinparam defaultFontSize 18
skinparam titleFontSize 28
skinparam rectangleBorderThickness 2

title **FAN-OUT ON WRITE (PUSH)**

rectangle "HOW IT WORKS" as how #E3F2FD {
    card "**When user posts:**" as h0 #BBDEFB
    card "**1.** Store the post in database" as h1 #90CAF9
    card "**2.** Find all followers (e.g., 100 users)" as h2 #64B5F6
    card "**3.** Copy post to EACH follower's timeline cache" as h3 #42A5F5
    card "**4.** Done - feeds are pre-computed!" as h4 #2196F3
}

rectangle "PROS" as pros #E8F5E9 {
    card "**Reads are O(1):** Just fetch pre-built timeline" as p1 #C8E6C9
    card "**No computation** at read time" as p2 #A5D6A7
    card "**Consistent fast** response for all users" as p3 #81C784
}

rectangle "CONS" as cons #FFEBEE {
    card "**Writes are O(N):** N = number of followers" as c1 #FFCDD2
    card "**Write amplification:** Same data copied N times" as c2 #EF9A9A
    card "**VIP problem:** 1M followers = 1M writes!" as c3 #E57373
}

@enduml
```

```plantuml
@startuml
skinparam backgroundColor #FAFAFA
skinparam defaultFontSize 18
skinparam titleFontSize 28
skinparam rectangleBorderThickness 2

title **FAN-OUT ON READ (PULL)**

rectangle "HOW IT WORKS" as how #E3F2FD {
    card "**When user opens app:**" as h0 #BBDEFB
    card "**1.** Find everyone they follow (e.g., 50 users)" as h1 #90CAF9
    card "**2.** Fetch recent posts from EACH followed user" as h2 #64B5F6
    card "**3.** Merge and sort all posts by timestamp" as h3 #42A5F5
    card "**4.** Return the feed" as h4 #2196F3
}

rectangle "PROS" as pros #E8F5E9 {
    card "**Writes are O(1):** Just store the post once" as p1 #C8E6C9
    card "**No write amplification**" as p2 #A5D6A7
    card "**VIPs don't cause spikes**" as p3 #81C784
}

rectangle "CONS" as cons #FFEBEE {
    card "**Reads are O(N):** N = number of followed users" as c1 #FFCDD2
    card "**Computation on every read**" as c2 #EF9A9A
    card "**Slower, inconsistent response times**" as c3 #E57373
}

@enduml
```

```plantuml
@startuml
skinparam backgroundColor #FAFAFA
skinparam defaultFontSize 18
skinparam titleFontSize 28
skinparam rectangleBorderThickness 2

title **HYBRID FAN-OUT (BEST APPROACH)**

rectangle "THE STRATEGY" as strategy #E3F2FD {
    card "**Regular users (< 10K followers):** Use PUSH" as s1 #BBDEFB
    card "**Pre-compute** their followers' timelines" as s2 #90CAF9
    card "**VIPs/Celebrities (1M+ followers):** Use PULL" as s3 #64B5F6
    card "**Fetch VIP posts** at read time, merge with timeline" as s4 #42A5F5
}

rectangle "WHY HYBRID WORKS" as why #E8F5E9 {
    card "**Regular users:** 100 followers x 10K users = 1M writes (fine)" as w1 #C8E6C9
    card "**VIP tweet:** Would be 1M writes = 2x normal load spike!" as w2 #A5D6A7
    card "**Solution:** Don't fan-out VIP tweets at all" as w3 #81C784
    card "**Only ~25 accounts** have 50M+ followers (Wikipedia)" as w4 #66BB6A
}

rectangle "NEWS FEED IMPLEMENTATION" as impl #FFF3E0 {
    card "**Timeline cache:** Pre-computed from regular users" as i1 #FFE0B2
    card "**VIP list:** Track which followed users are VIPs" as i2 #FFCC80
    card "**At read time:** Fetch timeline + merge VIP posts" as i3 #FFB74D
    card "**Result:** Fast reads, no VIP spikes" as i4 #FFA726
}

@enduml
```

---

# PAGE 10: INTERVIEW CHECKLIST

```plantuml
@startuml
skinparam backgroundColor #FAFAFA
skinparam defaultFontSize 18
skinparam titleFontSize 28
skinparam rectangleBorderThickness 2

title **STEP 1: CLARIFY REQUIREMENTS (5 min)**

rectangle "QUESTIONS TO ASK" as questions #E3F2FD {
    card "**Scale:** How many users? Requests per second?" as q1 #BBDEFB
    card "**Data:** How much storage? Retention period?" as q2 #90CAF9
    card "**Ratio:** Read-heavy or write-heavy?" as q3 #64B5F6
    card "**Latency:** Real-time needed? Eventual consistency OK?" as q4 #42A5F5
    card "**Consistency:** Can we show stale data?" as q5 #2196F3
}

rectangle "STEP 2: BACK-OF-ENVELOPE (5 min)" as boe #E8F5E9 {
    card "**RPS** = Total requests / time window" as b1 #C8E6C9
    card "**Peak** = Average x 10 (rule of thumb)" as b2 #A5D6A7
    card "**Storage** = Records x size x retention" as b3 #81C784
    card "**Identify bottleneck:** What will fail first?" as b4 #66BB6A
}

rectangle "STEP 3: HIGH-LEVEL DESIGN (10 min)" as hld #FFF3E0 {
    card "**Draw:** Client, LB, Services, Cache, DB" as h1 #FFE0B2
    card "**Show data flow** with labeled arrows" as h2 #FFCC80
    card "**Define APIs:** POST /resource, GET /resource/:id" as h3 #FFB74D
    card "**Choose database** with reasoning" as h4 #FFA726
}

@enduml
```

```plantuml
@startuml
skinparam backgroundColor #FAFAFA
skinparam defaultFontSize 18
skinparam titleFontSize 28
skinparam rectangleBorderThickness 2

title **STEP 4: DEEP DIVE (15 min)**

rectangle "DATABASE DESIGN" as db #E3F2FD {
    card "**Schema:** Tables, columns, types" as d1 #BBDEFB
    card "**Indexes:** What queries need to be fast?" as d2 #90CAF9
    card "**Relationships:** Foreign keys, denormalization" as d3 #64B5F6
}

rectangle "CACHING STRATEGY" as cache #E8F5E9 {
    card "**What to cache:** Hot data, computed results" as c1 #C8E6C9
    card "**Cache pattern:** Cache-aside, write-through" as c2 #A5D6A7
    card "**Invalidation:** TTL, event-based, manual" as c3 #81C784
}

rectangle "SCALING & CONCURRENCY" as scale #FFF3E0 {
    card "**Read scaling:** Replicas, cache tiers" as s1 #FFE0B2
    card "**Write scaling:** Sharding strategy" as s2 #FFCC80
    card "**Concurrency:** Optimistic, serialization, linearization" as s3 #FFB74D
    card "**Failure handling:** Retry, fallback, circuit breaker" as s4 #FFA726
}

@enduml
```

```plantuml
@startuml
skinparam backgroundColor #FAFAFA
skinparam defaultFontSize 18
skinparam titleFontSize 28
skinparam rectangleBorderThickness 2

title **STEP 5: TRADE-OFFS & COMMON MISTAKES**

rectangle "TRADE-OFFS TO DISCUSS" as tradeoffs #E3F2FD {
    card "**Consistency vs Availability:** Which matters more?" as t1 #BBDEFB
    card "**Latency vs Throughput:** Optimize for which?" as t2 #90CAF9
    card "**Cost vs Performance:** Where to invest?" as t3 #64B5F6
    card "**Complexity vs Simplicity:** Is it worth it?" as t4 #42A5F5
}

rectangle "COMMON MISTAKES" as mistakes #FFEBEE {
    card "**Jumping to solution** without clarifying requirements" as m1 #FFCDD2
    card "**Skipping calculations** - always do back-of-envelope" as m2 #EF9A9A
    card "**Over-engineering** for scale you don't need" as m3 #E57373
    card "**Ignoring failures** - discuss what breaks" as m4 #EF5350
    card "**No trade-offs** - everything has pros/cons" as m5 #F44336
}

rectangle "SYSTEM BENCHMARKS (MEMORIZE)" as bench #E8F5E9 {
    card "**PostgreSQL:** 10-50K writes/s" as b1 #C8E6C9
    card "**Redis:** 100K+ ops/s" as b2 #A5D6A7
    card "**Kafka:** 100K+ msgs/s per broker" as b3 #81C784
    card "**Service instance:** 1-10K RPS" as b4 #66BB6A
}

@enduml
```

---

## PRINT INSTRUCTIONS

| Page | Content |
|------|---------|
| **1** | Client Communication (HTTP, WebSockets, Polling, Webhooks) |
| **2** | Load Balancing (Round Robin, Hash, Geographic, L4/L7) |
| **3** | Caching Strategies (Cache-Aside, Write-Through, 1% Rule) |
| **4** | Database Selection (RDBMS, Key-Value, Document, Wide-Column) |
| **5** | Scaling Patterns (Read Replicas, Sharding, Examples) |
| **6** | Message Queues (Patterns, Technologies, Use Cases) |
| **7** | Concurrency Control (Race Conditions, Optimistic, Serialization) |
| **8** | Linearization (Best for Ticketing, State Machine) |
| **9** | Fan-Out Strategies (Push, Pull, Hybrid) |
| **10** | Interview Checklist (5 Steps, Trade-offs, Benchmarks) |

**Export Settings:**
- Format: PNG or SVG at 300 DPI
- Size: A2 Landscape (594mm x 420mm)
- Background: Light (#FAFAFA)
- Font: 18px uniform throughout

---

*Based on "Pragmatic System Design" by Alexey Soshin*
*Systems: Web Crawler, Auction, Coupon, URL Shortener, News Feed, Ticketing*
