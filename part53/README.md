# Part 53: Advanced ETS and Mnesia

> **"In-memory data structures that outlive their creators — the Erlang way"**  
> โครงสร้างข้อมูลใน memory ที่อยู่ยาวนานกว่า process ที่สร้างมัน — วิถี Erlang

---

## สารบัญ

1. [ETS Advanced Patterns](#1-ets-advanced-patterns)
2. [ETS Select ขั้นสูง](#2-ets-select-ขั้นสูง)
3. [ETS Counters และ Atomics](#3-ets-counters-และ-atomics)
4. [Mnesia Schema Design](#4-mnesia-schema-design)
5. [Mnesia Queries](#5-mnesia-queries)
6. [Mnesia Distribution](#6-mnesia-distribution)
7. [แบบฝึกหัด](#7-แบบฝึกหัด)

---

## 1. ETS Advanced Patterns

```erlang
%% ets_patterns.erl — production ETS patterns
-module(ets_patterns).

%% Pattern 1: Expiry-aware lookup
lookup_live(Table, Key) ->
    Now = os:system_time(second),
    case ets:lookup(Table, Key) of
        [{Key, Value, ExpiresAt}] when ExpiresAt > Now -> {ok, Value};
        [{Key, _, _}] -> expired;
        [] -> miss
    end.

%% Pattern 2: Atomic compare-and-swap
cas(Table, Key, Expected, NewValue) ->
    %% Not truly atomic — use select_replace for real CAS
    case ets:lookup(Table, Key) of
        [{Key, Expected}] ->
            ets:update_element(Table, Key, {2, NewValue}),
            {ok, swapped};
        [{Key, Current}] ->
            {error, {conflict, Current}};
        [] ->
            {error, not_found}
    end.

%% Pattern 3: Bulk operations
bulk_insert(Table, KeyValuePairs) ->
    ets:insert(Table, KeyValuePairs).

bulk_delete(Table, Keys) ->
    [ets:delete(Table, K) || K <- Keys].

%% Pattern 4: Size-bounded table with LRU eviction
bounded_insert(Table, Key, Value, MaxSize) ->
    case ets:info(Table, size) of
        N when N >= MaxSize ->
            %% Evict oldest (needs ordered_set with timestamp keys)
            [{OldestKey, _} | _] = ets:tab2list(Table),
            ets:delete(Table, OldestKey);
        _ -> ok
    end,
    ets:insert(Table, {Key, Value}).

%% Pattern 5: Bag table for multi-value keys
-define(TAGS_TABLE, tags_bag).

add_tag(EntityId, Tag) ->
    ets:insert(?TAGS_TABLE, {EntityId, Tag}).

get_tags(EntityId) ->
    [Tag || {_, Tag} <- ets:lookup(?TAGS_TABLE, EntityId)].

remove_tag(EntityId, Tag) ->
    ets:delete_object(?TAGS_TABLE, {EntityId, Tag}).

has_tag(EntityId, Tag) ->
    ets:member(?TAGS_TABLE, EntityId) andalso
    lists:member(Tag, get_tags(EntityId)).
```

---

## 2. ETS Select ขั้นสูง

```erlang
%% Advanced ets:select with match specifications

%% Example table: {user_id, name, age, score}
-define(USERS, user_ets_table).

%% Find users over 18 with score > 100
adult_high_scorers() ->
    ets:select(?USERS, [
        {
            {'$1', '$2', '$3', '$4'},           %% Pattern
            [{'>', '$3', 18}, {'>', '$4', 100}], %% Guards
            [['$1', '$2', '$3', '$4']]           %% Result
        }
    ]).

%% Using ets:fun2ms for more readable specs
-include_lib("stdlib/include/ms_transform.hrl").

adult_high_scorers_v2() ->
    MS = ets:fun2ms(fun({Id, Name, Age, Score})
                       when Age > 18, Score > 100 ->
        {Id, Name}
    end),
    ets:select(?USERS, MS).

%% Count users by age group
count_by_age_group() ->
    All = ets:tab2list(?USERS),
    lists:foldl(fun({_, _, Age, _}, Acc) ->
        Group = case Age of
            A when A < 18 -> under_18;
            A when A < 30 -> youth;
            A when A < 60 -> adult;
            _              -> senior
        end,
        maps:update_with(Group, fun(N) -> N+1 end, 1, Acc)
    end, #{}, All).

%% Efficient pagination
page(Table, AfterKey, PageSize) ->
    case AfterKey of
        undefined ->
            ets:select(Table, [{'$1', [], ['$1']}], PageSize);
        Key ->
            %% ordered_set only: start after the given key
            ets:select(Table,
                       [{{'$1', '$2'}, [{'>', '$1', Key}], [['$1', '$2']]}],
                       PageSize)
    end.
```

---

## 3. ETS Counters และ Atomics

```erlang
%% High-performance atomic counters

%% Method 1: ets:update_counter (atomic)
-define(COUNTERS_TABLE, atomic_counters).

init_counter(Name) ->
    ets:insert(?COUNTERS_TABLE, {Name, 0}).

increment(Name) ->
    ets:update_counter(?COUNTERS_TABLE, Name, 1).

increment_by(Name, By) ->
    ets:update_counter(?COUNTERS_TABLE, Name, By).

read(Name) ->
    [{Name, N}] = ets:lookup(?COUNTERS_TABLE, Name),
    N.

%% Method 2: counters module (even faster, no table overhead)
-module(fast_counters).
-export([new/1, inc/2, dec/2, get/2]).

new(N) ->
    counters:new(N, [atomics]).  %% array of N atomic integers

inc(Ref, Idx) ->
    counters:add(Ref, Idx, 1).

dec(Ref, Idx) ->
    counters:add(Ref, Idx, -1).

get(Ref, Idx) ->
    counters:get(Ref, Idx).

%% Method 3: atomics module (arbitrary integer ops)
-module(atomic_int).

new(InitVal) ->
    Ref = atomics:new(1, []),
    atomics:put(Ref, 1, InitVal),
    Ref.

compare_exchange(Ref, Expected, New) ->
    atomics:compare_exchange(Ref, 1, Expected, New).

fetch_add(Ref, Delta) ->
    atomics:add_get(Ref, 1, Delta) - Delta.
```

---

## 4. Mnesia Schema Design

```erlang
%% mnesia_schema.erl — production Mnesia design
-module(mnesia_schema).
-export([create/1]).

%% Record definitions
-record(session, {
    id,          %% binary, key
    user_id,     %% integer
    data,        %% map
    expires_at   %% integer (unix seconds)
}).

-record(rate_limit, {
    key,         %% {user_id, endpoint}
    count,
    window_start
}).

create(Nodes) ->
    mnesia:create_schema(Nodes),
    rpc:multicall(Nodes, application, start, [mnesia]),

    %% RAM table: fast, lost on restart
    {atomic, ok} = mnesia:create_table(session, [
        {attributes, record_info(fields, session)},
        {ram_copies, Nodes},
        {type, set}
    ]),

    %% Disc table: persistent across restarts
    {atomic, ok} = mnesia:create_table(rate_limit, [
        {attributes, record_info(fields, rate_limit)},
        {disc_copies, Nodes},
        {type, set}
    ]),

    %% Index for user_id lookups
    mnesia:add_table_index(session, user_id),

    ok.
```

---

## 5. Mnesia Queries

```erlang
%% mnesia_queries.erl — reading and writing Mnesia
-module(mnesia_queries).
-export([create_session/3, get_session/1, get_user_sessions/1,
         delete_expired/0]).

create_session(SessionId, UserId, Data) ->
    ExpiresAt = os:system_time(second) + 3600,
    Row = #session{id=SessionId, user_id=UserId,
                   data=Data, expires_at=ExpiresAt},
    mnesia:activity(transaction, fun() ->
        mnesia:write(Row)
    end).

get_session(SessionId) ->
    mnesia:activity(ets, fun() ->
        %% ets context = dirty read, no lock
        case mnesia:read(session, SessionId) of
            [#session{expires_at=E} = S] when E > os:system_time(second) ->
                {ok, S};
            [_] -> expired;
            []  -> not_found
        end
    end).

get_user_sessions(UserId) ->
    mnesia:activity(transaction, fun() ->
        mnesia:index_read(session, UserId, #session.user_id)
    end).

delete_expired() ->
    Now = os:system_time(second),
    mnesia:activity(transaction, fun() ->
        %% qlc: query list comprehension
        Q = qlc:q([S#session.id
                   || S <- mnesia:table(session),
                      S#session.expires_at < Now]),
        Expired = qlc:e(Q),
        [mnesia:delete({session, Id}) || Id <- Expired],
        length(Expired)
    end).

%% Complex query with qlc
sessions_by_user_count() ->
    mnesia:activity(transaction, fun() ->
        Q = qlc:q([{S#session.user_id, S#session.id}
                   || S <- mnesia:table(session)]),
        Sorted = qlc:sort(Q, {order, ascending}),
        Results = qlc:e(Sorted),
        %% Group by user_id
        lists:foldl(fun({UserId, _SessId}, Acc) ->
            maps:update_with(UserId, fun(N) -> N+1 end, 1, Acc)
        end, #{}, Results)
    end).
```

---

## 6. Mnesia Distribution

```erlang
%% mnesia_dist.erl — distributed Mnesia across nodes
-module(mnesia_dist).
-export([add_node/1, remove_node/1, replicate_table/2]).

add_node(NewNode) ->
    %% Must stop Mnesia on new node first
    rpc:call(NewNode, mnesia, stop, []),
    mnesia:change_config(extra_db_nodes, [NewNode]),
    mnesia:start(),
    %% Copy schema to new node
    mnesia:change_table_copy_type(schema, NewNode, disc_copies),
    ok.

remove_node(Node) ->
    [mnesia:del_table_copy(Table, Node)
     || Table <- mnesia:system_info(tables),
        Table =/= schema],
    ok.

replicate_table(Table, Nodes) ->
    [mnesia:add_table_copy(Table, N, disc_copies) || N <- Nodes].

%% Monitoring replication lag
check_consistency(Table) ->
    AllNodes = mnesia:table_info(Table, all_nodes),
    Counts = [{N, rpc:call(N, mnesia, table_info, [Table, size])}
              || N <- AllNodes],
    case lists:usort([C || {_, C} <- Counts]) of
        [_Same] -> {ok, consistent};
        Diffs   -> {error, {inconsistent, Diffs}}
    end.
```

---

## 7. แบบฝึกหัด

1. Benchmark: ETS vs Mnesia vs gen_server for 1M read/write operations
2. Implement `ets_lru_cache` using `ordered_set` and access timestamps
3. สร้าง Mnesia-based distributed config store ที่ replicate ไปทุก node
4. เพิ่ม ets:select-based report: "top 10 users by score in the last hour"

---

## สรุป Part 53

✅ ETS patterns: expiry, CAS, bulk, bounded  
✅ ets:select ด้วย match specifications  
✅ Atomic counters (ets/counters/atomics)  
✅ Mnesia schema design  
✅ Mnesia queries ด้วย QLC  
✅ Distributed Mnesia  

---

*Part 53/100 | [← ก่อนหน้า](../part52/README.md) | [ถัดไป →](../part54/README.md)*
