# Advanced System Design Reference Architecture

> Sharding, consistent hashing, circuit breakers, retry patterns, and data partitioning.

---

## Complete Advanced Architecture

```plantuml
@startuml
skinparam backgroundColor #FEFEFE
skinparam defaultFontSize 16
skinparam titleFontSize 24
skinparam rectangleBorderThickness 2
skinparam padding 2
skinparam nodesep 10
skinparam ranksep 10

title **Advanced System Design Reference Architecture**

' === EDGE + GATEWAY ===
rectangle "EDGE & GATEWAY" as edge #E8F5E9 {
    rectangle "CDN / DNS" as cdn #C8E6C9
    rectangle "API Gateway" as gateway #A5D6A7 {
        card "Rate Limit | Auth | SSL" as gwFeatures #81C784
        card "**Consistent Hash Router**" as hashRouter #4CAF50
    }
}

' === SERVICE LAYER ===
rectangle "SERVICE LAYER" as serviceLayer #F3E5F5 {
    rectangle "Microservices (N instances)" as services #E1BEE7 {
        rectangle "Service A" as svcA #CE93D8
        rectangle "Service B" as svcB #CE93D8
    }

    rectangle "**RESILIENCE**" as resilience #E1BEE7 {
        hexagon "Circuit\nBreaker" as cb #9C27B0
        card "Retry + Backoff" as retry #BA68C8
        card "Timeout" as timeout #BA68C8
        card "Fallback" as fallback #BA68C8
    }

    rectangle "WebSocket Server" as ws #BA68C8
}

' === ASYNC LAYER ===
rectangle "ASYNC (Partitioned)" as asyncLayer #FFEBEE {
    queue "Kafka" as queue #FFCDD2
    rectangle "Partitioned Consumers" as consumers #EF9A9A {
        rectangle "Worker P0" as w0 #E57373
        rectangle "Worker P1" as w1 #E57373
        rectangle "Worker PN" as wN #E57373
    }
    rectangle "Scheduler" as scheduler #E57373
}

' === CACHE LAYER ===
rectangle "CACHE (Consistent Hash)" as cacheLayer #E0F7FA {
    rectangle "Redis Cluster" as redisCluster #B2EBF2 {
        database "Redis 0" as redis0 #80DEEA
        database "Redis 1" as redis1 #80DEEA
        database "Redis N" as redisN #80DEEA
    }
    database "Local Cache" as localCache #4DD0E1
}

' === DATA LAYER - SHARDED ===
rectangle "DATA LAYER (Sharded)" as dataLayer #FBE9E7 {
    rectangle "**Shard Router** (Consistent Hash)" as shardRouter #FF7043

    rectangle "PostgreSQL Shards" as pgShards #FFCCBC {
        database "Shard 0\nuser_id % 4 = 0" as shard0 #FFAB91
        database "Shard 1\nuser_id % 4 = 1" as shard1 #FFAB91
        database "Shard 2\nuser_id % 4 = 2" as shard2 #FFAB91
        database "Shard 3\nuser_id % 4 = 3" as shard3 #FFAB91
    }

    rectangle "Read Replicas (Per Shard)" as replicaGroup #FFE0B2 {
        database "Replica 0a, 0b" as rep0 #FFF3E0
        database "Replica 1a, 1b" as rep1 #FFF3E0
        database "Replica 2a, 2b" as rep2 #FFF3E0
        database "Replica 3a, 3b" as rep3 #FFF3E0
    }

    database "NoSQL (DynamoDB)" as nosql #FF8A65
    database "Search (Elasticsearch)" as search #FF5722
}

' === STORAGE LAYER ===
rectangle "STORAGE" as storageLayer #F1F8E9 {
    database "Object Storage (S3)" as objectStorage #DCEDC8
}

' === EXTERNAL LAYER ===
rectangle "EXTERNAL" as externalLayer #ECEFF1 {
    rectangle "Payment (Stripe)" as payment #CFD8DC
    rectangle "Email/SMS" as notifications #B0BEC5
}

' === MONITORING ===
rectangle "OBSERVABILITY" as obsLayer #FCE4EC {
    rectangle "Metrics" as metrics #F8BBD9
    rectangle "Logging" as logging #F48FB1
    rectangle "Tracing" as tracing #EC407A
}

' === CONNECTIONS ===
cdn --> gateway
gateway --> hashRouter
hashRouter --> svcA : hash(user_id)
hashRouter --> svcB : hash(user_id)
gateway --> ws : WS Upgrade

svcA --> cb
cb --> svcB : Protected call
cb --> retry : On failure
retry --> fallback : After max retries

services --> redisCluster : Consistent hash
services --> localCache
services --> shardRouter

shardRouter --> shard0
shardRouter --> shard1
shardRouter --> shard2
shardRouter --> shard3

shard0 --> rep0 : Async
shard1 --> rep1 : Async
shard2 --> rep2 : Async
shard3 --> rep3 : Async

shardRouter ..> replicaGroup : Reads

svcB --> nosql
svcB --> search

svcA --> queue : Publish
queue --> w0 : Partition 0
queue --> w1 : Partition 1
queue --> wN : Partition N
scheduler --> queue

consumers --> notifications
consumers --> shardRouter

cb --> payment
payment --> gateway : Webhook

cdn --> objectStorage : Origin

services ..> metrics
services ..> logging
services ..> tracing

@enduml
```

