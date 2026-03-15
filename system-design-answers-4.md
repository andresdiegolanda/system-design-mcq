# System Design — Answers & Explanations
## Batch 4: Q151–Q200

---

## Topic 12: CDN, DNS & Edge Computing (continued)

### Q151. Edge Computing vs CDN
**Correct Answer: B** — CDN for static Scenario A; origin passthrough for personalized Scenario B. Static bundles are identical for all users and change weekly — perfect CDN fit, ~99% hit rate. Personalized recommendations require user-specific DB queries; CDN caches would either serve wrong user's data or have near-zero hit rates. CDN in passthrough mode still provides DDoS protection and TLS termination for Scenario B without incorrect caching.
**Why not A:** Caching per user_id creates one entry per user — 500K entries, near-zero hit rate, privacy risk (user A sees user B's recommendations on cache collision).
**Why not C:** Edge workers can run stateless logic but personalization requires user history DB not co-located at edge. Moving compute without the data doesn't reduce latency.
**Why not D:** CDN for static assets reduces APAC latency from 200ms (US-East origin) to 20-50ms (edge PoP). Not "unnecessary complexity."
**Interview tip:** CDN = shared cacheable content. Edge compute = stateless logic. Origin = DB-dependent, user-specific, compliance-sensitive responses.

---

### Q152. DNS TTL Strategy
**Correct Answer: B** — TTL ladder: high TTL normally (3600s), reduce 48h before planned changes, proactively reduce at first sign of trouble.
**Why not A:** 300s TTL fails Scenario C (60s failover requirement — clients cache for 300s).
**Why not C:** Always 30s TTL creates 3,333 DNS lookups/sec at 1M req/sec with 100K unique clients — unnecessary load for a service that changes IP every 2 years.
**Why not D:** TTL=0 creates per-request DNS lookups; 1M lookups/sec; not sustainable; many resolvers enforce minimum TTLs anyway.
**Interview tip:** The TTL ladder — key insight: TTL reduction must propagate BEFORE the IP change, not simultaneously. Reduce TTL → wait for propagation → change IP.

---

### Q153. Content Delivery Optimization
**Correct Answer: B** — Multi-tier CDN with regional origin shields. APAC shield (Singapore) sits between 50 APAC edge nodes and US-East origin. 50 trans-Pacific cache misses become 1 trans-Pacific request. APAC edge misses are served from the shield at intra-APAC latency (20-50ms) instead of trans-Pacific (150-200ms).
**Why not A:** More edge memory helps long-tail retention but doesn't fix the geographic network latency problem.
**Why not C:** Moving origin to Singapore helps APAC but not EU. Creates data sovereignty complexity. Shield achieves most of the benefit without moving primary origin.
**Why not D:** Video chunk size affects request count per playback, not CDN hit rate or geographic latency.
**Interview tip:** "Origin shield" / "shielding PoP" / "mid-tier cache" — know all three names. The benefit: aggregates demand from multiple edge nodes, reducing origin requests by the number of edge nodes in that region.

---

### Q154. Anycast vs Unicast Routing
**Correct Answer: B** — BGP anycast. Multiple datacenters announce the same IP prefix. Internet routers forward packets to the nearest BGP-announced location. Same IP, geographically distributed servers, network-layer routing.
**Why not A:** GeoDNS returns different IPs per client location — different IPs, not one IP to multiple servers.
**Why not C:** A centralized load balancer is a single physical location. Anycast distributes routing at the BGP level with no central ingress.
**Why not D:** DNS round-robin returns multiple different IPs per answer. Anycast is one IP, multiple physical destinations.
**Interview tip:** Anycast is what makes Google DNS (8.8.8.8) and Cloudflare (1.1.1.1) work globally. Also why DDoS attacks against CDNs fail — the attack traffic is distributed across hundreds of anycast PoPs.

---

### Q155. CDN for Dynamic Content
**Correct Answer: B** — `Cache-Control: public, max-age=300, stale-while-revalidate=60`. CDN caches for 5 minutes; serves stale while revalidating in background. Hot products (1,000 products, 80% of traffic): origin sees ~288 req/product/day instead of 86,400. 99.7% origin reduction for hot products; ~80% overall.
**Why not A:** 24-hour caching violates the 5-minute freshness requirement. Prices change 3x/day.
**Why not C:** Surrogate key purging requires event-driven integration between pricing system and CDN purge API. More complex than TTL for a 5-minute freshness requirement.
**Why not D:** Same product = same price for all users. This is shared cacheable content, opposite of Q19's personalization problem.
**Interview tip:** `stale-while-revalidate` eliminates cache expiry latency spikes — one request revalidates while all others serve stale. Essential for high-traffic dynamic content.

---

### Q156. DNS Load Balancing vs Layer 7
**Correct Answer: B** — Layer 7 ALB with active health checks. Failed servers removed within 60 seconds. Clients never routed to failed servers.
**Why not A:** DNS has no health awareness. Failed server stays in rotation until TTL expires AND DNS is manually updated. 33% of requests fail during the window.
**Why not C:** TTL=5s at 1M req/sec = ~20K DNS lookups/sec. Unnecessary overhead. Failed server still serves for 5 seconds per failure.
**Why not D:** Client retry is a last-resort mechanism, not load balancing. 33% initial failure rate with retry latency is poor UX.
**Interview tip:** One sentence: "DNS has no health awareness; Layer 7 LB has active health checks." Know the health check configuration: interval (30s), threshold (3 failures), failover time (~60s).

---

### Q157. Edge Caching for A/B Tests
**Correct Answer: C** — CDN cache key normalization to include only `ab_bucket` cookie value. Two cache entries: one per variant. Hit rate approaches 100%.
**Why not A:** Same URL, CDN serves first cached response to all users. 50% see wrong variant — correctness failure.
**Why not B:** `Vary: Cookie` uses the full cookie string. Every unique cookie combination = new cache entry. Near-zero hit rate at scale.
**Why not D:** Moving assignment to edge is a more complete solution but requires migrating experiment management to edge infrastructure. Cache key normalization is the minimal correct answer.
**Interview tip:** CDN cache key customization via edge workers (Cloudflare Workers, Lambda@Edge) is the precision tool. Default key = URL only. Extended key = URL + specific cookie value.

---

### Q158. CDN Security: DDoS Mitigation
**Correct Answer: B** — CDN anycast absorbs volumetric attack (100Gbps across 15+ Tbps CDN capacity); CDN rate-limits HTTP flood; CDN TCP SYN proxy absorbs SYN flood; origin IP hidden.
**Why not A:** Scaling origin doesn't help when origin bandwidth link (1Gbps) is the constraint. Volumetric attacks can exceed Tbps with amplification.
**Why not C:** WAF operates at Layer 7. 100Gbps UDP flood saturates origin network before TCP connections are established — WAF never sees the packets.
**Why not D:** Bandwidth arms race: UDP reflection can amplify 1Gbps attack to 50-100Gbps. CDN anycast distribution is the architectural answer.
**Interview tip:** Three DDoS attack types and defenses: volumetric (CDN anycast absorption), protocol/SYN (TCP SYN proxy), application/HTTP (rate limiting + CAPTCHA). CDN handles all three. Origin IP secrecy prevents bypass.

---

### Q159. Prefetching and Preloading
**Correct Answer: B** — `preload` for CSS and hero image; server-side prefetch for API data; `prefetch` for below-fold carousel; `async` for analytics.
**Why not A:** No prioritization — CSS, hero image, analytics all compete for bandwidth. Render-blocking CSS delays page.
**Why not C:** Inline CSS/JS breaks independent caching. Any change to CSS requires new HTML hash. Fine for tiny critical CSS, wrong for full stylesheets.
**Why not D:** Lazy loading the hero image (above-fold, visible immediately) delays the most prominent element. Hero should be `loading="eager"` with preload.
**Interview tip:** Four loading hints: `preload` (fetch now, high priority), `prefetch` (fetch in idle time), `preconnect` (establish connection early), `dns-prefetch` (resolve DNS early). Match to resource criticality.

---

### Q160. HTTP/2 vs HTTP/3
**Correct Answer: B** — HTTP/3 (QUIC). QUIC over UDP; independent streams at transport layer; packet loss on one stream doesn't block others; built-in TLS 1.3; 0-RTT resumption.
**Why not A:** HTTP/1.1 multiple connections: 6-connection browser limit; each connection has slow start + TLS overhead; doesn't solve high-packet-loss problem.
**Why not C:** Server push reduces round-trips but doesn't fix TCP head-of-line blocking. All pushed resources still stall on TCP packet loss.
**Why not D:** WebSockets are for specific bidirectional use cases. Not a general HTTP replacement.
**Interview tip:** HTTP/2 moved head-of-line blocking from HTTP layer to TCP layer. HTTP/3 eliminates it by using QUIC (UDP-based). TCP HoL blocking is the one-sentence answer.

---

### Q161. Cache Warming for New Edge PoPs
**Correct Answer: B** — Pre-warm before DNS cutover. Send synthetic requests for top 1,000 pages at 4K req/sec (80% of origin capacity). ~250 seconds to warm. Then cut over. Origin sees ~10K req/sec (20% cold tail) — manageable.
**Why not A:** Organic warming: 50K req/sec × 0% hit rate = 50K origin req/sec → 10× capacity → crash.
**Why not C:** Rate-limiting Mumbai PoP traffic defers the cold start problem. When limit is raised, cold cache problem recurs.
**Why not D:** Disabling cache = all requests hit origin. Same problem as A plus no cache benefit.
**Interview tip:** Pre-warming formula: (top_N_pages / sustainable_origin_rate) = warm-up time. Then calculate: traffic × (1 - hot_hit_rate) = origin load after launch.

---

### Q162. DNS-Based Service Discovery
**Correct Answer: B** — Short TTL (5-10s) + client retry on connection error. TTL limits stale window; retry makes failures transparent. Combined: short stale window + self-healing.
**Why not A:** TTL=0 creates per-request DNS lookups. 5-20ms added latency per service call. DNS infrastructure overwhelmed.
**Why not C:** 300s TTL with manual updates = minutes of 33% failure rate. Not automated.
**Why not D:** Service mesh eliminates DNS-based discovery entirely — correct long-term solution. But the question asks for DNS-specific mitigation.
**Interview tip:** DNS service discovery limitations: no real-time health, stale cache on failure. Mitigations: short TTL + client retry. Long-term: service mesh (Istio/Linkerd).

---

### Q163. Edge Computing Use Cases
**Correct Answer: B** — Edge: JWT validation (stateless crypto), A/B assignment (stateless bucketing), image resizing (no DB), SSR for static pages (cached data). Origin: personalized recommendations (user history DB), payment (PCI DSS), fraud detection (user history DB).
**Why not A:** Payment needs PCI DSS-compliant server environment. Edge workers are not PCI-certified. Recommendations need user DB not at edge.
**Why not C:** Personalized recommendations (B) require user history DB. Moving compute without the data doesn't reduce latency.
**Why not D:** Edge workers absolutely run application logic — Cloudflare Workers processes billions of requests per day with real code.
**Interview tip:** Edge decision rule: stateless + compute-bounded + no DB dependency = edge-appropriate. DB-dependent + compliance-sensitive + stateful = origin.

---

## Topic 13: Authentication, Authorization & Security

### Q164. OAuth 2.0 Flow Selection
**Correct Answer: B** — Authorization Code + PKCE for browser/mobile; Client Credentials for M2M.
PKCE replaces client_secret with a dynamic code_verifier/code_challenge pair. Safe for public clients (browser, mobile) that can't store secrets. Client Credentials: client_id + client_secret → access_token; no user involved; correct for service-to-service.
**Why not A:** Authorization Code requires client_secret stored server-side. Browser/mobile apps can't safely store secrets (visible in source/APK).
**Why not C:** Implicit flow deprecated in OAuth 2.1. Tokens in URL fragments are visible in browser history, referrer headers. PKCE is the modern replacement.
**Why not D:** ROPC requires app to handle user credentials directly — defeats SSO, enables phishing. Device Code is for TVs/CLIs, not mobile apps.
**Interview tip:** Know all four flows: Authorization Code + PKCE (browser/mobile), Client Credentials (M2M), Device Code (input-constrained), Authorization Code (server-side with secret). Implicit and ROPC are deprecated.

---

### Q165. JWT vs Session Token
**Correct Answer: B** — JWT with 15-min expiry + Redis revocation list (checked at API Gateway). Stateless validation per service; immediate revocation via JTI in Redis set; one Redis call at gateway perimeter (not per service).
**Why not A:** Pure JWT without revocation: compromised token valid until expiry (24h). Unacceptable for financial systems.
**Why not C:** Centralized session store: 500K × 20 services × 10 req/sec = 100M Redis reads/sec. Needs ~111 Redis primary nodes. Redis becomes bottleneck for every request to every service.
**Why not D:** Per-service session tokens: user must authenticate 20 times. No SSO. Not viable.
**Interview tip:** JWT + gateway revocation check = distributed validation (no auth service call per request) + revocation capability (one Redis lookup at perimeter). Check revocation at gateway, not at each service.

---

### Q166. RBAC vs ABAC
**Correct Answer: B** — RBAC for 3-role SaaS; ABAC for multi-condition healthcare policy.
RBAC: static role assignments, 3 roles, low overhead, easy audit. ABAC (OPA/Cedar): evaluates multiple attribute conditions at runtime — doctor's relationship to patient, consent flag, time, location.
**Why not A:** RBAC for healthcare: one role per doctor-patient pair = 10,000 doctors × 1,000 patients = 10M roles. Role explosion.
**Why not C:** ABAC can handle 3-role model but is over-engineered for it. RBAC's simplicity is a feature for simple permission models.
**Why not D:** if/else in application code: authorization logic scattered across codebase, impossible to audit, breaks on policy changes.
**Interview tip:** RBAC = "who you are determines access" (static). ABAC = "who you are + what you're accessing + when + where" (contextual). Healthcare, financial, multi-tenant = ABAC territory.

---

### Q167. API Key vs OAuth Token
**Correct Answer: B** — API key for server-to-server developer integration; OAuth access token for user-delegated scoped access.
API key: one string, long-lived, rate-limited per key, no user involved. OAuth: user grants limited scope, revocable by user, different apps get different scopes.
**Why not A:** API key for user delegation: no scope limitation, user can't revoke per-app, app has full key permissions.
**Why not C:** OAuth Client Credentials for Scenario A is technically correct but adds unnecessary flow complexity vs. simple API key.
**Why not D:** Session tokens are user-session-based. Not appropriate for API integrations.
**Interview tip:** API keys = developer authentication (who is calling). OAuth = user authorization (on whose behalf, with what permissions). Different problems.

---

### Q168. Password Hashing
**Correct Answer: D (Argon2id) or B (bcrypt) — D is modern recommendation.**
Argon2id: memory-hard (64MB/hash), GPU/ASIC attacks impractical (GPU can only run 256 parallel Argon2id at 64MB each). OWASP first choice for new systems. bcrypt (factor 12): 30-year standard, 300ms/hash, per-password salt, acceptable for existing systems.
**Why not A:** SHA-256: 10B hashes/sec on GPU, no salt = rainbow table vulnerable. Fast = wrong for passwords.
**Why not C:** MD5 with salt: 50B hashes/sec on GPU. Salt prevents rainbow tables but per-account brute force is trivial in seconds.
**Interview tip:** Password hashing must be: slow by design, per-password salt, memory-hard preferred. Argon2id > bcrypt > scrypt > PBKDF2 >> SHA-256 >> MD5.

---

### Q169. SQL Injection Prevention
**Correct Answer: B** — Parameterized queries (prepared statements). SQL structure compiled before user data provided. `?` placeholder always treated as data value, never SQL syntax. Cannot be injected regardless of content.
**Why not A:** Escaping is fragile — context-dependent, one missed escape = injection. Not the primary defense.
**Why not C:** Input validation limits damage scope but short injections (`OR 1=1`) still work. Insufficient alone.
**Why not D:** Stored procedures with parameterized inputs are safe; those doing internal string concatenation are equally vulnerable. Safety comes from parameterization, not the procedure wrapper.
**Interview tip:** Java: `PreparedStatement`. Spring: `@Query` with positional params or JPA criteria. Parameterized queries are the answer. No exceptions.

---

### Q170. HTTPS Certificate Management
**Correct Answer: B** — cert-manager for 500 Kubernetes services; Let's Encrypt ACME for public API; Istio CA for internal mTLS.
cert-manager: Kubernetes operator, auto-renews 30 days before expiry, 2,000 manual ops → 0. Let's Encrypt: free, browser-trusted, auto-renews via ACME. Istio CA: issues 24-hour certs to all pods automatically, rotates continuously.
**Why not A:** 2,000 manual rotations/year and 24-hour Istio rotation are both impossible to maintain manually.
**Why not C:** Self-signed not browser-trusted for public API.
**Why not D:** Wildcard covers one subdomain level. 500 services sharing one private key = compromise of one exposes all.
**Interview tip:** Three cert management tools: cert-manager (Kubernetes), Let's Encrypt ACME (public), service mesh CA (internal mTLS). The common thread: automate everything.

---

### Q171. CSRF Prevention
**Correct Answer: C** — `SameSite=Strict` cookie. Browser withholds cookie on cross-site requests. Attacker's page makes request to bank; browser recognizes cross-site; bank cookie not sent; request unauthenticated.
**Why B is also correct (classic approach):** CSRF token — server generates random token per session, embeds in forms, verifies on submission. Attacker can't read token (same-origin policy). Standard defense for legacy/older browsers.
**Why not A:** HTTPS prevents eavesdropping, not cross-site request forgery. Cookies still sent to legitimate site.
**Why not D:** CSP restricts resource loading. CSRF is about cookie behavior on cross-site requests, not resource loading.
**Interview tip:** `SameSite=Strict` = modern browser-native CSRF defense. CSRF token = classic server-side defense. Both correct; SameSite is simpler. Know three SameSite values: Strict (never cross-site), Lax (top-level nav OK), None (always send, needs Secure).

---

### Q172. Secrets Management
**Correct Answer: B** — HashiCorp Vault or AWS Secrets Manager. Dynamic secrets (short-lived DB credentials), automatic rotation, full audit log, Kubernetes integration via Vault Agent Injector (writes to in-memory volume, never in env vars or Kubernetes Secrets).
**Why not A:** Env variables visible in `kubectl describe pod`, `docker inspect`, `/proc/1/environ`, crash dumps. No rotation, no audit.
**Why not C:** Kubernetes Secrets: base64-encoded (not encrypted) in etcd by default. Readable with `kubectl get secret`. No rotation, no audit. Use for referencing Vault-managed secrets, not storing them.
**Why not D:** Encrypted config files in git: decryption key must be stored somewhere; git history retains deleted secrets permanently.
**Interview tip:** Complete secrets management = storage (Vault) + distribution (Vault Agent) + rotation (dynamic secrets) + audit (Vault audit log). Know all four components.

---

### Q173. Zero Trust Architecture
**Correct Answer: B** — Every request authenticated + authorized per service; mTLS between services; device posture verification; short-lived credentials; micro-segmentation; continuous evaluation.
**Why not A:** VPN with 2FA: strong perimeter, but once inside VPN all traffic trusted. Compromised device = lateral movement to all internal services.
**Why not C:** mTLS verifies service identity. Doesn't verify user identity, device health, or request context. One component of Zero Trust, not the complete model.
**Why not D:** Firewall rules restrict which services can communicate at network layer. Attacker with compromised allowed service can still call all its allowed targets. Network segmentation ≠ Zero Trust.
**Interview tip:** Zero Trust principle: "never trust, always verify." Six NIST SP 800-207 tenets: verify identity, device health, access rights; assume breach; enforce least privilege; monitor continuously.

---

### Q174. Content Security Policy
**Correct Answer: C** — Nonce-based CSP: `script-src 'nonce-{random}'`. Server generates unique nonce per response. Legitimate scripts have the nonce. Attacker's injected `<script>` lacks the nonce. Browser blocks it. Nonce must be cryptographically random and unique per response.
**Why A is partially correct:** `default-src 'self'` blocks all inline scripts (including attacker's). But also breaks all legitimate inline scripts in the application.
**Why not B:** `default-src *` allows any origin. Attacker-controlled external scripts can load.
**Why not D:** `unsafe-inline` explicitly allows all inline scripts. Completely defeats XSS protection.
**Interview tip:** CSP levels: `default-src 'self'` (all inline blocked), nonce-based (specific inline allowed), `unsafe-inline` (no protection). Nonce must be unique per response — static nonce can be found in source and reused.

---

### Q175. mTLS Implementation
**Correct Answer: B** — Service mesh sidecar (Istio/Envoy). Envoy sidecar injected via admission webhook. Service makes plain HTTP to localhost. Envoy handles mTLS transparently. Istio CA issues 24-hour certs automatically. PeerAuthentication enforces STRICT mode cluster-wide. Zero code changes.
**Why not A:** Modifying 100 services requires language-specific TLS libraries, cert loading code, rotation triggers. Hundreds of code changes across dozens of repos.
**Why not C:** API Gateway terminates external TLS. East-west internal traffic doesn't route through API Gateway. Doesn't help service-to-service mTLS.
**Why not D:** IPSec encrypts at network layer (bytes between IPs). Doesn't provide application-level service identity. Both services appear as IP addresses, not named services.
**Interview tip:** Service mesh mTLS = identity (SPIFFE X.509) + encryption (TLS) + zero code changes. Key components: SPIFFE identity format, Istio CA for issuance, Envoy sidecar for termination, PeerAuthentication for enforcement.

---

## Topic 14: Observability

### Q176. Three Pillars of Observability
**Correct Answer: B** — Q1→Metrics (aggregated error rate %), Q2→Traces (which service in chain is slow), Q3→Logs (what happened before failure).
Metrics: aggregated time-series numbers for "how many/how much." Traces: distributed request waterfall showing timing across services. Logs: structured text events showing what a specific service did.
**Why not A/C/D:** All other mappings mis-assign traces and metrics. Traces show individual flows; metrics show aggregates.
**Interview tip:** Metrics alert you (something wrong), traces locate it (which service), logs explain it (why). OpenTelemetry handles all three with one SDK.

---

### Q177. Structured Logging
**Correct Answer: B** — Structured JSON is machine-parseable. Enables exact-match queries (`order_id:"12345"`), aggregations, trace correlation via `trace_id`, field-based alerting. Unstructured log parsing requires regex that breaks silently when log format changes.
**Why not A:** Human readability matters for one engineer reading one log. At terabytes/day, engineers query logs, not read them.
**Why not C:** Format choice significantly affects operations. Unstructured regex patterns are maintenance burden and break on any message change.
**Why not D:** JSON with compression is often smaller than unstructured text. Operational value far outweighs storage cost.
**Interview tip:** Essential structured log fields: `timestamp`, `level`, `service`, `trace_id`, `span_id`, `event`, correlation IDs (order_id, user_id). `trace_id` connects logs to traces.

---

### Q178. Metrics Cardinality
**Correct Answer: B** — Remove `user_id` and `request_id` from metrics. Correct metric: `http_requests_total{service, endpoint, method, status_code}` = ~60 time series. Per-user data → logs; per-request data → traces.
Prometheus stores 1 time series per unique label combination. 50M user_ids × 50 endpoints × 10 status_codes = 25B time series × 3KB = 75 petabytes → OOM crash.
**Why not A:** No amount of RAM accommodates 25B time series. Reduce cardinality, don't add hardware.
**Why not C:** High-cardinality metrics are conceptually wrong regardless of backend. Metrics are for aggregates.
**Why not D:** 1% sampling loses accuracy for rare events. Root fix is cardinality reduction.
**Interview tip:** Rule of thumb: if a label has >1,000 unique values, it's too high-cardinality for metrics. Send to logs/traces instead.

---

### Q179. Alerting on Symptoms vs Causes
**Correct Answer: B** — Symptoms (alert immediately): C (error rate >1%), D (P99 >2,000ms). Causes (investigate, don't page): A (CPU), B (memory), E (connection pool), F (disk).
Error rate and latency directly measure user experience degradation. CPU/memory/disk/pool are internal resources — high CPU might just mean the service is legitimately busy.
**Why not A/C/D:** CPU is not a symptom; users don't experience "high CPU." Error rate and latency are precisely user-facing metrics.
**Interview tip:** Google SRE four golden signals: latency, traffic, errors, saturation. Alert on latency and errors (symptoms). Use traffic and saturation to investigate causes.

---

### Q180. Distributed Tracing Sampling
**Correct Answer: B** — Tail-based sampling: always store errors and slow requests; 1% random sample of fast/successful. Decision made at trace completion based on outcome.
Head-based 10% discards 90% of error traces — exactly when you need them most. Tail-based maximizes diagnostic value per storage byte.
Storage estimate: 100K × 0.1% errors = 100 error traces/sec × 50KB = 5MB/sec; 100K × 98.9% × 1% = 989 traces/sec × 50KB = 49MB/sec. Total ~54MB/sec — within 500MB budget.
**Why not A:** Head-based discards interesting traces upfront. During a 5% error incident, only 10% × 5% = 0.5% of errors captured.
**Why not C:** Adaptive sampling is correct but more complex infrastructure.
**Interview tip:** Tail-based vs head-based: tail decides after the fact (correct choice), head decides at entry (simple but loses diagnostic value). OTel Collector `tailsamplingprocessor` supports tail-based.

---

### Q181. SLO Error Budget
**Correct Answer: B** — Freeze risky deployments. 13.2 min remaining; deployment historically consumes 8 min; leaves 5.2 min for rest of month. Any incident = SLA breach.
Fix the deployment process: canary + auto-rollback, feature flags, improved pipeline. Deploy after month resets or after reducing deployment risk.
**Why not A:** Deploying with 13.2 min remaining knowing deployment costs 8 min = accepting SLA breach.
**Why not C:** Adjusting SLOs to create budget defeats the purpose. SLOs represent user experience requirements.
**Why not D:** Maintenance windows count against error budget. Week 3's 15-minute maintenance consumed budget; "maintenance window" doesn't make downtime free.
**Interview tip:** Error budget = reliability currency. Full budget: move fast. Low budget: slow down, focus on reliability. Formula: error_budget = (1 - SLO) × time_period.

---

### Q182. Log Aggregation Architecture
**Correct Answer: C** — stdout → Fluent Bit DaemonSet → Elasticsearch (30-day hot) + S3 (1-year cold). Two-tier reduces Elasticsearch storage cost from 1.825PB to 150TB (12× cheaper). S3 + Athena for rare cold queries.
Cost: Elasticsearch 30-day: ~$37.5K/year; S3 1-year: ~$42K/year; total ~$80K vs $456K for Elasticsearch-only (5.7× cheaper).
**Why not A:** 150TB Elasticsearch = $456K/year. No cold tier. No spike buffering.
**Why not B:** Kafka + Logstash correct for complex enrichment pipelines. Overkill for standard collection with two outputs.
**Why not D:** CloudWatch at 5TB/day = $2,500/day = $900K/year. Prohibitively expensive.
**Interview tip:** Two-tier log architecture: hot (Elasticsearch, 30 days, searchable) + cold (S3, 1 year, cheap). Fluent Bit as lightweight Kubernetes DaemonSet collector.

---

### Q183. Anomaly Detection in Metrics
**Correct Answer: B** — Seasonal decomposition + dynamic baseline. Model expected value per time-of-day × day-of-week. Alert when observed is >2σ below expected. Black Friday baseline = 8,000; alert if <6,000. Weekday baseline = 1,200; alert if <800.
**Why not A:** Higher static threshold still fails at Black Friday scale. Static thresholds can't adapt to seasonal patterns.
**Why not C:** Percentage drop needs per-season threshold tuning. 12.5% drop from 8,000 on Black Friday may be critical — needs context.
**Why not D:** Manual updates for each event = operational burden + inevitable human error.
**Interview tip:** Seasonal anomaly detection: Datadog Anomaly Monitor, AWS CloudWatch Anomaly Detection, Prometheus `holt_winters()`. Core concept: model expected behavior, alert on deviation.

---

### Q184. Health Check Design
**Correct Answer: B** — Liveness: `/actuator/health/liveness` (JVM only); Readiness: `/actuator/health/readiness` (DB + cache + circuit breakers); Startup: wait 50s then check context loaded.
Liveness failure → pod restart (correct for deadlock/OOM). Readiness failure → remove from LB (correct for DB outage — restart won't fix DB). Startup delay prevents premature traffic during 45-second Spring Boot startup.
**Why not A:** Same endpoint for all: liveness checks DB → DB down → liveness fails → restart loop during DB outage.
**Why not C:** Port opens before application context loads. Returns 200 while still initializing.
**Why not D:** DB as liveness check = restart loop. Also: 100 pods × health check every 10s = 10 wasted DB queries/second.
**Interview tip:** Three Kubernetes probes: liveness (restart on failure), readiness (remove from LB on failure), startup (delay other probes). Spring Boot Actuator provides all three natively.

---

### Q185. OpenTelemetry Collector
**Correct Answer: B** — OTel Collector as telemetry gateway. Services export all signals via OTLP to Collector. Collector routes to backends. Backend migration = update Collector config only. Services unchanged.
Benefits: single SDK per service (not 3), backend decoupling, sampling/batching/filtering in Collector, format conversion.
**Why not A:** 150 SDK configs; backend migration = 50 SDK changes per service. Tight coupling.
**Why not C:** Fluent Bit handles logs only. Traces and metrics need different collection.
**Why not D:** Service mesh collects traces and metrics via sidecars; logs still need application-level collection. Not a complete telemetry gateway.
**Interview tip:** OpenTelemetry Collector pipeline: receivers (OTLP, Jaeger, Prometheus) → processors (sampling, filtering, batching) → exporters (Jaeger, Prometheus, Elasticsearch). Single choke point for all telemetry decisions.

---

## Topic 15: Classic System Design Problems

### Q186. URL Shortener — Core Design
**Correct Answer: B** — hash(long_url) → 7 chars + collision detection; Cassandra for storage; Redis cache for hot URLs.
Hashing prevents sequential enumeration (unlike auto-increment IDs). Cassandra handles 100M writes/day with natural horizontal scaling. Redis absorbs 100:1 read:write ratio — 90%+ cache hit for viral URLs achieves 10ms P99 redirect.
**Why not A:** Sequential IDs are guessable (abc123 → abc124 → abc125). Security concern.
**Why not C:** UUIDs are 36 chars, not 7. CDN can cache redirects but target may be stale.
**Why not D:** 3.5 trillion (62^7) codes pre-generation is computationally and storage-infeasible.
**Interview tip:** URL shortener core: hash → 7 chars → collision check → store. Redirect path: Redis (1ms) → Cassandra (10ms) → 302. Analytics via Kafka async (don't block redirect).

---

### Q187. URL Shortener — Analytics
**Correct Answer: B** — Click events → Kafka → Flink aggregates per URL per minute → TimescaleDB; Redis sorted set (ZINCRBY) for real-time top-100.
Kafka decouples click recording from redirect path. Flink reduces 115K/sec to ~1 write/URL/minute to DB. Redis ZINCRBY for sub-millisecond top-100 updates.
**Why not A:** Hot row lock contention at 115K updates/sec on single URL row.
**Why not C:** Redis INCR works for total counts but loses time-series granularity (per-hour breakdown).
**Why not D:** Statistical estimation doesn't satisfy exact analytics requirements.
**Interview tip:** Counter analytics at high throughput: event stream (Kafka) → stream aggregation (Flink/Spark) → time-series DB. Redis sorted set for real-time leaderboards.

---

### Q188. Real-Time Chat System
**Correct Answer: B** — Connection servers + Redis Pub/Sub fan-out + Cassandra persistence.
Users connect to one of N connection servers (WebSocket). Each server subscribes to user's Redis channel. Sender publishes to recipient's channel. All servers with recipient's connections receive and deliver. Messages persisted to Cassandra. Push notifications for offline users.
**Why not A:** P2P doesn't work behind NAT; no persistence; no multi-device.
**Why not C:** Single server can't handle 1M connections (~50K per server max). SPOF.
**Why not D:** 50M × 1 poll/sec = 50M req/sec. Not real-time.
**Interview tip:** Chat architecture = connection servers (WebSocket routing) + message bus (Redis Pub/Sub for cross-server fan-out) + message store (Cassandra). Know all three layers.

---

### Q189. News Feed System
**Correct Answer: C** — Hybrid: fan-out on write for normal users (<1M followers); fan-out on read for celebrities (>1M followers).
Normal users: post → push to 200 followers' Redis feed caches. Manageable. Celebrities: post → not fanned out. Feed load: pull celebrity posts separately + merge with pre-computed cache. Twitter/Instagram use this exact pattern.
**Why not A:** Celebrity fan-out on write: 10M cache writes per celebrity post. Queue saturated. Write amplification.
**Why not B:** Fan-out on read for all: 200 DB queries × 5B feed loads/day = 1 trillion DB queries/day. Unscalable.
**Why not D:** Real-time computation: 200 DB queries × 500M users × 10 loads/day. Not feasible.
**Interview tip:** The celebrity threshold (e.g., >1M followers) triggers the fan-out strategy switch. Know both strategies and the hybrid — this is an Instagram/Twitter-level question.

---

### Q190. Notification System
**Correct Answer: B** — Event source → Kafka → Notification Service (preferences + deduplication via Redis) → channel-specific priority queues → channel workers (APNs/FCM/SES/Twilio) with exponential backoff retry + DLQ.
Kafka decouples event generation from delivery. Redis deduplication (event_id TTL=24h) prevents duplicate notifications. Separate queues per channel and priority enable independent scaling.
**Why not A:** Synchronous HTTP to 4 providers couples event source availability to all providers. Slow provider delays everything.
**Why not C:** No central preference management; deduplication impossible across services.
**Why not D:** Batch every hour violates <1-second urgent requirement.
**Interview tip:** Notification system layers: event ingestion (Kafka) → preference filtering → deduplication → channel routing → delivery workers. Each layer independently scalable.

---

### Q191. Search Autocomplete
**Correct Answer: B** — Pre-computed prefix trie in Redis: `GET autocomplete:ap` → sorted suggestions. Sub-millisecond. Rebuilt nightly from search log analytics.
1,160 queries/sec with 50ms P99 target — Redis GET is <1ms, easily achievable. Pre-computation eliminates real-time trie traversal cost.
**Why not A:** SQL LIKE `'ap%'` = range scan on indexed column. Fast for small tables; at 100M search terms, 50ms P99 not achievable under load.
**Why not C:** Elasticsearch completion suggester is purpose-built for autocomplete — valid heavier alternative.
**Why not D:** In-memory trie per server: ~1GB RAM per server; requires synchronization across servers on nightly update.
**Interview tip:** Autocomplete = pre-computed prefix map in Redis. At query time: O(1) Redis GET. Trade storage for query speed. Nightly rebuild keeps suggestions fresh with trending data.

---

### Q192. Rate-Limited API Gateway
**Correct Answer: B** — Local token bucket + Redis sync every 100ms. Local bucket absorbs burst within each node. Sync every 100ms reduces Redis calls from 500K/sec to ~12.5K/sec (40× reduction). ~1% overrun acceptable. Circuit breaker falls back to local enforcement if Redis down.
**Why not A:** Local counter without sync: 5 nodes each allow full limit = 5× effective limit. Underenforcement.
**Why not C:** Consistent hash per API key = one node, one counter, accurate. But sticky routing creates hotspots on popular keys; loses load balancing.
**Why not D:** PostgreSQL adds more latency than Redis. Wrong direction.
**Interview tip:** Distributed rate limiting tradeoff: accuracy (Redis) vs performance (local). Hybrid: local bucket + periodic sync = 40× Redis call reduction with ~1% accuracy trade-off.

---

### Q193. Distributed File Storage
**Correct Answer: B** — Block-level chunking + content-addressed storage. Files split into 4MB blocks; SHA-256 each block; store hash → S3 object. Deduplication: same block from different users = one S3 object. Sync: only upload changed blocks. Metadata in DynamoDB: (user_id, file_path, version, block_list).
Dropbox uses this exact architecture. 80-90% bandwidth savings for incremental changes.
**Why not A:** No deduplication; sync re-uploads entire file on any change.
**Why not C:** Compression reduces storage; no deduplication; still re-uploads entire file.
**Why not D:** NFS doesn't scale to 5 exabytes; SPOF; not distributed.
**Interview tip:** Content-addressed storage = store by hash of content, not by name. Deduplication is automatic: same content = same hash = one stored copy. Dropbox blog post describes this in detail.

---

### Q194. Ride-Sharing Location Service
**Correct Answer: B** — Redis Geospatial (live location) + Cassandra (location history).
Redis GEOADD handles 1.25M updates/sec across cluster (125K/node). GEOSEARCH for sub-10ms nearby queries. Cassandra persists location history for trip reconstruction and analytics.
**Why not A:** PostGIS + GIST correct geospatially; 1.25M writes/sec exceeds single PostgreSQL node without aggressive sharding.
**Why not C:** MongoDB 2dsphere index handles geo; scatter-gather for cross-shard queries more complex than Redis.
**Why not D:** In-memory grid: inter-cell cross-server calls; Redis Geospatial simpler with better tooling.
**Interview tip:** Redis Geospatial = GEOADD for writes + GEOSEARCH for radius queries. The combination handles both 1.25M writes/sec and sub-10ms reads. Cite Uber's early architecture (before switching to custom in-house system).

---

### Q195. Video Streaming Platform
**Correct Answer: B** — Upload → S3 (raw) → SQS → auto-scaled transcoding workers (parallel, one per resolution) → S3 (segments) → CDN with multi-tier PoPs; HLS/DASH for ABR; Redis for watch progress.
Auto-scaling workers handle burst uploads. Parallel transcoding of all 5 resolutions simultaneously (not sequential). S3 + CDN for exabyte scale and 1B views/day.
**Why not A:** Single server can't transcode 500 hours/minute.
**Why not C:** Transcoding on first playback: 30-60s delay; broken first-play UX.
**Why not D:** P2P unsuitable for on-demand with content protection requirements.
**Interview tip:** Video pipeline: upload → S3 → queue → parallel transcoding workers → S3 segments → CDN. Key: parallel (not sequential) transcoding of all resolutions.

---

### Q196. Web Crawler
**Correct Answer: B** — URL frontier partitioned by domain hash + Bloom filter deduplication + per-domain Redis rate limiter + priority queue by PageRank.
Bloom filter: O(1) dedup for 10B URLs at ~12GB memory, 1% false positive rate. Redis per-domain counter enforces 1 req/sec/domain across all 100 workers. Priority queue ensures high-value pages crawled first.
**Why not A:** Single node: 11,574 pages/sec required; single node bottleneck.
**Why not C:** 10B rows in database; slow lookup; 1,000× less memory-efficient than Bloom filter.
**Why not D:** Single priority queue: contention at 11,574 ops/sec; no domain-aware work partitioning.
**Interview tip:** Mercator/Heritrix architecture: domain-partitioned frontier + Bloom filter dedup + politeness per-domain rate limiting. Know all three components.

---

### Q197. Recommendation Engine
**Correct Answer: B** — Weekly batch training (two-tower model) → pre-computed embeddings in Redis (user: 51GB, items: 2.5GB) → Faiss ANN search at serve time (<10ms). Cold start: content-based fallback or popular items.
Pre-computed embeddings eliminate training cost at serve time. Faiss sub-millisecond ANN search. 100ms P99 achievable.
**Why not A:** Real-time collaborative filtering: O(N) computation across 100M users per request. Not 100ms.
**Why not C:** Popularity-based has no personalization. Cold-start fallback only.
**Why not D:** User-user collaborative filtering: O(N) similarity at 100M users = infeasible per request.
**Interview tip:** Recommendation serving = offline training + online inference with pre-computed vectors. ANN (approximate nearest neighbor) with Faiss is the serving mechanism. Two-tower model (user tower + item tower) is the modern architecture.

---

### Q198. Distributed Task Scheduler
**Correct Answer: B** — Redis sorted set delay queue (score=timestamp) + BRPOPLPUSH for at-most-once pickup + visibility timeout for crash recovery + priority sorted sets.
BRPOPLPUSH atomically moves task from delay queue to processing queue. If worker crashes and doesn't acknowledge in 30s, task returns to pending. At-most-once: each task dequeued by exactly one worker.
**Why not A:** Cron on each worker: each task executes on every worker simultaneously. Duplicate execution.
**Why not C:** Kafka doesn't natively support delayed message delivery.
**Why not D:** PostgreSQL `SELECT FOR UPDATE SKIP LOCKED` is also correct — valid relational alternative but higher latency than Redis.
**Interview tip:** Redis sorted set as delay queue: score = execution timestamp. ZRANGEBYSCORE finds due tasks. BRPOPLPUSH = atomic dequeue. Visibility timeout = crash recovery. This maps to SQS's visibility timeout mechanism.

---

### Q199. Consistent Hashing in Chat Routing
**Correct Answer: B** — Consistent hashing of room_id → server. All users in same room hash to same server. On server addition/removal: only ~1/N rooms remap. Minimizes reconnection during auto-scaling.
**Why not A:** Round-robin: users A and B in same room land on different servers. All messages need cross-server fan-out.
**Why not C:** Sticky sessions by user_id: two users in same room map to different servers. Cross-server messaging still required for same-room delivery.
**Why not D:** Random: no correlation. All rooms need cross-server messaging.
**Interview tip:** Consistent hashing for room routing = natural colocation of room members on same server. On topology change: only affected rooms remap (not all rooms). Compare with modulo hashing where adding one server remaps all rooms.

---

### Q200. End-to-End Instagram-Like Feed
**Correct Answer: B** — Photo storage: S3 + CDN; Metadata/social graph: Cassandra; Feed: hybrid fan-out (Redis pre-computed + celebrity pull); Likes/comments: Redis INCR; Real-time: WebSocket/SSE; Notifications: Kafka → notification service.
Each component matched to its optimal technology: S3+CDN for immutable media, Cassandra for social graph writes, Redis for hot feed cache and counters, Kafka for async fan-out.
**Why not A:** Single database at 1B DAU is architecturally impossible.
**Why not C:** RDS can't handle social graph at 1B users; no fan-out; photo BLOBs impractical at petabyte scale.
**Why not D:** Synchronous REST fan-out for celebrity post (10M followers) = 10M synchronous calls = timeout. Async via message queue required.
**Interview tip:** Instagram-scale design requires identifying which components have different scaling requirements: media storage (immutable, CDN-friendly), social graph (write-heavy, eventual consistency), feed generation (read-heavy, pre-computed), counters (atomic, high-concurrency).

---

*End of Batch 4 (Q151–Q200). All 200 questions complete.*
