# Part 86: Caching Architecture

> **"A cache is a lie that makes your system fast; design it to lie correctly"**  
> Cache คือการโกหกที่ทำให้ระบบเร็วขึ้น — ออกแบบมันให้โกหกอย่างถูกต้อง

---

## สารบัญ

1. [Cache Topology](#1-cache-topology)
2. [Multi-Layer Cache](#2-multi-layer-cache)
3. [Cache Invalidation Strategies](#3-cache-invalidation-strategies)
4. [Write-Through and Write-Behind](#4-write-through-and-write-behind)
5. [Distributed Cache with ETS](#5-distributed-cache-with-ets)
6. [Cache Stampede Protection](#6-cache-stampede-protection)
7. [แบบฝึกหัด](#7-แบบฝึกหัด)

---

## 1. Cache Topology

```
Cache Layers (nearest to farthest from request)
════════════════════════════════════════════════

L1: Process dictionary / local state  ~microseconds
    Pros: Zero overhead, private per process
    Cons: Not shared, lost on crash

L2: ETS (in-memory, node-local)       ~microseconds
    Pros: Shared across processes, survive process crash
    Cons: Single node only

L3: Mnesia (distributed ETS)          ~low milliseconds
    Pros: Replicated across cluster
    Cons: Write serialization, schema complexity

L4: Redis / Memcached (external)      ~1-5 milliseconds
    Pros: Shared across all nodes, large capacity
    Cons: Network hop, operational complexity

L5: CDN / Edge Cache                  ~10-100 milliseconds
    Pros: Geographic proximity to user
    Cons: Only for public, HTTP-level content

Cache Strategy by Data Type:
  User session:      L2 (ETS) or L4 (Redis) with 30-min TTL
  Product catalog:   L2 + L4, updated on change event
  User feed:         L4, fan-out-on-write
  Config/feature:    L2, refreshed every 60s
  Auth tokens:       L2 with exact TTL matching token expiry
```

---

## 2. Multi-Layer Cache

```erlang
%% cache.erl — transparent multi-layer cache
-module(cache).
-export([get/2, put/3, invalidate/1, invalidate_pattern/1]).

-define(L1_TTL, 5).     %% seconds — process-local hot cache
-define(L2_TABLE, cache_l2).

%% Get from nearest available cache layer
get(Key, FetchFun) ->
    case get_l1(Key) of
        {ok, Val} -> {ok, Val, l1};
        miss ->
            case get_l2(Key) of
                {ok, Val} ->
                    put_l1(Key, Val),
                    {ok, Val, l2};
                miss ->
                    case FetchFun() of
                        {ok, Val} ->
                            put_l2(Key, Val),
                            put_l1(Key, Val),
                            {ok, Val, origin};
                        Err -> Err
                    end
            end
    end.

put(Key, Val, TTL) ->
    put_l1(Key, Val),
    put_l2_ttl(Key, Val, TTL).

invalidate(Key) ->
    erase({cache_l1, Key}),
    ets:delete(?L2_TABLE, Key),
    ok.

invalidate_pattern(Pattern) ->
    %% Remove all keys matching pattern from L2
    ets:foldl(fun({Key, _, _}, Acc) ->
        case key_matches(Key, Pattern) of
            true  -> ets:delete(?L2_TABLE, Key);
            false -> ok
        end,
        Acc
    end, ok, ?L2_TABLE).

%% L1: process dictionary (hot path, per-process)
get_l1(Key) ->
    case get({cache_l1, Key}) of
        undefined -> miss;
        {Val, ExpiresAt} ->
            case ExpiresAt > erlang:monotonic_time(second) of
                true  -> {ok, Val};
                false ->
                    erase({cache_l1, Key}),
                    miss
            end
    end.

put_l1(Key, Val) ->
    put({cache_l1, Key}, {Val, erlang:monotonic_time(second) + ?L1_TTL}).

%% L2: shared ETS table
get_l2(Key) ->
    case ets:lookup(?L2_TABLE, Key) of
        [{Key, Val, ExpiresAt}] ->
            case ExpiresAt > erlang:system_time(second) of
                true  -> {ok, Val};
                false ->
                    ets:delete(?L2_TABLE, Key),
                    miss
            end;
        [] -> miss
    end.

put_l2(Key, Val) ->
    put_l2_ttl(Key, Val, 300).

put_l2_ttl(Key, Val, TTL) ->
    ExpiresAt = erlang:system_time(second) + TTL,
    ets:insert(?L2_TABLE, {Key, Val, ExpiresAt}).

key_matches(Key, Pattern) ->
    KeyStr     = binary_to_list(term_to_binary(Key)),
    PatternStr = binary_to_list(term_to_binary(Pattern)),
    string:find(KeyStr, PatternStr) =/= nomatch.
```

---

## 3. Cache Invalidation Strategies

```erlang
%% cache_invalidation.erl — event-driven cache invalidation
-module(cache_invalidation).
-behaviour(gen_server).

-export([start_link/0, subscribe/2, publish_change/2]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

%% Cache key patterns to invalidate on each domain event
-define(INVALIDATION_RULES, #{
    user_updated    => [fun(Id) -> {user, Id} end,
                        fun(Id) -> {user_profile, Id} end],
    order_placed    => [fun(Id) -> {order, Id} end,
                        fun(_)  -> orders_recent end],
    product_updated => [fun(Id) -> {product, Id} end,
                        fun(_)  -> product_list end,
                        fun(_)  -> product_search end]
}).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

%% Subscribe a cache module to invalidation events
subscribe(EventType, Module) ->
    gen_server:cast(?MODULE, {subscribe, EventType, Module}).

%% Publish a domain change — triggers cache invalidation
publish_change(EventType, Id) ->
    gen_server:cast(?MODULE, {change, EventType, Id}).

init([]) ->
    {ok, #{subscribers => #{}}}.

handle_cast({subscribe, EventType, Module}, #{subscribers := Subs} = State) ->
    Modules = maps:get(EventType, Subs, []),
    {noreply, State#{subscribers => maps:put(EventType, [Module | Modules], Subs)}};

handle_cast({change, EventType, Id}, #{subscribers := _Subs} = State) ->
    invalidate_for_event(EventType, Id),
    broadcast_to_cluster(EventType, Id),
    {noreply, State}.

handle_info({cluster_invalidate, EventType, Id}, State) ->
    %% Received from another node — invalidate locally
    invalidate_for_event(EventType, Id),
    {noreply, State}.

invalidate_for_event(EventType, Id) ->
    Rules = maps:get(EventType, ?INVALIDATION_RULES, []),
    lists:foreach(fun(KeyFun) ->
        Key = KeyFun(Id),
        cache:invalidate(Key)
    end, Rules).

broadcast_to_cluster(EventType, Id) ->
    Nodes = nodes(),
    lists:foreach(fun(Node) ->
        {?MODULE, Node} ! {cluster_invalidate, EventType, Id}
    end, Nodes).
```

---

## 4. Write-Through and Write-Behind

```erlang
%% write_cache.erl — write-through and write-behind cache patterns
-module(write_cache).
-behaviour(gen_server).

-export([start_link/1, write/3, flush/0]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

-define(FLUSH_INTERVAL_MS, 1000).
-define(MAX_BUFFER_SIZE, 500).

-record(state, {
    mode,           %% write_through | write_behind
    buffer = #{},   %% Key -> {Val, timestamp}
    persist_fun     %% fun(Key, Val) -> ok | error
}).

start_link(Opts) ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, Opts, []).

write(Key, Val, Metadata) ->
    gen_server:call(?MODULE, {write, Key, Val, Metadata}).

flush() ->
    gen_server:call(?MODULE, flush).

init(Opts) ->
    Mode       = maps:get(mode, Opts, write_through),
    PersistFun = maps:get(persist_fun, Opts),
    case Mode of
        write_behind -> schedule_flush();
        _            -> ok
    end,
    {ok, #state{mode = Mode, persist_fun = PersistFun}}.

handle_call({write, Key, Val, _Meta}, _From,
            #state{mode = write_through, persist_fun = PFun} = State) ->
    %% Write-through: persist immediately, update cache
    case PFun(Key, Val) of
        ok ->
            cache:put(Key, Val, 300),
            {reply, ok, State};
        Err ->
            {reply, Err, State}
    end;

handle_call({write, Key, Val, _Meta}, _From,
            #state{mode = write_behind, buffer = Buf} = State) ->
    %% Write-behind: buffer writes, flush async
    cache:put(Key, Val, 300),
    NewBuf = maps:put(Key, {Val, erlang:system_time(millisecond)}, Buf),
    NewState = State#state{buffer = NewBuf},
    case map_size(NewBuf) >= ?MAX_BUFFER_SIZE of
        true  -> do_flush(NewState);
        false -> {reply, ok, NewState}
    end;

handle_call(flush, _From, State) ->
    {reply, ok, State2} = do_flush(State),
    {reply, ok, State2}.

handle_info(flush_timer, State) ->
    {reply, ok, State2} = do_flush(State),
    schedule_flush(),
    {noreply, State2}.

do_flush(#state{buffer = Buf, persist_fun = PFun} = State) ->
    maps:foreach(fun(Key, {Val, _Ts}) ->
        case PFun(Key, Val) of
            ok    -> ok;
            Error -> logger:warning("Write-behind flush error for ~p: ~p", [Key, Error])
        end
    end, Buf),
    {reply, ok, State#state{buffer = #{}}}.

schedule_flush() ->
    erlang:send_after(?FLUSH_INTERVAL_MS, self(), flush_timer).
```

---

## 5. Distributed Cache with ETS

```erlang
%% distributed_cache.erl — ETS cache replicated across cluster nodes
-module(distributed_cache).
-behaviour(gen_server).

-export([start_link/0, get/1, put/2, put/3, delete/1]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

-define(TABLE, distributed_cache).
-define(DEFAULT_TTL, 300).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

get(Key) ->
    case ets:lookup(?TABLE, Key) of
        [{Key, Val, ExpiresAt}] ->
            case ExpiresAt > erlang:system_time(second) of
                true  -> {ok, Val};
                false -> ets:delete(?TABLE, Key), miss
            end;
        [] -> miss
    end.

put(Key, Val) -> put(Key, Val, ?DEFAULT_TTL).
put(Key, Val, TTL) ->
    gen_server:cast(?MODULE, {put, Key, Val, TTL}).

delete(Key) ->
    gen_server:cast(?MODULE, {delete, Key}).

init([]) ->
    ets:new(?TABLE, [named_table, public, {read_concurrency, true}]),
    net_kernel:monitor_nodes(true),
    sync_from_peers(),
    {ok, #{}}.

handle_cast({put, Key, Val, TTL}, State) ->
    ExpiresAt = erlang:system_time(second) + TTL,
    ets:insert(?TABLE, {Key, Val, ExpiresAt}),
    %% Broadcast to all peers
    broadcast({replicate, Key, Val, TTL}),
    {noreply, State};

handle_cast({delete, Key}, State) ->
    ets:delete(?TABLE, Key),
    broadcast({replicate_delete, Key}),
    {noreply, State};

handle_cast({replicate, Key, Val, TTL}, State) ->
    ExpiresAt = erlang:system_time(second) + TTL,
    ets:insert(?TABLE, {Key, Val, ExpiresAt}),
    {noreply, State};

handle_cast({replicate_delete, Key}, State) ->
    ets:delete(?TABLE, Key),
    {noreply, State}.

handle_info({nodeup, Node}, State) ->
    %% New node joined: send our full cache to it
    All = ets:tab2list(?TABLE),
    Now = erlang:system_time(second),
    lists:foreach(fun({Key, Val, ExpiresAt}) ->
        case ExpiresAt > Now of
            true ->
                TTL = ExpiresAt - Now,
                gen_server:cast({?MODULE, Node}, {replicate, Key, Val, TTL});
            false -> ok
        end
    end, All),
    {noreply, State};

handle_info(_, State) ->
    {noreply, State}.

broadcast(Msg) ->
    lists:foreach(fun(Node) ->
        gen_server:cast({?MODULE, Node}, Msg)
    end, nodes()).

sync_from_peers() ->
    case nodes() of
        [] -> ok;
        [Peer | _] -> gen_server:cast({?MODULE, Peer}, sync_request)
    end.
```

---

## 6. Cache Stampede Protection

```erlang
%% cache_lock.erl — prevent thundering herd / cache stampede
-module(cache_lock).
-export([get_or_compute/3, get_or_compute/4]).

-define(LOCK_TABLE, cache_locks).
-define(LOCK_TIMEOUT_MS, 5000).

%% Single-flight: only one process computes; others wait
get_or_compute(Key, ComputeFun, TTL) ->
    get_or_compute(Key, ComputeFun, TTL, #{}).

get_or_compute(Key, ComputeFun, TTL, Opts) ->
    case cache:get(Key, fun() -> {error, miss} end) of
        {ok, Val, _Layer} -> {ok, Val};
        _ ->
            case try_acquire_lock(Key) of
                acquired ->
                    try
                        %% Double-check after acquiring lock
                        case cache:get(Key, fun() -> {error, miss} end) of
                            {ok, Val, _} ->
                                {ok, Val};
                            _ ->
                                Result = ComputeFun(),
                                case Result of
                                    {ok, Val} ->
                                        cache:put(Key, Val, TTL),
                                        notify_waiters(Key, Val),
                                        {ok, Val};
                                    Err ->
                                        notify_waiters_error(Key, Err),
                                        Err
                                end
                        end
                    after
                        release_lock(Key)
                    end;
                waiting ->
                    %% Someone else is computing — wait for their result
                    wait_for_result(Key, maps:get(timeout, Opts, ?LOCK_TIMEOUT_MS))
            end
    end.

try_acquire_lock(Key) ->
    case ets:insert_new(?LOCK_TABLE, {Key, self(), []}) of
        true  -> acquired;
        false -> waiting
    end.

release_lock(Key) ->
    ets:delete(?LOCK_TABLE, Key).

notify_waiters(Key, Val) ->
    case ets:lookup(?LOCK_TABLE, Key) of
        [{Key, _, Waiters}] ->
            lists:foreach(fun(W) -> W ! {cache_result, Key, {ok, Val}} end, Waiters);
        [] -> ok
    end.

notify_waiters_error(Key, Err) ->
    case ets:lookup(?LOCK_TABLE, Key) of
        [{Key, _, Waiters}] ->
            lists:foreach(fun(W) -> W ! {cache_result, Key, Err} end, Waiters);
        [] -> ok
    end.

wait_for_result(Key, Timeout) ->
    %% Register as a waiter
    ets:update_element(?LOCK_TABLE, Key, {3, [self() | element(3, hd(ets:lookup(?LOCK_TABLE, Key)))]}),
    receive
        {cache_result, Key, Result} -> Result
    after Timeout ->
        {error, cache_lock_timeout}
    end.
```

---

## 7. แบบฝึกหัด

1. สร้าง cache warming job ที่โหลด popular products เข้า cache ก่อน traffic เช้า
2. Implement cache metrics: hit rate, miss rate, eviction rate per cache layer
3. เพิ่ม circuit breaker เมื่อ Redis ไม่ response — fall through ไป DB โดยตรง
4. สร้าง tag-based invalidation: `cache:invalidate_tag(product_category)` ลบ cache ทุกตัวที่ tagged

---

## สรุป Part 86

✅ Cache topology: 5 layers จาก process dict ถึง CDN พร้อม trade-offs  
✅ Multi-layer cache: L1 process + L2 ETS แบบ transparent  
✅ Event-driven invalidation: domain change triggers cluster-wide invalidation  
✅ Write-through vs write-behind: choose based on consistency requirements  
✅ Distributed ETS: replicate via node broadcast, sync on nodeup  
✅ Stampede protection: single-flight lock, waiters queue  

---

*Part 86/100 | [← ก่อนหน้า](../part85/README.md) | [ถัดไป →](../part87/README.md)*
