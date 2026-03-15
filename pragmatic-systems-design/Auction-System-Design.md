# Auction System Design

> Based on "Design an auction" by Alexey Soshin

## Table of Contents

1. [Problem Introduction](#problem-introduction)
2. [Requirements Gathering](#requirements-gathering)
3. [Basic Design](#basic-design)
4. [The Concurrency Problem](#the-concurrency-problem)
5. [Optimistic Locking](#optimistic-locking)
6. [Serialization with Queues](#serialization-with-queues)
7. [Event-Driven Architecture](#event-driven-architecture)
8. [Client Communication Protocols](#client-communication-protocols)
9. [Scaling Calculations](#scaling-calculations)
10. [Final Architecture](#final-architecture)

---

## Problem Introduction

### What is an Auction System?

An auction system allows users to bid on items, with the highest bidder winning when the auction ends. The core challenge is handling **concurrent bids** while ensuring **fairness** and **data consistency**.

```plantuml
@startuml auction-concept
!theme plain
skinparam backgroundColor #FEFEFE

start
:User sees **Current Price: $3**;
:User enters **New Bid: $4**;
:User clicks **BID**;

if (Highest bidder?) then (Yes)
    #2ECC71:Congratulations!\nYou're winning!;
else (No)
    #E74C3C:Outbid!\nTry again;
endif

stop

@enduml
```

### The Basic Auction Flow

1. User sees current highest price
2. User enters a higher bid amount
3. System validates if bid is higher than current price
4. If yes: User becomes highest bidder
5. If no: User is notified they've been outbid

---

## Requirements Gathering

### Key Questions to Ask

Before designing any system, you must understand the scale and constraints:

| Question | Why It Matters |
|----------|----------------|
| How many simultaneous auctions? | Determines database and service capacity |
| How many bidders per auction? | Affects concurrency handling |
| Average bids per auction? | Determines write throughput |
| Auction duration? | Affects how bids are distributed over time |

### Our Requirements

| Parameter | Value |
|-----------|-------|
| Simultaneous auctions | **100,000** |
| Bidders per auction | **10 - 1,000** |
| Average bids per user | **5** |
| Auction duration | **24 hours** |

These numbers will drive our architectural decisions and help us calculate required throughput.

---

## Basic Design

### Initial Architecture

The simplest design uses a synchronous HTTP-based approach:

```plantuml
@startuml basic-design
!theme plain
skinparam backgroundColor #FEFEFE

actor "User" as user
boundary "HTTP" as http #D5DBDB
rectangle "Bids Service" as service #4A90D9
database "Database" as db #3498DB {
    rectangle "item_id | price" as table
}

user -> http : POST /items/:id/bids
http -> service
service -> db

@enduml
```

**Components:**
- **Client**: Web browser or mobile app
- **Bids Service**: Handles bid validation and storage
- **Database**: Stores auction items and current prices

### Basic Bid Flow

```plantuml
@startuml basic-flow
!theme plain
skinparam backgroundColor #FEFEFE

participant "Client" as client
participant "Bids Service" as service #4A90D9
database "Database" as db #3498DB

client -> service : New bid ($4)
service -> db : Get current top price
db --> service : Current price: $3

alt bid > current price
    service -> db : Update top price to $4
    service --> client : "You are the highest bidder!"
else bid <= current price
    service --> client : "Sorry, you've been outbid!"
end

@enduml
```

**The Logic:**
1. Receive new bid from client
2. Query database for current highest price
3. Compare new bid against current price
4. If higher: update database and confirm
5. If lower: reject and notify user

---

## The Concurrency Problem

### Why Simple Design Fails

The basic design has a critical flaw: **race conditions**. When two users bid simultaneously, both may read the same "current price" before either update completes.

```plantuml
@startuml race-condition
!theme plain
skinparam backgroundColor #FEFEFE

participant "Rick" as rick
participant "Bids Service\n(Instance 1)" as service1 #4A90D9
database "Database" as db #3498DB
participant "Bids Service\n(Instance 2)" as service2 #4A90D9
participant "Morty" as morty

rick -> service1 : Bid $3
morty -> service2 : Bid $5

service1 -> db : Get current price
db --> service1 : $2

service2 -> db : Get current price
db --> service2 : $2

note over db : Both see $2 as current price!

service2 -> db : Update to $5
service1 -> db : Update to $3

note over db #FFAAAA : Final price: $3\nMorty's $5 bid was lost!

service1 --> rick : "You are the highest bidder!"
service2 --> morty : "You are the highest bidder!"

note over rick, morty #FFAAAA : Both think they won!\nBut Rick's lower bid overwrote Morty's

@enduml
```

### The Concurrency Error Window

```plantuml
@startuml error-window
!theme plain
skinparam backgroundColor #FEFEFE

participant "Client" as client
participant "Bids Service" as service #4A90D9
database "Database" as db #3498DB

client -> service : New bid
service -> db : Get current top price
db --> service : Return price

note right of service #FFAAAA
    **DANGER ZONE**

    Between reading the price
    and updating it, another
    bid could change the value!

    This is the "Concurrency
    Error Window"
end note

service -> service : Is new bid higher?

alt Yes
    service -> db : Update the top price
    service --> client : Success!
else No
    service --> client : Outbid!
end

@enduml
```

### What Goes Wrong?

1. **Lost Updates**: Higher bids get overwritten by lower ones
2. **False Confirmations**: Users told they're winning when they're not
3. **Data Inconsistency**: Database state doesn't reflect actual bid order

---

## Optimistic Locking

### The Solution: Conditional Updates

Instead of blindly updating, we **include the expected current value** in our update query:

```sql
UPDATE bids
SET price = 3
WHERE item_id = ? AND price = 2
```

This query will only succeed if the price is still what we expected ($2). If someone else changed it, **zero rows are updated**.

```plantuml
@startuml optimistic-locking
!theme plain
skinparam backgroundColor #FEFEFE

participant "Client" as client
participant "Bids Service" as service #4A90D9
database "Database" as db #3498DB

client -> service : New bid ($3)
service -> db : Get current top price
db --> service : Current price: $2

service -> service : Is $3 > $2? Yes!

service -> db : UPDATE SET price=3\nWHERE item_id=? AND price=2

alt 1 row updated
    db --> service : Success (1 row)
    service --> client : "You are the highest bidder!"
else 0 rows updated
    db --> service : Failed (0 rows)
    note right: Someone else updated\nthe price first!
    service --> client : "Sorry, you've been outbid!"
end

@enduml
```

### How Optimistic Locking Resolves Conflicts

```plantuml
@startuml optimistic-resolution
!theme plain
skinparam backgroundColor #FEFEFE

participant "Rick" as rick
participant "Bids Service\n(Instance 1)" as service1 #4A90D9
database "Database" as db #3498DB
participant "Bids Service\n(Instance 2)" as service2 #4A90D9
participant "Morty" as morty

rick -> service1 : Bid $5
morty -> service2 : Bid $3

service1 -> db : Get current price
db --> service1 : $2

service2 -> db : Get current price
db --> service2 : $2

service2 -> db : UPDATE price=3\nWHERE price=2
db --> service2 : 1 row updated
note over db : Price is now $3

service1 -> db : UPDATE price=5\nWHERE price=2
db --> service1 : 0 rows updated
note over db #AAFFAA : Price stays $3\n(condition failed)

service1 --> rick : "Sorry, you've been outbid!"
service2 --> morty : "You are the highest bidder!"

@enduml
```

### The Remaining Problem

Optimistic locking prevents data corruption, but creates a **fairness issue**:
- Rick bid $5 (higher!) but lost
- Morty bid $3 (lower) but won
- The **order of arrival** matters, not just the **bid amount**

This happens because Rick's update arrived *after* Morty's completed, even though Rick submitted a higher bid.

---

## Serialization with Queues

### Guaranteeing Order with Message Queues

To ensure fair ordering, we introduce a **message queue** between the service and database:

```plantuml
@startuml queue-architecture
!theme plain
skinparam backgroundColor #FEFEFE

actor "Rick\n($5)" as rick
actor "Morty\n($3)" as morty

rectangle "Bids Service" as service #4A90D9
queue "Bids Queue" as queue #9B59B6
rectangle "Serializer" as serializer #4A90D9
database "DB" as db #3498DB

rick --> service
morty --> service
service --> queue
queue --> serializer
serializer --> db

note bottom of queue
    Messages are processed
    in order of arrival
end note

@enduml
```

### How Queue Serialization Works

```plantuml
@startuml queue-processing
!theme plain
skinparam backgroundColor #FEFEFE

actor "Rick ($5)" as rick #E67E22
actor "Morty ($3)" as morty #F1C40F

rectangle "Bids Service" as service #4A90D9
queue "Bids Queue" as queue #9B59B6 {
    rectangle "5" as bid1 #E67E22
    rectangle "3" as bid2 #F1C40F
}
rectangle "Serializer" as serializer #4A90D9
database "DB" as db #3498DB

rick --> service
morty --> service
service --> queue

note right of queue
    Bids enter queue in
    arrival order:
    1. Rick's $5 (first)
    2. Morty's $3 (second)
end note

queue --> serializer : Process one at a time
serializer --> db

note bottom of serializer
    Serializer processes sequentially:
    1. Rick's $5 -> Accepted (5 > current)
    2. Morty's $3 -> Rejected (3 < 5)
end note

@enduml
```

### Benefits of Queue-Based Serialization

| Benefit | Explanation |
|---------|-------------|
| **Ordering Guarantee** | First-come, first-served processing |
| **Fairness** | Higher bid that arrived first wins |
| **Decoupling** | Service doesn't wait for DB operations |
| **Backpressure** | Queue absorbs traffic spikes |
| **Retry Capability** | Failed operations can be reprocessed |

---

## Event-Driven Architecture

### From Synchronous to Asynchronous

The synchronous approach blocks the client until processing completes. With an **event-driven** design, we can respond faster and handle higher load.

#### Synchronous (Before)

```plantuml
@startuml sync-approach
!theme plain
skinparam backgroundColor #FEFEFE

actor "User" as user
boundary "HTTP" as http #D5DBDB
rectangle "Bids Service" as service #4A90D9
database "Database" as db #3498DB

user -> http : POST /items/:id/bids
http -> service : Bid request
service -> db : Read + Write
db --> service : Result
service --> http : Response
http --> user : Success/Failure

note right of user
    User waits for
    entire operation
    to complete
end note

@enduml
```

#### Asynchronous (After)

```plantuml
@startuml async-approach
!theme plain
skinparam backgroundColor #FEFEFE

actor "User" as user
rectangle "Bids Service" as service #4A90D9
queue "Bids Queue" as bidQueue #9B59B6
rectangle "Serializer" as serializer #4A90D9
database "DB" as db #3498DB
queue "Response Queue" as respQueue #9B59B6

user -> service : Submit bid
service -> bidQueue : Enqueue bid
service --> user : "Bid received"

note right of user #AAFFAA
    Immediate response!
    User doesn't wait
    for processing
end note

bidQueue -> serializer : Process bid
serializer -> db : Update
serializer -> respQueue : Result

respQueue --> service : Accepted/Rejected
service --> user : Push notification

@enduml
```

### Complete Event Flow

```plantuml
@startuml event-flow
!theme plain
skinparam backgroundColor #FEFEFE

actor "Rick ($5)" as rick #E67E22
actor "Morty ($3)" as morty #F1C40F

rectangle "Bids Service" as service #4A90D9
queue "Bids Queue" as bidQueue #9B59B6 {
    rectangle "5" as b1 #E67E22
    rectangle "3" as b2 #F1C40F
}
rectangle "Serializer" as serializer #4A90D9
database "DB" as db #3498DB
queue "Response Queue" as respQueue #9B59B6 {
    rectangle "Accepted" as r1 #2ECC71
    rectangle "Rejected" as r2 #E74C3C
}

rick --> service
morty --> service
service --> bidQueue
bidQueue --> serializer
serializer --> db
serializer --> respQueue
respQueue --> service
service --> rick : "You're winning!"
service --> morty : "Outbid!"

@enduml
```

---

## Client Communication Protocols

### The Challenge: Notifying Clients

With asynchronous processing, how do clients know the result of their bid?

Two main approaches:

### Option 1: Polling

Client repeatedly asks "Is my bid processed yet?"

```plantuml
@startuml polling
!theme plain
skinparam backgroundColor #FEFEFE

participant "Client" as client
participant "Bids Service" as service #4A90D9
database "DB" as db #3498DB

client -> service : Submit bid
service --> client : "Bid ID: 12345"

... Poll every few seconds ...

client -> service : Check status (ID: 12345)
service -> db : Query bid status
db --> service : "Processing"
service --> client : "Still processing..."

... Poll again ...

client -> service : Check status (ID: 12345)
service -> db : Query bid status
db --> service : "Accepted"
service --> client : "You're the highest bidder!"

@enduml
```

**Polling Trade-offs:**

| Pros | Cons |
|------|------|
| Simple to implement | Wasted requests (most return "no change") |
| Works through firewalls | Higher latency (wait for next poll) |
| Stateless server | Increased server load |

### Option 2: WebSockets

Server pushes updates to client over persistent connection.

```plantuml
@startuml websockets
!theme plain
skinparam backgroundColor #FEFEFE

actor "Client" as client
rectangle "Bids Service" as service #4A90D9
queue "Bids Queue" as bidQueue #9B59B6
rectangle "Serializer" as serializer #4A90D9
database "DB" as db #3498DB
queue "Response Queue" as respQueue #9B59B6

client <--> service : WebSocket connection

client -> service : Submit bid
service -> bidQueue : Enqueue
service --> client : "Bid received"

bidQueue -> serializer : Process
serializer -> db : Update
serializer -> respQueue : Result

respQueue -> service : "Accepted"
service --> client : Push: "You're winning!"

note right of client #AAFFAA
    Instant notification!
    No polling needed
end note

@enduml
```

**WebSocket Trade-offs:**

| Pros | Cons |
|------|------|
| Real-time updates | More complex implementation |
| Lower latency | Requires connection management |
| Efficient (no wasted requests) | Stateful (harder to scale) |

### Two-Way Communication Example

```plantuml
@startuml two-way
!theme plain
skinparam backgroundColor #FEFEFE

participant "Rick" as rick #E67E22
participant "Bids Service" as service #4A90D9
participant "Morty" as morty #F1C40F

rick -> service : Bid $5 (via WebSocket)
morty -> service : Bid $3 (via WebSocket)

service -> service : Queue bids for processing

note right of service
    After processing:
    Rick's $5: Accepted
    Morty's $3: Rejected
end note

service --> rick : Push: "You're winning!"
service --> morty : Push: "Outbid! Current: $5"

note over morty #AAFFAA
    Morty immediately knows
    he needs to bid higher!
end note

@enduml
```

---

## Scaling Calculations

### Back-of-the-Envelope Math

Understanding your throughput requirements is crucial for capacity planning.

### Write Throughput (Bids Queue)

```
Given:
- 5 bids per user (average)
- 1,000 peak users per auction
- 100,000 simultaneous auctions

Total writes per day:
5 × 1,000 × 100,000 = 500,000,000 writes/day

Breaking it down:
500,000,000 / 24 hours   = 20,833,333 writes/hour
20,833,333 / 60 minutes  = 347,222 writes/minute
347,222 / 60 seconds     = ~8,000 writes/second
```

### Summary of Throughput Requirements

| Metric | Value |
|--------|-------|
| **Writes to Bids Queue** | ~8K/second |
| **Reads from DB** | ~8K/second (one per bid) |
| **Response Messages** | ~8K/second |

### Is This Achievable?

| Component | Typical Capacity | Our Requirement |
|-----------|------------------|-----------------|
| PostgreSQL writes | 10-50K/sec | 8K/sec |
| Redis operations | 100K+/sec | 8K/sec |
| Kafka messages | 100K+/sec | 8K/sec |
| WebSocket connections | 100K+ per server | Depends on bidders |

The requirements are well within modern infrastructure capabilities.

---

## Final Architecture

### Complete System Design

```plantuml
@startuml final-architecture
!theme plain
skinparam backgroundColor #FEFEFE

actor "Users" as users

rectangle "Client Layer" {
    rectangle "WebSockets" as ws #9B59B6
}

rectangle "Service Layer" {
    rectangle "Bids Service\n(Multiple Instances)" as service #4A90D9
}

rectangle "Message Layer" {
    queue "Bids Queue\n(Kafka)" as bidQueue #9B59B6
    queue "Response Queue\n(Kafka)" as respQueue #9B59B6
}

rectangle "Processing Layer" {
    rectangle "Bid Processor\n(Serializer)" as processor #4A90D9
}

rectangle "Data Layer" {
    database "Database\n(PostgreSQL)" as db #3498DB
}

users <--> ws
ws <--> service
service --> bidQueue : Publish bids
bidQueue --> processor : Consume bids
processor <--> db : Read/Write
processor --> respQueue : Publish results
respQueue --> service : Consume results
service --> ws : Push to clients

@enduml
```

### Component Summary

| Component | Responsibility | Technology Options |
|-----------|---------------|-------------------|
| **Bids Service** | Accept bids, manage WebSockets, route messages | Node.js, Go, Java |
| **Bids Queue** | Order and buffer incoming bids | Kafka, RabbitMQ, SQS |
| **Bid Processor** | Validate and process bids sequentially | Dedicated workers |
| **Response Queue** | Deliver results back to services | Kafka, RabbitMQ, SQS |
| **Database** | Store auction state and bid history | PostgreSQL, MySQL |

### Data Flow Summary

```plantuml
@startuml data-flow
!theme plain
skinparam backgroundColor #FEFEFE

|User|
start
:Submit Bid;

|Bids Service|
:Receive bid via WebSocket;
:Publish to Bids Queue;
:Send "Bid Received" acknowledgment;

|Bid Processor|
:Consume from Bids Queue;
:Read current price from DB;
if (Bid > Current Price?) then (yes)
    :Update DB with new price;
    :Publish "Accepted" to Response Queue;
else (no)
    :Publish "Rejected" to Response Queue;
endif

|Bids Service|
:Consume from Response Queue;
:Push result via WebSocket;

|User|
:Receive notification;
stop

@enduml
```

---

## Key Takeaways

### Design Principles Demonstrated

1. **Start Simple, Then Iterate**
   - Begin with basic synchronous design
   - Identify problems through analysis
   - Add complexity only when needed

2. **Understand Concurrency Challenges**
   - Race conditions can corrupt data
   - "Read-then-write" patterns are dangerous
   - Always consider what happens with simultaneous requests

3. **Optimistic Locking for Consistency**
   - Include expected state in updates
   - Let the database enforce constraints
   - Handle conflicts gracefully

4. **Queues for Ordering and Decoupling**
   - Serialize operations that must be ordered
   - Decouple components for scalability
   - Buffer against traffic spikes

5. **Event-Driven for Responsiveness**
   - Don't block clients on slow operations
   - Use async patterns for better user experience
   - Push updates instead of polling when possible

6. **Always Do the Math**
   - Calculate expected throughput
   - Validate assumptions with numbers
   - Design for peak load, not average

---

## Comparison: Approaches Summary

| Approach | Consistency | Fairness | Complexity | Latency |
|----------|-------------|----------|------------|---------|
| Basic (no protection) | Poor | Poor | Low | Low |
| Optimistic Locking | Good | Fair | Medium | Medium |
| Queue Serialization | Excellent | Excellent | High | Higher |
| Event-Driven + WebSocket | Excellent | Excellent | Highest | Best UX |

---

*Document generated from "Design an auction" presentation by Alexey Soshin*
