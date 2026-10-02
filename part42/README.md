# Part 42: Event Sourcing and CQRS

> **"Instead of storing just the current state, store every change that led to it"**  
> แทนที่จะเก็บแค่ state ปัจจุบัน — เก็บทุกการเปลี่ยนแปลงที่นำไปสู่มัน

---

## สารบัญ

1. [Event Sourcing พื้นฐาน](#1-event-sourcing-พื้นฐาน)
2. [Aggregate Root Pattern](#2-aggregate-root-pattern)
3. [Event Store](#3-event-store)
4. [CQRS: Command and Query Separation](#4-cqrs)
5. [Projections / Read Models](#5-projections--read-models)
6. [Event Replay and Snapshots](#6-event-replay-and-snapshots)
7. [Sagas with Events](#7-sagas-with-events)
8. [แบบฝึกหัด](#8-แบบฝึกหัด)

---

## 1. Event Sourcing พื้นฐาน

```erlang
%% แนวคิด Event Sourcing:
%%
%% Traditional:   State ← UPDATE users SET balance = 100 WHERE id = 1
%%
%% Event Sourced: Events:
%%   {account_opened,   #{user_id=>1, initial=>0}}
%%   {money_deposited,  #{user_id=>1, amount=>150}}
%%   {money_withdrawn,  #{user_id=>1, amount=>50}}
%%   → Current state = replay(events) = balance: 100

%% event.erl — event type definitions
-module(event).

-type event_type() ::
    account_opened |
    money_deposited |
    money_withdrawn |
    account_closed.

-type event() :: #{
    id         := binary(),
    type       := event_type(),
    aggregate_id := binary(),
    version    := pos_integer(),
    data       := map(),
    metadata   := map(),
    created_at := integer()
}.

new(Type, AggregateId, Data) ->
    #{
        id           => generate_id(),
        type         => Type,
        aggregate_id => AggregateId,
        data         => Data,
        metadata     => #{},
        created_at   => os:system_time(millisecond)
    }.

generate_id() ->
    <<I:128>> = crypto:strong_rand_bytes(16),
    list_to_binary(io_lib:format("~32.16.0b", [I])).
```

---

## 2. Aggregate Root Pattern

```erlang
%% bank_account.erl — Account aggregate
-module(bank_account).
-export([open/2, deposit/2, withdraw/2, close/1, apply_event/2]).

%% State
-record(account, {
    id       = undefined,
    owner    = undefined,
    balance  = 0,
    status   = new,   %% new | active | closed
    version  = 0
}).

%% Commands → produce events
open(AccountId, Owner) ->
    E = event:new(account_opened, AccountId,
                  #{account_id => AccountId, owner => Owner, initial_balance => 0}),
    {ok, [E]}.

deposit(#account{status=active, id=Id} = Acc, Amount) when Amount > 0 ->
    E = event:new(money_deposited, Id, #{amount => Amount}),
    {ok, Acc, [E]};
deposit(#account{status=closed}, _) -> {error, account_closed};
deposit(_, Amount) when Amount =< 0 -> {error, invalid_amount}.

withdraw(#account{status=active, id=Id, balance=Bal} = Acc, Amount)
        when Amount > 0, Bal >= Amount ->
    E = event:new(money_withdrawn, Id, #{amount => Amount}),
    {ok, Acc, [E]};
withdraw(#account{balance=Bal}, Amount) when Amount > Bal ->
    {error, insufficient_funds};
withdraw(#account{status=closed}, _) ->
    {error, account_closed}.

close(#account{status=active, id=Id} = Acc) ->
    E = event:new(account_closed, Id, #{}),
    {ok, Acc, [E]}.

%% Apply events to evolve state
apply_event(#account{version=V} = Acc,
            #{type := account_opened, data := #{owner := O}}) ->
    Acc#account{owner=O, status=active, version=V+1};

apply_event(#account{balance=Bal, version=V} = Acc,
            #{type := money_deposited, data := #{amount := A}}) ->
    Acc#account{balance=Bal+A, version=V+1};

apply_event(#account{balance=Bal, version=V} = Acc,
            #{type := money_withdrawn, data := #{amount := A}}) ->
    Acc#account{balance=Bal-A, version=V+1};

apply_event(#account{version=V} = Acc, #{type := account_closed}) ->
    Acc#account{status=closed, version=V+1}.

%% Rebuild state from events
rebuild(Events) ->
    lists:foldl(fun apply_event/2, #account{}, Events).
```

---

## 3. Event Store

```erlang
%% event_store.erl — append-only event storage
-module(event_store).
-behaviour(gen_server).
-export([start_link/0, append/3, load/1, load_from/2, subscribe/2]).
-export([init/1, handle_call/3, handle_cast/2]).

-define(EVENTS_TABLE, es_events).
-define(SUBS_TABLE,   es_subscriptions).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

%% Append events for an aggregate, with optimistic concurrency check
append(AggregateId, ExpectedVersion, Events) ->
    gen_server:call(?MODULE, {append, AggregateId, ExpectedVersion, Events}).

load(AggregateId) ->
    load_from(AggregateId, 0).

load_from(AggregateId, FromVersion) ->
    All = ets:match_object(?EVENTS_TABLE,
                           {AggregateId, '$1', '_'}),
    Sorted = lists:sort(fun({_, V1, _}, {_, V2, _}) -> V1 < V2 end, All),
    Events = [E || {_, V, E} <- Sorted, V >= FromVersion],
    {ok, Events}.

subscribe(Type, Subscriber) ->
    gen_server:call(?MODULE, {subscribe, Type, Subscriber}).

init([]) ->
    ets:new(?EVENTS_TABLE, [named_table, bag, protected]),
    ets:new(?SUBS_TABLE,   [named_table, set, protected]),
    {ok, #{versions => #{}}}.

handle_call({append, AggId, ExpectedVsn, Events}, _From,
            #{versions := Versions} = State) ->
    CurrentVsn = maps:get(AggId, Versions, 0),
    case CurrentVsn =:= ExpectedVsn of
        false ->
            {reply, {error, {version_conflict, CurrentVsn}}, State};
        true ->
            NewVsn = ExpectedVsn + length(Events),
            Versioned = lists:zip(
                lists:seq(ExpectedVsn+1, NewVsn),
                Events),
            [ets:insert(?EVENTS_TABLE, {AggId, V, E#{version => V}})
             || {V, E} <- Versioned],
            notify_subscribers(Events),
            {reply, ok, State#{versions := maps:put(AggId, NewVsn, Versions)}}
    end;

handle_call({subscribe, Type, Pid}, _From, State) ->
    Existing = case ets:lookup(?SUBS_TABLE, Type) of
        [{Type, Subs}] -> Subs;
        [] -> []
    end,
    ets:insert(?SUBS_TABLE, {Type, [Pid | Existing]}),
    {reply, ok, State}.

handle_cast(_, S) -> {noreply, S}.

notify_subscribers(Events) ->
    lists:foreach(fun(#{type := Type} = Event) ->
        case ets:lookup(?SUBS_TABLE, Type) of
            [{Type, Subs}] ->
                [Pid ! {event, Event} || Pid <- Subs,
                                        is_process_alive(Pid)];
            [] -> ok
        end
    end, Events).
```

---

## 4. CQRS

```erlang
%% command_bus.erl — route commands to handlers
-module(command_bus).
-export([dispatch/1]).

dispatch(#{type := open_account, data := #{account_id := Id, owner := Owner}}) ->
    {ok, Events} = bank_account:open(Id, Owner),
    save_and_publish(Id, 0, Events);

dispatch(#{type := deposit, data := #{account_id := Id, amount := A}}) ->
    {ok, Account} = load_account(Id),
    case bank_account:deposit(Account, A) of
        {ok, _, Events} ->
            save_and_publish(Id, Account#account.version, Events);
        {error, _} = Err ->
            Err
    end;

dispatch(#{type := withdraw, data := #{account_id := Id, amount := A}}) ->
    {ok, Account} = load_account(Id),
    case bank_account:withdraw(Account, A) of
        {ok, _, Events} ->
            save_and_publish(Id, Account#account.version, Events);
        {error, _} = Err ->
            Err
    end.

load_account(Id) ->
    {ok, Events} = event_store:load(Id),
    case Events of
        [] -> {error, not_found};
        _  -> {ok, bank_account:rebuild(Events)}
    end.

save_and_publish(AggId, ExpectedVsn, Events) ->
    case event_store:append(AggId, ExpectedVsn, Events) of
        ok -> {ok, Events};
        {error, _} = E -> E
    end.
```

---

## 5. Projections / Read Models

```erlang
%% account_projection.erl — build query-optimized view from events
-module(account_projection).
-behaviour(gen_server).
-export([start_link/0, get_balance/1, get_all/0]).
-export([init/1, handle_info/2, handle_call/3, handle_cast/2]).

-define(READ_MODEL, account_read_model).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

get_balance(AccountId) ->
    case ets:lookup(?READ_MODEL, AccountId) of
        [{AccountId, Data}] -> {ok, Data};
        [] -> {error, not_found}
    end.

get_all() ->
    ets:tab2list(?READ_MODEL).

init([]) ->
    ets:new(?READ_MODEL, [named_table, set, protected]),
    %% Subscribe to all account events
    event_store:subscribe(account_opened,  self()),
    event_store:subscribe(money_deposited, self()),
    event_store:subscribe(money_withdrawn, self()),
    event_store:subscribe(account_closed,  self()),
    %% Rebuild from existing events
    rebuild_all(),
    {ok, #{}}.

handle_info({event, Event}, State) ->
    handle_event(Event),
    {noreply, State};
handle_info(_, S) -> {noreply, S}.

handle_call(_, _, S) -> {reply, ok, S}.
handle_cast(_, S)    -> {noreply, S}.

handle_event(#{type := account_opened, aggregate_id := Id,
               data := #{owner := Owner}}) ->
    ets:insert(?READ_MODEL, {Id, #{id=>Id, owner=>Owner, balance=>0, status=>active}});

handle_event(#{type := money_deposited, aggregate_id := Id,
               data := #{amount := A}}) ->
    case ets:lookup(?READ_MODEL, Id) of
        [{Id, D}] -> ets:insert(?READ_MODEL, {Id, D#{balance => maps:get(balance, D) + A}});
        [] -> ok
    end;

handle_event(#{type := money_withdrawn, aggregate_id := Id,
               data := #{amount := A}}) ->
    case ets:lookup(?READ_MODEL, Id) of
        [{Id, D}] -> ets:insert(?READ_MODEL, {Id, D#{balance => maps:get(balance, D) - A}});
        [] -> ok
    end;

handle_event(#{type := account_closed, aggregate_id := Id}) ->
    case ets:lookup(?READ_MODEL, Id) of
        [{Id, D}] -> ets:insert(?READ_MODEL, {Id, D#{status => closed}});
        [] -> ok
    end.

rebuild_all() ->
    %% In practice: scan event store, apply to projection
    ok.
```

---

## 6. Event Replay and Snapshots

```erlang
%% snapshot_store.erl — avoid replaying all events every time
-module(snapshot_store).
-export([save/3, load/1]).

-define(SNAP_TABLE, snapshots).

save(AggregateId, Version, State) ->
    ets:insert(?SNAP_TABLE,
               {AggregateId, #{version => Version, state => State,
                               saved_at => os:system_time(second)}}),
    ok.

load(AggregateId) ->
    case ets:lookup(?SNAP_TABLE, AggregateId) of
        [{_, Snapshot}] -> {ok, Snapshot};
        [] -> {error, not_found}
    end.

%% Using snapshots to load aggregates efficiently
load_with_snapshot(AggregateId) ->
    {BaseVersion, BaseState} = case snapshot_store:load(AggregateId) of
        {ok, #{version := V, state := S}} -> {V, S};
        {error, not_found} -> {0, #account{}}
    end,
    {ok, Events} = event_store:load_from(AggregateId, BaseVersion),
    FinalState = lists:foldl(fun bank_account:apply_event/2, BaseState, Events),
    {ok, FinalState}.

%% Auto-snapshot every N events
maybe_snapshot(AggregateId, Account, Events) ->
    #account{version = V} = Account,
    case V rem 50 of
        0 -> snapshot_store:save(AggregateId, V, Account);
        _ -> ok
    end.
```

---

## 7. Sagas with Events

```erlang
%% order_saga.erl — distributed transaction using events
-module(order_saga).
-behaviour(gen_statem).
-export([start/1]).
-export([callback_mode/0, init/1,
         pending/3, reserving/3, paying/3, confirming/3, failed/3]).

callback_mode() -> state_functions.

-record(saga, {order_id, items, payment, reservations=[]}).

start(Order) ->
    gen_statem:start(?MODULE, Order, []).

init(#{id:=Id, items:=Items, payment:=P}) ->
    Data = #saga{order_id=Id, items=Items, payment=P},
    self() ! start,
    {ok, pending, Data}.

pending(info, start, #saga{items=Items} = Data) ->
    %% Step 1: Reserve all items
    [inventory:reserve(Item) || Item <- Items],
    {next_state, reserving, Data, [{state_timeout, 30000, timeout}]}.

reserving(info, {reserved, ItemId}, #saga{items=Items, reservations=R} = Data) ->
    NewR = [ItemId | R],
    case length(NewR) =:= length(Items) of
        true ->
            %% All reserved, charge payment
            payment:charge(Data#saga.payment),
            {next_state, paying, Data#saga{reservations=NewR},
             [{state_timeout, 30000, timeout}]};
        false ->
            {keep_state, Data#saga{reservations=NewR}}
    end;

reserving(state_timeout, timeout, Data) ->
    compensate_reservations(Data),
    {next_state, failed, Data}.

paying(info, {charged, _Ref}, Data) ->
    %% Step 3: Confirm order
    order_db:confirm(Data#saga.order_id),
    {next_state, confirming, Data};

paying(info, {charge_failed, Reason}, Data) ->
    compensate_reservations(Data),
    {next_state, failed, Data#{reason => Reason}};

paying(state_timeout, timeout, Data) ->
    compensate_reservations(Data),
    {next_state, failed, Data}.

confirming(info, confirmed, Data) ->
    %% Saga complete
    {stop, normal, Data}.

failed(info, _, Data) -> {keep_state, Data}.

compensate_reservations(#saga{reservations=R}) ->
    [inventory:release(ItemId) || ItemId <- R].
```

---

## 8. แบบฝึกหัด

1. เพิ่ม `transfer` command ใน bank_account: debit จาก account A, credit ไปยัง account B (ต้อง atomic)
2. สร้าง `balance_history` projection ที่เก็บ history ของ balance เปลี่ยนแปลง
3. เขียน property test: `rebuild(events) = apply all events sequentially`
4. เพิ่ม snapshot ทุก 100 events และวัด load time ก่อน/หลัง

---

## สรุป Part 42

✅ Event Sourcing concept  
✅ Aggregate Root (bank_account)  
✅ Event Store ด้วย ETS  
✅ CQRS: command_bus  
✅ Projections/Read models  
✅ Snapshots  
✅ Sagas ด้วย gen_statem  

---

*Part 42/100 | [← ก่อนหน้า](../part41/README.md) | [ถัดไป →](../part43/README.md)*
