# Designing an Asynchronous Assessment Processing System

* **Category:** System Design
* **Target level:** Mid-level to Senior
* **Practice scenario:** Design a backend that accepts assessments, processes them asynchronously, and returns results to an integrating platform.
* **Format:** Use the 45-minute framework as a practice structure; adapt it to the actual prompt and interview duration.

> This scenario is a practice exercise. Clarify the requirements in the interview rather than assuming that every assessment platform works the same way.

## What a strong answer demonstrates

Show that you can take an ambiguous backend problem, clarify the requirements, design a simple end-to-end solution, and reason about correctness and operations under failure. State your assumptions, invite the interviewer to challenge them, and avoid adding infrastructure before the scale or requirements justify it.

For an assessment-processing system, useful concerns include traceability, consistent results, customer integration, and recoverability. Treat these as questions to explore, not assumptions about the product.

## 45-minute answer plan

### Phase 1 — Scope and requirements (0–5 minutes)

Open with:

> “I’ll focus on submitting an assessment, processing it, and returning a result. Before drawing the design, I’d like to clarify the expected volume and turnaround time, whether marking is asynchronous, and whether an assessor must approve a result before it is released.”

Clarify:

- Who submits assessments: learners, customer platforms, or both?
- Are submissions individual, batch, or both?
- What does the caller need immediately: a final result or an accepted submission ID and status?
- Does a human need to review results before learners see them?
- What are normal and peak submission rates, payload sizes, and acceptable processing times?
- Are customer data and assessment configurations isolated by tenant?
- What retention, audit, and availability requirements apply?

Keep the first version to three core capabilities: accept a submission, process it, and return its status/result. Defer analytics, adaptive learning, and other adjacent features unless asked.

### Phase 2 — Estimates and API/data contracts (5–10 minutes)

Estimate only enough to guide architecture. For example, divide the stated daily submission volume by seconds per day to estimate average rate, then apply an agreed peak multiplier. Ask about batch sizes and processing latency because worker concurrency and queue depth depend on them.

If the interviewer does not give numbers, state a default aloud and ask them to correct it:

| Assumption | Value | Implication |
|---|---|---|
| Volume | 5M assessments/month | ≈ 2/s average |
| Peak | 10x average, driven by batch uploads | ≈ 20/s submissions |
| Marking time | 5-30 s per assessment (LLM call) | ≈ 100-600 jobs in flight at peak |
| Payload | ~10-100 KB text; larger if files/audio | Text fits in Postgres; files go to S3 |

The takeaway: submission rate is trivial for FastAPI + Postgres. The pressure is **in-flight marking work and the LLM provider's rate limits**, not the API.

Possible API contract:

- `POST /v1/assessments` accepts a submission and an idempotency key; returns `202 Accepted` with an assessment ID and status URL.
- `GET /v1/assessments/{id}` returns processing state or the result when available.
- Optional customer webhook notifies a platform when processing completes; delivery is retried and is independently observable.

If submissions can include files or audio, return a presigned S3 upload URL (or accept a file reference) and store only the object key and metadata in PostgreSQL, not the blob.

Use PostgreSQL as the source of truth for submissions, processing state, and results. Define the idempotency scope, such as `(tenant_id, idempotency_key)`, and make status transitions explicit. Keep sensitive submission content out of logs.

### Phase 3 — High-level architecture and happy path (10–20 minutes)

**Start with the simplest viable version, then evolve it.** At ~2-20 submissions/s, PostgreSQL can act as the job queue itself:

```text
Customer platform → FastAPI → PostgreSQL (assessments + jobs table)
                                   ▲
                    Python workers poll with
                    SELECT … FOR UPDATE SKIP LOCKED
                                   │
                            Marking service / AI
```

- The submission and its job row are written in **one transaction**, so there is no dual-write problem and no outbox to build or operate.
- Workers claim a row with `FOR UPDATE SKIP LOCKED`, set a lease (`locked_until`), and commit the claim *before* calling the marking service.
- For a lean team this means one fewer system to run, monitor and debug.

Say when you would evolve it: sustained throughput that strains Postgres, the need for independent consumers or fan-out, or strict isolation between workloads. At that point move to the managed-queue design below, which needs the outbox to keep Postgres and the queue consistent.

**Evolved design (managed queue + outbox):**

```text
Customer platform
       │ HTTPS API
       ▼
FastAPI service ─────── PostgreSQL
                           │ submission + outbox event
                           ▼
                     Outbox publisher
                           │
                           ▼
                    Managed queue (e.g. SQS)
                           │
                           ▼
                    Python worker pool
                           │
                           ▼
                  Marking service / AI
                           │
                           ▼
              PostgreSQL result + audit data
                           │
                 API status / webhook delivery
                           ▼
                  Customer platform
```

