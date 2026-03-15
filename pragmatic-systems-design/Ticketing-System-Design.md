# Ticketing System Design

## Table of Contents
1. [Introduction](#introduction)
2. [Requirements Gathering](#requirements-gathering)
3. [User Perspective](#user-perspective)
4. [Basic Design](#basic-design)
5. [Database Schema](#database-schema)
6. [Checking Ticket Availability](#checking-ticket-availability)
7. [Handling Payment Failures](#handling-payment-failures)
8. [Payment Integration](#payment-integration)
9. [Scaling the System](#scaling-the-system)
10. [Conflict Resolution](#conflict-resolution)
11. [Final Architecture](#final-architecture)

---

## Introduction

A **Ticketing System** allows users to purchase tickets for events at various venues. Think of platforms like Ticketmaster, Eventbrite, or concert booking systems. The core challenge is handling high concurrency when thousands of users compete for limited tickets, while ensuring no overselling occurs.

### The Core Problem

```plantuml
@startuml
skinparam backgroundColor #FEFEFE

actor "User" as user
participant "Ticketing\nSystem" as system
participant "Payments" as payments

user -> system : Select event + tickets
system -> system : Check availability
system -> user : Show payment screen
user -> payments : Pay
payments -> system : Confirm payment
system -> user : Issue tickets

note right of system
  Challenge: What if 1000 users
  try to buy the last 10 tickets
  at the same time?
end note
@enduml
```

---

## Requirements Gathering

Before designing, we need to understand the scale and constraints:

| Question | Answer | Impact |
|----------|--------|--------|
| How many venues? | **1,000** | Determines sharding strategy |
| Max tickets per venue? | **100,000** | Affects storage and query design |
| Ticket type? | **Entrance tickets** (all same) | Simplifies logic (no seat selection) |
| Payment handling? | **3rd-party integration** | Async webhook-based flow |

### Key Insight: Entrance vs Seat Tickets

- **Entrance tickets**: All tickets are identical (like festival entry)
- **Seat tickets**: Each ticket is unique (like cinema, airplane)

For entrance tickets, we don't need to track individual seats - just the count of available tickets.

---

## User Perspective

The user experience is straightforward:

```plantuml
@startuml
skinparam backgroundColor #FEFEFE

rectangle "1. Select Event" as step1 {
  card "Event Name" as e1
  card "Tickets: [-] [+]" as e2
}

rectangle "2. Payment" as step2 {
  card "Card Details" as p1
  card "[Pay!]" as p2
}

rectangle "3. Confirmation" as step3 {
  card "Tickets Ready!" as c1
}

step1 -right-> step2
step2 -right-> step3
@enduml
```

**User Flow:**
1. Browse events and select number of tickets
2. Enter payment details
3. Receive ticket confirmation

---

## Basic Design

### Initial Booking Flow

```plantuml
@startuml
skinparam backgroundColor #FEFEFE

actor "User" as user
participant "Ticketing\nSystem" as system
participant "Payments" as payments

user -> system : event + number of tickets
system -> system : Check enough tickets?
system -> user : credit card screen
user -> system : Credit card details
system -> payments : Charge credit card
payments -> system : Success
system -> user : Tickets

@enduml
```

### Initial Database Schema

We start with a simple two-table design:

```plantuml
@startuml
skinparam backgroundColor #FEFEFE

entity "Events" as events {
  * event_id : BIGINT <<PK>>
  --
  venue : VARCHAR
  name : VARCHAR
  date : DATE
  max_tickets : INT
}

entity "Tickets" as tickets {
  * ticket_id : BIGINT <<PK>>
  --
  * event_id : BIGINT <<FK>>
  user_email : VARCHAR
  num_tickets : INT
}

events ||--o{ tickets : has
@enduml
```

**Problem**: How do we know if enough tickets are available?

---

## Database Schema

### Evolution: Adding Venues

Since venues host multiple events, we normalize the schema:

```plantuml
@startuml
skinparam backgroundColor #FEFEFE

entity "Venues" as venues {
  * venue_id : BIGINT <<PK>>
  --
  name : VARCHAR
  max_tickets : INT
}

entity "Events" as events {
  * event_id : BIGINT <<PK>>
  --
  * venue_id : BIGINT <<FK>>
  name : VARCHAR
  date : DATE
}

entity "Tickets" as tickets {
  * ticket_id : BIGINT <<PK>>
  --
  * event_id : BIGINT <<FK>>
  user_email : VARCHAR
  num_tickets : INT
}

venues ||--o{ events : hosts
events ||--o{ tickets : has
@enduml
```

---

## Checking Ticket Availability

The fundamental check before any booking:

```
current_tickets_count + booked_tickets <= max_tickets
```

### Approach 1: Counter Field (Denormalized)

Add `curr_tickets` field to Events table:

| Column | Source |
|--------|--------|
| `max_tickets` | Events table |
| `booked_tickets` | User request |
| `curr_tickets` | **New field** - running count |

**Pros**: Fast reads (single row lookup)
**Cons**: Must keep in sync with Tickets table

### Approach 2: Aggregate Query (Normalized)

Calculate from Tickets table:

```sql
SELECT SUM(num_tickets)
FROM Tickets
WHERE event_id = ?
GROUP BY event_id
```

**Pros**: Always accurate, no sync issues
**Cons**: Slower (aggregation on every request)

---

## Handling Payment Failures

### The Problem

What happens when payment fails after we've already counted the tickets?

```plantuml
@startuml
skinparam backgroundColor #FEFEFE

actor "User" as user
participant "Ticketing\nSystem" as system
participant "Payments" as payments

user -> system : event + number of tickets
system -> system : Check enough tickets?
system -> system : current_tickets += booked_tickets
system -> user : credit card screen
user -> system : Credit card details
system -> payments : Charge credit card

note right of payments #ffcccc
  Payment FAILS!
  But tickets already
  counted as sold!
end note

payments --> system : Failure
system -> user : Tickets (incorrectly)
@enduml
```

### Solution: Status Field

Add a `status` field to track ticket state:

```plantuml
@startuml
skinparam backgroundColor #FEFEFE

entity "Tickets" as tickets {
  * ticket_id : BIGINT <<PK>>
  --
  * event_id : BIGINT <<FK>>
  user_email : VARCHAR
  num_tickets : INT
  **status** : ENUM
}

note right of tickets
  Status values:
  - 'reserved'
  - 'paid'
  - 'reversed'
  - 'free'
end note
@enduml
```

Now our availability query becomes:

```sql
SELECT SUM(num_tickets)
FROM Tickets
WHERE event_id = ?
  AND status IN ('reserved', 'paid')
GROUP BY event_id
```

---

## Payment Integration

### The Challenge of 3rd-Party Payments

Payment processing is asynchronous. The user leaves our site, pays externally, and we receive confirmation via webhook.

### Reservation UUID Pattern

```plantuml
@startuml
skinparam backgroundColor #FEFEFE

actor "User" as user
participant "Ticketing" as system
participant "Payments" as payments

user -> system : event + number of tickets
system -> system : Check enough tickets?
system -> system : Create ticket reservation
system -> user : credit card screen\n+ reservation UUID

user -> payments : Credit card details\n+ reservation UUID
system -> system : Poll every X seconds\nwith reservation UUID

payments --> system : Payment status (webhook)
system -> user : Tickets
@enduml
```

**Key Points:**
1. System creates a **reservation** with unique UUID
2. UUID is passed to payment provider
3. Webhook returns with same UUID to correlate payment
4. User polls system for status updates

---

## Scaling the System

### Back-of-Envelope Calculations

```
Storage over 5 years:
  100,000 (max venue size)
  x 365 x 5 (five years)
  = 182,500,000 => ~200M per venue
  x 1,000 (venues)
  = 200 BILLION rows
```

### Traffic Calculations

| Question | Answer |
|----------|--------|
| People competing per ticket? | 100 people |
| Total users for max venue | 100 x 100,000 = 10M users |
| Time to sell out? | 30 minutes |

```
10M users / 30 minutes
= 10M users / 1,800 seconds
= ~6K requests/second (average)
```

### Handling Peak Load (10x)

What if we have a 10x peak? That's **60K requests/second**.

```plantuml
@startuml
skinparam backgroundColor #FEFEFE

actor "User" as user
database "Cache" as cache
participant "Ticketing" as ticketing
database "Database" as db

user -> ticketing : 60K req/s
ticketing -> cache : Read availability
cache -> db : Sync X times/second

note right of cache
  Cache handles high read load
  Database updated periodically
end note
@enduml
```

### Architecture with DB Writer

```plantuml
@startuml
skinparam backgroundColor #FEFEFE

rectangle "User\n60K req/s" as user
rectangle "Cache" as cache #ffcccc
rectangle "Ticketing" as ticketing #FF6B6B
rectangle "DB Writer" as writer #FF6B6B
database "Database" as db

user --> ticketing
ticketing --> cache : Read/Write
ticketing --> writer : Reservations
writer --> db : Batch writes
db --> cache : Sync periodically

note bottom of writer
  DB Writer batches writes
  to reduce database load
end note
@enduml
```

### Sharding by Venue

With 200B rows, we need to shard. Venue is a natural shard key:

```plantuml
@startuml
skinparam backgroundColor #FEFEFE

rectangle "Ticketing" as ticketing #FF6B6B

database "Shard 1\n(Venue 1)" as s1
database "Shard 2\n(Venue 2, 3)" as s2
database "Shard N\n(Venue N)" as sn

ticketing --> s1
ticketing --> s2
ticketing --> sn
@enduml
```

---

## Conflict Resolution

### The Race Condition Problem

What happens when Alice and Bob both try to buy the last ticket?

| Time | Alice | Bob |
|------|-------|-----|
| T1 | Check enough tickets? | |
| T2 | | Check enough tickets? |
| T3 | Make Reservation | |
| T4 | | Make Reservation |
| T5 | | Make Payment |
| T6 | **Make Payment (FAILS!)** | |

Both users see 1 ticket available, both reserve, but only one can actually buy!

### Solution 1: Serialization (Single DB Writer)

Route all writes through a single DB Writer to serialize operations:

```plantuml
@startuml
skinparam backgroundColor #FEFEFE

rectangle "User" as user
rectangle "Cache" as cache #ffcccc
rectangle "Ticketing" as ticketing #FF6B6B
rectangle "WebHook" as webhook #FF6B6B
rectangle "DB Writer" as writer #FF6B6B
database "Database" as db
rectangle "Payments" as payments #ffcc00

user --> ticketing
ticketing --> cache
payments --> webhook
webhook --> writer : Serialize writes
writer --> db
writer --> cache : Update status
@enduml
```

**How it works:**
- All reservation and payment confirmations go through DB Writer
- DB Writer processes requests sequentially
- No two users can reserve the same ticket simultaneously

### Solution 2: Linearization (Pre-created Tickets)

Instead of counting tickets, **pre-create individual ticket records**:

```plantuml
@startuml
skinparam backgroundColor #FEFEFE

card "Tickets Table - Initial State" as initial {
  card "| ID | User | Status |" as h1
  card "| 1  |      | Free   |" as r1
  card "| 2  |      | Free   |" as r2
}

card "Tickets Table - After Reservations" as reserved {
  card "| ID | User    | Status   |" as h2
  card "| 1  | Alice   | Reserved |" as r3
  card "| 2  | Bob     | Reserved |" as r4
}

card "Tickets Table - After Payments" as paid {
  card "| ID | User    | Status   |" as h3
  card "| 1  | Alice   | Approved |" as r5
  card "| 2  | Bob     | Approved |" as r6
}

initial -down-> reserved : Users reserve
reserved -down-> paid : Payments confirmed
@enduml
```

**The Magic: Optimistic Locking**

```sql
UPDATE Tickets
SET user = 'Alice', status = 'Reserved'
WHERE id = 1 AND status = 'Free'
```

- Only ONE user can successfully update a ticket
- If `status != 'Free'`, update affects 0 rows
- Application checks rows affected and handles failure

### Ticket Status State Machine

```plantuml
@startuml
skinparam backgroundColor #FEFEFE

[*] --> Free : Ticket created

Free --> Reserved : User reserves
Reserved --> Paid : Payment success
Reserved --> Reversed : Payment failed\nor timeout
Reversed --> Free : Ticket released

Paid --> [*] : Final state

note right of Reserved
  Reservation has TTL
  Auto-reverses if not paid
end note
@enduml
```

### Linearization Trade-offs

| Pros | Cons |
|------|------|
| No overbooking possible | Waste of space (pre-created rows) |
| Easy optimistic locking | Less efficient for group reservations |
| Simple concurrent handling | Requires ticket pre-population |

---

## Final Architecture

The complete system combines all components:

```plantuml
@startuml
skinparam backgroundColor #FEFEFE

rectangle "User" as user
rectangle "Cache" as cache #ffcccc
rectangle "Ticketing" as ticketing #FF6B6B
rectangle "DB Writer" as writer #FF6B6B
database "Database" as db
rectangle "WebHook" as webhook #FF6B6B
rectangle "Payments" as payments #ffcc00

user --> ticketing
ticketing --> cache : Read availability
ticketing --> writer : Reservations

writer --> db : Batch writes
db --> cache : Sync X times/second

payments --> webhook : Payment status
webhook --> writer : Update status
writer --> db : Update ticket status
@enduml
```

### Component Summary

| Component | Purpose | Technology |
|-----------|---------|------------|
| Ticketing Service | API, business logic | Stateless microservice |
| Cache | Fast reads, ticket availability | Redis |
| DB Writer | Serialize writes, conflict resolution | Worker service |
| Database | Persistent storage | PostgreSQL (sharded by venue) |
| WebHook Handler | Receive payment confirmations | HTTP endpoint |
| Payments | 3rd-party payment processing | Stripe, PayPal, etc. |

### Key Design Decisions

1. **Caching** for high read throughput (60K req/s peak)
2. **DB Writer** for serialized writes and conflict resolution
3. **Sharding by venue** for horizontal scaling
4. **Status field** to handle payment failures gracefully
5. **Reservation UUID** for payment correlation
6. **Webhook-based** async payment confirmation
7. **Linearization** or **Serialization** for race condition handling

---

## Key Takeaways

| Challenge | Solution |
|-----------|----------|
| 60K requests/second peak | Cache + DB Writer pattern |
| 200B rows over 5 years | Shard by venue |
| Payment failures | Status field (reserved/paid/reversed) |
| Race conditions | Serialization (DB Writer) or Linearization (optimistic locking) |
| 3rd-party payments | Reservation UUID + Webhook pattern |
| Availability checking | Aggregate query with status filter |

### Trade-offs Accepted

- **Eventual consistency**: Cache may lag behind database briefly
- **Complexity**: Multiple components (cache, DB writer, webhook handler)
- **Storage overhead**: Linearization pre-creates empty ticket records

This design handles concert-scale load while preventing overselling and gracefully handling payment failures.
