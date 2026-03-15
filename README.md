# System Design — Decision-Based Interview Questions

320 multiple-choice questions for Principal/Staff Backend Engineer interviews. Every question presents a real scenario and forces a tradeoff decision. Wrong answers are plausible — the kind of thing a competent engineer might consider but that has specific downsides in that context.

---

## Structure

| File | Questions | Topics |
|------|-----------|--------|
| `system-design-questions.md` | Q1–Q50 | Load Balancing, Caching, Database Selection, SQL vs NoSQL (partial) |
| `system-design-answers.md` | Q1–Q50 | Full explanations with PlantUML diagrams |
| `system-design-questions-2.md` | Q51–Q100 | SQL vs NoSQL (cont.), Message Queues & Kafka, Microservices vs Monolith, API Design (partial) |
| `system-design-answers-2.md` | Q51–Q100 | Full explanations with PlantUML diagrams |
| `system-design-questions-3.md` | Q101–Q150 | API Design (cont.), CAP & Consistency, Distributed Patterns, Rate Limiting, Sharding, CDN & DNS (partial) |
| `system-design-answers-3.md` | Q101–Q150 | Full explanations with PlantUML diagrams |
| `system-design-questions-4.md` | Q151–Q200 | CDN & DNS (cont.), Auth & Security, Observability, Classic Problems |
| `system-design-answers-4.md` | Q151–Q200 | Explanations with interview tips |
| `system-design-questions-5.md` | Q201–Q220 | Back-of-Envelope Estimation (QPS, storage, bandwidth, cache sizing) |
| `system-design-answers-5.md` | Q201–Q220 | Step-by-step math with sequence diagrams |
| `system-design-questions-6.md` | Q221–Q270 | Concurrency Control, DB Sharding, Real-Time Protocols, Caching & Resilience |
| `system-design-answers-6.md` | Q221–Q270 | Full explanations with sequence diagrams |
| `system-design-questions-7.md` | Q271–Q320 | Inventory Patterns, Fan-out & Feeds, URL Shortening & Crawling, Chat & Presence, Resilience |
| `system-design-answers-7.md` | Q271–Q320 | Full explanations with sequence diagrams |

---

## Topics Covered

| # | Topic | Questions |
|---|-------|-----------|
| 1 | Load Balancing & Reverse Proxies | Q1–Q12 |
| 2 | Caching Strategies | Q13–Q27 |
| 3 | Database Selection & Modeling | Q28–Q47 |
| 4 | SQL vs NoSQL Tradeoffs | Q48–Q59 |
| 5 | Message Queues & Event Streaming (Kafka, RabbitMQ, SQS) | Q60–Q74 |
| 6 | Microservices vs Monolith | Q75–Q86 |
| 7 | API Design (REST, gRPC, GraphQL, WebSocket) | Q87–Q101 |
| 8 | Consistency, Availability & Partition Tolerance (CAP) | Q102–Q113 |
| 9 | Distributed Systems Patterns (Saga, CQRS, Event Sourcing, Circuit Breaker) | Q114–Q131 |
| 10 | Rate Limiting, Throttling & Backpressure | Q132–Q141 |
| 11 | Data Partitioning & Sharding | Q142–Q153 |
| 12 | CDN, DNS & Edge Computing | Q149–Q163 |
| 13 | Authentication, Authorization & Security | Q164–Q175 |
| 14 | Observability (Logging, Metrics, Tracing, Alerting) | Q176–Q185 |
| 15 | Classic System Design Problems | Q186–Q200 |
| 16 | Back-of-Envelope Estimation | Q201–Q220 |
| 17 | Concurrency Control & Race Conditions | Q221–Q232 |
| 18 | Database Selection & Sharding (Applied) | Q233–Q242 |
| 19 | Communication Protocols & Real-Time Systems | Q243–Q252 |
| 20 | Caching, Scaling & Resilience Patterns | Q253–Q262 |
| 21 | System-Specific Design Decisions | Q263–Q270 |
| 22 | Linearization, Inventory & Reservation Patterns | Q271–Q280 |
| 23 | Fan-out, Feed & Notification Architectures | Q281–Q290 |
| 24 | URL Shortening, Crawling & Content Systems | Q291–Q300 |
| 25 | Chat, Presence & Real-Time Messaging | Q301–Q310 |
| 26 | Resilience, Observability & Interview Patterns | Q311–Q320 |

---

## Difficulty