---

## 1. Consistent Hashing

```plantuml
@startuml
skinparam backgroundColor #FEFEFE
skinparam defaultFontSize 16
skinparam titleFontSize 22
skinparam padding 2
skinparam nodesep 8
skinparam ranksep 8

title **Consistent Hashing Ring**

rectangle "Hash Ring (0 to 2^32)" as ring #E3F2FD {
    card "**Node A** Position: 1000" as nodeA #C8E6C9
    card "**Node B** Position: 3000" as nodeB #C8E6C9
    card "**Node C** Position: 6000" as nodeC #C8E6C9
    card "**Node D** Position: 9000" as nodeD #C8E6C9

    card "key1 hash=500" as key1 #BBDEFB
    card "key2 hash=2500" as key2 #BBDEFB
    card "key3 hash=7000" as key3 #BBDEFB
}

key1 --> nodeA : Routes to A (next clockwise)
key2 --> nodeB : Routes to B
key3 --> nodeD : Routes to D

note bottom of ring
  **Why Consistent Hashing?**

  When Node C fails:
  - Only keys 6001-9000 rehash (to D)
  - Keys 0-6000 stay on same nodes

  With simple hash(key) % N:
  - ALL keys would rehash!
end note

@enduml
```

#### When to Use

| Scenario | Use Consistent Hashing |
|----------|----------------------|
| **Cache cluster** | Redis/Memcached nodes - minimize cache misses on node failure |
| **Database shards** | Add/remove shards without full data migration |
| **Load balancer** | Sticky sessions without centralized state |
| **Distributed storage** | Cassandra, DynamoDB partition assignment |

---

## 2. Circuit Breaker Pattern

```plantuml
@startuml
skinparam backgroundColor #FEFEFE
skinparam defaultFontSize 16
skinparam titleFontSize 22
skinparam padding 2

title **Circuit Breaker State Machine**

state "CLOSED" as closed #C8E6C9 {
    closed : Requests flow normally
    closed : Track failure count
}

state "OPEN" as open #FFCDD2 {
    open : Requests fail immediately
    open : Don't call downstream
    open : Return fallback/error
}

state "HALF-OPEN" as halfOpen #FFF9C4 {
    halfOpen : Allow ONE test request
    halfOpen : Check if service recovered
}

[*] --> closed

closed --> open : Failure threshold exceeded (e.g., 5 failures)
open --> halfOpen : Timeout expires (e.g., 30 seconds)
halfOpen --> closed : Test request succeeds
halfOpen --> open : Test request fails

note right of open
  **Benefits:**
  - Fail fast (don't wait for timeout)
  - Prevent cascade failures
  - Give downstream time to recover
  - Save resources
end note

@enduml
```

