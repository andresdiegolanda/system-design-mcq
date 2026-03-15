# News Feed System Design

## Table of Contents
1. [Introduction](#introduction)
2. [Requirements Gathering](#requirements-gathering)
3. [Back-of-Envelope Calculations](#back-of-envelope-calculations)
4. [Basic Design](#basic-design)
5. [Database Schema](#database-schema)
6. [Scaling Reads](#scaling-reads)
7. [Storage Architecture](#storage-architecture)
8. [Feed Optimization with Fan-Out](#feed-optimization-with-fan-out)
9. [The VIP/Celebrity Problem](#the-vipcelebrity-problem)
10. [Final Architecture](#final-architecture)

---

## Introduction

A **News Feed** is a core feature of social media platforms like Twitter, Facebook, and Instagram. When a user posts content, that content needs to appear in the feeds of all their followers. This seemingly simple requirement becomes extremely complex at scale.

### How News Feed Works

```plantuml
@startuml
skinparam backgroundColor #FEFEFE
skinparam defaultFontSize 14

actor "User A" as A
hexagon "News Feed\nSystem" as NFS
actor "User B\n(Follower)" as B
actor "User C\n(Follower)" as C

A -> NFS : Posts "Doing system design!"
NFS -> B : Feed shows A's post
NFS -> C : Feed shows A's post

note right of B
  **B's Feed:**
  - A: Doing system design!
  - C: Excellent weather
end note

note right of C
  **C's Feed:**
  - A: Doing system design!
  - B: Ate a donut...
end note
@enduml
```

The challenge is delivering these updates to potentially millions of followers with low latency while handling massive read and write volumes.

---

## Requirements Gathering

Before designing, we need to clarify the scale and constraints. The key questions to ask:

| Question | Answer | Impact |
|----------|--------|--------|
| How many new tweets per second? | **10,000 tweets/sec** | Determines write throughput |
| What's the read-to-write ratio? | **30:1** | Reads dominate; optimize for reads |
| Average followers per user? | **100 followers** | Affects fan-out calculations |
| Maximum followers per user? | **1 million followers** | Introduces VIP/celebrity problem |

### Derived Metrics

- **Write throughput**: 10K tweets/second
- **Read throughput**: 10K × 30 = **300K reads/second**
- **Timeline writes per second** (fan-out): 10K × 100 = **1M timeline updates/second**

---

## Back-of-Envelope Calculations

### Tweet Volume

```
10K tweets/second × 60 = 600K tweets/minute
600K tweets/minute × 60 = 36M tweets/hour
36M tweets/hour × 24 = 864M tweets/day
```

### Storage Requirements

Assuming **100 bytes per tweet** (user_id, message, timestamp):

```
Daily:   864M × 100 bytes = 86.4 GB/day
Yearly:  86.4 GB × 365 = ~31 TB/year
```

```plantuml
@startuml
skinparam backgroundColor #FEFEFE

rectangle "Storage Growth" {
  card "Daily\n86 GB" as daily
  card "Monthly\n2.6 TB" as monthly
  card "Yearly\n31 TB" as yearly
}

daily -right-> monthly
monthly -right-> yearly
@enduml
```

---

## Basic Design

### Tweet Creation Flow

When a user posts a tweet, it's stored in the database:

```plantuml
@startuml
skinparam backgroundColor #FEFEFE

participant "Mobile\nClient" as client
participant "News Feed\nService" as service
database "Database" as db

== Tweet Creation ==
client -> service : POST /tweet\n{message: "Hello World"}
service -> db : INSERT tweet
db --> service : Success
service --> client : 201 Created

@enduml
```

### Feed Retrieval Flow (Naive Approach)

Followers poll for updates periodically:

```plantuml
@startuml
skinparam backgroundColor #FEFEFE

participant "Follower\nClient" as client
participant "News Feed\nService" as service
database "Database" as db

== Feed Retrieval ==
client -> service : GET /tweets?limit=10
service -> db : Query tweets from\nfollowed users
db --> service : Tweet list
service --> client : JSON response

note right of client
  Polls every minute
end note
@enduml
```

**Problem**: This approach requires complex queries joining tweets with follows for every read request - 300K times per second!

---

## Database Schema

### Tweets Table

| Column | Type | Description |
|--------|------|-------------|
| user_id | BIGINT | Author of the tweet |
| message | VARCHAR(280) | Tweet content |
| timestamp | DATETIME | When tweet was created |

### Follows Table

| Column | Type | Description |
|--------|------|-------------|
| source_id | BIGINT | User who follows |
| target_id | BIGINT | User being followed |

```plantuml
@startuml
skinparam backgroundColor #FEFEFE

entity "Tweets" as tweets {
  * tweet_id : BIGINT <<PK>>
  --
  * user_id : BIGINT <<FK>>
  * message : VARCHAR(280)
  * timestamp : DATETIME
}

entity "Users" as users {
  * user_id : BIGINT <<PK>>
  --
  username : VARCHAR(50)
  is_vip : BOOLEAN
}

entity "Follows" as follows {
  * source_id : BIGINT <<FK>>
  * target_id : BIGINT <<FK>>
  --
  created_at : DATETIME
}

users ||--o{ tweets : writes
users ||--o{ follows : "source (follower)"
users ||--o{ follows : "target (followed)"
@enduml
```

---

## Scaling Reads

With 300K reads per second, a single database won't suffice. Let's evaluate our options:

### Option 1: Read Replicas

```
300K reads/second ÷ 20K reads/replica = 15 replicas
300K reads/second ÷ 50K reads/replica = 6 replicas
```

| Pros | Cons |
|------|------|
| Simple architecture | High cost (full data copies) |
| Easy to implement | High space requirements |
| Strong consistency | Replication lag |

**Verdict**: Not ideal due to cost and space requirements.

### Option 2: Sharding

```
300K reads/second ÷ 10K reads/shard = 30 shards
300K reads/second ÷ 25K reads/shard = 12 shards
```

With 30TB of data:
- 30 shards → 1TB per shard
- 12 shards → 3TB per shard

| Pros | Cons |
|------|------|
| Less costly than replicas | Architectural complexity |
| Each shard is smaller | Cross-shard queries are hard |
| Scales horizontally | Resharding is painful |

### Option 3: Caching (Recommended)

```
300K reads/second ÷ 100K reads/instance = 3 Redis instances
```

```plantuml
@startuml
skinparam backgroundColor #FEFEFE

participant "Client" as client
participant "News Feed\nService" as service
database "Cache\n(Redis)" as cache
database "Database" as db

== Cache-Aside Pattern ==
client -> service : GET /tweets
service -> cache : Check cache
alt Cache Hit
  cache --> service : Return tweets
else Cache Miss
  service -> db : Query tweets
  db --> service : Tweet data
  service -> cache : Populate cache
end
service --> client : Tweet response
@enduml
```

| Pros | Cons |
|------|------|
| Fewest instances needed | More complex than replicas |
| Sub-millisecond reads | Cache invalidation challenges |
| Cost effective | Data consistency concerns |

**Verdict**: Caching is the best option for this read-heavy workload.

---

## Storage Architecture

### Short-Term vs Long-Term Storage

Most users only care about recent tweets (last 5 days). We can optimize by separating hot and cold data:

```plantuml
@startuml
skinparam backgroundColor #FEFEFE

participant "Client" as client
participant "News Feed\nService" as service
database "Redis Cache\n(5 days)" as cache
participant "Storage Writer" as writer
database "File System" as fs

== Write Path ==
client -> service : POST /tweet
service -> cache : Write to cache
cache --> writer : Async read new entries
writer -> fs : Persist to file system

note right of fs : Organized by /<user-id>/<date>
@enduml
```

### Cache Configuration

- **Cache size**: 86GB/day × 5 days = ~430GB hot data
- **Cluster**: 3 instances × 32GB = 96GB (storing ~1% hot data)
- **File system**: Long-term storage organized by `/<user-id>/<date>`

---

## Feed Optimization with Fan-Out

### The Problem with Pull-Based Feeds

When User A requests their feed:
1. Query Follows DB: "Who does A follow?"
2. For each followed user, query their tweets
3. Merge and sort all tweets
4. Apply limit/offset

**Issue**: More people you follow = more cache requests per feed load.

```plantuml
@startuml
skinparam backgroundColor #FEFEFE

database "Follows DB" as follows {
}
database "Tweets Cache" as cache {
}

note right of follows
  | user-id | follows |
  |---------|---------|
  | Luke    | Leia    |
  | Luke    | Han     |
  | Han     | Leia    |
end note

note right of cache
  luke => [luke-tweets]
  leia => [leia-tweets]
  han => [han-tweets]
end note

rectangle "Feed Generation Steps" as steps {
  card "1. Get Leia's tweets" as s1
  card "2. Get Han's tweets" as s2
  card "3. Merge results" as s3
  card "4. Apply limit/offset" as s4
}

s1 -down-> s2
s2 -down-> s3
s3 -down-> s4
@enduml
```

### Fan-Out on Write (Push Model)

Instead of computing feeds on read, **pre-compute them on write**:

```plantuml
@startuml
skinparam backgroundColor #FEFEFE

participant "User A" as userA
participant "News Feed\nService" as service
database "Follows DB" as follows
database "Timeline Cache" as cache

== Fan-Out on Write ==
userA -> service : POST /tweet\n"Hello World"
service -> follows : Get A's followers
follows --> service : [B, C, D, ...]
loop For each follower
  service -> cache : Append tweet to\nfollower's timeline
end
service --> userA : 201 Created
@enduml
```

### Pre-Computed Timelines

After fan-out, each user's timeline is pre-built:

| User | Timeline Cache |
|------|---------------|
| Luke | [leia-tweets, han-tweets] |
| Han | [leia-tweets] |
| Leia | [] |

**Result**: **100x read optimization** on average! Instead of N cache lookups (one per followed user), just 1 lookup for the pre-computed timeline.

---

## The VIP/Celebrity Problem

### The Problem

With fan-out on write, a single VIP tweet triggers massive writes:

```
Normal load:
  10,000 tweets/sec × 100 followers = 1,000,000 timeline writes/sec

VIP with 1M followers posts:
  1 tweet × 1,000,000 followers = 1,000,000 timeline writes
  = 2x spike instantly!
```

```plantuml
@startuml
skinparam backgroundColor #FEFEFE

participant "VIP\n(1M followers)" as vip
participant "News Feed\nService" as service
database "Cache" as cache
participant "Followers" as followers

== VIP Tweet Problem ==
vip -> service : POST /tweet
service -> cache : Single write
note right
  But to fan-out:
  1M timeline updates!
end note

followers -> service : GET /tweets
service -> cache : 1M reads in total
cache --> followers : Timeline
@enduml
```

### VIP Statistics

According to Wikipedia, only **25 accounts** have more than 50M followers (as of 2021). VIPs are rare but impactful.

### Solution: Hybrid Fan-Out

Use **different strategies** for regular users vs VIPs:

```plantuml
@startuml
skinparam backgroundColor #FEFEFE

start
:New Tweet Arrives;

if (Is user in VIP list?) then (Yes)
  :Store in VIP Storage;
  note right: No fan-out\n(pull on read)
else (No)
  :Fan-out to each\nfollower's timeline;
  note right: Push to all\nfollower caches
  :Store in Regular\nUser Cache;
endif

stop
@enduml
```

### Hybrid Feed Generation

When a user requests their feed:

```plantuml
@startuml
skinparam backgroundColor #FEFEFE

participant "Feed\nClient" as client
participant "News Feed\nService" as service
database "Feed Cache\n(Pre-computed)" as cache
database "Followers DB" as follows
database "VIP Storage" as vip

== Hybrid Feed Retrieval ==
client -> service : GET /feed
service -> cache : Get regular updates
cache --> service : Regular tweets

service -> follows : Which VIPs do I follow?
follows --> service : List of VIP IDs

service -> vip : Get VIP tweets
vip --> service : VIP tweets

service -> service : Merge results
service --> client : Combined feed
@enduml
```

---

## Final Architecture

The complete News Feed system combines all optimization strategies:

```plantuml
@startuml
skinparam backgroundColor #FEFEFE
skinparam defaultFontSize 12

actor "New Tweet\nAuthor" as author
actor "Feed\nReader" as reader
rectangle "News Feed Service" as service

database "Follows\nDB" as follows
database "Feed Cache\n(Redis)\nPre-computed\nTimelines" as cache
database "VIP\nTweets" as vip
rectangle "Storage\nWriter" as writer
database "File System\nLong-term\nStorage" as fs

author --> service : POST /tweets
reader --> service : GET /tweets?limit=10

service --> follows : Query relationships
service --> cache : Read/Write timelines
service --> vip : Read/Write VIP tweets

cache ..> writer : Async drain
writer --> fs : Persist old tweets
@enduml
```

### Component Summary

| Component | Purpose | Technology |
|-----------|---------|------------|
| News Feed Service | API gateway, business logic | Stateless microservice |
| Follows DB | User relationships | PostgreSQL/MySQL (sharded) |
| Feed Cache | Pre-computed timelines | Redis Cluster |
| VIP Tweets | Celebrity content (pull-based) | Redis/Cassandra |
| Storage Writer | Async persistence | Background workers |
| File System | Long-term archive | S3/HDFS |

### Key Design Decisions

1. **Fan-out on write** for regular users (optimize reads)
2. **Fan-out on read** for VIPs (avoid write amplification)
3. **Tiered storage**: Hot data in cache, cold data in file system
4. **Caching over sharding** for read scalability
5. **Pre-computed timelines** for 100x read optimization

---

## Key Takeaways

| Challenge | Solution |
|-----------|----------|
| 300K reads/second | Redis cache cluster (3 instances) |
| 31TB/year storage | Tiered storage (cache + file system) |
| Feed computation cost | Pre-computed timelines (fan-out on write) |
| VIP spike problem | Hybrid fan-out (push for regular, pull for VIP) |
| Data freshness | Polling every minute + eventual consistency |

### Trade-offs Accepted

- **Eventual consistency**: Timelines may lag slightly behind
- **Complexity**: Hybrid approach adds architectural overhead
- **Storage duplication**: Same tweet stored in multiple timelines

This design handles Twitter-scale load while remaining cost-effective and maintainable.
