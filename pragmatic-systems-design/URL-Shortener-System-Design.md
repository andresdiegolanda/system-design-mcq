# URL Shortener System Design

A comprehensive guide to designing a URL shortening service like bit.ly or TinyURL.

## Table of Contents

1. [Problem Introduction](#problem-introduction)
2. [Requirements & Capacity Planning](#requirements--capacity-planning)
3. [Short Identifier Generation](#short-identifier-generation)
4. [Basic Design](#basic-design)
5. [Scaling Reads](#scaling-reads)
6. [Caching Strategies](#caching-strategies)
7. [Final Architecture](#final-architecture)

---

## Problem Introduction

A URL shortener converts long URLs into short, memorable links that redirect users to the original destination.

```plantuml
@startuml url-shortener-concept
!theme plain
skinparam backgroundColor #FEFEFE

title URL Shortener - Basic Flow

actor User as user
participant "URL Shortener\nService" as service #FF6B6B

== Creating a Short URL ==

user -> service : Submit long URL\nhttps://example.com/very/long/path/to/page
service --> user : Return short URL\nhttps://url.short/XyZ123

== Using a Short URL ==

user -> service : Visit https://url.short/XyZ123
service --> user : HTTP 301/302 Redirect\nto https://example.com/very/long/path/to/page

@enduml
```

### Benefits of Short URLs

| Benefit | Description |
|---------|-------------|
| **Easier to type** | Short codes are memorable and quick to enter |
| **Space efficient** | Ideal for character-limited platforms (Twitter, SMS) |
| **Trackable** | Can collect analytics on link clicks |
| **Clean appearance** | More professional in marketing materials |

---

## Requirements & Capacity Planning

### Clarifying Questions

Before designing, we need to understand the scale:

```plantuml
@startuml requirements-questions
!theme plain
skinparam backgroundColor #FEFEFE

rectangle "Questions to Ask" {
    card "How many new URLs\nper month?" as q1 #E8F5E9
    card "URL retention\nperiod?" as q2 #E3F2FD
    card "Read:Write\nratio?" as q3 #FFF3E0
}

rectangle "Answers (Assumptions)" {
    card "500 Million\nURLs/month" as a1 #C8E6C9
    card "5 Years" as a2 #BBDEFB
    card "100:1" as a3 #FFE0B2
}

q1 -right-> a1
q2 -right-> a2
q3 -right-> a3

@enduml
```

### Back-of-Envelope Calculations

#### Total URLs Over 5 Years

```
500M URLs/month x 12 months x 5 years = 30 Billion URLs
```

#### Write Throughput

```plantuml
@startuml write-calculations
!theme plain
skinparam backgroundColor #FEFEFE
skinparam defaultTextAlignment center

rectangle "Write Throughput Calculation" #F5F5F5 {
    card "500M URLs/month" as step1 #FFCDD2
    card "~20M URLs/day\n(500M / 30)" as step2 #FFCDD2
    card "~1M URLs/hour\n(20M / 24)" as step3 #FFCDD2
    card "~20K URLs/minute\n(1M / 60)" as step4 #FFCDD2
    card "**~300 URLs/second**" as step5 #EF5350
    card "Peak: **3K URLs/sec**\n(10x multiplier)" as peak #B71C1C
}

step1 -down-> step2
step2 -down-> step3
step3 -down-> step4
step4 -down-> step5
step5 -down-> peak

@enduml
```

#### Read Throughput

With a 100:1 read-to-write ratio:

| Metric | Average | Peak (10x) |
|--------|---------|------------|
| Writes | 300/sec | 3K/sec |
| Reads | 30K/sec | 300K/sec |

#### Storage Requirements

```plantuml
@startuml storage-calculation
!theme plain
skinparam backgroundColor #FEFEFE

rectangle "Storage Calculation" #E3F2FD {
    card "30 Billion URLs" as urls
    card "x 100 bytes avg per URL" as size
    card "= 3 Trillion bytes" as bytes
    card "= 3 TB total storage" as result #1976D2
}

urls -down-> size
size -down-> bytes
bytes -down-> result

note right of result
    This is manageable
    on a single large disk,
    but we'll need replication
    for availability
end note

@enduml
```

---

## Short Identifier Generation

The core challenge: generating unique, short, human-readable identifiers.

### Requirements for Short IDs

```plantuml
@startuml id-requirements
!theme plain
skinparam backgroundColor #FEFEFE

rectangle "ID Requirements" {
    card "Short\n(6-8 chars)" as short #FFCDD2
    card "Random\n(unpredictable)" as random #C8E6C9
    card "Human Readable\n(easy to type)" as readable #BBDEFB
}

@enduml
```

### Option Analysis

#### Option 1: UUID

```plantuml
@startuml uuid-analysis
!theme plain
skinparam backgroundColor #FEFEFE

rectangle "UUID Analysis" #FFF3E0 {
    card "Characters: 0-9 + a-f = 16" as chars
    card "Length: 32 hex characters" as length
    card "Keyspace: 16^32 = 2^128" as space
    card "Example:\na1b2c3d4-e5f6-7890-abcd-ef1234567890" as example #FFE0B2
}

rectangle "Verdict" #FFCDD2 {
    card "Short" as r1
    card "Random" as r2
    card "Human Readable" as r3
}

note right of r1 : FAIL - 32 chars is too long!
note right of r2 : PASS
note right of r3 : PASS

@enduml
```

**UUID Verdict**: Too long to type manually.

#### Option 2: MD5 Hash

```
MD5(URL) -> 128 bits -> Same size as UUID

Approach: substring(Base58(MD5(URL)), 8)

Problem: Same URL always produces same hash - NOT RANDOM!
         Different users shortening same URL get same ID.
```

**MD5 Verdict**: Not random, causes conflicts for same URLs.

#### Option 3: Base62 Encoding

```plantuml
@startuml base62-analysis
!theme plain
skinparam backgroundColor #FEFEFE

rectangle "Base62 Character Set" #E8F5E9 {
    card "a-z = 26 lowercase" as lower
    card "A-Z = 26 uppercase" as upper
    card "0-9 = 10 digits" as digits
    card "Total = 62 characters" as total #4CAF50
}

rectangle "Keyspace by Length" #E3F2FD {
    card "62^5 = 916 Million" as k5
    card "62^6 = 56 Billion" as k6 #2196F3
    card "62^7 = 3.5 Trillion" as k7
    card "62^8 = 218 Trillion" as k8
}

note right of k6 : 6 chars covers\n30B URLs with margin

@enduml
```

**Base62 Problem**: Characters like `0` vs `O` and `l` vs `1` are confusing!

#### Option 4: Base58 (Recommended)

```plantuml
@startuml base58-analysis
!theme plain
skinparam backgroundColor #FEFEFE

rectangle "Base58 - Removing Confusing Characters" #E8F5E9 {
    card "Base62 (62 chars)" as base62
    card "Remove: 0, O, I, l" as remove #FFCDD2
    card "Base58 (58 chars)" as base58 #4CAF50
}

base62 -down-> remove
remove -down-> base58

rectangle "Keyspace by Length" #E3F2FD {
    card "58^6 = 38 Billion" as k6 #2196F3
    card "58^7 = 2.2 Trillion" as k7
    card "58^8 = 128 Trillion" as k8
}

note right of k6 : 6 chars is enough\nfor 30B URLs!

@enduml
```

### Comparison Summary

| Method | Short | Random | Human Readable | Verdict |
|--------|:-----:|:------:|:--------------:|---------|
| UUID | No | Yes | Yes | Too long |
| MD5 | No | No | Yes | Not random |
| Base62 | Yes | Yes | No | Confusing chars |
| **Base58** | **Yes** | **Yes** | **Yes** | **Recommended** |

### Base58 Generation Algorithm

Two approaches to generate Base58 IDs:

```plantuml
@startuml base58-generation
!theme plain
skinparam backgroundColor #FEFEFE

rectangle "Approach 1: UUID-based" #E3F2FD {
    card "1. Generate UUID" as a1
    card "2. Convert to Base58" as a2
    card "3. Take first 6 characters" as a3
    card "substring(Base58(UUID), 6)" as code1 #BBDEFB
}

a1 -down-> a2
a2 -down-> a3
a3 -down-> code1

rectangle "Approach 2: Random Generation" #E8F5E9 {
    card "For i = 1 to 6:" as b1
    card "  Pick random(0, 57)" as b2
    card "  Append Base58 char" as b3
    card "Direct random selection" as code2 #C8E6C9
}

b1 -down-> b2
b2 -down-> b3
b3 -down-> code2

@enduml
```

---

## Basic Design

### URL Shortening Flow

```plantuml
@startuml shortening-flow
!theme plain
skinparam backgroundColor #FEFEFE

participant "Client" as client
participant "URL Shortener\nService" as service #4A90D9
participant "ID Generator" as generator #F39C12
database "Storage" as db #27AE60

client -> service : PUT /shorten\n{ url: "https://long-url.com/..." }
service -> generator : Generate unique ID
generator --> service : "XyZ123"
service -> db : Store(XyZ123, long_url)
db --> service : Success
service --> client : { short_url: "https://url.short/XyZ123" }

@enduml
```

### URL Redirection Flow

```plantuml
@startuml redirection-flow
!theme plain
skinparam backgroundColor #FEFEFE

participant "Client\nBrowser" as client
participant "URL Shortener\nService" as service #4A90D9
database "Storage" as db #27AE60

client -> service : GET /XyZ123
service -> db : Lookup("XyZ123")
db --> service : "https://long-url.com/..."

alt URL Found
    service --> client : HTTP 301/302 Redirect\nLocation: https://long-url.com/...
else URL Not Found
    service --> client : HTTP 404 Not Found
end

@enduml
```

### Redirect Types: 301 vs 302

```plantuml
@startuml redirect-comparison
!theme plain
skinparam backgroundColor #FEFEFE

rectangle "301 Permanent Redirect" #E8F5E9 {
    card "Browser caches the redirect" as p1
    card "Subsequent visits skip our service" as p2
    card "Lower server load" as p3
    card "Cannot track repeat visits" as p4 #FFCDD2
}

rectangle "302 Temporary Redirect" #E3F2FD {
    card "Browser always hits our service" as t1
    card "Every visit goes through us" as t2
    card "Higher server load" as t3
    card "Can track all visits" as t4 #C8E6C9
}

note bottom of p4 : Use for simple\nredirection only

note bottom of t4 : Use when analytics\nare important

@enduml
```

---

## Scaling Reads

With 30K reads/second (300K peak), we need to scale our read path.

### Option 1: Read Replicas

```plantuml
@startuml read-replicas
!theme plain
skinparam backgroundColor #FEFEFE

rectangle "URL Shortener\nService" as service #4A90D9

database "Primary\nRDBMS" as primary #27AE60
database "Read Replica 1" as replica1 #A9DFBF
database "Read Replica 2" as replica2 #A9DFBF
database "Read Replica 3" as replica3 #A9DFBF
database "Read Replica 4" as replica4 #A9DFBF
database "Read Replica 5" as replica5 #A9DFBF
database "Read Replica 6" as replica6 #A9DFBF

service -down-> primary : Writes
primary -down-> replica1 : Replication
primary -down-> replica2
primary -down-> replica3
primary -down-> replica4
primary -down-> replica5
primary -down-> replica6

service -right-> replica1 : Reads\n~10K rps each
service -right-> replica2
service -right-> replica3

note bottom of replica6
    6 replicas x 10K rps = 60K rps
    Each holds full 3TB dataset
end note

@enduml
```

**Trade-off**: Each replica stores 3TB - expensive!

### Option 2: Redis Cluster

```plantuml
@startuml redis-cluster
!theme plain
skinparam backgroundColor #FEFEFE

rectangle "URL Shortener\nService" as service #4A90D9

database "Redis 1" as redis1 #E74C3C
database "Redis 2" as redis2 #E74C3C
database "Redis 3" as redis3 #E74C3C

service -down-> redis1 : 100K rps
service -down-> redis2 : 100K rps
service -down-> redis3 : 100K rps

note bottom of redis2
    3 Redis nodes x 100K rps = 300K rps
    But storing 3TB in Redis is expensive!
    64GB per node = 50 nodes needed
end note

@enduml
```

**Trade-off**: Redis is fast but storing 3TB is expensive.

### Option 3: DynamoDB (Managed NoSQL)

```plantuml
@startuml dynamodb-option
!theme plain
skinparam backgroundColor #FEFEFE

rectangle "URL Shortener\nService" as service #4A90D9

database "DynamoDB" as dynamo #F39C12 {
    card "Auto-sharded" as shard
    card "Managed scaling" as scale
    card "300K+ rps capable" as rps
}

service -right-> dynamo : Direct queries\n300K rps

note bottom of dynamo
    Pros:
    - Key-value fits perfectly
    - Auto-sharding built-in
    - Handles 300K rps easily

    Cons:
    - Proprietary (AWS lock-in)
    - Cost scales with usage
end note

@enduml
```

---

## Caching Strategies

### The 1% Rule

Most URL shorteners follow a power law distribution - a small percentage of URLs receive most of the traffic.

```plantuml
@startuml caching-principle
!theme plain
skinparam backgroundColor #FEFEFE

rectangle "The 1% Hot Data Principle" #FFF3E0 {
    card "Total Data: 3TB" as total
    card "Hot Data (1%): 30GB" as hot #FF9800
    card "30GB fits in memory!" as fits #4CAF50
}

total -down-> hot
hot -down-> fits

note right of hot
    Only ~1% of URLs
    receive most traffic.
    Cache these!
end note

@enduml
```

### Multi-Tier Caching Architecture

```plantuml
@startuml multi-tier-cache
!theme plain
skinparam backgroundColor #FEFEFE

participant "Client" as client
participant "URL Shortener" as service #4A90D9
participant "In-Memory\nCache" as inmem #F39C12
participant "Redis\nCache" as redis #E74C3C
database "RDBMS" as db #27AE60

client -> service : GET /XyZ123

service -> inmem : Check local cache
alt In-Memory HIT
    inmem --> service : Found!
    service --> client : Redirect
else In-Memory MISS
    service -> redis : Check Redis
    alt Redis HIT
        redis --> service : Found!
        service -> inmem : Update local cache
        service --> client : Redirect
    else Redis MISS
        service -> db : Query database
        db --> service : URL data
        service -> redis : Update Redis
        service -> inmem : Update local cache
        service --> client : Redirect or 404
    end
end

@enduml
```

### Cache Sizing

```plantuml
@startuml cache-sizing
!theme plain
skinparam backgroundColor #FEFEFE

rectangle "Cache Configuration" #E3F2FD {

    rectangle "In-Memory Cache\n(per service instance)" #FFF3E0 {
        card "Size: 1-2 GB" as inmem_size
        card "Speed: Nanoseconds" as inmem_speed
        card "Scope: Local only" as inmem_scope
    }

    rectangle "Redis Cluster\n(shared cache)" #FFEBEE {
        card "Size: 30GB total\n(~8GB x 4 nodes)" as redis_size
        card "Speed: Microseconds" as redis_speed
        card "Scope: Shared across services" as redis_scope
    }
}

note bottom of redis_size
    4 Redis nodes handles:
    - 300K rps / 4 = 75K rps each
    - Well within Redis capacity
end note

@enduml
```

---

## Final Architecture

```plantuml
@startuml final-architecture
!theme plain
skinparam backgroundColor #FEFEFE

rectangle "Clients" as clients {
    actor "Web" as web
    actor "Mobile" as mobile
    actor "API" as api
}

rectangle "Load Balancer" as lb #9B59B6

rectangle "URL Shortener Service\n(Multiple Instances)" as services #4A90D9 {
    rectangle "Instance 1" as s1 {
        card "Service" as svc1
        card "In-Memory\nCache" as cache1 #F39C12
    }
    rectangle "Instance 2" as s2 {
        card "Service" as svc2
        card "In-Memory\nCache" as cache2 #F39C12
    }
    rectangle "Instance N" as sn {
        card "Service" as svcn
        card "In-Memory\nCache" as cachen #F39C12
    }
}

rectangle "Redis Cache Cluster" as redis #E74C3C {
    database "Redis 1" as r1
    database "Redis 2" as r2
    database "Redis 3" as r3
    database "Redis 4" as r4
}

rectangle "Database Layer" as dblayer #27AE60 {
    database "Primary\nRDBMS" as primary
    database "Replica 1" as rep1
    database "Replica 2" as rep2
    database "Replica 3" as rep3
    database "Replica 4" as rep4
    database "Replica 5" as rep5
    database "Replica 6" as rep6
}

clients -down-> lb
lb -down-> services
services -down-> redis : Cache\nMisses
redis -down-> dblayer : Cache\nMisses
primary -right-> rep1 : Replication
primary -right-> rep2
primary -right-> rep3

note right of services
    **Traffic Summary**
    Writes: 300 rps (3K peak)
    Reads: 30K rps (300K peak)
end note

note right of redis
    **Cache Layer**
    30GB total (1% of data)
    4 nodes for redundancy
end note

note right of dblayer
    **Storage Layer**
    3TB total data
    6 read replicas
end note

@enduml
```

### System Specifications Summary

| Component | Specification | Purpose |
|-----------|--------------|---------|
| **ID Format** | Base58, 6 characters | Human-readable short codes |
| **Writes** | 300 rps (3K peak) | URL creation |
| **Reads** | 30K rps (300K peak) | URL redirection |
| **Storage** | 3TB (30B URLs) | Persistent URL storage |
| **Cache** | 30GB Redis (1% hot data) | Fast reads |
| **Replicas** | 6x read replicas | Read scaling |

---

## Advanced Topics (Interview Deep Dives)

### Topics Not Covered in This Design

The following are important for senior-level interviews:

```plantuml
@startuml advanced-topics
!theme plain
skinparam backgroundColor #FEFEFE

rectangle "Missing from Basic Design" #FFEBEE {
    card "Collision Handling" as collision #EF5350
    card "API Design" as api #EF5350
    card "URL Expiration/TTL" as ttl #EF5350
    card "Analytics & Tracking" as analytics #EF5350
    card "Rate Limiting" as ratelimit #EF5350
    card "Security (malicious URLs)" as security #EF5350
}

note right of collision
    What happens if two IDs collide?
    - Retry with new ID
    - Use DB uniqueness constraint
    - Pre-generate keys (KGS pattern)
end note

note right of api
    POST /api/v1/shorten
    GET /{shortCode}
    DELETE /api/v1/url/{shortCode}
end note

note right of ttl
    - TTL field in database
    - Background cleanup job
    - Lazy deletion on access
end note

@enduml
```

### Key Generation Service (Advanced Pattern)

For guaranteed uniqueness without collision checks:

```plantuml
@startuml kgs-pattern
!theme plain
skinparam backgroundColor #FEFEFE

rectangle "Key Generation Service (KGS)" #E8F5E9 {
    database "Unused Keys\nTable" as unused #C8E6C9
    database "Used Keys\nTable" as used #FFCDD2
    rectangle "KGS Server" as kgs #4CAF50
}

rectangle "URL Shortener\nService" as service #4A90D9

kgs -> unused : Pre-generate\nmillions of keys
service -> kgs : Request key
kgs -> unused : Mark as used
kgs -> used : Move to used table
kgs --> service : Return key "XyZ123"

note bottom of kgs
    Benefits:
    - No collision possible
    - No duplicate checks needed
    - Pre-computed = fast
end note

@enduml
```

---

## Interview Checklist

Use this checklist to ensure comprehensive coverage:

- [ ] **Requirements**: Clarify scale (URLs/month, retention, read:write ratio)
- [ ] **Back-of-envelope**: Calculate storage, throughput, bandwidth
- [ ] **ID Generation**: Explain Base58 choice and alternatives
- [ ] **API Design**: Define REST endpoints with parameters
- [ ] **Database**: Choose SQL vs NoSQL with justification
- [ ] **Caching**: Multi-tier strategy with sizing
- [ ] **Scaling**: Read replicas, sharding considerations
- [ ] **Collision Handling**: Retry logic or KGS pattern
- [ ] **Analytics**: Click tracking, geographic data
- [ ] **Security**: Rate limiting, malicious URL detection
- [ ] **Availability**: Failover, cross-region replication
