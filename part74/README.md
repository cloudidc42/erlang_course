# Part 74: CQRS and Event Sourcing at Scale

> **"The past is immutable; only the present can be changed"**  
> อดีตเปลี่ยนแปลงไม่ได้ — มีเพียงปัจจุบันเท่านั้นที่เปลี่ยนได้

---

## สารบัญ

1. [Event Sourcing Fundamentals](#1-event-sourcing-fundamentals)
2. [CQRS Write Side](#2-cqrs-write-side)
3. [CQRS Read Side — Projections](#3-cqrs-read-side--projections)
4. [Event Store Implementation](#4-event-store-implementation)
5. [Sagas and Process Managers](#5-sagas-and-process-managers)
6. [Snapshots for Performance](#6-snapshots-for-performance)
7. [แบบฝึกหัด](#7-แบบฝึกหัด)

---

## 1. Event Sourcing Fundamentals

```
Event Sourcing: Store EVENTS, not STATE
════════════════════════════════════════════════════════

Traditional approach:
  DB row: {id: 1, balance: 950, status: active}

Event sourced:
  Event 1: AccountOpened    {id: 1, initial_balance: 1000}
  Event 2: MoneyDeposited   {id: 1, amount: 200}
  Event 3: MoneyWithdrawn   {id: 1, amount: 250}
  Current state = fold(events, initial)  →  balance: 950

BENEFITS:
  ✓ Complete audit trail (regulatory compliance)
  ✓ Temporal queries: "what was balance on 2024-01-01?"
  ✓ Replay: fix bugs by replaying events with new logic
  ✓ Projections: derive any read model from events
  ✓ Event-driven integration: emit domain events naturally

TRADE-OFFS:
  ✗ More complex than CRUD
  ✗ Read performance needs projections
  ✗ Schema evolution requires versioning
  ✗ Eventual consistency in read models
```

---

## 2. CQRS Write Side

```erlang
%% account_aggregate.erl — event-sourced bank account aggregate
-module(account_aggregate).
-behaviour(gen_server).

-export([start_link/1, open_account/2, deposit/3, withdraw/3,
         get_balance/1, get_events/1]).
-export([init/1, handle_call/3, handle_cast/2]).

-record(state, {
    id          :: binary(),
    balance     = 0       :: integer(),
    status      = new     :: new | active | closed,
    version     = 0       :: integer(),
    pending_events = []   :: list()
}).

start_link(AccountId) ->
    gen_server:start_link(?MODULE, AccountId, []).

open_account(Pid, OwnerId) ->
    gen_server:call(Pid, {command, {open_account, OwnerId}}).

deposit(Pid, Amount, Ref) ->
    gen_server:call(Pid, {command, {deposit, Amount, Ref}}).

withdraw(Pid, Amount, Ref) ->
    gen_server:call(Pid, {command, {withdraw, Amount, Ref}}).

get_balance(Pid) ->
    gen_server:call(Pid, get_balance).

get_events(Pid) ->
    gen_server:call(Pid, get_events).

init(AccountId) ->
    %% Load historical events from store and rebuild state
    HistoricalEvents = event_store:load_events(AccountId),
    State = lists:foldl(fun apply_event/2, #state{id = AccountId},
                        HistoricalEvents),
    {ok, State}.

handle_call({command, Command}, _From, State) ->
    case execute_command(Command, State) of
        {ok, Events} ->
            NewState = lists:foldl(fun apply_event/2, State, Events),
            %% Persist events
            ok = event_store:append_events(State#state.id, Events,
                                           State#state.version),
            %% Publish to event bus
            lists:foreach(fun(E) -> event_bus:publish(E) end, Events),
            NewState2 = NewState#state{
                pending_events = [],
                version = State#state.version + length(Events)
            },
            {reply, ok, NewState2};
        {error, _} = Err ->
            {reply, Err, State}
    end;

handle_call(get_balance, _From, State) ->
    {reply, State#state.balance, State};

handle_call(get_events, _From, State) ->
    Events = event_store:load_events(State#state.id),
    {reply, Events, State}.

%% ─────────────────────────────────────────────
%%  Command handlers — validate and produce events
%% ─────────────────────────────────────────────
execute_command({open_account, OwnerId}, #state{status = new} = S) ->
    Event = #{type => account_opened,
              account_id => S#state.id,
              owner_id   => OwnerId,
              timestamp  => timestamp()},
    {ok, [Event]};

execute_command({open_account, _}, _State) ->
    {error, account_already_open};

execute_command({deposit, Amount, Ref}, #state{status = active})
  when Amount > 0 ->
    Event = #{type => money_deposited, amount => Amount, ref => Ref,
              timestamp => timestamp()},
    {ok, [Event]};

execute_command({deposit, Amount, _}, _State) when Amount =< 0 ->
    {error, invalid_amount};

execute_command({withdraw, Amount, Ref},
                #state{status = active, balance = Balance})
  when Amount > 0, Balance >= Amount ->
    Event = #{type => money_withdrawn, amount => Amount, ref => Ref,
              timestamp => timestamp()},
    {ok, [Event]};

execute_command({withdraw, Amount, _}, #state{balance = Balance})
  when Amount > Balance ->
    {error, insufficient_funds};

execute_command(_, _) ->
    {error, invalid_command}.

%% ─────────────────────────────────────────────
%%  Event applicators — mutate state
%% ─────────────────────────────────────────────
apply_event(#{type := account_opened}, State) ->
    State#state{status = active};

apply_event(#{type := money_deposited, amount := Amount}, State) ->
    State#state{balance = State#state.balance + Amount};

apply_event(#{type := money_withdrawn, amount := Amount}, State) ->
    State#state{balance = State#state.balance - Amount};

apply_event(_, State) -> State.

timestamp() -> erlang:system_time(millisecond).
```

---

## 3. CQRS Read Side — Projections

```erlang
%% account_projection.erl — builds read model from events
-module(account_projection).
-behaviour(gen_server).

-export([start_link/0, handle_event/1, get_account/1, list_accounts/0]).
-export([init/1, handle_call/3, handle_cast/2]).

%% Read model stored in ETS — optimized for queries
start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

get_account(Id) ->
    gen_server:call(?MODULE, {get, Id}).

list_accounts() ->
    gen_server:call(?MODULE, list).

handle_event(Event) ->
    gen_server:cast(?MODULE, {event, Event}).

init([]) ->
    ets:new(accounts_read, [named_table, public, {read_concurrency, true}]),
    %% Subscribe to event bus
    event_bus:subscribe(account_events, ?MODULE),
    %% Rebuild from all historical events
    rebuild_from_store(),
    {ok, #{}}.

handle_cast({event, Event}, State) ->
    project_event(Event),
    {noreply, State}.

handle_call({get, Id}, _From, State) ->
    Result = case ets:lookup(accounts_read, Id) of
        [{_, Account}] -> {ok, Account};
        []             -> {error, not_found}
    end,
    {reply, Result, State};

handle_call(list, _From, State) ->
    Accounts = [A || {_, A} <- ets:tab2list(accounts_read)],
    {reply, Accounts, State}.

project_event(#{type := account_opened,
                account_id := Id, owner_id := OwnerId}) ->
    Account = #{id => Id, owner_id => OwnerId,
                balance => 0, status => active,
                created_at => timestamp()},
    ets:insert(accounts_read, {Id, Account});

project_event(#{type := money_deposited,
                account_id := Id, amount := Amount}) ->
    update_account(Id, fun(A) -> maps:update_with(balance,
                                    fun(B) -> B + Amount end, A) end);

project_event(#{type := money_withdrawn,
                account_id := Id, amount := Amount}) ->
    update_account(Id, fun(A) -> maps:update_with(balance,
                                    fun(B) -> B - Amount end, A) end);

project_event(_) -> ok.

update_account(Id, Fun) ->
    case ets:lookup(accounts_read, Id) of
        [{_, Account}] -> ets:insert(accounts_read, {Id, Fun(Account)});
        []             -> ok
    end.

rebuild_from_store() ->
    Events = event_store:load_all_events(account_events),
    lists:foreach(fun project_event/1, Events).

timestamp() -> erlang:system_time(millisecond).
```

---

## 4. Event Store Implementation

```erlang
%% event_store.erl — append-only event store backed by PostgreSQL
-module(event_store).
-export([append_events/3, load_events/1, load_all_events/1]).

%% Optimistic concurrency: expectedVersion must match current
append_events(AggregateId, Events, ExpectedVersion) ->
    Sql = "
        INSERT INTO events (aggregate_id, event_type, data, version, created_at)
        SELECT $1, $2, $3::jsonb, $4 + ROW_NUMBER() OVER (), NOW()
        FROM UNNEST($5::text[], $6::jsonb[]) AS t(type, data)
        WHERE (SELECT COALESCE(MAX(version), 0) FROM events
               WHERE aggregate_id = $1) = $4",

    Types = [maps:get(type, E) || E <- Events],
    Datas = [json:encode(maps:without([type], E)) || E <- Events],

    case db:execute(Sql, [AggregateId, Types, Datas, ExpectedVersion,
                          Types, Datas]) of
        {ok, _} -> ok;
        {error, {unique_violation, _}} ->
            {error, optimistic_concurrency_conflict}
    end.

load_events(AggregateId) ->
    Sql = "SELECT event_type, data, version FROM events
           WHERE aggregate_id = $1
           ORDER BY version ASC",
    case db:query(Sql, [AggregateId]) of
        {ok, Rows} ->
            [decode_event(Type, Data, Version)
             || {Type, Data, Version} <- Rows];
        {error, _} = Err -> Err
    end.

load_all_events(Category) ->
    Sql = "SELECT event_type, data, version FROM events
           WHERE event_type LIKE $1 || '%'
           ORDER BY created_at ASC, version ASC",
    case db:query(Sql, [atom_to_binary(Category)]) of
        {ok, Rows} ->
            [decode_event(T, D, V) || {T, D, V} <- Rows];
        {error, _} = Err -> Err
    end.

decode_event(Type, DataJson, Version) ->
    Data = json:decode(DataJson),
    Data#{type => binary_to_existing_atom(Type), version => Version}.
```

```sql
-- events table schema
CREATE TABLE events (
    id           BIGSERIAL PRIMARY KEY,
    aggregate_id VARCHAR(255) NOT NULL,
    event_type   VARCHAR(255) NOT NULL,
    data         JSONB NOT NULL,
    version      INTEGER NOT NULL,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (aggregate_id, version)
);

CREATE INDEX idx_events_aggregate ON events(aggregate_id, version);
CREATE INDEX idx_events_type ON events(event_type);
CREATE INDEX idx_events_created ON events(created_at);
```

---

## 5. Sagas and Process Managers

```erlang
%% order_saga.erl — orchestration saga for order fulfillment
%% Manages multi-step process with compensating transactions
-module(order_saga).
-behaviour(gen_statem).

-export([start_link/2, callback_mode/0, init/1]).
-export([awaiting_payment/3, reserving_inventory/3,
         shipping/3, completed/3, compensating/3]).

-record(saga, {
    order_id    :: binary(),
    order_data  :: map(),
    payment_id  :: binary() | undefined,
    reservation_id :: binary() | undefined,
    shipment_id :: binary() | undefined,
    error       :: term() | undefined
}).

start_link(OrderId, OrderData) ->
    gen_statem:start_link(?MODULE, [OrderId, OrderData], []).

callback_mode() -> [state_functions, state_enter].

init([OrderId, OrderData]) ->
    Data = #saga{order_id = OrderId, order_data = OrderData},
    {ok, awaiting_payment, Data}.

%% Step 1: Charge payment
awaiting_payment(enter, _, Data) ->
    payment_service:charge_async(Data#saga.order_id,
                                  maps:get(amount, Data#saga.order_data),
                                  self()),
    {keep_state, Data, [{state_timeout, 30000, timeout}]};

awaiting_payment(cast, {payment_success, PaymentId}, Data) ->
    {next_state, reserving_inventory, Data#saga{payment_id = PaymentId}};

awaiting_payment(cast, {payment_failed, Reason}, Data) ->
    {next_state, compensating, Data#saga{error = {payment_failed, Reason}}};

awaiting_payment(state_timeout, timeout, Data) ->
    {next_state, compensating, Data#saga{error = payment_timeout}}.

%% Step 2: Reserve inventory
reserving_inventory(enter, _, Data) ->
    Items = maps:get(items, Data#saga.order_data),
    inventory_service:reserve_async(Data#saga.order_id, Items, self()),
    {keep_state, Data, [{state_timeout, 15000, timeout}]};

reserving_inventory(cast, {reserved, ReservationId}, Data) ->
    {next_state, shipping, Data#saga{reservation_id = ReservationId}};

reserving_inventory(cast, {reservation_failed, Reason}, Data) ->
    {next_state, compensating,
     Data#saga{error = {reservation_failed, Reason}}};

reserving_inventory(state_timeout, timeout, Data) ->
    {next_state, compensating, Data#saga{error = reservation_timeout}}.

%% Step 3: Create shipment
shipping(enter, _, Data) ->
    Address = maps:get(address, Data#saga.order_data),
    shipping_service:create_shipment_async(Data#saga.order_id, Address, self()),
    {keep_state, Data, [{state_timeout, 10000, timeout}]};

shipping(cast, {shipment_created, ShipmentId}, Data) ->
    {next_state, completed, Data#saga{shipment_id = ShipmentId}};

shipping(cast, {shipment_failed, Reason}, Data) ->
    {next_state, compensating,
     Data#saga{error = {shipment_failed, Reason}}};

shipping(state_timeout, timeout, Data) ->
    {next_state, compensating, Data#saga{error = shipment_timeout}}.

%% Success
completed(enter, _, Data) ->
    logger:info("Order saga completed", #{order_id => Data#saga.order_id}),
    order_events:publish(order_completed, Data#saga.order_id,
                         #{shipment_id => Data#saga.shipment_id}),
    {keep_state, Data}.

%% Compensation: undo completed steps in reverse order
compensating(enter, _, Data) ->
    logger:warning("Saga compensating", #{order_id => Data#saga.order_id,
                                          error => Data#saga.error}),
    %% Undo in reverse order
    maybe_cancel_reservation(Data),
    maybe_refund_payment(Data),
    order_events:publish(order_failed, Data#saga.order_id,
                         #{error => Data#saga.error}),
    {keep_state, Data};

compensating(cast, _, Data) ->
    {keep_state, Data}.

maybe_cancel_reservation(#saga{reservation_id = undefined}) -> ok;
maybe_cancel_reservation(#saga{reservation_id = Rid}) ->
    inventory_service:cancel_reservation(Rid).

maybe_refund_payment(#saga{payment_id = undefined}) -> ok;
maybe_refund_payment(#saga{payment_id = Pid}) ->
    payment_service:refund(Pid).
```

---

## 6. Snapshots for Performance

```erlang
%% snapshot_store.erl — cache aggregate state for fast loading
-module(snapshot_store).
-export([save_snapshot/3, load_snapshot/1, should_snapshot/2]).

-define(SNAPSHOT_INTERVAL, 50).  % Every 50 events

should_snapshot(Version, _AggregateId) ->
    Version rem ?SNAPSHOT_INTERVAL =:= 0.

save_snapshot(AggregateId, Version, State) ->
    Sql = "INSERT INTO snapshots (aggregate_id, version, state, created_at)
           VALUES ($1, $2, $3::jsonb, NOW())
           ON CONFLICT (aggregate_id)
           DO UPDATE SET version = $2, state = $3::jsonb, created_at = NOW()",
    db:execute(Sql, [AggregateId, Version, json:encode(State)]).

load_snapshot(AggregateId) ->
    Sql = "SELECT version, state FROM snapshots
           WHERE aggregate_id = $1",
    case db:query(Sql, [AggregateId]) of
        {ok, [{Version, StateJson}]} ->
            {ok, Version, json:decode(StateJson)};
        {ok, []} ->
            {error, no_snapshot}
    end.

%% Modified init to use snapshots
init_with_snapshot(AggregateId) ->
    {StartVersion, InitialState} =
        case snapshot_store:load_snapshot(AggregateId) of
            {ok, Ver, State} ->
                {Ver, State};
            {error, no_snapshot} ->
                {0, #state{id = AggregateId}}
        end,
    %% Load only events AFTER the snapshot
    Events = event_store:load_events_from(AggregateId, StartVersion),
    FinalState = lists:foldl(fun apply_event/2, InitialState, Events),
    FinalState.
```

---

## 7. แบบฝึกหัด

1. เพิ่ม optimistic concurrency check ที่แท้จริงใน `event_store:append_events`
2. สร้าง second projection ที่ track transaction history per account
3. เขียน saga ที่ handle timeout แต่ละ step แตกต่างกัน
4. Implement event schema versioning: เมื่อ event format เปลี่ยน

---

## สรุป Part 74

✅ Event sourcing: store events not state, fold to rebuild  
✅ CQRS write side: command validation, event generation, optimistic concurrency  
✅ CQRS read side: projections rebuild read models from events  
✅ Event store: append-only with unique (aggregate_id, version)  
✅ Sagas: multi-step process with gen_statem + compensating transactions  
✅ Snapshots: checkpoint state every N events for fast load  

---

*Part 74/100 | [← ก่อนหน้า](../part73/README.md) | [ถัดไป →](../part75/README.md)*
