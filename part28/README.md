# Part 28: Caching Strategies

> **"Cache is king — but stale cache is worse than no cache"**  
> Cache คือ king — แต่ stale cache แย่กว่าไม่มี cache

---

## สารบัญ

1. [Caching Concepts](#1-caching-concepts)
2. [ETS Cache พื้นฐาน](#2-ets-cache-พื้นฐาน)
3. [TTL Cache](#3-ttl-cache)
4. [LRU Cache](#4-lru-cache)
5. [Write-Through Cache](#5-write-through-cache)
6. [Cache-Aside Pattern](#6-cache-aside-pattern)
7. [Distributed Cache ด้วย ETS + pg](#7-distributed-cache-ด้วย-ets--pg)
8. [Cache Stampede Prevention](#8-cache-stampede-prevention)
9. [แบบฝึกหัด](#9-แบบฝึกหัด)

---

## 1. Caching Concepts

```
Cache Strategies:

Cache-Aside (Lazy Loading):
  Read: check cache → miss → load DB → write cache → return
  Write: write DB → invalidate cache

Write-Through:
  Write: write cache + write DB (synchronous)
  Read: check cache → (always hit)

Write-Behind (Write-Back):
  Write: write cache → async write DB
  Risk: data loss on crash

Read-Through:
  Cache handles DB loading automatically

Cache Eviction Policies:
  LRU  — Least Recently Used
  LFU  — Least Frequently Used
  TTL  — Time To Live
  FIFO — First In First Out

Cache Invalidation (hardest problem):
  1. TTL-based expiry
  2. Event-driven invalidation
  3. Version keys
```

---

## 2. ETS Cache พื้นฐาน

```erlang
%% simple_cache.erl
-module(simple_cache).
-export([start/0, get/1, put/2, delete/1, clear/0, size/0]).

start() ->
    ets:new(simple_cache, [
        named_table,
        set,
        public,
        {read_concurrency, true},
        {write_concurrency, true}
    ]),
    ok.

get(Key) ->
    case ets:lookup(simple_cache, Key) of
        [{Key, Value}] -> {ok, Value};
        []             -> miss
    end.

put(Key, Value) ->
    ets:insert(simple_cache, {Key, Value}),
    ok.

delete(Key) ->
    ets:delete(simple_cache, Key),
    ok.

clear() ->
    ets:delete_all_objects(simple_cache),
    ok.

size() ->
    ets:info(simple_cache, size).

%% ใช้งาน
example() ->
    simple_cache:start(),
    simple_cache:put(user_1, #{name => <<"Alice">>, age => 30}),
    {ok, User} = simple_cache:get(user_1),
    io:format("User: ~p~n", [User]).
```

---

## 3. TTL Cache

```erlang
%% ttl_cache.erl — Cache พร้อม Time-To-Live
-module(ttl_cache).
-behaviour(gen_server).

-export([start_link/0, start_link/1,
         get/1, put/2, put/3, delete/1, flush/0]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

-define(DEFAULT_TTL,     300).  %% 5 minutes
-define(CLEANUP_INTERVAL, 60000). %% 1 minute

start_link() -> start_link([]).
start_link(Opts) ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, Opts, []).

get(Key) ->
    Now = erlang:system_time(second),
    case ets:lookup(ttl_cache, Key) of
        [{Key, Value, Exp}] when Exp > Now ->
            {ok, Value};
        [{Key, _, _}] ->
            ets:delete(ttl_cache, Key),
            miss;
        [] ->
            miss
    end.

put(Key, Value) -> put(Key, Value, ?DEFAULT_TTL).
put(Key, Value, TTL) ->
    Exp = erlang:system_time(second) + TTL,
    ets:insert(ttl_cache, {Key, Value, Exp}),
    ok.

delete(Key) ->
    ets:delete(ttl_cache, Key),
    ok.

flush() ->
    ets:delete_all_objects(ttl_cache),
    ok.

init(Opts) ->
    ets:new(ttl_cache, [
        named_table, set, public,
        {read_concurrency, true}
    ]),
    Interval = proplists:get_value(cleanup_interval, Opts, ?CLEANUP_INTERVAL),
    erlang:send_after(Interval, self(), cleanup),
    {ok, #{interval => Interval}}.

handle_info(cleanup, #{interval:=Interval}=S) ->
    Now = erlang:system_time(second),
    %% Delete expired entries
    ets:select_delete(ttl_cache, [{{'_', '_', '$1'}, [{'<','$1',Now}], [true]}]),
    erlang:send_after(Interval, self(), cleanup),
    {noreply, S};
handle_info(_, S) -> {noreply, S}.

handle_call(_, _, S) -> {noreply, S}.
handle_cast(_, S)    -> {noreply, S}.
```

---

## 4. LRU Cache

```erlang
%% lru_cache.erl — Least Recently Used cache with max size
-module(lru_cache).
-behaviour(gen_server).

-export([start_link/1, get/1, put/2, size/0]).
-export([init/1, handle_call/3, handle_cast/2]).

start_link(MaxSize) ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [MaxSize], []).

get(Key) ->
    gen_server:call(?MODULE, {get, Key}).

put(Key, Value) ->
    gen_server:call(?MODULE, {put, Key, Value}).

size() ->
    gen_server:call(?MODULE, size).

init([MaxSize]) ->
    %% data: #{key => value}
    %% order: [{ts, key}] sorted by timestamp (most recent last)
    {ok, #{max_size => MaxSize, data => #{}, order => []}}.

handle_call({get, Key}, _From, #{data:=Data, order:=Order}=S) ->
    case maps:find(Key, Data) of
        {ok, Value} ->
            %% Update access time (move to front)
            Now = erlang:monotonic_time(),
            NewOrder = [{Now, Key} | lists:keydelete(Key, 2, Order)],
            {reply, {ok, Value}, S#{order := NewOrder}};
        error ->
            {reply, miss, S}
    end;

handle_call({put, Key, Value}, _From,
            #{max_size:=Max, data:=Data, order:=Order}=S) ->
    Now = erlang:monotonic_time(),
    Data2  = Data#{Key => Value},
    Order2 = [{Now, Key} | lists:keydelete(Key, 2, Order)],

    %% Evict LRU if over capacity
    {Data3, Order3} = maybe_evict(Data2, Order2, Max),
    {reply, ok, S#{data := Data3, order := Order3}};

handle_call(size, _From, #{data:=Data}=S) ->
    {reply, maps:size(Data), S}.

handle_cast(_, S) -> {noreply, S}.

maybe_evict(Data, Order, Max) when map_size(Data) > Max ->
    %% Remove oldest (last in Order list)
    Oldest = lists:last(Order),
    {_, OldKey} = Oldest,
    {maps:remove(OldKey, Data),
     lists:droplast(Order)};
maybe_evict(Data, Order, _Max) ->
    {Data, Order}.
```

---

## 5. Write-Through Cache

```erlang
%% write_through_cache.erl
-module(write_through_cache).
-export([get/1, put/2, delete/1]).

%% Read: cache-first, fallback to DB
get(Key) ->
    case ttl_cache:get(Key) of
        {ok, Value} ->
            {ok, Value};
        miss ->
            case db:get(Key) of
                {ok, Value} ->
                    ttl_cache:put(Key, Value),
                    {ok, Value};
                {error, not_found} ->
                    %% Cache negative result (prevent DB hammering)
                    ttl_cache:put(Key, undefined, 60),
                    {error, not_found};
                Error ->
                    Error
            end
    end.

%% Write: write to both cache and DB atomically
put(Key, Value) ->
    case db:put(Key, Value) of
        ok ->
            ttl_cache:put(Key, Value),
            ok;
        Error ->
            Error
    end.

%% Delete: remove from both
delete(Key) ->
    db:delete(Key),
    ttl_cache:delete(Key),
    ok.
```

---

## 6. Cache-Aside Pattern

```erlang
%% cache_aside.erl
-module(cache_aside).
-export([get_user/1, update_user/2, delete_user/1]).

cache_key(Type, Id) ->
    iolist_to_binary([atom_to_binary(Type), ":", integer_to_binary(Id)]).

get_user(UserId) ->
    Key = cache_key(user, UserId),
    case ttl_cache:get(Key) of
        {ok, User} ->
            {ok, User};
        miss ->
            case user_repo:find(UserId) of
                {ok, User} ->
                    ttl_cache:put(Key, User, 300),
                    {ok, User};
                Error ->
                    Error
            end
    end.

update_user(UserId, Changes) ->
    case user_repo:update(UserId, Changes) of
        {ok, UpdatedUser} ->
            %% Invalidate cache
            Key = cache_key(user, UserId),
            ttl_cache:delete(Key),
            {ok, UpdatedUser};
        Error ->
            Error
    end.

delete_user(UserId) ->
    case user_repo:delete(UserId) of
        ok ->
            Key = cache_key(user, UserId),
            ttl_cache:delete(Key),
            ok;
        Error ->
            Error
    end.
```

---

## 7. Distributed Cache ด้วย ETS + pg

```erlang
%% distributed_cache.erl — Cache invalidation across nodes
-module(distributed_cache).
-behaviour(gen_server).

-export([start_link/0, get/1, put/2, invalidate/1, broadcast_invalidate/1]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

get(Key) ->
    case ets:lookup(dist_cache, Key) of
        [{Key, Value}] -> {ok, Value};
        []             -> miss
    end.

put(Key, Value) ->
    ets:insert(dist_cache, {Key, Value}),
    ok.

invalidate(Key) ->
    ets:delete(dist_cache, Key),
    ok.

broadcast_invalidate(Key) ->
    %% ส่ง invalidation ไปทุก node ใน cluster
    Pids = pg:get_members(dist_cache_group),
    lists:foreach(fun(Pid) ->
        gen_server:cast(Pid, {invalidate, Key})
    end, Pids),
    ok.

init([]) ->
    ets:new(dist_cache, [named_table, set, public,
                          {read_concurrency, true}]),
    %% Join process group
    pg:start_link(),
    pg:join(dist_cache_group, self()),
    {ok, #{}}.

handle_cast({invalidate, Key}, S) ->
    ets:delete(dist_cache, Key),
    {noreply, S};

handle_call(_, _, S) -> {noreply, S}.
handle_info(_, S)    -> {noreply, S}.
```

---

## 8. Cache Stampede Prevention

```erlang
%% cache_lock.erl — Prevent multiple processes loading same key simultaneously
-module(cache_lock).
-behaviour(gen_server).

-export([start_link/0, get_or_load/2]).
-export([init/1, handle_call/3, handle_cast/2]).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

%% Caller-side: get from cache OR load once (others wait)
get_or_load(Key, LoadFun) ->
    case ttl_cache:get(Key) of
        {ok, V} -> {ok, V};
        miss    -> load_with_lock(Key, LoadFun)
    end.

load_with_lock(Key, LoadFun) ->
    case gen_server:call(?MODULE, {lock, Key}) of
        acquired ->
            %% We won the lock: load the data
            Result = try LoadFun()
                     catch _:E -> {error, E}
                     end,
            case Result of
                {ok, Value} ->
                    ttl_cache:put(Key, Value),
                    gen_server:cast(?MODULE, {unlock, Key, {ok, Value}}),
                    {ok, Value};
                Error ->
                    gen_server:cast(?MODULE, {unlock, Key, Error}),
                    Error
            end;
        {wait, From} ->
            %% Another process is loading: wait for result
            receive
                {cache_result, Key, Result} -> Result
            after 5000 ->
                {error, timeout}
            end
    end.

init([]) ->
    {ok, #{}}.  %% #{key => {owner_pid, [waiting_pids]}}

handle_call({lock, Key}, {CallerPid,_}, Locks) ->
    case maps:find(Key, Locks) of
        error ->
            %% No lock: acquire it
            {reply, acquired, Locks#{Key => {CallerPid, []}}};
        {ok, {_Owner, Waiters}} ->
            %% Already locked: add to waiters
            Locks2 = Locks#{Key => {_Owner, [CallerPid | Waiters]}},
            {reply, {wait, CallerPid}, Locks2}
    end.

handle_cast({unlock, Key, Result}, Locks) ->
    case maps:find(Key, Locks) of
        {ok, {_, Waiters}} ->
            lists:foreach(fun(Pid) ->
                Pid ! {cache_result, Key, Result}
            end, Waiters),
            {noreply, maps:remove(Key, Locks)};
        error ->
            {noreply, Locks}
    end.
```

---

## 9. แบบฝึกหัด

### Exercise: Multi-Level Cache

```erlang
%% สร้าง L1 (process memory) + L2 (ETS) + L3 (DB) cache

-module(multi_level_cache).
-export([get/1, put/2]).

-define(L1_SIZE, 100).    %% process dict
-define(L2_TTL,  300).    %% ETS 5 min
-define(L3_TTL,  3600).   %% DB 1 hr

get(Key) ->
    case get_l1(Key) of
        {ok, V} ->
            {ok, V};
        miss ->
            case get_l2(Key) of
                {ok, V} ->
                    put_l1(Key, V),
                    {ok, V};
                miss ->
                    case get_l3(Key) of
                        {ok, V} ->
                            put_l1(Key, V),
                            ttl_cache:put(Key, V, ?L2_TTL),
                            {ok, V};
                        Error ->
                            Error
                    end
            end
    end.

put(Key, Value) ->
    put_l1(Key, Value),
    ttl_cache:put(Key, Value, ?L2_TTL),
    db:put(Key, Value).

%% L1: process dictionary (ultra fast, per-process)
get_l1(Key) ->
    case get({l1, Key}) of
        undefined -> miss;
        Value     -> {ok, Value}
    end.

put_l1(Key, Value) ->
    put({l1, Key}, Value).

%% L2: shared ETS
get_l2(Key) -> ttl_cache:get(Key).

%% L3: database
get_l3(Key) -> db:get(Key).
```

---

## สรุป Part 28

✅ Caching concepts: strategies, eviction, invalidation  
✅ ETS cache พื้นฐาน  
✅ TTL cache พร้อม cleanup  
✅ LRU cache ด้วย GenServer  
✅ Write-through cache  
✅ Cache-aside pattern  
✅ Distributed cache ด้วย pg  
✅ Cache stampede prevention  
✅ Multi-level cache

---

*Part 28/100 | [← ก่อนหน้า](../part27/README.md) | [ถัดไป →](../part29/README.md)*
