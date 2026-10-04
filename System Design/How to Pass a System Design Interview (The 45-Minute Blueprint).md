# How to Pass a System Design Interview (The 45-Minute Blueprint)

* **Source:** [YouTube - Code with Lucian](https://www.youtube.com/watch?v=HcC9Du6RWwk)
* **Category:** System Design
* **Target Level:** Mid-Level → Senior

---

## The Reality of System Design Interviews

The funnel is steep: most applicants are filtered out at the HR screen and the coding round, and system design cuts the remaining pool again. (The video quotes rough numbers - roughly 100 → 75 → 20 → 10 - but they are unsourced, so treat them as illustrative only.)

But here's the flip side: **system design is also the ultimate differentiator.** It is the round where you directly dictate your seniority level and the salary band you land in.

Interviewers are not counting how many buzzwords you drop. They are measuring **signal versus noise** - how you navigate ambiguity, how you justify trade-offs, and whether you think like a senior engineer when a system is under stress.

The biggest mistake mid-level engineers make is sitting back and waiting to be asked questions one by one. **Senior engineers take control and drive the interview.** The 45-minute blueprint below gives you the exact structure to do that.

> **Caveat - drive, don't steamroll.** Leading is not monologuing. Check in at every phase boundary ("Does this scope work for you? Anything you'd like me to go deeper on?") and treat interviewer hints as signals to pivot. Candidates who ignore a nudge fail just as surely as those who wait passively.

---

## The 45-Minute Blueprint

| Phase | Time | Focus |
|---|---|---|
| 1 - Scope & Requirements | 0–5 min | Define the problem before drawing anything |
| 2 - Estimations & Contracts | 5–10 min | Justify hardware and protocol choices with math |
| 3 - High-Level Architecture | 10–20 min | Draw a minimal, functional 5-box system |
| 4 - Deep Dives & Bottlenecks | 20–35 min | Stress test and solve the hard scaling problems |
| 5 - Resilience & Observability | 35–45 min | Make the system reliable and monitorable |

![The 45-Minute Matrix](assets/matrix_recap.jpg)

---

## Phase 1: Scope & Requirements (0–5 min)

You are handed a vague prompt - *"Design Uber."* A mid-level engineer panics and immediately starts drawing database schemas. A senior engineer stops, takes a breath, and uses the first five minutes to **define the problem**.

### Functional Requirements
Limit yourself to **three core features maximum**. For Uber:
- Rider requests a ride
- Driver accepts the ride
- Real-time driver location tracking

That's it. Do not try to design surge pricing, ratings, or ride-sharing pools in the first five minutes.

### Non-Functional Requirements
![Phase 1: Non-Functional Requirements Up Front](assets/phase_1_requirements.jpg)

This is where you establish SLAs (Service Level Agreements). Say this out loud to the interviewer:

> *"Before we look at components, I want to clarify our scale and SLAs. Do we need strong consistency for payments but eventual consistency for driver locations? What is the latency budget for location tracking - is under 500ms acceptable?"*

Those answers will shape every architectural decision downstream. Also ask about the **read-to-write ratio**. A 100:1 read-heavy system (like a news feed) looks completely different from an IoT logging service that is 99% writes.

### The Cardinal Sin of Phase 1
**Premature over-engineering.** If you pitch multi-region database sharding before confirming the system handles more than 10 requests per second, you are signalling zero engineering maturity.

**Rule of thumb:** Scope it tight, establish the SLAs, get agreement from your interviewer, then move on.

---

## Phase 2: Estimations & API Contracts (5–10 min)

Many engineers dread this phase because they think they are being tested on precise math. They burn 15 minutes obsessing over exact decimal places for storage calculations.

Your interviewer **does not care** if your multiplication is off by a few decimal points. They care about **order of magnitude** and what it implies for your hardware choices.

### Back-of-the-Envelope Estimation
![Phase 2: Math as Architectural Justification](assets/phase_2_estimations.jpg)

Round aggressively:

- 10 million daily users doing 10 actions a day = 100 million events/day
- 100 million ÷ 100,000 seconds in a day = **~1,000 average QPS**, or ~2,000 at peak
- At 10 KB payload × 2,000 QPS = **20 MB/s of network bandwidth** - easily handled by a single instance
- If it were 2 GB/s, you would immediately know you need horizontal partitioning
- **Estimate each flow separately.** Ride bookings are low-volume, but driver location pings are not: 1M active drivers × 1 ping / 4 s = **~250K writes/s**. That single number is what justifies the queue and sharding in Phase 4 - and it is why the "20,000 writes/s" stress test there is realistic, not arbitrary.

Math is not busywork - it is **architectural justification**.

### API Contracts
Do not just say "we'll use APIs." Make specific statements:

- **REST over HTTPS** for ride booking - it is a stateless transactional operation. One request, one confirmation, connection closed.
- **WebSockets or Server-Sent Events (SSE)** for real-time driver location streaming - to avoid costly HTTP polling overhead.

### Data Model
Justify your storage choice based on access patterns:
- Need ACID compliance and relational integrity for payments? → **Relational (SQL)**
- Need ultra-low latency writes and "find drivers near me" queries for driver coordinates? → **In-memory store (e.g. Redis) keyed by a geospatial cell** (geohash, S2 or H3), with Cassandra or similar for durable history. Note that Cassandra has no native geospatial indexing - the spatial index comes from the cell-based key design, not the database.
- Also state your **API details**: key endpoints, pagination, and **idempotency keys** on any write that costs money (e.g. `POST /rides`, `POST /payments`).

**Rule of thumb:** Keep calculations simple, define protocols explicitly, and always justify your storage choices.

---

## Phase 3: High-Level Architecture (10–20 min)

This is where mid-level engineers make a fatal mistake: trying to design Day 1,000 architecture on Day 1.

**Senior engineers build iteratively.** Start with a minimal foundation:

```
Client → Load Balancer → API Gateway → Application Service → Storage
```

### Trace the Happy Path
![Phase 3: Trace the Happy Path First](assets/phase_3_architecture.jpg)

Walk your interviewer through a single end-to-end request:

> *"The mobile client sends a ride request to our load balancer, which terminates TLS and forwards it to the API gateway for authentication and rate limiting. The gateway routes it to the ride service, which writes to the database and returns a ride ID."*

### The Golden Rule
**Never draw a box without stating its purpose out loud.** Don't silently add a load balancer - say:

> *"We place a load balancer here to distribute peak traffic of 2,000 QPS across stateless application instances and perform health checks."*

### Separate Read and Write Paths
If 95% of traffic is users viewing driver locations on a map, those read queries should hit an in-memory cache or read replica - completely bypassing the transactional write database.

**Rule of thumb:** Keep the initial design embarrassingly simple. Prove the core system works logically before you try to scale it.

---

## Phase 4: Deep Dives & Bottlenecks (20–35 min)

This is the most critical phase. Your interviewer will stress-test the baseline:

> *"Your database is taking 20,000 writes per second during peak hours. How do you stop it from collapsing?"*

This is your opportunity to demonstrate senior engineering thinking.

### Caching Strategy
Don't just say "add a cache." Specify the pattern:

> *"We implement a cache-aside pattern using Redis. Read requests hit the cache first. On a miss, we load from the primary database, write back to the cache, and return the response."*

Manage memory with TTL and LRU (Least Recently Used) eviction policies. Also say how you handle **invalidation** (write-through or explicit delete on update) and **cache stampedes** (request coalescing, jittered TTLs) - TTL alone is not an invalidation strategy.

### Database Scaling
![Phase 4: Database Scaling & Sharding Keys](assets/phase_4_deep_dives.jpg)

- **Read pressure?** Add read replicas with asynchronous replication.
- **Write pressure?** Introduce sharding (horizontal partitioning).

**Critical: Choose your sharding key carefully - and match it to the query.** If you shard an Uber database by `city_id`, what happens during New Year's Eve in New York? A massive hotspot on a single partition.

Be careful with the obvious fix. A key like `city_id + hash(driver_id)` spreads load evenly, but it **scatters nearby drivers across every shard**, so the core query ("drivers within 2 km of this rider") becomes a scatter-gather across all of them. Choose the key by access pattern:

- **Location lookups and matching** → shard by **geospatial cell** (geohash / S2 / H3). Nearby drivers live together, so a query touches one or a few shards. Handle hot cells (Times Square on NYE) by **splitting them into finer cells**, or by adding a salt suffix and querying the small fixed set of salted keys.
- **Trip and payment records** (no spatial query) → shard by `trip_id` / `user_id` hash, where even distribution is exactly what you want.

Saying "different data, different key" is a stronger answer than one key for everything.

### Asynchronous Decoupling
For write surges, introduce a message queue:

> *"Instead of synchronously writing location updates to the database, we push events into a distributed queue (e.g., Kafka). Worker nodes pull from the queue at a controlled rate, giving us automatic backpressure protection during traffic spikes."*

### CAP Theorem Trade-offs
During a network partition you cannot have both strong consistency and high availability - you must explicitly choose one and justify it.

- **Driver location tracking** → prioritise **Availability (AP)**. A 2-second stale driver icon is acceptable.
- **Financial transactions** → prioritise **Consistency (CP)**. It is far better to reject a payment cleanly than to risk data corruption or double charging. Consistency alone does not prevent double charging, though: pair it with **idempotency keys**, so a client retry after a timeout returns the original result instead of charging twice.

CAP only describes behaviour *during a partition*. Partitions are rare; the trade-off you live with every day is **PACELC** - *else* (no partition), latency vs consistency. Naming it ("synchronous replication to a quorum costs us a few ms of write latency but gives us read-your-writes") signals real depth.

### Eliminate Single Points of Failure
Proactively scan your diagram for risks before the interviewer does:

> *"Our primary database node is currently a single point of failure. We'll set up multi-region automated failover with heartbeat health checks and a standby replica continuously streaming updates."*

State the cost of that choice. **Asynchronous replication means a failover can lose the last few writes (non-zero RPO).** For location data that is fine; for payments, use synchronous or quorum replication within a region and accept the extra write latency. Also mention **WebSocket scaling**: connection state lives on specific servers, so you need sticky routing or a pub/sub layer, plus client reconnect with backoff.

**Rule of thumb:** Welcome bottlenecks as opportunities to showcase advanced patterns - event-driven queues, caching strategies, failover mechanics. Use them to demonstrate judgment, not ego.

---

## Phase 5: Resilience & Observability (35–45 min)

Most candidates think the interview ends when the architecture works on paper. Senior candidates know that **unmonitored systems are broken systems waiting to happen.**

Spend two minutes covering:

![Phase 5: Resilience & Observability](assets/phase_5_observability.jpg)

- **Distributed Tracing:** Inject a unique `trace_id` at the API gateway to track a single request across all microservices end-to-end.
- **Circuit Breakers:** If a downstream payment service slows down, a circuit breaker trips to prevent cascading failures across the entire system.
- **Timeouts, Retries and Bulkheads:** Set a timeout on every remote call, retry only idempotent operations with exponential backoff and jitter (otherwise retries amplify an outage), and isolate resource pools so one slow dependency cannot exhaust all threads.
- **Metrics and Alerting:** Name the signals - latency percentiles (p50/p99), error rate, queue lag, cache hit ratio - and alert on SLO burn rather than raw CPU.
- **Security (one line):** Authentication at the gateway, authorisation per service, TLS everywhere, and rate limiting per client.

### Final Synthesis
Close by walking through the blueprint you have delivered:
- 0–5 min: Defined scope and SLAs
- 5–10 min: Calculated order of magnitude and established API contracts
- 10–20 min: Built a simple, functional 5-box architecture
- 20–35 min: Solved bottlenecks and justified trade-offs
- 35–45 min: Added reliability and observability

---

## The 3 Red Flags That Trigger Rejection

1. **Over-engineering too early** - pitching multi-region sharding before confirming baseline QPS.
2. **Defensiveness** - treating the interviewer's questions as attacks rather than opportunities to explore trade-offs.
3. **Bad clock management** - spending 25 minutes on math and running out of time for the architecture diagram.

---

## Worked Example: Uber Location Pipeline

**Scale assumptions:** 1M active drivers, ping every 4 s, 10M riders/day.

| Step | Reasoning |
|---|---|
| Writes | 1M ÷ 4 s = **~250K location writes/s** (peak ~400K). Far beyond one DB node → queue + partitioning |
| Payload | ~100 bytes per ping × 250K/s = **~25 MB/s**. Bandwidth is trivial; write *rate* is the problem |
| Ingest | Drivers → WebSocket gateway → **Kafka** (partitioned by geo cell) → consumers update the in-memory index |
| Hot store | Redis (or in-memory grid) keyed by **cell ID** → set of driver IDs + last position, with a short TTL (~10-30 s) so offline drivers expire |
| Match query | Rider location → compute cell + 8 neighbours → read those cells → rank by distance/ETA. Touches 1-9 keys, not every shard |
| Hot cells | Split dense cells into finer resolution (H3 res +1) or salt the key |
| History | Async consumer batches pings to Cassandra / object storage for analytics, off the critical path |
| Consistency | Locations: **AP**, 2 s stale is fine. Ride assignment: **CP** with a lock/transaction so two riders never get the same driver |
| Failure | Redis replica per shard with auto-failover; if a cell is lost, it repopulates within one ping interval (~4 s) - no durable recovery needed |

**Why this beats `city_id + hash(driver_id)`:** the match query stays local to a few shards, and the repopulate-on-ping property makes a location store cheap to lose.

---

## Traps Checklist (scan before you finish)

- [ ] Did I state an **idempotency** story for every write that moves money or creates a ride?
- [ ] Does my **sharding key match my main read query**? (Even distribution is not the only goal.)
- [ ] Have I identified **hot keys** and how to split them?
- [ ] Do I know the **RPO/RTO** implied by my replication choice?
- [ ] Have I addressed **cache invalidation**, not just eviction?
- [ ] Do **retries** have backoff + jitter, and only apply to idempotent calls?
- [ ] Where does **connection state** live (WebSockets), and what happens on reconnect?
- [ ] Is there a **geospatial** or other domain-specific structure the problem actually hinges on?
- [ ] Did I **check in** with the interviewer at each phase boundary?

---

## What Good Looks Like

If you execute this structure, you separate yourself from candidates who wander without direction. You demonstrate:

- **Leadership** - you drive the conversation
- **Structure** - you work through a clear, repeatable framework
- **Technical depth** - you reason from first principles
- **Communication** - you make your thinking visible and defensible under pressure

That is what a senior engineer looks like in a room.