#### Circuit Breaker Flow

```plantuml
@startuml
skinparam backgroundColor #FEFEFE
skinparam defaultFontSize 16
skinparam titleFontSize 22

participant "Service A" as svcA
participant "Circuit Breaker" as cb
participant "Service B (Downstream)" as svcB
participant "Fallback" as fallback

== Normal Operation (CLOSED) ==
svcA -> cb : Request
cb -> svcB : Forward
svcB --> cb : Success
cb --> svcA : Response

== Failures Accumulate ==
svcA -> cb : Request
cb -> svcB : Forward
svcB --> cb : Timeout/Error
cb --> svcA : Error (failure count: 3)

== Circuit OPENS ==
svcA -> cb : Request
cb --> svcA : Fail fast (circuit open)
note right: No call to Service B!\nReturn cached/default value

== After Timeout (HALF-OPEN) ==
svcA -> cb : Request
cb -> svcB : Test request
svcB --> cb : Success!
cb --> svcA : Response
note right: Circuit CLOSES\nNormal operation resumes

@enduml
```

---

## 3. Retry with Exponential Backoff

```plantuml
@startuml
skinparam backgroundColor #FEFEFE
skinparam defaultFontSize 16
skinparam titleFontSize 22
skinparam padding 2

title **Retry with Exponential Backoff + Jitter**

start

:Initial Request;

repeat
    :Attempt request;

    if (Success?) then (Yes)
        #C8E6C9:Return success;
        stop
    else (No)
        :Increment attempt count;

        if (Max retries exceeded?) then (Yes)
            #FFCDD2:Return failure / Use fallback;
            stop
        else (No)
            :Calculate wait time;
            note right
              **Exponential Backoff:**
              wait = base * 2^attempt

              Attempt 1: 100ms
              Attempt 2: 200ms
              Attempt 3: 400ms
              Attempt 4: 800ms

              **+ Jitter (randomness):**
              wait = wait * random(0.5, 1.5)

              Prevents thundering herd
            end note

            :Sleep(wait time);
        endif
    endif

repeat while (true)

@enduml
```

#### Retry Strategy Comparison

| Strategy | Formula | When to Use |
|----------|---------|-------------|
| **Fixed delay** | `wait = 1s` | Simple, predictable timing |
| **Linear backoff** | `wait = attempt * 1s` | Gradual increase |
| **Exponential backoff** | `wait = 2^attempt * 100ms` | Fast recovery, prevents overload |
| **Exponential + Jitter** | `wait = random(0, 2^attempt * 100ms)` | Distributed systems (prevents thundering herd) |

---

## 4. Database Sharding Strategy

```plantuml
@startuml
skinparam backgroundColor #FEFEFE
skinparam defaultFontSize 16
skinparam titleFontSize 22
skinparam padding 2
skinparam nodesep 8
skinparam ranksep 8

title **Data Partitioning Strategies**

rectangle "HASH-BASED SHARDING" as hash #E3F2FD {
    card "**How:** shard = hash(user_id) % N" as h1 #BBDEFB
    card "**Distribution:** Even across shards" as h2 #90CAF9
    card "**Resharding:** Requires data migration" as h3 #64B5F6
    card "**Best for:** User data, sessions, coupons" as h4 #42A5F5
}

rectangle "RANGE-BASED SHARDING" as range #E8F5E9 {
    card "**How:** Shard by value range" as r1 #C8E6C9
    card "**Example:** 2024-01 -> Shard1, 2024-02 -> Shard2" as r2 #A5D6A7
    card "**Hotspot risk:** Recent data is hot" as r3 #81C784
    card "**Best for:** Time-series, logs, events" as r4 #66BB6A
}

rectangle "DIRECTORY-BASED SHARDING" as directory #FFF3E0 {
    card "**How:** Lookup table maps key -> shard" as d1 #FFE0B2
    card "**Flexibility:** Any routing logic" as d2 #FFCC80
    card "**Overhead:** Extra lookup required" as d3 #FFB74D
    card "**Best for:** Complex multi-tenant, migrations" as d4 #FFA726
}

rectangle "ENTITY/GEOGRAPHIC SHARDING" as entity #F3E5F5 {
    card "**How:** Natural partitions (venue, region)" as e1 #E1BEE7
    card "**Locality:** Related data together" as e2 #CE93D8
    card "**No cross-shard:** Queries stay local" as e3 #BA68C8
    card "**Best for:** Ticketing (by venue), multi-region" as e4 #AB47BC
}

@enduml
```

