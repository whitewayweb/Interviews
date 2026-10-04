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

Possible API contract:

- `POST /v1/assessments` accepts a submission and an idempotency key; returns `202 Accepted` with an assessment ID and status URL.
- `GET /v1/assessments/{id}` returns processing state or the result when available.
- Optional customer webhook notifies a platform when processing completes; delivery is retried and is independently observable.

Use PostgreSQL as the source of truth for submissions, processing state, and results. Define the idempotency scope, such as `(tenant_id, idempotency_key)`, and make status transitions explicit. Keep sensitive submission content out of logs.

### Phase 3 — High-level architecture and happy path (10–20 minutes)

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

State each component's purpose. Keep the initial architecture small; use a managed queue if its delivery semantics and operational burden fit the requirements. Avoid introducing Kafka, Redis, microservices, or multiple regions without a demonstrated need.

### Phase 4 — Deep dives, trade-offs, and failure modes (20–35 minutes)

**Duplicate requests and at-least-once delivery**

A queue may deliver a message more than once. Make the worker idempotent using a stable job/assessment ID, database uniqueness constraints, and conditional state transitions. A repeated customer request with the same idempotency key should return the original assessment rather than create another one.

**Database and queue consistency**

A database commit followed by a failed queue publish can strand work; publishing before commit can create work for a submission that rolls back. The transactional outbox records the submission and event in one transaction. A publisher retries delivery. Consumers still need idempotency because the publisher can send more than once.

**Worker failures and slow marking**

Set explicit timeouts and bounded retries with backoff for transient failures. Distinguish retryable errors from invalid input or permanent failures. Send exhausted jobs to a dead-letter/review path, expose their state, and provide a safe replay procedure. Limit concurrency to protect PostgreSQL and the marking service; scale workers based on queue age and throughput.

**Results, versions, and human review**

Store the assessment/rubric version and marking-service or model version with the result, plus timestamps and any reviewer decision. Preserve an audit trail when a human changes or approves a result. Define whether a re-mark creates a new version or supersedes the previous one.

**Customer integrations**

Authenticate each integration, enforce tenant boundaries, validate webhook signatures, and give webhook delivery its own retry state. A customer endpoint outage should not block the marking pipeline. Make status and error responses actionable without leaking another tenant's data.

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

- Request validation, dependency boundaries, authentication, and async I/O versus CPU-bound work.
- ORM transaction/session lifecycle, unique constraints, migrations, indexes, and avoiding long transactions around external calls.
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
- Use concrete examples from your own work; be precise about your contribution and the outcomes you can support.