Trace one request aloud:

1. FastAPI authenticates the customer, checks tenant access, validates the payload, and applies size/rate limits.
2. In one PostgreSQL transaction, it writes the submission, initial status, and outbox event. It returns the assessment ID without waiting for marking.
3. A publisher sends committed outbox events to the queue. A worker claims a job, invokes the marking capability, and persists the result and completion state.
4. The customer polls the status endpoint or receives a webhook. Webhook retries do not rerun marking.

Make the status an explicit state machine: `RECEIVED → QUEUED → PROCESSING → COMPLETED | FAILED | NEEDS_REVIEW`, with only allowed transitions enforced by conditional updates (`UPDATE … WHERE status = 'QUEUED'`).

State each component's purpose. Keep the initial architecture small; use a managed queue if its delivery semantics and operational burden fit the requirements. Avoid introducing Kafka, Redis, microservices, or multiple regions without a demonstrated need.

### Phase 4 — Deep dives, trade-offs, and failure modes (20–35 minutes)

**Duplicate requests and at-least-once delivery**

A queue may deliver a message more than once. Make the worker idempotent using a stable job/assessment ID, database uniqueness constraints, and conditional state transitions. A repeated customer request with the same idempotency key should return the original assessment rather than create another one.

**Database and queue consistency**

A database commit followed by a failed queue publish can strand work; publishing before commit can create work for a submission that rolls back. The transactional outbox records the submission and event in one transaction. A publisher retries delivery. Consumers still need idempotency because the publisher can send more than once.

**Worker failures and slow marking**

Set explicit timeouts and bounded retries with backoff for transient failures. Distinguish retryable errors from invalid input or permanent failures. Send exhausted jobs to a dead-letter/review path, expose their state, and provide a safe replay procedure. Limit concurrency to protect PostgreSQL and the marking service; scale workers based on queue age and throughput.

**Stuck jobs and lease recovery**

A worker can die after claiming a job. Give each claim a lease (`locked_until`) that the worker extends with a heartbeat on long jobs. A reconciler periodically finds `PROCESSING` rows whose lease has expired and returns them to `QUEUED` (incrementing an attempt count, and moving to the dead-letter/review path after the limit). With SQS, the equivalent is a visibility timeout longer than the maximum job time, plus a heartbeat extension. Because jobs can then run twice, the result write must be conditional and idempotent.

**Tenant fairness and noisy neighbours**

One customer submitting a 100K-assessment batch must not starve everyone else. Options, in order of simplicity: a per-tenant cap on concurrently `PROCESSING` jobs (enforced in the claim query), per-tenant rate limits at the API, and weighted or priority queues for different customer tiers. Return `429` with `Retry-After` or accept batches into a throttled backlog, and make the backlog visible to the customer through status.

**The AI marking pipeline**

The LLM call is usually the bottleneck and the least predictable component, so design for it explicitly:

- **Rate limits and cost:** the provider's tokens-per-minute limit caps throughput. Use a shared limiter across workers, honour `429`/`Retry-After`, and track cost per tenant.
- **Consistent results:** pin the model and prompt version, use low temperature, and request structured output validated against a schema before it is saved. Invalid output is a retryable failure with a bounded attempt count.
- **Quality control:** keep a golden set of human-marked assessments and run it whenever a prompt or model changes, so regressions are caught before release. Compare score distributions in production for drift.
- **Human review routing:** results with low confidence, schema repair, disagreement between runs, or high-stakes flags go to `NEEDS_REVIEW` instead of being released automatically.
- **Resilience:** circuit-break a failing provider, optionally fail over to a second model, and queue work instead of dropping it during an outage.
- **Efficiency:** cache identical inputs, and batch where the provider and turnaround allow it.
- **Boundary with AI engineers:** treat the marking capability as a versioned interface (input, output schema, version) so the AI team can change internals without a backend release.

**Results, versions, and human review**

Store the assessment/rubric version and marking-service or model version with the result, plus timestamps and any reviewer decision. Preserve an audit trail when a human changes or approves a result. Define whether a re-mark creates a new version or supersedes the previous one.

**Customer integrations**

Authenticate each integration, enforce tenant boundaries, sign webhooks (HMAC over the body plus a timestamp, so customers can verify and reject replays), and give webhook delivery its own retry state. Delivery is at-least-once and unordered, so include a unique event ID and the assessment version so customers can deduplicate and ignore stale updates. A customer endpoint outage should not block the marking pipeline. Make status and error responses actionable without leaking another tenant's data.

**Data protection and retention**

Assessment answers are likely personal data, possibly about minors. Encrypt in transit and at rest, scope every query by `tenant_id`, keep secrets in a managed store, and redact content from logs. Ask about retention and deletion requirements (for example GDPR erasure) early, because they affect whether you can keep an immutable audit trail and how you handle backups.