#### Sharding Architecture Detail

```plantuml
@startuml
skinparam backgroundColor #FEFEFE
skinparam defaultFontSize 16
skinparam titleFontSize 22
skinparam padding 2
skinparam nodesep 8
skinparam ranksep 8

title **Sharded Database Architecture**

rectangle "Application Layer" as app #E3F2FD {
    rectangle "Service" as svc #BBDEFB
}

rectangle "Routing Layer" as routing #FFF3E0 {
    rectangle "Shard Router" as router #FFE0B2 {
        card "1. Extract shard key (user_id from request)" as step1 #FFCC80
        card "2. Apply hash function hash(user_id)" as step2 #FFCC80
        card "3. Map to shard hash % num_shards" as step3 #FFCC80
    }
}

rectangle "Data Layer" as data #E8F5E9 {
    rectangle "Shard 0 (users 0, 4, 8...)" as s0 #C8E6C9 {
        database "Primary" as p0 #A5D6A7
        database "Replica" as r0a #81C784
        database "Replica" as r0b #81C784
    }

    rectangle "Shard 1 (users 1, 5, 9...)" as s1 #C8E6C9 {
        database "Primary" as p1 #A5D6A7
        database "Replica" as r1a #81C784
        database "Replica" as r1b #81C784
    }

    rectangle "Shard 2 (users 2, 6, 10...)" as s2 #C8E6C9 {
        database "Primary" as p2 #A5D6A7
        database "Replica" as r2a #81C784
        database "Replica" as r2b #81C784
    }

    rectangle "Shard 3 (users 3, 7, 11...)" as s3 #C8E6C9 {
        database "Primary" as p3 #A5D6A7
        database "Replica" as r3a #81C784
        database "Replica" as r3b #81C784
    }
}

svc --> router
router --> p0 : user_id=0
router --> p1 : user_id=1
router --> p2 : user_id=2
router --> p3 : user_id=3

p0 --> r0a : Async replication
p0 --> r0b
p1 --> r1a
p1 --> r1b
p2 --> r2a
p2 --> r2b
p3 --> r3a
p3 --> r3b

note bottom of data
  **Key Points:**
  - Each shard has its own replicas
  - Writes go to shard primary
  - Reads can go to replicas
  - No cross-shard JOINs!
end note

@enduml
```

---

## 5. Complete Resilience Pattern

