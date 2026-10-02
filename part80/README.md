# Part 80: Milestone — World-Class Patterns Preview

> **"The difference between good and great is knowing WHEN to apply each pattern"**  
> ความแตกต่างระหว่างดีและดีเลิศคือการรู้ว่า เมื่อไหร่ควรใช้แต่ละ pattern

---

## สารบัญ

1. [Pattern Catalog Summary](#1-pattern-catalog-summary)
2. [Choosing the Right Pattern](#2-choosing-the-right-pattern)
3. [Anti-Patterns to Avoid](#3-anti-patterns-to-avoid)
4. [Performance Trade-off Map](#4-performance-trade-off-map)
5. [Career-Level Competency Guide](#5-career-level-competency-guide)
6. [What's Coming in Parts 81-100](#6-whats-coming-in-parts-81-100)

---

## 1. Pattern Catalog Summary

```
Patterns Covered So Far (Parts 1-80)
════════════════════════════════════════════════════════

PROCESS PATTERNS
  ✓ gen_server: mutable state server
  ✓ gen_statem: complex state machines
  ✓ gen_event: event handler dispatch
  ✓ Supervisor: fault tolerance tree
  ✓ Pool (poolboy): bounded concurrency
  ✓ Circuit breaker: failure isolation
  ✓ Rate limiter: token bucket / sliding window
  ✓ Worker pool: demand-driven processing

DATA PATTERNS
  ✓ ETS: lock-free concurrent read
  ✓ Mnesia: distributed transactions
  ✓ Event sourcing: append-only with replay
  ✓ CQRS: separate read and write models
  ✓ Saga: distributed transaction compensation
  ✓ Snapshot: aggregate state checkpoint
  ✓ Time-series: bucketed aggregations
  ✓ Inverted index: full-text search

COMMUNICATION PATTERNS
  ✓ Request-Reply (call/cast)
  ✓ Pub/Sub (pg, event_bus)
  ✓ Fan-out (scatter_gather)
  ✓ Pipeline (stage chain)
  ✓ Backpressure (demand producer)
  ✓ Dead letter queue
  ✓ Outbox pattern (transactional events)

RELIABILITY PATTERNS
  ✓ Let it crash
  ✓ Timeout + retry
  ✓ Idempotency keys
  ✓ Optimistic concurrency
  ✓ Pessimistic locking (SELECT FOR UPDATE)
  ✓ Immutable audit log
  ✓ Health checks (liveness / readiness)
  ✓ SLO tracking with error budgets
```

---

## 2. Choosing the Right Pattern

```erlang
%% decision_guide.erl — pattern selection guide
%% This is educational pseudocode — use as a guide

choose_process_pattern(Requirements) ->
    case Requirements of
        %% Need to hold mutable state and respond to queries?
        #{mutable_state := true, query_driven := true} ->
            gen_server;

        %% Complex lifecycle with many states?
        #{states := N} when N > 3 ->
            gen_statem;

        %% Multiple handlers for same events?
        #{multiple_handlers := true} ->
            gen_event;

        %% One-shot async computation?
        #{one_shot := true, no_state := true} ->
            spawn_link;

        %% Need fault isolation without OTP overhead?
        #{simple := true, monitored := true} ->
            spawn_monitor
    end.

choose_data_pattern(Requirements) ->
    case Requirements of
        %% Need audit trail? Replay capability?
        #{audit_required := true} ->
            event_sourcing;

        %% Write-heavy with separate read concerns?
        #{write_read_different := true} ->
            cqrs;

        %% Sub-millisecond reads, concurrent access?
        #{latency := sub_ms, concurrent_reads := true} ->
            ets;

        %% Shared state across nodes?
        #{distributed := true, consistency := strong} ->
            mnesia;

        %% Time-based analytics?
        #{time_series := true} ->
            time_series_buckets;

        %% Search with ranking?
        #{full_text_search := true} ->
            inverted_index
    end.

choose_failure_pattern(Requirements) ->
    case Requirements of
        %% External service calls?
        #{external_service := true} ->
            circuit_breaker;

        %% Rate limiting needed?
        #{rate_limiting := true, burst_allowed := true} ->
            token_bucket;

        %% Strict rate?
        #{rate_limiting := true, burst_allowed := false} ->
            sliding_window;

        %% Multi-step distributed process?
        #{distributed_transaction := true} ->
            saga_pattern;

        %% Idempotent writes?
        #{exactly_once := true} ->
            idempotency_key
    end.
```

---

## 3. Anti-Patterns to Avoid

```
TOP 10 ERLANG ANTI-PATTERNS
════════════════════════════════════════════════════════

1. THE GOD PROCESS
   Problem: One gen_server handles everything
   Symptom: Message queue grows under load
   Fix: Split by domain, add pools

2. SYNCHRONOUS CHAIN
   Problem: A→B→C→D all with gen_server:call
   Symptom: Total latency = sum of all parts
   Fix: Async where possible, parallelize independent calls

3. CATCH ALL WITHOUT LOGGING
   Problem: catch _:_ -> ok
   Symptom: Silent failures, debugging nightmare
   Fix: At minimum log the error with context

4. PROCESS PER REQUEST (Unbounded)
   Problem: spawn(fun() -> handle(Req) end) for every request
   Symptom: Memory exhaustion under load
   Fix: poolboy or demand-driven producer

5. ETS WITHOUT CLEANUP
   Problem: Insert to ETS, never delete
   Symptom: Memory grows until OOM
   Fix: TTL field + periodic cleanup, or ordered_set with eviction

6. BINARY STRING OPERATIONS IN HOT PATH
   Problem: <<A/binary, B/binary>> in loop
   Symptom: High GC pressure, copies
   Fix: Use iolist, binary:join/2

7. RECURSIVE LOOP WITHOUT tail_call
   Problem: f(X) -> ... f(X-1).   (not tail recursive)
   Symptom: Stack overflow on large inputs
   Fix: Accumulator pattern, lists:foldl

8. TIMER MESSAGE FLOOD
   Problem: Every process sends itself periodic messages
   Symptom: Scheduler overloaded with timer messages
   Fix: Shared timer process, coalesce timers

9. MNESIA FOR ANALYTICS
   Problem: Using Mnesia for time-series or analytics queries
   Symptom: Very slow aggregations
   Fix: PostgreSQL or ClickHouse for analytical queries

10. HOT ATOM CREATION
    Problem: binary_to_atom/1 with user input
    Symptom: Atom table fills up, node crashes
    Fix: binary_to_existing_atom/2 for known atoms,
         keep binaries for unknowns
```

---

## 4. Performance Trade-off Map

```
Erlang Data Structure Decision Matrix
════════════════════════════════════════════════════════

                    Read      Write    Memory    Dist   Persist
Process dict         O(1)      O(1)     Low      No      No
ETS (set)            O(1)      O(log)   Med      No      No
ETS (ordered_set)    O(log)    O(log)   Med      No      No
Mnesia (ram)         O(log)    O(log)   High     Yes     No
Mnesia (disk)        O(log)    O(log)   High     Yes     Yes
PostgreSQL           O(log)    O(log)   Med      No      Yes
Redis                O(1)      O(1)     Med      Yes     Opt

Recommended uses:
  Process dict   → scratch state within a single gen_server
  ETS set        → fast key lookup, session cache, counters
  ETS ord_set    → sorted data, range queries, leaderboards
  Mnesia ram     → distributed coordination, soft-realtime
  Mnesia disk    → configuration, routing tables
  PostgreSQL     → domain data, relationships, ACID
  Redis          → distributed cache, rate limits, pubsub
```

---

## 5. Career-Level Competency Guide

```
Erlang Engineer Competency Matrix
════════════════════════════════════════════════════════

JUNIOR (Parts 1-20)
  □ Understand functional programming concepts
  □ Write basic modules with exports
  □ Use lists, maps, tuples, binary
  □ Basic pattern matching and guards
  □ Simple gen_server
  □ Basic supervision tree

MID-LEVEL (Parts 21-50)
  □ Design supervision hierarchies
  □ Write gen_statem for complex state machines
  □ Use ETS for caching and counters
  □ Build REST APIs with Cowboy
  □ Connect to PostgreSQL with poolboy
  □ Write properties and unit tests
  □ Debug with observer and process_info

SENIOR (Parts 51-70)
  □ Implement circuit breakers and rate limiters
  □ Design CQRS + event sourcing architectures
  □ Write parse transforms
  □ Understand BEAM internals (schedulers, GC)
  □ Performance profiling (fprof, eprof, cprof)
  □ Hot code upgrades (appup/relup)
  □ Distributed systems basics (RPC, Mnesia)

STAFF/PRINCIPAL (Parts 71-100)
  □ Architect multi-service platforms
  □ Implement consensus algorithms
  □ Design for 99.99% availability
  □ Production monitoring and SLOs
  □ Mentor team on OTP patterns
  □ Contribute to open source Erlang libraries
  □ Make language/framework selection decisions

WORLD-CLASS (Parts 81-100)
  □ Contribute to OTP itself
  □ Design novel distributed algorithms
  □ Lead cross-team architectural decisions
  □ Published conference talks
  □ Author widely-used libraries
```

---

## 6. What's Coming in Parts 81-100

```
Parts 81-100: World-Class Erlang Engineering
════════════════════════════════════════════════════════

81. Security Engineering — JWT, OAuth2, RBAC, audit
82. Multi-Tenancy Architecture — tenant isolation, quotas
83. API Design at Scale — versioning, deprecation, SDKs
84. Testing Strategies — property-based, integration, load
85. Database Patterns Deep Dive — sharding, read replicas
86. Caching Architecture — multi-layer, invalidation
87. Message Queue Patterns — ordering, deduplication
88. gRPC and Protocol Buffers — high-performance APIs
89. Contributing to OTP — understanding internals
90. Building Your Own Behaviour — advanced metaprogramming
91. Game Server — low-latency, real-time, physics tick
92. IoT Platform — device management, MQTT, data pipeline
93. Financial Systems — double-entry bookkeeping, reconciliation
94. Developer Tools — CLI, code generators, scaffolding
95. Open Source Library — design, documentation, versioning
96. Production War Stories — real incidents, postmortems
97. Architecture Reviews — how to critique and improve designs
98. Team Processes — code review, RFC, ADR practices
99. Capstone Project — design a complete platform
100. Graduation — next steps in your Erlang journey
```

---

## สรุป Part 80 — Second Milestone

✅ 80 parts complete — halfway to world-class  
✅ Pattern catalog: 30+ patterns across process/data/communication/reliability  
✅ Decision framework: when to use each pattern  
✅ Top 10 anti-patterns: what to avoid and why  
✅ Performance matrix: data structure tradeoffs  
✅ Career competency guide: junior → world-class progression  

---

**🎯 Checkpoint: Parts 1-80 Complete**
- You now know more Erlang than 99% of developers
- Parts 81-100 = true world-class mastery

---

*Part 80/100 | [← ก่อนหน้า](../part79/README.md) | [ถัดไป →](../part81/README.md)*
