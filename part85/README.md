# Part 85: Database Architecture at Scale

> **"Your database is the most honest part of your system — it never lies about what happened"**  
> ฐานข้อมูลคือส่วนที่ซื่อสัตย์ที่สุดในระบบ — มันไม่เคยโกหกว่าเกิดอะไรขึ้น

---

## สารบัญ

1. [Database Architecture Patterns](#1-database-architecture-patterns)
2. [Connection Pool Management](#2-connection-pool-management)
3. [Read Replicas](#3-read-replicas)
4. [Sharding Strategy](#4-sharding-strategy)
5. [Database Migration Framework](#5-database-migration-framework)
6. [Query Optimization](#6-query-optimization)
7. [แบบฝึกหัด](#7-แบบฝึกหัด)

---

## 1. Database Architecture Patterns

```
Database Scaling Strategies
════════════════════════════════════════════════════════

VERTICAL SCALING (Scale Up)
  Single powerful server
  Pros: Simple, ACID, no complexity
  Limit: ~32 cores, ~2TB RAM, ~100k QPS

HORIZONTAL READ SCALING
  One primary (writes), multiple replicas (reads)
  Pros: 5-10x read capacity, no sharding complexity
  Limit: Write throughput still single node

SHARDING (Horizontal Write Scaling)
  Partition data by key (user_id, tenant_id, etc.)
  Pros: Near-linear write scaling
  Cons: Complex, cross-shard queries impossible

CQRS with Separate Stores
  Write: PostgreSQL (ACID, relational)
  Read:  Elasticsearch/ClickHouse (fast analytics)
  Pros: Each store optimized for its workload
  Cons: Eventual consistency, data sync complexity

RECOMMENDED PROGRESSION:
  Phase 1: Single PostgreSQL with good indexes (0-100k users)
  Phase 2: Add read replica (100k-1M users)
  Phase 3: Add connection pooling (PgBouncer) and caching
  Phase 4: Shard by tenant_id if multi-tenant (1M+ users)
  Phase 5: CQRS with analytics store for reporting
```

---

## 2. Connection Pool Management

```erlang
%% db_pool.erl — multi-pool database layer
-module(db_pool).
-export([query/2, query/3, execute/2, transaction/1]).
-export([start_link/0]).

%% Pool configuration
-define(POOLS, #{
    primary => #{
        host => "db-primary.internal",
        pool_size => 20,
        mode => read_write
    },
    replica1 => #{
        host => "db-replica1.internal",
        pool_size => 30,
        mode => read_only
    },
    replica2 => #{
        host => "db-replica2.internal",
        pool_size => 30,
        mode => read_only
    }
}).

start_link() ->
    %% Start pools for each DB endpoint
    maps:foreach(fun(Name, Config) ->
        start_pool(Name, Config)
    end, ?POOLS),
    ok.

%% Route reads to replicas, writes to primary
query(Sql, Params) ->
    Pool = choose_pool(read, Sql),
    with_connection(Pool, fun(Conn) ->
        epgsql:equery(Conn, Sql, Params)
    end).

query(Sql, Params, #{pool := PoolName}) ->
    with_connection(PoolName, fun(Conn) ->
        epgsql:equery(Conn, Sql, Params)
    end).

execute(Sql, Params) ->
    with_connection(primary, fun(Conn) ->
        epgsql:equery(Conn, Sql, Params)
    end).

transaction(Fun) ->
    with_connection(primary, fun(Conn) ->
        epgsql:with_transaction(Conn, Fun)
    end).

choose_pool(read, Sql) ->
    %% Use primary for consistency-sensitive reads
    case needs_primary(Sql) of
        true  -> primary;
        false -> choose_replica()
    end.

needs_primary(Sql) ->
    %% Force primary for: SELECT FOR UPDATE, from a transaction, etc.
    SqlUpper = string:uppercase(binary_to_list(Sql)),
    lists:any(fun(Keyword) ->
        string:find(SqlUpper, Keyword) =/= nomatch
    end, ["FOR UPDATE", "FOR SHARE"]).

choose_replica() ->
    %% Round-robin across replicas
    Replicas = [replica1, replica2],
    Idx = atomics:add_get(replica_counter, 1, 1) rem length(Replicas) + 1,
    lists:nth(Idx, Replicas).

with_connection(Pool, Fun) ->
    poolboy:transaction(Pool, Fun).

start_pool(Name, Config) ->
    PoolArgs = [
        {name, {local, Name}},
        {worker_module, db_worker},
        {size, maps:get(pool_size, Config, 10)},
        {max_overflow, 5}
    ],
    WorkerArgs = [maps:get(host, Config)],
    {ok, _} = poolboy:start_link(PoolArgs, WorkerArgs),
    ok.
```

---

## 3. Read Replicas

```erlang
%% replica_router.erl — intelligent read routing with lag awareness
-module(replica_router).
-behaviour(gen_server).

-export([start_link/0, get_pool/1, report_lag/2]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

-define(LAG_CHECK_INTERVAL, 5000).
-define(MAX_ACCEPTABLE_LAG_MB, 100).

-record(replica, {
    name :: atom(),
    lag_bytes = 0 :: integer(),
    healthy = true :: boolean(),
    last_check :: integer()
}).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

%% Get best pool for a query type
%% Types: read_fresh (needs very recent data), read_stale (can use replica)
get_pool(read_fresh) -> {ok, primary};
get_pool(read_stale) ->
    gen_server:call(?MODULE, get_best_replica).

report_lag(ReplicaName, LagBytes) ->
    gen_server:cast(?MODULE, {lag, ReplicaName, LagBytes}).

init([]) ->
    Replicas = #{
        replica1 => #replica{name = replica1},
        replica2 => #replica{name = replica2}
    },
    schedule_lag_check(),
    {ok, Replicas}.

handle_call(get_best_replica, _From, Replicas) ->
    %% Find healthy replica with least lag
    Healthy = [{Lag, Name}
               || {Name, #replica{lag_bytes = Lag, healthy = H}} <- maps:to_list(Replicas),
                  H =:= true,
                  Lag < ?MAX_ACCEPTABLE_LAG_MB * 1024 * 1024],
    case lists:sort(Healthy) of
        [{_, Best} | _] -> {reply, {ok, Best}, Replicas};
        [] ->
            %% No healthy replicas, fall back to primary
            {reply, {ok, primary}, Replicas}
    end.

handle_cast({lag, Name, LagBytes}, Replicas) ->
    Replica = maps:get(Name, Replicas, #replica{name = Name}),
    Updated = Replica#replica{
        lag_bytes = LagBytes,
        healthy   = LagBytes < ?MAX_ACCEPTABLE_LAG_MB * 1024 * 1024,
        last_check = erlang:system_time(second)
    },
    {noreply, maps:put(Name, Updated, Replicas)}.

handle_info(check_lag, Replicas) ->
    %% Check replication lag for each replica
    maps:foreach(fun(Name, _) ->
        spawn(fun() ->
            Lag = check_replica_lag(Name),
            report_lag(Name, Lag)
        end)
    end, Replicas),
    schedule_lag_check(),
    {noreply, Replicas}.

check_replica_lag(ReplicaName) ->
    %% Query pg_stat_replication on primary
    case db_pool:query(
        "SELECT pg_wal_lsn_diff(sent_lsn, replay_lsn) as lag_bytes
         FROM pg_stat_replication
         WHERE application_name = $1",
        [atom_to_binary(ReplicaName)],
        #{pool => primary}) of
        {ok, [{LagBytes}]} when is_integer(LagBytes) -> LagBytes;
        _ -> 0
    end.

schedule_lag_check() ->
    erlang:send_after(?LAG_CHECK_INTERVAL, self(), check_lag).
```

---

## 4. Sharding Strategy

```erlang
%% shard_router.erl — route queries to correct database shard
-module(shard_router).
-export([get_shard/1, shard_query/3, scatter_gather/2]).

-define(NUM_SHARDS, 8).

%% Consistent hash shard assignment
get_shard(ShardKey) ->
    Hash  = erlang:phash2(ShardKey, ?NUM_SHARDS),
    ShardId = Hash + 1,
    shard_name(ShardId).

shard_name(N) ->
    list_to_atom("shard_" ++ integer_to_list(N)).

%% Route single query to correct shard
shard_query(ShardKey, Sql, Params) ->
    Shard = get_shard(ShardKey),
    db_pool:query(Sql, Params, #{pool => Shard}).

%% Fan out query to all shards and merge results (scatter-gather)
scatter_gather(Sql, Params) ->
    Shards  = [shard_name(N) || N <- lists:seq(1, ?NUM_SHARDS)],
    Parent  = self(),
    Workers = [spawn_link(fun() ->
        Result = db_pool:query(Sql, Params, #{pool => Shard}),
        Parent ! {shard_result, Shard, Result}
    end) || Shard <- Shards],
    collect_shard_results(length(Workers), []).

collect_shard_results(0, Acc) ->
    {ok, lists:flatten(Acc)};
collect_shard_results(N, Acc) ->
    receive
        {shard_result, _Shard, {ok, Rows}} ->
            collect_shard_results(N - 1, [Rows | Acc]);
        {shard_result, Shard, {error, Reason}} ->
            logger:warning("Shard ~p failed: ~p", [Shard, Reason]),
            collect_shard_results(N - 1, Acc)
    after 5000 ->
        {error, scatter_gather_timeout}
    end.

%% Example: user lookup always goes to correct shard
get_user(UserId) ->
    shard_query(UserId,
        "SELECT * FROM users WHERE id = $1", [UserId]).

%% Example: admin report gathers from all shards
total_revenue() ->
    case scatter_gather(
        "SELECT SUM(amount) FROM orders WHERE created_at > NOW() - INTERVAL '1 day'",
        []) of
        {ok, Rows} ->
            Total = lists:sum([R || {R} <- Rows, is_number(R)]),
            {ok, Total};
        Err -> Err
    end.
```

---

## 5. Database Migration Framework

```erlang
%% migrations.erl — ordered database migrations
-module(migrations).
-export([run_pending/0, list_migrations/0, rollback/1]).

%% Migration module convention:
%%   Name: migration_YYYYMMDD_HHMMSS_description
%%   Exports: up/0, down/0

run_pending() ->
    {ok, Applied} = get_applied(),
    Available     = discover_migrations(),
    Pending = [M || M <- Available, not lists:member(M, Applied)],
    case Pending of
        [] ->
            io:format("No pending migrations~n"),
            ok;
        _ ->
            io:format("Running ~p migrations...~n", [length(Pending)]),
            lists:foreach(fun(Migration) ->
                run_migration(up, Migration)
            end, lists:sort(Pending))
    end.

rollback(N) ->
    {ok, Applied} = get_applied(),
    ToRollback = lists:sublist(lists:reverse(lists:sort(Applied)), N),
    lists:foreach(fun(Migration) ->
        run_migration(down, Migration)
    end, ToRollback).

run_migration(Direction, Name) ->
    Module = list_to_atom(Name),
    io:format("  [~s] ~s...", [Direction, Name]),
    try
        ok = db_pool:transaction(fun(Conn) ->
            case Direction of
                up   -> Module:up(Conn);
                down -> Module:down(Conn)
            end
        end),
        case Direction of
            up   -> record_migration(Name);
            down -> remove_migration(Name)
        end,
        io:format(" OK~n")
    catch
        E:R ->
            io:format(" FAILED: ~p:~p~n", [E, R]),
            error({migration_failed, Name, Direction, {E, R}})
    end.

discover_migrations() ->
    %% Find all migration_* modules
    [atom_to_list(M) || M <- erlang:loaded(),
                         is_migration_module(M)].

is_migration_module(M) ->
    ModStr = atom_to_list(M),
    lists:prefix("migration_", ModStr) andalso
    erlang:function_exported(M, up, 0).

get_applied() ->
    case db_pool:query(
        "SELECT name FROM schema_migrations ORDER BY name ASC", []) of
        {ok, Rows} -> {ok, [binary_to_list(N) || {N} <- Rows]};
        _ -> {ok, []}
    end.

record_migration(Name) ->
    db_pool:execute(
        "INSERT INTO schema_migrations (name, applied_at) VALUES ($1, NOW())",
        [Name]).

remove_migration(Name) ->
    db_pool:execute(
        "DELETE FROM schema_migrations WHERE name = $1", [Name]).

list_migrations() ->
    Available = lists:sort(discover_migrations()),
    {ok, Applied} = get_applied(),
    [{M, lists:member(M, Applied)} || M <- Available].
```

---

## 6. Query Optimization

```erlang
%% query_optimizer.erl — tools for identifying and fixing slow queries
-module(query_optimizer).
-export([explain/2, detect_n_plus_1/2, batch_load/3]).

%% Get EXPLAIN ANALYZE output for a query
explain(Sql, Params) ->
    ExplainSql = <<"EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) ", Sql/binary>>,
    case db_pool:query(ExplainSql, Params) of
        {ok, [{PlanJson}]} ->
            Plan = json:decode(PlanJson),
            print_plan(Plan),
            {ok, Plan};
        Err -> Err
    end.

print_plan(Plan) ->
    Node     = hd(Plan),
    PlanNode = maps:get(<<"Plan">>, Node),
    TotalMs  = maps:get(<<"Actual Total Time">>, PlanNode, 0),
    io:format("Total time: ~.2fms~n", [TotalMs]),
    print_node(PlanNode, 0).

print_node(Node, Depth) ->
    Indent   = lists:duplicate(Depth * 2, $\s),
    NodeType = maps:get(<<"Node Type">>, Node, <<"Unknown">>),
    RowsEst  = maps:get(<<"Plan Rows">>, Node, 0),
    RowsAct  = maps:get(<<"Actual Rows">>, Node, 0),
    TimeMs   = maps:get(<<"Actual Total Time">>, Node, 0),
    io:format("~s~s (est: ~p rows, actual: ~p rows, time: ~.2fms)~n",
              [Indent, NodeType, RowsEst, RowsAct, TimeMs]),
    case maps:get(<<"Plans">>, Node, undefined) of
        undefined -> ok;
        SubPlans  -> lists:foreach(fun(S) -> print_node(S, Depth + 1) end, SubPlans)
    end.

%% Detect N+1 queries: load collection, then load each item separately
%% Fix: use a single JOIN or batch load
batch_load(Ids, Sql, KeyField) ->
    %% Load all items in ONE query instead of N queries
    case db_pool:query(Sql, [Ids]) of
        {ok, Rows} ->
            %% Index by key for O(1) lookup
            Index = maps:from_list([
                {maps:get(KeyField, format_row(Row)), format_row(Row)}
                || Row <- Rows
            ]),
            %% Return in same order as input
            [{Id, maps:get(Id, Index, undefined)} || Id <- Ids];
        {error, _} = Err -> Err
    end.

%% Example: instead of N user lookups, batch them
load_users_for_posts(Posts) ->
    UserIds = lists:usort([maps:get(author_id, P) || P <- Posts]),
    UserRows = batch_load(UserIds,
        "SELECT id, username, avatar_url FROM users WHERE id = ANY($1)",
        id),
    UserMap = maps:from_list(UserRows),
    [P#{author => maps:get(maps:get(author_id, P), UserMap, undefined)}
     || P <- Posts].

format_row(_Row) -> #{}.  % simplified

detect_n_plus_1(_BaseQuery, _ItemQuery) ->
    %% In real implementation: wrap queries with a counter,
    %% alert if same query pattern repeated > threshold times
    ok.
```

---

## 7. แบบฝึกหัด

1. Implement advisory locks สำหรับ distributed cron jobs (only one node runs)
2. สร้าง migration ที่ rename column โดยใช้ expand-contract pattern จาก Part 73
3. เพิ่ม automatic EXPLAIN logging สำหรับ queries ที่ใช้เวลา > 1 วินาที
4. Implement tenant-aware connection pool ที่ route ตาม tenant database

---

## สรุป Part 85

✅ Database scaling: vertical → read replicas → sharding progression  
✅ Connection pool: multi-pool with primary/replica routing  
✅ Read replica routing: lag-aware, automatic fallback to primary  
✅ Sharding: consistent hash routing, scatter-gather for all-shard queries  
✅ Migration framework: ordered, transactional, with rollback support  
✅ Query optimization: EXPLAIN ANALYZE, batch loading to prevent N+1  

---

*Part 85/100 | [← ก่อนหน้า](../part84/README.md) | [ถัดไป →](../part86/README.md)*
