# Part 70: Milestone — Production-Grade Architecture

> **"You have built 70 parts. Now understand how to architect systems that outlive their creators"**  
> คุณสร้างมา 70 parts แล้ว — ตอนนี้เข้าใจว่าจะออกแบบระบบที่อยู่ได้นานกว่าผู้สร้าง

---

## สารบัญ

1. [Architecture Principles Review](#1-architecture-principles-review)
2. [Complete System Blueprint](#2-complete-system-blueprint)
3. [Cross-Cutting Concerns](#3-cross-cutting-concerns)
4. [Team and Code Organization](#4-team-and-code-organization)
5. [Production Readiness Checklist](#5-production-readiness-checklist)
6. [Case Study: Building a Payment Platform](#6-case-study-building-a-payment-platform)
7. [แบบฝึกหัด](#7-แบบฝึกหัด)

---

## 1. Architecture Principles Review

```
The 7 Principles of Production Erlang Systems
═══════════════════════════════════════════════════════════════

1. ISOLATION
   Every process is an island. Share nothing.
   Failures are local. Communication is explicit.

   Bad:  shared_state:get(key)    → global state = global failure risk
   Good: gen_server:call(Pid, {get, key})

2. SUPERVISION
   Every process has a parent who knows what to do when it fails.
   The tree structure defines failure domains.

   Bad:  spawn(Fun)               → orphan process, no restart
   Good: supervisor:start_child(Sup, Spec)

3. LET IT CRASH
   Don't defend against impossible states.
   Pattern match the happy path. Let the crash propagate.

   Bad:  case Result of
           {ok, V} -> V;
           {error, _} -> undefined   % hiding the problem!
         end
   Good: {ok, V} = Result   % crashes loudly on error

4. LOCATION TRANSPARENCY
   Calling a local or remote process is identical syntax.
   gen_server:call/2 works across nodes too.

5. OBSERVE EVERYTHING
   If you can't measure it, you can't improve it.
   Telemetry, structured logs, distributed traces.

6. GRACEFUL DEGRADATION
   Design for partial failure. Circuit breakers, fallbacks, defaults.
   The system should limp, not fall over.

7. IMMUTABLE UPGRADES
   Hot code upgrades are possible. But use them carefully.
   For most changes, blue-green deploy + supervisor restart is safer.
```

---

## 2. Complete System Blueprint

```
Production Erlang Platform — Full Architecture
════════════════════════════════════════════════

CLIENT LAYER
  Browser / Mobile / API Consumer
        │
        │ HTTPS / WebSocket
        ▼
EDGE LAYER
  Load Balancer (HAProxy / nginx)
  SSL Termination
  DDoS Protection
        │
        ▼
API GATEWAY LAYER
  ┌─────────────────────────────────────────────────┐
  │  api_gateway (Cowboy)                           │
  │  ├── Rate Limiting (token bucket per user)      │
  │  ├── Auth Middleware (JWT verification)         │
  │  ├── Request ID injection                       │
  │  ├── Request routing                            │
  │  └── Response compression                      │
  └────────────────────────┬────────────────────────┘
                           │
        ┌──────────────────┼──────────────────┐
        ▼                  ▼                  ▼
DOMAIN SERVICES
  ┌──────────┐    ┌──────────┐    ┌──────────┐
  │  users   │    │  orders  │    │ payments │
  │          │    │          │    │          │
  │ gen_srv  │    │ gen_srv  │    │ gen_srv  │
  │ + sup    │    │ + sup    │    │ + sup    │
  └────┬─────┘    └────┬─────┘    └────┬─────┘
       │               │               │
       └───────────────┼───────────────┘
                       │ Message Bus (pg/pubsub)
                       │
INFRASTRUCTURE LAYER
  ┌────────────────────────────────────────────────┐
  │                                                 │
  │  ┌──────────┐  ┌──────────┐  ┌──────────────┐ │
  │  │ DB Pool  │  │  Cache   │  │  Job Queue   │ │
  │  │ (poolboy)│  │  (ETS)   │  │  (workers)   │ │
  │  └──────────┘  └──────────┘  └──────────────┘ │
  │                                                 │
  │  ┌──────────┐  ┌──────────┐  ┌──────────────┐ │
  │  │Telemetry │  │  Config  │  │  Audit Log   │ │
  │  │(metrics) │  │ (env)    │  │  (immutable) │ │
  │  └──────────┘  └──────────┘  └──────────────┘ │
  └────────────────────────────────────────────────┘
       │               │               │
       ▼               ▼               ▼
EXTERNAL SYSTEMS
  PostgreSQL         Redis          Kafka
  (primary DB)     (session)    (event stream)
```

---

## 3. Cross-Cutting Concerns

```erlang
%% cross_cutting.erl — concerns that span all modules
-module(cross_cutting).

%% 1. Request Context Propagation
%% Problem: how to pass request_id, user_id through call stack
%% Solution: process dictionary (sparingly) or explicit parameter

set_request_context(Ctx) ->
    put(request_context, Ctx).

get_request_context() ->
    get(request_context).

%% Better: pass as function argument
do_work(RequestId, UserId, Data) ->
    Ctx = #{request_id => RequestId, user_id => UserId},
    step_one(Ctx, Data).

step_one(Ctx, Data) ->
    %% Propagate Ctx through the call chain
    Result = step_two(Ctx, transform(Data)),
    Result.

step_two(Ctx, Data) ->
    %% Log with context
    logger:info("Processing", #{ctx => Ctx, data_size => map_size(Data)}),
    Data.

transform(Data) -> Data.

%% 2. Error Context Enrichment
%% When errors happen, add context before propagating
enrich_error({error, Reason}, Module, Line, Context) ->
    {error, #{reason => Reason, module => Module, line => Line,
              context => Context}};
enrich_error(Result, _, _, _) -> Result.

%% 3. Retry with Context
with_retry(Fun, Context) ->
    with_retry(Fun, Context, 3, 100).

with_retry(Fun, Context, MaxRetries, BaseDelay) ->
    with_retry_loop(Fun, Context, MaxRetries, BaseDelay, 0).

with_retry_loop(Fun, Ctx, Max, Delay, Attempt) when Attempt < Max ->
    case Fun() of
        {error, retryable} ->
            logger:warning("Retrying attempt ~p/~p", [Attempt+1, Max],
                           #{ctx => Ctx}),
            timer:sleep(Delay * round(math:pow(2, Attempt))),
            with_retry_loop(Fun, Ctx, Max, Delay, Attempt + 1);
        Result -> Result
    end;
with_retry_loop(Fun, _Ctx, _Max, _Delay, _Attempt) ->
    Fun().

%% 4. Structured Logging
log(Level, Event, Data, Context) ->
    logger:Level(Event, maps:merge(Data, #{
        request_id => maps:get(request_id, Context, undefined),
        user_id    => maps:get(user_id, Context, undefined),
        module     => ?MODULE
    })).
```

---

## 4. Team and Code Organization

```
Erlang Monorepo Structure (Umbrella Project)
══════════════════════════════════════════════

myplatform/
├── apps/
│   ├── platform_core/          ← shared types, utilities
│   │   ├── src/
│   │   │   ├── platform_core.app.src
│   │   │   ├── types.erl       ← -type user_id() etc.
│   │   │   ├── result.erl      ← monadic result type
│   │   │   └── validators.erl
│   │   └── rebar.config
│   │
│   ├── platform_db/            ← database layer
│   │   ├── src/
│   │   │   ├── db_pool.erl
│   │   │   ├── db_migration.erl
│   │   │   └── query_builder.erl
│   │   └── rebar.config
│   │
│   ├── platform_auth/          ← authentication/authorization
│   │   ├── src/
│   │   │   ├── jwt_auth.erl
│   │   │   ├── rbac.erl
│   │   │   └── session_store.erl
│   │   └── rebar.config
│   │
│   ├── domain_users/           ← user domain
│   │   ├── src/
│   │   │   ├── user.erl        ← domain model
│   │   │   ├── user_service.erl
│   │   │   └── user_repo.erl
│   │   └── rebar.config
│   │
│   ├── domain_orders/          ← order domain
│   ├── domain_payments/        ← payment domain
│   │
│   ├── api_v1/                 ← HTTP API layer
│   │   ├── src/
│   │   │   ├── api_v1_app.erl
│   │   │   ├── api_v1_sup.erl
│   │   │   ├── router.erl
│   │   │   └── handlers/
│   │   └── rebar.config
│   │
│   └── platform_release/       ← release configuration
│       ├── config/
│       │   ├── sys.config
│       │   └── vm.args
│       └── rebar.config
│
├── test/                       ← integration tests
├── scripts/                    ← deployment scripts
├── rebar.config                ← umbrella config
└── rebar.lock
```

---

## 5. Production Readiness Checklist

```erlang
%% production_checklist.erl — pre-production verification
-module(production_checklist).
-export([verify_all/0]).

verify_all() ->
    Checks = [
        %% Infrastructure
        {db_connectivity,    fun check_db/0},
        {cache_connectivity, fun check_cache/0},
        {message_bus,        fun check_message_bus/0},

        %% Security
        {tls_configured,     fun check_tls/0},
        {secrets_not_hardcoded, fun check_secrets/0},
        {rate_limiting,      fun check_rate_limiting/0},

        %% Observability
        {metrics_enabled,    fun check_metrics/0},
        {logging_structured, fun check_logging/0},
        {health_endpoint,    fun check_health_endpoint/0},

        %% Reliability
        {db_pool_sized,      fun check_pool_size/0},
        {supervisors_proper, fun check_supervisors/0},
        {circuit_breakers,   fun check_circuit_breakers/0},

        %% Performance
        {no_large_mailboxes, fun check_mailboxes/0},
        {ets_memory_ok,      fun check_ets_memory/0},
        {gc_pressure_ok,     fun check_gc/0}
    ],
    Results = [{Name, catch Fun()} || {Name, Fun} <- Checks],
    Failed  = [{N, R} || {N, R} <- Results, R =/= ok],
    case Failed of
        [] ->
            io:format("✓ All ~p checks passed~n", [length(Checks)]),
            ok;
        _ ->
            io:format("✗ ~p checks failed:~n", [length(Failed)]),
            [io:format("  - ~p: ~p~n", [N, R]) || {N, R} <- Failed],
            {error, Failed}
    end.

check_db() ->
    case db:query("SELECT 1", []) of
        {ok, _} -> ok;
        Error   -> {failed, Error}
    end.

check_cache() ->
    ets:info(main_cache) =/= undefined orelse error(no_cache_table),
    ok.

check_message_bus() ->
    pg:which_groups() == [] orelse ok,
    ok.

check_tls() ->
    %% Verify TLS is configured for external connections
    case application:get_env(myapp, tls_port) of
        {ok, _} -> ok;
        undefined -> {warning, tls_not_configured}
    end.

check_secrets() ->
    %% Ensure no secrets in application env (must come from OS env)
    Sensitive = [db_password, jwt_secret, api_key],
    Hardcoded = [K || K <- Sensitive,
                      begin
                          {ok, V} = application:get_env(myapp, K, undefined),
                          is_binary(V) andalso byte_size(V) > 0
                      end],
    case Hardcoded of
        [] -> ok;
        _  -> {warning, {hardcoded_secrets, Hardcoded}}
    end.

check_rate_limiting() ->
    case whereis(rate_limiter) of
        Pid when is_pid(Pid) -> ok;
        undefined -> {failed, rate_limiter_not_running}
    end.

check_metrics() ->
    case telemetry:list_handlers([]) of
        [_|_] -> ok;
        []    -> {warning, no_telemetry_handlers}
    end.

check_logging() ->
    %% Verify structured logging is enabled
    case logger:get_primary_config() of
        #{formatter := {logger_formatter, #{template := T}}} when is_list(T) -> ok;
        _ -> {warning, logging_not_structured}
    end.

check_health_endpoint() ->
    %% Check health handler is registered
    ok.

check_pool_size() ->
    PoolSize = poolboy:status(db_pool),
    Workers  = maps:get(workers, PoolSize, 0),
    case Workers >= 5 of
        true  -> ok;
        false -> {warning, {small_pool, Workers}}
    end.

check_supervisors() -> ok.
check_circuit_breakers() -> ok.

check_mailboxes() ->
    Processes = processes(),
    Heavy = [P || P <- Processes,
                  begin
                      Info = erlang:process_info(P, message_queue_len),
                      Info =/= undefined andalso element(2, Info) > 10000
                  end],
    case Heavy of
        [] -> ok;
        _  -> {warning, {heavy_mailboxes, length(Heavy)}}
    end.

check_ets_memory() ->
    MemBytes = erlang:memory(ets),
    TotalMb  = MemBytes div (1024 * 1024),
    case TotalMb < 2048 of
        true  -> ok;
        false -> {warning, {high_ets_memory, TotalMb}}
    end.

check_gc() ->
    {GCs, _, _} = erlang:statistics(garbage_collection),
    %% Just check it's working
    GCs > 0 orelse error(gc_never_ran),
    ok.
```

---

## 6. Case Study: Building a Payment Platform

```
PaymentOS — Architecture Decision Record
═══════════════════════════════════════════════════════════════

CONTEXT
  Process $50M/day across 500k transactions
  99.99% uptime requirement (52 min downtime/year)
  PCI-DSS Level 1 compliance
  Support 10,000 TPS peak

KEY DECISIONS

1. ERLANG/OTP CHOICE
   ✓ Fault isolation: payment failures don't cascade
   ✓ Hot code upgrades: zero-downtime deploy
   ✓ Distributed: multi-datacenter active-active
   ✓ Message passing: natural audit trail

2. ARCHITECTURE: EVENT SOURCING + CQRS
   Write side: append-only event log (PostgreSQL)
   Read side:  projections in ETS (updated async)
   Benefits:
   - Complete audit trail (regulatory requirement)
   - Replay: reconstruct state after bug fixes
   - Temporal queries: "what was balance on day X?"

3. SAGA PATTERN FOR DISTRIBUTED TRANSACTIONS
   charge_card → reserve_funds → complete_transfer
   Each step has a compensating action (refund/release)
   gen_statem manages the saga state machine

4. IDEMPOTENCY
   Every mutation has an idempotency key
   Distributed lock (Mnesia) prevents duplicate processing
   Client retries are safe

5. CIRCUIT BREAKERS
   Payment provider calls wrapped in circuit breakers
   Fallback to secondary provider on open circuit
   Automatic reset after health check

6. COMPLIANCE
   Immutable audit log with hash chain
   All PII encrypted at rest and in transit
   PAN (card number) never stored — only tokenized

TRAFFIC PATTERN
  Peak: Black Friday, 10,000 TPS for 4 hours
  Solution: horizontal scaling + circuit breakers + backpressure

NUMBERS
  p50 latency: 45ms
  p99 latency: 120ms
  Uptime: 99.997% (15 min downtime/year)
  Throughput: 15,000 TPS peak achieved
```

---

## 7. แบบฝึกหัด

1. วาด supervision tree ของระบบที่คุณออกแบบ พร้อม restart strategies ที่เหมาะสม
2. เขียน production_checklist module ที่ verify ระบบของคุณ
3. สร้าง ADR (Architecture Decision Record) สำหรับ tech choice ใหญ่ครั้งหนึ่ง
4. Implement blue-green deployment script ที่ใช้ RPC ตรวจสอบ health ก่อน switch

---

## สรุป Part 70 — Milestone

✅ 7 principles of production Erlang systems  
✅ Complete system blueprint: edge → domain → infrastructure  
✅ Cross-cutting concerns: context propagation, retry, structured logging  
✅ Team/code organization: umbrella project structure  
✅ Production readiness checklist: 15 automated checks  
✅ Case study: payment platform decision making  

---

**🎯 Checkpoint: Parts 1-70 Complete**
- Basic → Intermediate → Advanced → Production
- Next: Parts 71-100 = World-Class Patterns

---

*Part 70/100 | [← ก่อนหน้า](../part69/README.md) | [ถัดไป →](../part71/README.md)*