```plantuml
@startuml
skinparam backgroundColor #FEFEFE
skinparam defaultFontSize 16
skinparam titleFontSize 22

title **Service-to-Service Resilience Stack**

participant "Service A" as svcA
participant "Timeout (5s max)" as timeout
participant "Retry (3 attempts)" as retry
participant "Circuit Breaker" as cb
participant "Bulkhead (10 concurrent)" as bulkhead
participant "Service B" as svcB

== Request Flow ==

svcA -> timeout : 1. Set deadline
timeout -> retry : 2. Wrap with retry
retry -> cb : 3. Check circuit

alt Circuit OPEN
    cb --> retry : Fail fast
    retry --> timeout : Return error
    timeout --> svcA : Fallback response
else Circuit CLOSED
    cb -> bulkhead : 4. Check concurrency

    alt Bulkhead full
        bulkhead --> cb : Reject (too many requests)
        cb --> retry : Trigger retry
    else Bulkhead OK
        bulkhead -> svcB : 5. Actual call

        alt Success
            svcB --> bulkhead : Response
            bulkhead --> cb : Success (reset failures)
            cb --> retry : Pass through
            retry --> timeout : Pass through
            timeout --> svcA : Success!
        else Failure/Timeout
            svcB --> bulkhead : Error
            bulkhead --> cb : Failure (increment count)
            cb --> retry : Trigger retry
            retry -> cb : Retry attempt 2...
        end
    end
end

note right of svcA
  **Resilience Stack Order:**
  1. Timeout - Don't wait forever
  2. Retry - Handle transient failures
  3. Circuit Breaker - Fail fast when broken
  4. Bulkhead - Limit concurrency

  **Libraries:**
  - Java: Resilience4j, Hystrix
  - Go: go-kit, sony/gobreaker
  - Node: cockatiel, opossum
end note

@enduml
```

---

## 6. Request Flow Through Sharded System

```plantuml
@startuml
skinparam backgroundColor #FEFEFE
skinparam defaultFontSize 16
skinparam titleFontSize 22

title **Request Flow: User 12345 Buys a Coupon**

actor "User (id=12345)" as user
participant "Load Balancer (Consistent Hash)" as lb
participant "Service Instance (for user 12345)" as svc
participant "Redis Cluster (Consistent Hash)" as redis
participant "Shard Router" as router
database "Shard 1 (12345 % 4 = 1)" as db

== Step 1: Route to Consistent Service ==
user -> lb : POST /coupons/claim
lb -> lb : hash(12345) % 10 = 5
lb -> svc : Route to instance 5
note right: Same user always hits\nsame instance (cache locality)

== Step 2: Check Cache ==
svc -> redis : GET coupon:available:ABC
redis -> redis : hash("coupon:ABC") -> Node 2
redis --> svc : 5 remaining

== Step 3: Claim with Retry ==
svc -> router : UPDATE coupons SET claimed...
router -> router : hash(12345) % 4 = 1
router -> db : Route to Shard 1

alt Success
    db --> router : 1 row affected
    router --> svc : Success
    svc -> redis : DECR coupon:available:ABC
    svc --> user : 200 OK - Coupon claimed!
else Contention (optimistic lock fail)
    db --> router : 0 rows affected
    router --> svc : Conflict
    svc -> svc : Retry with backoff
    note right: Exponential backoff\n100ms, 200ms, 400ms
end

@enduml
```

---

## Summary: When to Use Each Pattern

| Pattern | Problem It Solves | Example |
|---------|------------------|---------|
| **Consistent Hashing** | Minimize rehashing when nodes change | Redis cluster, DB shards |
| **Circuit Breaker** | Prevent cascade failures | Payment service down |
| **Retry + Backoff** | Handle transient failures | Network glitches |
| **Timeout** | Don't wait forever | Slow downstream |
| **Bulkhead** | Isolate failures | Limit concurrent calls |
| **Hash Sharding** | Scale writes | User data |
| **Range Sharding** | Time-based queries | Logs, events |
| **Entity Sharding** | Data locality | Ticketing by venue |

---

## Quick Reference: Resilience Configuration

```
Circuit Breaker:
  - Failure threshold: 5 consecutive failures
  - Open duration: 30 seconds
  - Half-open test: 1 request

Retry:
  - Max attempts: 3
  - Backoff: Exponential (100ms, 200ms, 400ms)
  - Jitter: ±50%
  - Retry on: 5xx, timeout, connection error
  - Don't retry: 4xx (client error)

Timeout:
  - HTTP call: 5 seconds
  - Database query: 3 seconds
  - Cache lookup: 100ms

Bulkhead:
  - Max concurrent: 10 per downstream
  - Queue size: 20
  - Queue timeout: 500ms
```

---

*Extended from Reference-Architecture.md with advanced patterns.*