**Scaling choices**

Start with indexes for actual query paths, such as tenant plus assessment ID and status plus creation time for worker/reconciliation tasks. Add connection pooling, read replicas, partitioning, or caching only in response to measured bottlenecks and consistency needs. Explain the operational costs of each step.

### Phase 5 — Resilience and observability (35–45 minutes)

Track and alert on:

- API error and latency rates, submission acceptance rate, and idempotency conflicts.
- Queue depth and oldest-message age, worker throughput, retry counts, and dead-letter volume.
- Marking-service latency/errors and end-to-end time from submission to result.
- Webhook success rate, delivery lag, and repeated failures.
- PostgreSQL saturation, connection pool use, slow queries, and storage growth.

Propagate a correlation ID from API request through outbox, queue, worker, and webhook. Use structured logs and distributed traces, while excluding assessment answers and other sensitive content. Describe backup/restore expectations and how operators can inspect and safely replay a failed job.

Close with:

> “The core design accepts work durably and responds quickly, then processes it asynchronously. PostgreSQL owns the authoritative state, the outbox prevents lost handoffs to the queue, and idempotent workers make retries safe. I’d scale workers based on queue age and protect result integrity with versioning and an audit trail. I’d validate the details against the volume, turnaround, and review requirements we established.”

## Implementation details to be ready to discuss

Translate the design into the technologies named in the job description. Depending on the stack, be ready to discuss:

- Request validation, dependency boundaries, authentication, and async I/O versus CPU-bound work. A concrete position: use `async def` endpoints with an async driver (`asyncpg`) for I/O-bound handlers; never call blocking code (sync SDKs, CPU-heavy parsing) inside them, and push heavy work to the worker.
- ORM transaction/session lifecycle, unique constraints, migrations, indexes, and avoiding long transactions around external calls. For SQLAlchemy: one session per request (via a FastAPI dependency) or per job, commit explicitly, and use `INSERT … ON CONFLICT DO NOTHING` (or catch `IntegrityError`) for idempotency keys.
- Connection budgeting: `pool_size × (API processes + worker processes)` must stay below Postgres `max_connections`, otherwise use PgBouncer. This is a common cause of production incidents when workers scale out.
- Worker framework choice: a plain SQS or Postgres-polling consumer is simpler to reason about than Celery; choose Celery/arq only if you need its scheduling, chaining or ecosystem, and know its acks-late and visibility-timeout behaviour.
- Migrations without downtime: add nullable columns first, backfill, then enforce constraints; avoid long locks on large tables (for example `CREATE INDEX CONCURRENTLY`).
- Safe concurrency and worker idempotency; never hold a database transaction open while waiting on a slow external service.
- Cloud choices for queues, compute, secrets, object storage, and monitoring, tied to concrete requirements.
- API compatibility, pagination, rate limiting, webhook authentication, retries, timeouts, and tenant-specific configuration.
- Deployment, migrations, rollback, health checks, and operational ownership in a lean team.

## Questions to ask your interviewers

- “Which backend reliability problem is most important for this team to solve next?”
- “Where does the system spend most of its time: submission, processing, integrations, or review?”
- “How do you represent and audit changes to customer-specific assessment criteria?”
- “How do you balance fast feedback with review and control over high-stakes outcomes?”
- “What would success look like in the first three months?”

## Interview reminders

- Drive the conversation, but make assumptions explicit and invite correction.
- Explain why each box exists and trace one request end to end.
- Prefer a correct, observable baseline over premature sharding or multi-region design.
- When challenged, revisit the requirement and trade-off calmly.
- Use concrete examples from your own work; be precise about your contribution and the outcomes you can support. Prepare two or three short stories (a duplicate-processing bug, a retry storm, a migration or scaling decision) with the problem, your action and the result, and attach them to the matching section of the design.
- Keep the diagram small. Show the simplest version first and say where it would grow, instead of drawing eight boxes in the first ten minutes.
- Expect the interviewer to steer toward their own stack and pain points (AI latency, customer integrations). Follow the steer and tie it back to the design.

## Pre-interview checklist

- [ ] Do I have default volume and latency numbers ready if the interviewer gives none?
- [ ] Can I explain Postgres-as-queue **and** when I would move to SQS + outbox?
- [ ] Have I covered idempotency at three layers: API key, queue/job claim, result write?
- [ ] Do I have an answer for stuck jobs (lease + reconciler)?
- [ ] Do I have an answer for one tenant flooding the system?
- [ ] Can I describe the AI-call risks: rate limits, cost, nondeterminism, schema validation, human-review routing?
- [ ] Can I explain webhook signing, deduplication and retries?
- [ ] Have I connected each concern to a real example from my own work?
