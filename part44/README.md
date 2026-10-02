# Part 44: Database Optimization

> **"The slowest query is the one you didn't know was running"**  
> Query ที่ช้าที่สุดคือ Query ที่คุณไม่รู้ว่ากำลังรันอยู่

---

## สารบัญ

1. [Query Analysis](#1-query-analysis)
2. [Connection Pool Tuning](#2-connection-pool-tuning)
3. [Read Replicas](#3-read-replicas)
4. [Database Sharding](#4-database-sharding)
5. [Bulk Operations](#5-bulk-operations)
6. [Query Caching](#6-query-caching)
7. [Schema Migrations](#7-schema-migrations)
8. [แบบฝึกหัด](#8-แบบฝึกหัด)

---

## 1. Query Analysis

```erlang
%% db_profiler.erl — track slow queries
-module(db_profiler).
-export([query/3, get_slow_queries/0, reset/0]).

-define(SLOW_THRESHOLD_MS, 100).
-define(TABLE, slow_queries).

query(Pool, SQL, Params) ->
    T1 = erlang:monotonic_time(millisecond),
    Result = poolboy:transaction(Pool, fun(Worker) ->
        gen_server:call(Worker, {query, SQL, Params})
    end),
    T2 = erlang:monotonic_time(millisecond),
    DurationMs = T2 - T1,

    %% Emit telemetry
    telemetry:execute([db, query, stop],
                      #{duration => DurationMs},
                      #{query => SQL}),

    %% Track slow queries
    if DurationMs > ?SLOW_THRESHOLD_MS ->
        record_slow(SQL, Params, DurationMs);
       true -> ok
    end,
    Result.

record_slow(SQL, Params, DurationMs) ->
    try
        Entry = ets:lookup(?TABLE, SQL),
        Count = case Entry of
            [{SQL, C, _, _}] -> C + 1;
            [] -> 1
        end,
        ets:insert(?TABLE, {SQL, Count, DurationMs, Params})
    catch _:_ -> ok
    end.

get_slow_queries() ->
    All = ets:tab2list(?TABLE),
    Sorted = lists:sort(fun({_, _, D1, _}, {_, _, D2, _}) ->
        D1 > D2
    end, All),
    [#{sql => SQL, count => C, max_ms => D, sample_params => P}
     || {SQL, C, D, P} <- Sorted].

reset() ->
    ets:delete_all_objects(?TABLE).
```

---

## 2. Connection Pool Tuning

```erlang
%% pool_config.erl — tuning poolboy for your workload
-module(pool_config).
-export([specs/0, optimal_size/0]).

%% Calculate optimal pool size based on system
optimal_size() ->
    CoreCount    = erlang:system_info(logical_processors),
    %% Rule of thumb: (cores * 2) + effective spindle count
    %% For I/O bound (DB queries): more threads OK
    BaseSize = CoreCount * 2,
    io:format("Optimal pool size: ~p~n", [BaseSize]),
    BaseSize.

specs() ->
    Size = optimal_size(),
    [
        poolboy:child_spec(
            db_pool,
            [
                {name,       {local, db_pool}},
                {worker_module, db_worker},
                {size,          Size},
                {max_overflow,  Size div 2},
                {strategy,      lifo}   %% or fifo
            ],
            [{host, "localhost"}, {database, "myapp"}]
        )
    ].

%% Monitor pool utilization
pool_stats() ->
    Status  = poolboy:status(db_pool),
    #{
        state        => element(1, Status),
        workers      => element(2, Status),
        overflow     => element(3, Status),
        monitors     => element(4, Status)
    }.
```

---

## 3. Read Replicas

```erlang
%% db_router.erl — route reads to replica, writes to primary
-module(db_router).
-export([query/2, execute/2]).

-define(PRIMARY, db_pool_primary).
-define(REPLICA, db_pool_replica).

query(SQL, Params) ->
    Pool = choose_pool(SQL),
    db:raw_query(Pool, SQL, Params).

execute(SQL, Params) ->
    %% Writes always go to primary
    db:raw_query(?PRIMARY, SQL, Params).

choose_pool(SQL) ->
    Upper = string:uppercase(binary_to_list(SQL)),
    case lists:prefix("SELECT", Upper) of
        true  -> ?REPLICA;   %% reads → replica
        false -> ?PRIMARY    %% writes → primary
    end.

%% Lag-aware routing: avoid stale reads on fresh writes
-define(LAG_TABLE, replica_lag).

record_write() ->
    %% After a write, mark when it happened
    ets:insert(?LAG_TABLE, {last_write, os:system_time(millisecond)}).

query_with_lag_check(SQL, Params) ->
    case {lists:prefix("SELECT", string:uppercase(binary_to_list(SQL))),
          check_lag()} of
        {true, ok}     -> db:raw_query(?REPLICA, SQL, Params);
        {true, lagged} -> db:raw_query(?PRIMARY, SQL, Params);
        {false, _}     -> execute(SQL, Params)
    end.

check_lag() ->
    case ets:lookup(?LAG_TABLE, last_write) of
        [{_, T}] ->
            case os:system_time(millisecond) - T < 500 of
                true  -> lagged;
                false -> ok
            end;
        [] -> ok
    end.
```

---

## 4. Database Sharding

```erlang
%% db_shard.erl — horizontal sharding by key
-module(db_shard).
-export([pool_for/1, query/3, execute/3]).

-define(NUM_SHARDS, 4).
-define(SHARD_POOLS, [shard_0, shard_1, shard_2, shard_3]).

pool_for(ShardKey) ->
    ShardIdx = erlang:phash2(ShardKey, ?NUM_SHARDS),
    lists:nth(ShardIdx + 1, ?SHARD_POOLS).

query(ShardKey, SQL, Params) ->
    Pool = pool_for(ShardKey),
    db:raw_query(Pool, SQL, Params).

execute(ShardKey, SQL, Params) ->
    Pool = pool_for(ShardKey),
    db:raw_query(Pool, SQL, Params).

%% Cross-shard query: scatter-gather
scatter_gather(SQL, Params) ->
    Parent  = self(),
    Workers = [spawn(fun() ->
                 Result = db:raw_query(Pool, SQL, Params),
                 Parent ! {shard_result, Pool, Result}
             end) || Pool <- ?SHARD_POOLS],
    Results = [receive
                   {shard_result, Pool, R} -> {Pool, R}
               after 5000 -> {Pool, {error, timeout}}
               end || Pool <- ?SHARD_POOLS],
    %% Merge results
    Rows = lists:flatten([Rows || {_, {ok, Rows}} <- Results]),
    {ok, Rows}.
```

---

## 5. Bulk Operations

```erlang
%% db_bulk.erl — efficient bulk inserts and updates
-module(db_bulk).
-export([bulk_insert/2, bulk_upsert/2, bulk_update/3]).

%% Bulk INSERT using unnest
bulk_insert(Table, Rows) when length(Rows) > 0 ->
    %% PostgreSQL: INSERT INTO table SELECT * FROM unnest($1::int[], $2::text[])
    Cols   = maps:keys(hd(Rows)),
    Params = extract_params(Rows, Cols),
    SQL    = build_bulk_insert_sql(Table, Cols, length(Rows)),
    db:execute(SQL, Params).

build_bulk_insert_sql(Table, Cols, NumRows) ->
    ColList   = string:join([atom_to_list(C) || C <- Cols], ", "),
    Placeholders = build_placeholders(length(Cols), NumRows),
    iolist_to_binary([
        "INSERT INTO ", atom_to_list(Table),
        " (", ColList, ") VALUES ", Placeholders
    ]).

build_placeholders(NumCols, NumRows) ->
    RowPlaceholders = fun(RowIdx) ->
        Params = [io_lib:format("$~p", [(RowIdx-1)*NumCols + ColIdx])
                  || ColIdx <- lists:seq(1, NumCols)],
        ["(", string:join(Params, ", "), ")"]
    end,
    Rows = [RowPlaceholders(I) || I <- lists:seq(1, NumRows)],
    string:join(Rows, ", ").

extract_params(Rows, Cols) ->
    lists:flatten([
        [maps:get(Col, Row) || Col <- Cols]
        || Row <- Rows
    ]).

%% Bulk UPSERT with ON CONFLICT
bulk_upsert(Table, Rows) ->
    Cols = maps:keys(hd(Rows)),
    SQL  = iolist_to_binary([
        build_bulk_insert_sql(Table, Cols, length(Rows)),
        " ON CONFLICT (id) DO UPDATE SET ",
        build_update_set(Cols)
    ]),
    db:execute(SQL, extract_params(Rows, Cols)).

build_update_set(Cols) ->
    Updates = [io_lib:format("~s = EXCLUDED.~s", [C, C])
               || C <- Cols, C =/= id],
    string:join(Updates, ", ").

%% Chunked bulk operations to avoid OOM
bulk_insert_chunked(Table, Rows, ChunkSize) ->
    Chunks = chunk_list(Rows, ChunkSize),
    Results = [bulk_insert(Table, Chunk) || Chunk <- Chunks],
    TotalInserted = lists:sum([N || {ok, N} <- Results]),
    {ok, TotalInserted}.

chunk_list([], _)       -> [];
chunk_list(List, N) ->
    {Chunk, Rest} = lists:split(min(N, length(List)), List),
    [Chunk | chunk_list(Rest, N)].
```

---

## 6. Query Caching

```erlang
%% query_cache.erl — cache query results with TTL
-module(query_cache).
-export([cached_query/4]).

-define(CACHE_TABLE, query_cache_table).
-define(DEFAULT_TTL, 300).   %% 5 minutes

cached_query(Pool, SQL, Params, Opts) ->
    TTL      = maps:get(ttl, Opts, ?DEFAULT_TTL),
    CacheKey = erlang:phash2({SQL, Params}),
    Now      = os:system_time(second),

    case ets:lookup(?CACHE_TABLE, CacheKey) of
        [{CacheKey, Result, ExpiresAt}] when ExpiresAt > Now ->
            {cached, Result};
        _ ->
            Result = db:raw_query(Pool, SQL, Params),
            ets:insert(?CACHE_TABLE, {CacheKey, Result, Now + TTL}),
            {fresh, Result}
    end.

invalidate_pattern(Pattern) ->
    %% Remove all cache entries matching a table name
    All = ets:tab2list(?CACHE_TABLE),
    [ets:delete(?CACHE_TABLE, K)
     || {K, _, _} <- All,
        cache_matches_pattern(K, Pattern)].

cache_matches_pattern(_Key, _Pattern) ->
    %% Simplified: in practice compare SQL against pattern
    false.
```

---

## 7. Schema Migrations

```erlang
%% migration.erl — versioned schema migrations
-module(migration).
-export([run/0, status/0]).

-define(MIGRATIONS_TABLE, "schema_migrations").

migrations() ->
    [
        {1, "Create users",
         "CREATE TABLE users (id SERIAL PRIMARY KEY, "
         "email TEXT UNIQUE NOT NULL, created_at TIMESTAMPTZ DEFAULT NOW())",
         "DROP TABLE users"},

        {2, "Add user name",
         "ALTER TABLE users ADD COLUMN name TEXT",
         "ALTER TABLE users DROP COLUMN name"},

        {3, "Create indexes",
         "CREATE INDEX CONCURRENTLY idx_users_email ON users(email)",
         "DROP INDEX CONCURRENTLY idx_users_email"}
    ].

run() ->
    ensure_migrations_table(),
    Applied  = get_applied_versions(),
    Pending  = [{V, D, Up, Down} || {V, D, Up, Down} <- migrations(),
                                     not lists:member(V, Applied)],
    lists:foreach(fun({Version, Desc, SQL, _Down}) ->
        io:format("Applying migration ~p: ~s~n", [Version, Desc]),
        case db:execute(SQL, []) of
            {ok, _} ->
                record_migration(Version),
                io:format("  ✓ Done~n");
            {error, Reason} ->
                io:format("  ✗ Failed: ~p~n", [Reason]),
                error({migration_failed, Version, Reason})
        end
    end, Pending).

status() ->
    Applied = get_applied_versions(),
    [#{version => V, description => D,
       applied => lists:member(V, Applied)}
     || {V, D, _, _} <- migrations()].

ensure_migrations_table() ->
    db:execute(
        "CREATE TABLE IF NOT EXISTS schema_migrations "
        "(version INTEGER PRIMARY KEY, applied_at TIMESTAMPTZ DEFAULT NOW())",
        []).

get_applied_versions() ->
    case db:query("SELECT version FROM schema_migrations ORDER BY version", []) of
        {ok, Rows} -> [V || #{<<"version">> := V} <- Rows];
        _          -> []
    end.

record_migration(Version) ->
    db:execute("INSERT INTO schema_migrations (version) VALUES ($1)", [Version]).
```

---

## 8. แบบฝึกหัด

1. วัด query time ก่อน/หลังเพิ่ม index
2. ทดสอบ bulk_insert 10,000 rows และวัด throughput เทียบกับ single inserts
3. เพิ่ม `rollback/1` function ใน migration module
4. สร้าง read replica pool และ benchmark latency difference

---

## สรุป Part 44

✅ Query profiling และ slow query tracking  
✅ Connection pool tuning  
✅ Read replica routing  
✅ Database sharding  
✅ Bulk inserts ด้วย unnest  
✅ Query result caching  
✅ Versioned schema migrations  

---

*Part 44/100 | [← ก่อนหน้า](../part43/README.md) | [ถัดไป →](../part45/README.md)*