| Level | Count | Description |
|-------|-------|-------------|
| ★☆☆ | 96 (30%) | Warm-up. Senior engineer answers in 10 seconds. |
| ★★☆ | 160 (50%) | Interview level. Requires weighing 2–3 tradeoffs. |
| ★★★ | 64 (20%) | Principal/Staff level. Deep understanding of system behavior under failure or scale. |

---

## Question Format

Each question follows the same structure:

```
### Q{N}. {Scenario Title} [★★☆]

[PlantUML diagram — current state before the decision]

[Scenario — 2–3 sentences, specific numbers: users, RPS, latency, data size]

**What is the most appropriate approach?**

- A) [Option — specific technology or pattern]
- B) [Option — specific technology or pattern]
- C) [Option — specific technology or pattern]
- D) [Option — specific technology or pattern]
```

Each answer includes:
- **Correct answer** with the tradeoff reasoning for this specific scenario
- **Why not A/B/C/D** — the scenario where each wrong answer *would* be correct
- **PlantUML diagram** — the correct architecture (Batches 1–3)
- **Interview tip** — what an interviewer wants to hear

---

## How to Use

**Self-testing:** Work through the questions file. Write your answer and reasoning before checking. The wrong answers are designed to be tempting — if you can articulate *why* each wrong answer fails in this specific scenario, you're ready.

**Weak area drill:** Use the topic table to jump to the sections you need most. CAP theorem and distributed patterns (Q102–Q131) are the highest-density ★★★ section.

**Interview prep pattern:**
1. Answer the question cold
2. Compare with the correct answer
3. Read the "why not" for each wrong option
4. Read the interview tip — this is the sentence that signals depth to an interviewer
5. If you got it right but couldn't articulate the tradeoff, mark it for review

---

## Tech Stack Coverage

Java/Spring ecosystem is used where relevant for implementation options:
- Spring Boot, Spring Cloud, Resilience4j (circuit breaker, bulkhead, retry)
- HikariCP (connection pool sizing via Little's Law)
- Spring Security, OAuth2 client
- Micrometer, Spring Boot Actuator (health checks, metrics)
- OpenTelemetry Java SDK

Infrastructure and platform coverage: AWS (ALB, SQS, SNS, EventBridge, RDS Proxy, ElastiCache, CloudFront, Route 53), Kubernetes (cert-manager, Istio, Envoy), Kafka, Redis, Cassandra, PostgreSQL, Elasticsearch.

---

## Classic System Design Problems (Q186–Q200)

| Q | Problem |
|---|---------|
| Q186–Q187 | URL Shortener (core + analytics) |
| Q188 | Real-Time Chat System |
| Q189 | News Feed (hybrid fan-out) |
| Q190 | Notification System |
| Q191 | Search Autocomplete |
| Q192 | Rate-Limited API Gateway |
| Q193 | Distributed File Storage (Dropbox-like) |
| Q194 | Ride-Sharing Location Service |
| Q195 | Video Streaming Platform |
| Q196 | Web Crawler |
| Q197 | Recommendation Engine |
| Q198 | Distributed Task Scheduler |
| Q199 | Consistent Hashing in Chat Routing |
| Q200 | Instagram-Like Feed (end-to-end) |

---

## Back-of-Envelope Estimation (Q201–Q220)

20 questions covering the quantitative skills interviewers expect: QPS calculations, storage sizing, bandwidth estimation, cache capacity planning, server count estimation, and database throughput modeling. Each answer includes step-by-step math.

---

## Pragmatic System Design Patterns (Q221–Q320)

100 questions drawn from real-world system design case studies covering:
- **Concurrency control** — optimistic locking, CAS, TOCTOU bugs, linearization
- **Applied sharding & DB selection** — partition strategies, hot-key mitigation, polyglot persistence
- **Real-time protocols** — WebSocket vs SSE vs long polling, presence systems, typing indicators
- **Fan-out architectures** — push vs pull vs hybrid feeds, notification delivery
- **Domain-specific designs** — auction bidding, coupon distribution, ticketing, URL shorteners, web crawlers, chat systems
- **Resilience patterns** — circuit breakers, bulkheads, retry budgets, graceful degradation

---

## Cheat Sheet

[`CHEAT-SHEET.md`](CHEAT-SHEET.md) — A comprehensive pre-session review covering all 26 topics with PlantUML diagrams, estimation formulas, decision trees, and quick-reference tables. Read it once before a practice session to prime your memory on key patterns, thresholds, and default technology choices.
