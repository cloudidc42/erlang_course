# Part 99: Capstone Project — Design a Complete Platform

> **"Now you have the tools — build something that matters"**  
> ตอนนี้คุณมีเครื่องมือแล้ว — สร้างสิ่งที่มีความหมาย

---

## สารบัญ

1. [Capstone: Collaborative Task Platform](#1-capstone-collaborative-task-platform)
2. [System Requirements](#2-system-requirements)
3. [Architecture Design](#3-architecture-design)
4. [Data Model](#4-data-model)
5. [Core Implementation](#5-core-implementation)
6. [Production Readiness Checklist](#6-production-readiness-checklist)
7. [แบบฝึกหัด: Complete the Platform](#7-แบบฝึกหัดcomplete-the-platform)

---

## 1. Capstone: Collaborative Task Platform

```
PROJECT REQUIREMENTS: "Tasky" — Real-time task collaboration tool

Target: 10,000 concurrent users, 100 organizations
Features:
  - Organizations with multiple workspaces
  - Real-time collaborative task editing (like Jira + Figma)
  - Task assignment, status, due dates, comments
  - Activity feed (who did what, when)
  - File attachments (up to 10MB per file)
  - Role-based access: admin, member, viewer
  - REST API + WebSocket for real-time
  - Email notifications (digest or immediate)
  - Full-text search across tasks

Non-functional requirements:
  - API P99 < 100ms
  - WebSocket message delivery < 50ms
  - 99.9% uptime (8.7 hours/year downtime budget)
  - Data encrypted at rest and in transit
  - GDPR compliant (data deletion on request)

Tech stack chosen:
  - Erlang/OTP 27 + Cowboy
  - PostgreSQL 16 (primary + read replica)
  - Redis (sessions, pub/sub)
  - S3-compatible storage (files)
  - Prometheus + Grafana (metrics)
```

---

## 2. System Requirements

```
Capacity Planning
═════════════════════════════════════════════════════

LOAD ESTIMATION:
  10,000 concurrent users
  Each user: 1 API request every 30s + 1 WebSocket connection
  Peak API load: 10,000 / 30 = ~333 RPS
  WebSocket connections: 10,000 simultaneous
  Database queries: ~1,000 QPS (reads) + ~100 QPS (writes)

STORAGE ESTIMATION:
  Tasks: 10,000 org × 1,000 tasks/org = 10M tasks
  Comments: 5 per task = 50M comments
  Activity events: 20 per task = 200M events
  Storage: (10M + 50M + 200M) × 500 bytes ≈ 130GB
  Files: 10,000 orgs × 100 files × 1MB avg = 1TB

SCALING TARGETS:
  Single node: handles ~1,000 concurrent WS + 500 RPS (Cowboy default)
  Target: 3 API nodes behind load balancer for redundancy + capacity
  Database: primary (writes) + replica (reads)

BEAM PROCESS BUDGET:
  Per-org supervisor: 10,000 processes
  Per-user WS handler: 10,000 processes
  Background workers: ~100 processes
  Total: ~20,100 processes (BEAM handles millions easily)
```

---

## 3. Architecture Design

```
Tasky System Architecture
════════════════════════════════════════════════════

Browser / Mobile App
        │ HTTPS / WSS
        ▼
   Load Balancer (HAProxy / nginx)
        │
        ├─── api-1 ─┐
        ├─── api-2 ─┤  Erlang/OTP + Cowboy nodes
        └─── api-3 ─┘
                │
        ┌───────┼────────────────────────────┐
        │       │                            │
        ▼       ▼                            ▼
   PostgreSQL  Redis                    S3 Storage
   (primary)   (sessions,               (file uploads)
               pub/sub,
   PostgreSQL  rate limits)
   (read
   replica)

Supervision tree per node:
  tasky_sup (one_for_one)
    ├── db_pool_sup         (manages PG connections)
    ├── redis_client        (gen_server)
    ├── cowboy_http         (HTTP/WS server)
    ├── org_registry        (gen_server: org_id → pid)
    ├── org_sup             (simple_one_for_one: per-org state)
    ├── notification_sup    (email + push workers)
    └── background_sup      (search indexer, event aggregator)

Real-time flow:
  Client edits task
    → WebSocket handler → task_service:update/3
      → DB write (primary)
      → Publish to Redis channel "org:{org_id}"
        → All nodes subscribed to channel
          → Each node broadcasts to relevant WS connections
```

---

## 4. Data Model

```sql
-- Core tables (PostgreSQL)

CREATE TABLE organizations (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        VARCHAR(200) NOT NULL,
    slug        VARCHAR(50) UNIQUE NOT NULL,
    plan        VARCHAR(20) DEFAULT 'free',  -- free | pro | enterprise
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE workspaces (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id      UUID NOT NULL REFERENCES organizations(id),
    name        VARCHAR(200) NOT NULL,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE tasks (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id),
    title        TEXT NOT NULL,
    description  TEXT,
    status       VARCHAR(20) DEFAULT 'todo',  -- todo|in_progress|done|cancelled
    priority     SMALLINT DEFAULT 2,           -- 1=low 2=medium 3=high 4=critical
    assignee_id  UUID REFERENCES users(id),
    due_date     DATE,
    created_by   UUID NOT NULL REFERENCES users(id),
    created_at   TIMESTAMPTZ DEFAULT NOW(),
    updated_at   TIMESTAMPTZ DEFAULT NOW(),
    deleted_at   TIMESTAMPTZ  -- soft delete
);
CREATE INDEX tasks_workspace_status ON tasks(workspace_id, status) WHERE deleted_at IS NULL;
CREATE INDEX tasks_assignee ON tasks(assignee_id) WHERE deleted_at IS NULL;

CREATE TABLE activity_events (
    id          BIGSERIAL PRIMARY KEY,
    org_id      UUID NOT NULL,
    entity_type VARCHAR(50) NOT NULL,  -- 'task' | 'comment' | 'workspace'
    entity_id   UUID NOT NULL,
    event_type  VARCHAR(50) NOT NULL,  -- 'created' | 'updated' | 'assigned'
    actor_id    UUID NOT NULL REFERENCES users(id),
    changes     JSONB,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX activity_org_time ON activity_events(org_id, created_at DESC);
```

---

## 5. Core Implementation

```erlang
%% task_service.erl — core task domain logic
-module(task_service).
-export([create/3, update/3, assign/3, change_status/3,
         list_for_workspace/2, get/2]).

create(WorkspaceId, CreatorId, Attrs) ->
    #{title := Title} = Attrs,
    case validate_task_attrs(Attrs) of
        ok ->
            db_pool:transaction(fun(_Conn) ->
                {ok, [{TaskId}]} = db_pool:execute(
                    "INSERT INTO tasks
                       (workspace_id, title, description, priority, created_by)
                     VALUES ($1, $2, $3, $4, $5)
                     RETURNING id",
                    [WorkspaceId, Title,
                     maps:get(description, Attrs, null),
                     maps:get(priority, Attrs, 2),
                     CreatorId]),
                OrgId = workspace_service:get_org_id(WorkspaceId),
                record_activity(OrgId, task, TaskId, created, CreatorId, #{}),
                broadcast_event(OrgId, #{
                    type        => task_created,
                    task_id     => TaskId,
                    workspace_id => WorkspaceId,
                    actor_id    => CreatorId
                }),
                {ok, TaskId}
            end);
        {error, _} = Err -> Err
    end.

update(TaskId, ActorId, Changes) ->
    case get(TaskId, ActorId) of
        {ok, Task} ->
            OrgId = maps:get(org_id, Task),
            UpdateSql = build_update_sql(Changes),
            db_pool:execute(UpdateSql, [TaskId]),
            record_activity(OrgId, task, TaskId, updated, ActorId, Changes),
            broadcast_event(OrgId, #{
                type    => task_updated,
                task_id => TaskId,
                changes => Changes,
                actor_id => ActorId
            }),
            ok;
        Err -> Err
    end.

assign(TaskId, ActorId, AssigneeId) ->
    update(TaskId, ActorId, #{assignee_id => AssigneeId}).

change_status(TaskId, ActorId, NewStatus)
  when NewStatus =:= todo;
       NewStatus =:= in_progress;
       NewStatus =:= done;
       NewStatus =:= cancelled ->
    update(TaskId, ActorId, #{status => atom_to_binary(NewStatus)}).

list_for_workspace(WorkspaceId, Opts) ->
    Status   = maps:get(status, Opts, undefined),
    Limit    = min(maps:get(limit, Opts, 50), 100),
    Cursor   = maps:get(cursor, Opts, undefined),
    {WhereSql, Params} = build_where(WorkspaceId, Status, Cursor),
    db_pool:query(
        "SELECT id, title, status, priority, assignee_id, due_date, created_at
         FROM tasks
         WHERE " ++ WhereSql ++ "
         ORDER BY created_at DESC, id DESC
         LIMIT $" ++ integer_to_list(length(Params) + 1),
        Params ++ [Limit]).

get(TaskId, ActorId) ->
    case db_pool:query(
        "SELECT t.*, w.org_id
         FROM tasks t
         JOIN workspaces w ON w.id = t.workspace_id
         WHERE t.id = $1 AND t.deleted_at IS NULL", [TaskId]) of
        {ok, [Row]} ->
            Task = row_to_map(Row),
            case rbac:can(ActorId, read, Task) of
                true  -> {ok, Task};
                false -> {error, forbidden}
            end;
        {ok, []} -> {error, not_found}
    end.

record_activity(OrgId, EntityType, EntityId, EventType, ActorId, Changes) ->
    db_pool:execute(
        "INSERT INTO activity_events
           (org_id, entity_type, entity_id, event_type, actor_id, changes)
         VALUES ($1, $2, $3, $4, $5, $6)",
        [OrgId, atom_to_binary(EntityType), EntityId,
         atom_to_binary(EventType), ActorId, json:encode(Changes)]).

broadcast_event(OrgId, Event) ->
    Channel = <<"org:", OrgId/binary>>,
    redis_client:publish(Channel, json:encode(Event)).

validate_task_attrs(#{title := Title}) when byte_size(Title) > 0, byte_size(Title) < 500 ->
    ok;
validate_task_attrs(_) ->
    {error, {validation, <<"title must be 1-500 characters">>}}.

build_where(WorkspaceId, undefined, undefined) ->
    {"workspace_id = $1 AND deleted_at IS NULL", [WorkspaceId]};
build_where(WorkspaceId, Status, undefined) when Status =/= undefined ->
    {"workspace_id = $1 AND status = $2 AND deleted_at IS NULL",
     [WorkspaceId, Status]};
build_where(WorkspaceId, _, _) ->
    {"workspace_id = $1 AND deleted_at IS NULL", [WorkspaceId]}.

build_update_sql(_Changes) -> "UPDATE tasks SET updated_at = NOW() WHERE id = $1".
row_to_map(_Row) -> #{}.
workspace_service() -> erlang:module_info().
workspace_service:get_org_id(_) -> <<>>.
```

---

## 6. Production Readiness Checklist

```
Tasky Production Readiness
═════════════════════════════════════════════════════

SECURITY:
  □ All endpoints require authentication
  □ RBAC enforced at service layer (not just HTTP layer)
  □ SQL uses parameterized queries (no string interpolation)
  □ File uploads: type validation, size limit, virus scan
  □ Rate limiting: per-IP and per-user
  □ HTTPS only, HSTS header set
  □ Secrets in environment variables, not source code

RELIABILITY:
  □ All external calls have timeouts
  □ Circuit breakers on Redis, S3, email provider
  □ Supervisor restart strategies match each process type
  □ Graceful shutdown: drain connections before stopping
  □ Health endpoint: /health (liveness) and /ready (readiness)
  □ Database connection pool with overflow protection

OBSERVABILITY:
  □ Structured logs with request_id, org_id, user_id
  □ Prometheus metrics: latency, errors, queue depths
  □ Distributed tracing for request flow
  □ Alerting: P99 latency, error rate, process queue length
  □ Dashboard: request rate, DB connections, WS connections

DEPLOYMENT:
  □ Blue-green deployment for zero downtime
  □ Database migrations are backwards compatible
  □ Feature flags for risky changes
  □ Rollback plan documented and tested
  □ Load tested to 2x expected peak before launch
```

---

## 7. แบบฝึกหัด: Complete the Platform

สิ่งที่ต้องสร้างให้ครบ:

1. **WebSocket handler** — real-time task updates ด้วย pattern จาก Part 91
2. **Search service** — full-text search บน tasks ด้วย PostgreSQL `tsvector`
3. **Notification system** — email digest ด้วย pattern จาก Part 87 (outbox)
4. **File upload** — S3 pre-signed URLs + metadata stored in DB
5. **Activity feed** — cursor-paginated API ด้วย pattern จาก Part 83
6. **Load test** — รัน load_test.erl จาก Part 84 ยืนยัน P99 < 100ms

---

## สรุป Part 99

✅ Capstone platform: real requirements, capacity planning, non-functional constraints  
✅ Architecture: 3-node Erlang + PG + Redis + S3, full supervision tree  
✅ Data model: schema with indexes for hot query patterns  
✅ Core service: task CRUD with RBAC, activity recording, real-time broadcast  
✅ Production checklist: security, reliability, observability, deployment  
✅ Implementation path: connects patterns from Parts 69-98  

---

*Part 99/100 | [← ก่อนหน้า](../part98/README.md) | [ถัดไป →](../part100/README.md)*
