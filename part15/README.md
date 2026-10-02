# Part 15: ETS — Erlang Term Storage

> **"ETS gives you in-memory tables as fast as a hash map, shared across processes"**  
> ETS ให้ตาราง in-memory เร็วเท่า hash map และแชร์ระหว่าง processes ได้

---

## สารบัญ

1. [ETS คืออะไร?](#1-ets-คืออะไร)
2. [สร้างและจัดการ Table](#2-สร้างและจัดการ-table)
3. [Table Types](#3-table-types)
4. [Insert และ Lookup](#4-insert-และ-lookup)
5. [Query ด้วย Match Spec](#5-query-ด้วย-match-spec)
6. [Select และ Continuation](#6-select-และ-continuation)
7. [Update Operations](#7-update-operations)
8. [Table Traversal](#8-table-traversal)
9. [ETS Patterns](#9-ets-patterns)
10. [Performance](#10-performance)
11. [แบบฝึกหัด](#11-แบบฝึกหัด)

---

## 1. ETS คืออะไร?

```
ETS (Erlang Term Storage):
├── In-memory key-value store
├── ไม่ใช่ process — เป็น table ใน BEAM
├── แชร์ระหว่าง processes ได้
├── No garbage collection (นอก BEAM process GC)
├── เร็วมาก: O(1) lookup สำหรับ set/bag
├── ไม่ persistent — node crash → ข้อมูลหาย
└── เหมาะสำหรับ: cache, shared state, lookup tables

เปรียบเทียบ:
- ETS: fast, shared, not persistent
- Mnesia: persistent, distributed, slower
- gen_server state: private, slowest
```

---

## 2. สร้างและจัดการ Table

```erlang
%% สร้าง table
TabId = ets:new(my_table, [Options]).

%% Options:
%% Type: set, ordered_set, bag, duplicate_bag
%% Access: public, protected, private
%% named_table — ใช้ชื่อแทน TabId
%% {keypos, N} — key อยู่ที่ position N ใน tuple
%% {heir, Pid, Data} — ถ้า owner ตาย ส่งให้ Pid
%% {read_concurrency, true}  — optimize concurrent reads
%% {write_concurrency, true} — optimize concurrent writes
%% compressed — เก็บข้อมูลแบบ compressed

%% ตัวอย่าง
Tab = ets:new(cache, [set, public, named_table,
                      {read_concurrency, true}]).
%% Tab = cache (เพราะ named_table)

%% ลบ table
ets:delete(Tab).

%% ดู info
ets:info(Tab).
ets:info(Tab, size).
ets:info(Tab, memory).
ets:info(Tab, type).
ets:info(Tab, protection).

%% List ทุก tables
ets:all().
```

---

## 3. Table Types

```erlang
%% set — key ไม่ซ้ำ, เร็วที่สุด
Tab = ets:new(t, [set]),
ets:insert(Tab, {key1, val1}),
ets:insert(Tab, {key1, val2}),  %% overwrite key1
ets:lookup(Tab, key1).
%% [{key1, val2}]

%% ordered_set — key ไม่ซ้ำ, เรียงลำดับ (B-tree)
Tab = ets:new(t, [ordered_set]),
%% ใช้ first/next สำหรับ ordered traversal

%% bag — key ซ้ำได้, values ต้องต่างกัน
Tab = ets:new(t, [bag]),
ets:insert(Tab, {key1, val1}),
ets:insert(Tab, {key1, val2}),  %% both stored
ets:lookup(Tab, key1).
%% [{key1, val1}, {key1, val2}]

%% duplicate_bag — key ซ้ำได้, values ก็ซ้ำได้
Tab = ets:new(t, [duplicate_bag]),
ets:insert(Tab, {key1, val1}),
ets:insert(Tab, {key1, val1}),  %% both stored
ets:lookup(Tab, key1).
%% [{key1, val1}, {key1, val1}]
```

---

## 4. Insert และ Lookup

```erlang
%% Insert
ets:insert(Tab, {key, value}).
ets:insert(Tab, [{k1, v1}, {k2, v2}]).  %% batch insert

%% insert_new: insert เฉพาะถ้า key ไม่มีอยู่
ets:insert_new(Tab, {key, value}).  %% true | false

%% Lookup
ets:lookup(Tab, key).            %% [{key, value}] | []
ets:lookup_element(Tab, key, 2). %% value (field 2) | error

%% Member
ets:member(Tab, key).  %% true | false

%% Delete
ets:delete(Tab, key).       %% ลบทุก records ที่มี key นี้
ets:delete_object(Tab, Obj). %% ลบ exact object

%% ตัวอย่าง cache pattern
cache_get(Tab, Key) ->
    case ets:lookup(Tab, Key) of
        [{Key, Value}] -> {ok, Value};
        []             -> miss
    end.

cache_put(Tab, Key, Value) ->
    ets:insert(Tab, {Key, Value}).

cache_delete(Tab, Key) ->
    ets:delete(Tab, Key).
```

---

## 5. Query ด้วย Match Spec

```erlang
%% match: ใช้ pattern
ets:match(Tab, {_, value}).        %% คืน [[Key], ...]
ets:match(Tab, {'$1', '$2'}).      %% คืน [[Key, Val], ...]
ets:match_object(Tab, {_, val}).   %% คืน [{Key, val}, ...]

%% match spec: pattern + guard + return
MS = ets:fun2ms(fun({Key, Value}) when Value > 10 -> {Key, Value} end).
ets:select(Tab, MS).

%% Manual match spec
%% [{MatchHead, Guards, ReturnBody}]
MS2 = [{{user, '$1', '$2', '$3'},  %% MatchHead: {user, Name, Age, Score}
        [{'>', '$3', 100}],         %% Guard: Score > 100
        [{{'$1', '$2'}}]}].         %% Return: {Name, Age}
ets:select(Tab, MS2).

%% ตัวอย่าง: หา users ที่ active = true
Users = ets:match_object(Tab, {user, '_', '_', true}).

%% ตัวอย่าง: ใช้ fun2ms
MS3 = ets:fun2ms(fun(#user{name=N, active=true}) -> N end).
%% ต้องใช้ include_lib("stdlib/include/ms_transform.hrl")
```

---

## 6. Select และ Continuation

```erlang
%% select_count: นับ
Count = ets:select_count(Tab, MS).

%% select_delete: ลบที่ match
Deleted = ets:select_delete(Tab, MS).

%% Paginated select (Continuation)
{Results, Continuation} = ets:select(Tab, MS, 100).
%% ดึง 100 records ต่อครั้ง

fetch_all(Tab, MS) ->
    fetch_all(Tab, MS, ets:select(Tab, MS, 100), []).

fetch_all(_Tab, _MS, '$end_of_table', Acc) ->
    lists:flatten(lists:reverse(Acc));
fetch_all(Tab, MS, {Rows, Cont}, Acc) ->
    fetch_all(Tab, MS, ets:select(Cont), [Rows|Acc]).
```

---

## 7. Update Operations

```erlang
%% update_counter: atomic increment/decrement
%% {Tab, Key, Increment}
ets:update_counter(Tab, key, 1).     %% increment by 1
ets:update_counter(Tab, key, -5).    %% decrement by 5
ets:update_counter(Tab, key, {2, 1}).  %% field 2, increment 1

%% update_element: เปลี่ยน field
ets:update_element(Tab, key, {2, new_value}).
ets:update_element(Tab, key, [{2, v1}, {3, v2}]).

%% Atomic insert_or_update pattern
atomic_update(Tab, Key, Fun) ->
    case ets:lookup(Tab, Key) of
        [{Key, OldVal}] ->
            NewVal = Fun(OldVal),
            ets:insert(Tab, {Key, NewVal}),
            NewVal;
        [] ->
            Default = Fun(undefined),
            ets:insert_new(Tab, {Key, Default}),
            Default
    end.
%% หมายเหตุ: ไม่ thread-safe ถ้าหลาย process เขียนพร้อมกัน

%% Thread-safe counter pattern
init_counter(Tab, Name) ->
    ets:insert_new(Tab, {Name, 0}).

increment(Tab, Name) ->
    ets:update_counter(Tab, Name, 1).
%% update_counter เป็น atomic operation!
```

---

## 8. Table Traversal

```erlang
%% first/next สำหรับ ordered_set
Key = ets:first(Tab),
next_key(Tab, Key) ->
    ets:next(Tab, Key).  %% '$end_of_table' เมื่อหมด

%% traverse ทั้งหมด (ordered_set)
traverse_ordered(Tab) ->
    traverse_ordered(Tab, ets:first(Tab), []).

traverse_ordered(_Tab, '$end_of_table', Acc) ->
    lists:reverse(Acc);
traverse_ordered(Tab, Key, Acc) ->
    [{Key, Value}] = ets:lookup(Tab, Key),
    traverse_ordered(Tab, ets:next(Tab, Key), [{Key, Value}|Acc]).

%% tab2list — ดึงทั้ง table
All = ets:tab2list(Tab).

%% foldl
Result = ets:foldl(fun({Key, Val}, Acc) ->
    maps:put(Key, Val, Acc)
end, #{}, Tab).

%% to_dets
ets:tab2file(Tab, "backup.dets").
{ok, _} = ets:file2tab("backup.dets").
```

---

## 9. ETS Patterns

### Cache with TTL

```erlang
-module(ttl_cache).
-export([new/0, put/3, get/2, cleanup/1]).

new() ->
    ets:new(ttl_cache, [set, public, named_table,
                        {read_concurrency, true}]).

put(Tab, Key, Value) ->
    put(Tab, Key, Value, 300).  %% default 5 min TTL

put(Tab, Key, Value, TTL) ->
    Expires = erlang:system_time(second) + TTL,
    ets:insert(Tab, {Key, Value, Expires}).

get(Tab, Key) ->
    Now = erlang:system_time(second),
    case ets:lookup(Tab, Key) of
        [{Key, Value, Expires}] when Expires > Now ->
            {ok, Value};
        [{Key, _, _}] ->
            ets:delete(Tab, Key),
            miss;
        [] ->
            miss
    end.

%% Periodic cleanup (ส่ง cleanup message ทุก N ms)
cleanup(Tab) ->
    Now = erlang:system_time(second),
    MS = ets:fun2ms(fun({_K, _V, Exp}) when Exp =< Now -> true end),
    Deleted = ets:select_delete(Tab, MS),
    Deleted.
```

### Distributed Counter

```erlang
%% ใช้ ETS สำหรับ atomic counters
-module(counters_ets).
-export([new/1, inc/2, dec/2, get/2, reset/2]).

new(Tab) ->
    ets:new(Tab, [set, public, named_table]).

inc(Tab, Name) ->
    try ets:update_counter(Tab, Name, 1)
    catch error:badarg ->
        ets:insert_new(Tab, {Name, 0}),
        ets:update_counter(Tab, Name, 1)
    end.

dec(Tab, Name) ->
    try ets:update_counter(Tab, Name, -1)
    catch error:badarg ->
        0
    end.

get(Tab, Name) ->
    case ets:lookup(Tab, Name) of
        [{Name, N}] -> N;
        []          -> 0
    end.

reset(Tab, Name) ->
    ets:insert(Tab, {Name, 0}).
```

### Index Table

```erlang
%% Primary table: {id, data}
%% Index table: {email, id}
-module(indexed_store).
-export([new/0, insert/3, find_by_id/2, find_by_email/2, delete/2]).

new() ->
    Primary = ets:new(users, [set, public, {keypos, 1}]),
    Index   = ets:new(email_index, [set, public, {keypos, 1}]),
    {Primary, Index}.

insert({Primary, Index}, User = #{id := Id, email := Email}) ->
    ets:insert(Primary, {Id, User}),
    ets:insert(Index, {Email, Id}).

find_by_id({Primary, _Index}, Id) ->
    case ets:lookup(Primary, Id) of
        [{Id, User}] -> {ok, User};
        []           -> not_found
    end.

find_by_email({Primary, Index}, Email) ->
    case ets:lookup(Index, Email) of
        [{Email, Id}] -> find_by_id({Primary, Index}, Id);
        []            -> not_found
    end.

delete({Primary, Index}, Id) ->
    case ets:lookup(Primary, Id) of
        [{Id, #{email := Email}}] ->
            ets:delete(Primary, Id),
            ets:delete(Index, Email);
        [] -> ok
    end.
```

---

## 10. Performance

```erlang
%% read_concurrency: optimize concurrent reads (default false)
%% → ดีสำหรับ read-heavy workloads

%% write_concurrency: reduce lock contention on writes (default false)
%% → ดีสำหรับ write-heavy workloads

%% compressed: ลด memory แต่ช้ากว่า

%% Tips:
%% 1. ใช้ named_table เพื่อไม่ต้อง pass TabId
%% 2. {keypos, N} สำหรับ records ที่ key ไม่ใช่ field แรก
%% 3. ใช้ ets:update_counter สำหรับ atomic counters (ไม่ต้องใช้ GenServer)
%% 4. ets:select แทน ets:tab2list สำหรับ filtered queries
%% 5. อย่า ets:tab2list บน table ขนาดใหญ่ (copies ทั้ง table)

%% Benchmark
bench_ets() ->
    Tab = ets:new(bench, [set, public]),
    N = 100000,
    
    %% Insert
    T1 = erlang:monotonic_time(millisecond),
    [ets:insert(Tab, {I, I}) || I <- lists:seq(1, N)],
    T2 = erlang:monotonic_time(millisecond),
    io:format("Insert ~p: ~pms~n", [N, T2-T1]),
    
    %% Lookup
    T3 = erlang:monotonic_time(millisecond),
    [ets:lookup(Tab, I) || I <- lists:seq(1, N)],
    T4 = erlang:monotonic_time(millisecond),
    io:format("Lookup ~p: ~pms~n", [N, T4-T3]),
    
    ets:delete(Tab).
```

---

## 11. แบบฝึกหัด

### Exercise: Session Store

```erlang
%% สร้าง session store ด้วย ETS
%% - สร้าง session, อ่าน session, ลบ session
%% - TTL support
%% - Periodic cleanup

-module(session_store).
-behaviour(gen_server).
-export([start_link/0, create/2, get/1, delete/1]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2, terminate/2]).

-define(TAB, session_ets).
-define(TTL, 1800).  %% 30 minutes
-define(CLEANUP_INTERVAL, 60000).  %% 1 minute

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

create(UserId, Data) ->
    Token = crypto:strong_rand_bytes(16),
    gen_server:call(?MODULE, {create, Token, UserId, Data}).

get(Token) ->
    case ets:lookup(?TAB, Token) of
        [{Token, UserId, Data, Expires}] ->
            Now = erlang:system_time(second),
            case Now < Expires of
                true  -> {ok, #{user_id => UserId, data => Data}};
                false ->
                    ets:delete(?TAB, Token),
                    {error, expired}
            end;
        [] ->
            {error, not_found}
    end.

delete(Token) ->
    ets:delete(?TAB, Token),
    ok.

init([]) ->
    ets:new(?TAB, [set, public, named_table,
                   {read_concurrency, true}]),
    erlang:send_after(?CLEANUP_INTERVAL, self(), cleanup),
    {ok, #{}}.

handle_call({create, Token, UserId, Data}, _From, State) ->
    Expires = erlang:system_time(second) + ?TTL,
    ets:insert(?TAB, {Token, UserId, Data, Expires}),
    {reply, {ok, Token}, State}.

handle_cast(_Msg, State) ->
    {noreply, State}.

handle_info(cleanup, State) ->
    Now = erlang:system_time(second),
    MS = ets:fun2ms(fun({_T, _U, _D, Exp}) when Exp =< Now -> true end),
    Deleted = ets:select_delete(?TAB, MS),
    logger:debug("Session cleanup: deleted ~p expired sessions", [Deleted]),
    erlang:send_after(?CLEANUP_INTERVAL, self(), cleanup),
    {noreply, State}.

terminate(_Reason, _State) ->
    ok.
```

---

## สรุป Part 15

✅ ETS คืออะไร — in-memory shared table  
✅ Table types: set, ordered_set, bag, duplicate_bag  
✅ Access modes: public, protected, private  
✅ Insert, lookup, delete operations  
✅ Match spec สำหรับ complex queries  
✅ Select with continuation (pagination)  
✅ Atomic update_counter  
✅ Patterns: TTL Cache, Counters, Index tables  
✅ Performance tips

---

*Part 15/100 | [← ก่อนหน้า](../part14/README.md) | [ถัดไป →](../part16/README.md)*
