# Part 97: Architecture Reviews — Critique and Improve Designs

> **"Architecture review is not about finding fault; it's about finding better paths"**  
> การ review architecture ไม่ใช่การหาข้อผิดพลาด — แต่คือการหาเส้นทางที่ดีกว่า

---

## สารบัญ

1. [How to Review an Architecture](#1-how-to-review-an-architecture)
2. [Case Study: Monolith to Microservices](#2-case-study-monolith-to-microservices)
3. [Case Study: Synchronous to Async Pipeline](#3-case-study-synchronous-to-async-pipeline)
4. [Case Study: Scaling the Database Layer](#4-case-study-scaling-the-database-layer)
5. [Architecture Decision Records](#5-architecture-decision-records)
6. [Common Architecture Smells](#6-common-architecture-smells)
7. [แบบฝึกหัด](#7-แบบฝึกหัด)

---

## 1. How to Review an Architecture

```
Architecture Review Framework
═══════════════════════════════════════════════════════

QUESTIONS TO ASK:

1. CORRECTNESS
   - What are the consistency guarantees?
   - What happens when component X fails?
   - Is there a single point of failure?
   - Can data be lost? Under what conditions?

2. PERFORMANCE
   - Where are the bottlenecks? (hint: usually the database)
   - What is the hot path latency budget?
   - What happens at 10x current load?
   - Are there O(n) operations disguised as O(1)?

3. OPERABILITY
   - Can it be deployed without downtime?
   - How do we debug it when it fails at 3am?
   - What does healthy look like? (metrics, alerts)
   - Can we roll back?

4. SIMPLICITY
   - Is each component doing one thing?
   - Can a new team member understand it in a day?
   - Are there accidental complexities? (vs essential)

5. EVOLUTION
   - Which parts will change most often?
   - Are the stable and volatile parts separated?
   - Does the design constrain future changes?

REVIEW PROCESS:
  □ Read the design doc (ADR, RFC, or diagram)
  □ Walk through the happy path
  □ Walk through every failure mode
  □ Challenge assumptions explicitly stated
  □ Challenge unstated assumptions too
  □ Propose alternatives, not just problems
  □ Estimate cost of each concern (probability × impact)
```

---

## 2. Case Study: Monolith to Microservices

```
BEFORE: Single Erlang application
════════════════════════════════════════════════

myapp (monolith)
  ├── user management
  ├── order processing
  ├── inventory
  ├── notifications
  └── billing

Problems identified:
  1. Deployment: change to notifications requires full deploy
  2. Scaling: can't scale just inventory during peak
  3. Team: 8 engineers modifying same codebase → conflicts
  4. Technology: need ML (Python) for recommendations

AFTER: Service boundary proposal
════════════════════════════════════════════════

user-service      → PostgreSQL (users DB)
order-service     → PostgreSQL (orders DB) + Kafka events
inventory-service → PostgreSQL (inventory DB)
notification-svc  → Twilio, SendGrid  
billing-service   → Stripe, PostgreSQL (billing DB)
recommendation    → Python ML service

REVIEW FINDINGS:

CONCERN 1 (High): Distributed transactions
  Before: Creating order atomically updates inventory (same DB)
  After:  order-service and inventory-service are separate DBs
  Risk:   Order placed but inventory not reserved (data inconsistency)
  Fix:    Saga pattern (Part 74): order-service publishes event,
          inventory-service listens and reserves, publishes confirm/fail
          order-service listens and confirms/cancels order

CONCERN 2 (Medium): Network latency added
  Before: order.create() calls inventory.reserve() in ~1ms (same process)
  After:  HTTP call adds ~5-10ms per hop
  Fix:    Accept the latency for small operations;
          use async events for non-critical paths

CONCERN 3 (Low): Operational complexity
  Before: 1 service to monitor, deploy, restart
  After:  6 services with their own pipelines
  Fix:    Invest in platform tooling first (Kubernetes, service mesh);
          don't split until team is > 10 engineers

RECOMMENDATION:
  Do NOT migrate to microservices yet.
  Instead: split monolith into separate OTP applications
  within same repo (umbrella project). Same benefits, less overhead.
```

---

## 3. Case Study: Synchronous to Async Pipeline

```
BEFORE: Synchronous order processing
════════════════════════════════════════════════

HTTP POST /orders
  │
  ▼ (1) Validate input              ~1ms
  ▼ (2) Check inventory             ~5ms (DB)
  ▼ (3) Create order record         ~10ms (DB)
  ▼ (4) Send confirmation email     ~200ms (SMTP)
  ▼ (5) Update analytics             ~50ms (DB)
  ▼ (6) Notify warehouse            ~100ms (HTTP)
  │
  └─ Total: ~366ms P50, ~2s P99 (SMTP timeout)

PROBLEMS:
  - P99 dominated by email delivery
  - Failure in step 5 or 6 rolls back entire order
  - Can't retry individual steps

PROPOSED: Async pipeline
════════════════════════════════════════════════

HTTP POST /orders
  │
  ▼ (1) Validate input              ~1ms
  ▼ (2) Check + reserve inventory   ~5ms (DB with SELECT FOR UPDATE)
  ▼ (3) Create order + publish event~10ms (DB + Kafka, same txn via outbox)
  │
  └─ Return 202 Accepted: {order_id, status: processing}

Background (async, each independently retryable):
  ▼ email_worker: send confirmation
  ▼ analytics_worker: update dashboards
  ▼ warehouse_notifier: send picking request

REVIEW FINDINGS:

IMPROVEMENT (High): P50 drops from 366ms to ~16ms
IMPROVEMENT (High): Each async step retries independently on failure
TRADEOFF (Medium): Order confirmation email may arrive 2-3s late
CONCERN (Medium): Order status = 'processing' briefly — client must poll or listen

IMPLEMENTATION NOTE:
  The outbox pattern (Part 87) guarantees that if the DB transaction
  commits, the Kafka event WILL be published eventually.
  This prevents the "order created but no email sent" ghost order problem.

```erlang
%% The key: write order + outbox message in ONE transaction
create_order_with_outbox(CustomerId, Items) ->
    db_pool:transaction(fun(Conn) ->
        {ok, OrderId} = orders:create(Conn, CustomerId, Items),
        outbox:send_in_txn(Conn, OrderId, order_created, #{
            customer_id => CustomerId,
            items       => Items
        }),
        {ok, OrderId}
    end).
```

---

## 4. Case Study: Scaling the Database Layer

```
BEFORE: Single PostgreSQL, 500 QPS, P99 = 80ms
════════════════════════════════════════════════

Problem: approaching 80% CPU on PostgreSQL
Write operations: 100 QPS
Read operations:  400 QPS (mostly user profile, product catalog)

OPTION A: Vertical scaling (bigger machine)
  Pros: Simple, no code change
  Cons: Expensive, temporary, doesn't scale writes
  Recommendation: Quick fix only, not long-term

OPTION B: Read replicas
  Pros: 4x read capacity, simple Erlang change (see Part 85)
  Cons: Adds replica lag (~50ms), consistency-sensitive reads must use primary
  Recommendation: DO THIS FIRST

OPTION C: Caching layer (ETS + Redis, see Part 86)
  Cache user profiles (read heavy, rarely updated)
  Cache product catalog (read heavy, updated ~hourly)
  Estimated: reduces DB reads by 60%
  Recommendation: DO THIS SECOND

OPTION D: Sharding by user_id
  Pros: Near-linear write scaling
  Cons: Complex, no cross-shard queries, hard to rebalance
  When to do: writes > 10k QPS, after read replicas and caching exhausted
  Recommendation: NOT YET

OPTION E: CQRS with separate read store
  Write to PostgreSQL, sync to Elasticsearch for search/analytics
  Pros: Read store optimized for query patterns
  Cons: Eventual consistency, sync complexity
  Recommendation: Only if complex query patterns can't be served by PostgreSQL

IMPLEMENTATION PLAN:
  Week 1-2:  Add read replicas (Part 85 code, test lag tolerance)
  Week 3-4:  Add ETS cache for hot paths
  Week 5-6:  Add Redis cache for cross-node sharing
  Month 2+:  Measure — if still bottlenecked, consider sharding
```

---

## 5. Architecture Decision Records

```markdown
# ADR-042: Use Outbox Pattern for Event Publishing

**Date:** 2026-03-10
**Status:** Accepted
**Authors:** Alice, Bob
**Deciders:** Engineering Lead

## Context

We need to publish Kafka events when orders are created.
Direct Kafka publish after DB commit risks: DB commits but Kafka publish fails
→ order exists but downstream services never notified.

## Decision

Use the transactional outbox pattern:
1. Write order AND outbox record in same DB transaction
2. Background process polls outbox and publishes to Kafka
3. Mark outbox records as published after successful Kafka delivery

## Consequences

**Positive:**
- Guaranteed delivery: if DB commits, event will be published
- Event delivery failures don't affect the order creation response
- Outbox records provide audit trail

**Negative:**
- Events are delivered with slight delay (0-1 second typical)
- Additional table to manage (cleanup job needed)
- Outbox poller is a new component to operate

## Alternatives Considered

**A: Direct Kafka publish after DB commit**
  Rejected: Silent data loss if Kafka unavailable

**B: Kafka exactly-once transactions**
  Rejected: Requires Kafka transactions which add complexity and latency;
  outbox is simpler and sufficient for our consistency requirements

## Review Date

2026-09-10 — review if outbox performance becomes a bottleneck
```

---

## 6. Common Architecture Smells

```
Architecture Smell Catalog
═══════════════════════════════════════════════════════

SMELL: Distributed Monolith
  Signs: Services communicate synchronously for every request;
         one service deployment requires coordinated deploy of others
  Fix:   Async events; accept eventual consistency

SMELL: Chatty API
  Signs: Client makes 10+ requests to render one page
  Fix:   Aggregation layer (BFF - Backend for Frontend)
        GraphQL for flexible queries; batch endpoints

SMELL: Shared Database
  Signs: Two services write to the same DB tables
  Fix:   One service owns each table; others read via API or events

SMELL: God Service
  Signs: One service has 50+ endpoints, 20+ DB tables
  Fix:   Domain-driven decomposition; identify bounded contexts

SMELL: Synchronous Chain
  Signs: A → B → C → D all synchronous calls in the hot path
  Fix:   Async at earliest possible point; fan-out where independent

SMELL: Premature Microservices
  Signs: 3-person team, 12 services, 80% time spent on infrastructure
  Fix:   Consolidate; use OTP umbrella before true microservices

SMELL: Missing Circuit Breaker
  Signs: One slow dependency can hang all requests
  Fix:   Circuit breaker + fallback for every external call

SMELL: No Health Check
  Signs: Load balancer can't tell if service is actually working
  Fix:   /health (liveness) and /ready (readiness) endpoints
        Readiness checks DB, cache, upstream dependencies

SMELL: Leaky Abstraction
  Signs: Domain service returns database rows or SQL errors to callers
  Fix:   Domain services return domain types; translate at boundary
```

---

## 7. แบบฝึกหัด

1. Review architecture ของ project ที่ทำงานอยู่ด้วย framework ใน Section 1
2. เขียน ADR สำหรับ decision ที่สำคัญที่สุดใน project ของคุณ (ที่ไม่ได้บันทึกไว้)
3. วาด sequence diagram สำหรับ happy path + 3 failure modes
4. ระบุ architecture smells ใน codebase ที่คุณรู้จัก พร้อม priority ว่าจะแก้ตามลำดับใด

---

## สรุป Part 97

✅ Review framework: correctness, performance, operability, simplicity, evolution  
✅ Monolith → microservices: distributed transaction risk, saga fix, when not to do it  
✅ Sync → async: outbox-based pipeline, P50/P99 improvement, client UX tradeoffs  
✅ Database scaling: progression from read replicas → cache → sharding → CQRS  
✅ ADR format: context, decision, consequences, alternatives, review date  
✅ Architecture smells: 8 patterns with detection signs and fixes  

---

*Part 97/100 | [← ก่อนหน้า](../part96/README.md) | [ถัดไป →](../part98/README.md)*
