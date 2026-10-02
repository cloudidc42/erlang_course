# Part 65: Advanced Actor Patterns

> **"The actor model is not just about concurrency — it is about designing systems that think in terms of isolated, communicating entities"**  
> Actor model ไม่ใช่แค่เรื่อง concurrency — แต่คือการออกแบบระบบที่คิดในแง่ของ entities ที่แยกตัวและสื่อสารกัน

---

## สารบัญ

1. [Actor Hierarchies](#1-actor-hierarchies)
2. [Routing Actors](#2-routing-actors)
3. [Pooling Patterns](#3-pooling-patterns)
4. [Backpressure with GenStage](#4-backpressure-with-genstage)
5. [Circuit Breaker Actor](#5-circuit-breaker-actor)
6. [Actor Supervision Trees](#6-actor-supervision-trees)
7. [แบบฝึกหัด](#7-แบบฝึกหัด)

---

## 1. Actor Hierarchies

```erlang
%% actor_tree.erl — building actor hierarchies
-module(actor_tree).

%% Erlang supervisor trees form natural actor hierarchies
%% Root supervisor → sub-supervisors → workers

%% Anti-pattern: flat actor topology
%% flat_sup: [worker1, worker2, ..., worker100]
%%   Problem: one restart policy for all

%% Good pattern: layered hierarchy
%% root_sup
%%   ├── infra_sup (one_for_one)
%%   │   ├── db_pool_worker
%%   │   ├── cache_worker
%%   │   └── metrics_worker
%%   ├── domain_sup (one_for_one)
%%   │   ├── user_registry
%%   │   ├── order_manager
%%   │   └── payment_processor
%%   └── http_sup (one_for_one)
%%       ├── cowboy_listener
%%       └── ws_acceptor

%% Actor communication patterns

%% 1. Request-Reply (synchronous)
request_reply_example(ServerPid, Request) ->
    gen_server:call(ServerPid, Request).

%% 2. Tell (fire-and-forget, asynchronous)
tell_example(ServerPid, Message) ->
    gen_server:cast(ServerPid, Message).

%% 3. Forward (relay to another actor)
-module(relay_actor).
-behaviour(gen_server).

handle_call(Request, From, #{target := Target} = State) ->
    %% Forward to another actor and reply when it responds
    Result = gen_server:call(Target, Request),
    {reply, Result, State}.

%% 4. Scatter-Gather: broadcast and collect
scatter_gather(Workers, Request, Timeout) ->
    Ref   = make_ref(),
    Self  = self(),
    Count = length(Workers),
    %% Scatter
    [spawn_link(fun() ->
        Result = gen_server:call(W, Request),
        Self ! {result, Ref, W, Result}
    end) || W <- Workers],
    %% Gather
    gather(Ref, Count, [], Timeout).

gather(_Ref, 0, Acc, _Timeout) -> Acc;
gather(Ref, Remaining, Acc, Timeout) ->
    receive
        {result, Ref, Worker, Result} ->
            gather(Ref, Remaining - 1, [{Worker, Result} | Acc], Timeout)
    after Timeout ->
        {partial, Acc, Remaining}
    end.
```

---

## 2. Routing Actors

```erlang
%% router_actor.erl — content-based routing
-module(router_actor).
-behaviour(gen_server).
-export([start_link/1, route/2]).
-export([init/1, handle_call/3, handle_cast/2]).

-record(state, {
    routes :: [{predicate(), pid()}]
}).
-type predicate() :: fun((term()) -> boolean()).

start_link(Routes) ->
    gen_server:start_link(?MODULE, Routes, []).

route(Router, Message) ->
    gen_server:cast(Router, {route, Message}).

init(Routes) ->
    {ok, #state{routes=Routes}}.

handle_cast({route, Message}, #state{routes=Routes} = State) ->
    case find_route(Message, Routes) of
        {ok, Target} ->
            gen_server:cast(Target, Message);
        no_match ->
            logger:warning("No route for message: ~p", [Message])
    end,
    {noreply, State}.

handle_call(_, _, State) -> {reply, ok, State}.

find_route(_Message, []) -> no_match;
find_route(Message, [{Pred, Target} | Rest]) ->
    case Pred(Message) of
        true  -> {ok, Target};
        false -> find_route(Message, Rest)
    end.

%% Content-based routing example
example_routes() ->
    HighPriorityWorker = spawn(fun worker_loop/0),
    NormalWorker       = spawn(fun worker_loop/0),
    DeadLetterQueue    = spawn(fun dlq_loop/0),

    [
        {fun(#{priority := high}) -> true; (_) -> false end, HighPriorityWorker},
        {fun(#{type := order})    -> true; (_) -> false end, NormalWorker},
        {fun(_) -> true end, DeadLetterQueue}   %% catch-all
    ].

worker_loop() ->
    receive
        Msg -> logger:info("Processing: ~p", [Msg]), worker_loop()
    end.

dlq_loop() ->
    receive
        Msg -> logger:warning("DLQ: ~p", [Msg]), dlq_loop()
    end.

%% Round-robin router
-module(round_robin_router).
-behaviour(gen_server).
-export([start_link/1, send/2]).
-export([init/1, handle_call/3, handle_cast/2]).

start_link(Workers) ->
    gen_server:start_link(?MODULE, Workers, []).

send(Router, Message) ->
    gen_server:cast(Router, {send, Message}).

init(Workers) ->
    {ok, {queue:from_list(Workers)}}.

handle_cast({send, Message}, {Queue}) ->
    {{value, Worker}, Q1} = queue:out(Queue),
    gen_server:cast(Worker, Message),
    %% Rotate: put worker at back of queue
    {noreply, {queue:in(Worker, Q1)}}.

handle_call(_, _, State) -> {reply, ok, State}.
```

---

## 3. Pooling Patterns

```erlang
%% worker_pool.erl — dynamic worker pool
-module(worker_pool).
-behaviour(gen_server).
-export([start_link/3, checkout/1, checkout/2, checkin/2, run/2]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

-record(state, {
    module   :: module(),
    size     :: pos_integer(),
    workers  :: queue:queue(pid()),
    waiting  :: queue:queue({pid(), reference()}),
    monitors :: #{reference() => pid()}
}).

start_link(Name, Module, Size) ->
    gen_server:start_link({local, Name}, ?MODULE, {Module, Size}, []).

checkout(Pool) -> checkout(Pool, 5000).
checkout(Pool, Timeout) ->
    gen_server:call(Pool, checkout, Timeout).

checkin(Pool, Worker) ->
    gen_server:cast(Pool, {checkin, Worker}).

%% Convenience: borrow, run, return
run(Pool, Fun) ->
    Worker = checkout(Pool),
    try
        Fun(Worker)
    after
        checkin(Pool, Worker)
    end.

init({Module, Size}) ->
    process_flag(trap_exit, true),
    Workers = [start_worker(Module) || _ <- lists:seq(1, Size)],
    {ok, #state{
        module   = Module,
        size     = Size,
        workers  = queue:from_list(Workers),
        waiting  = queue:new(),
        monitors = #{}
    }}.

handle_call(checkout, {From, _} = _Caller, State) ->
    #state{workers=Workers, waiting=Waiting, monitors=Monitors} = State,
    case queue:out(Workers) of
        {{value, Worker}, NewWorkers} ->
            %% Worker available: give it and monitor the borrower
            Ref = monitor(process, From),
            {reply, Worker, State#state{
                workers  = NewWorkers,
                monitors = Monitors#{Ref => Worker}
            }};
        {empty, _} ->
            %% No worker available: queue the caller
            Ref = make_ref(),
            NewWaiting = queue:in({From, Ref}, Waiting),
            {noreply, State#state{waiting = NewWaiting}}
    end.

handle_cast({checkin, Worker}, State) ->
    #state{workers=Workers, waiting=Waiting} = State,
    case queue:out(Waiting) of
        {{value, {WaitPid, _Ref}}, NewWaiting} ->
            %% Give worker directly to next waiter
            WaitPid ! {worker, Worker},
            {noreply, State#state{waiting = NewWaiting}};
        {empty, _} ->
            {noreply, State#state{workers = queue:in(Worker, Workers)}}
    end.

handle_info({'DOWN', Ref, process, _Pid, _Reason},
            #state{monitors=Monitors, workers=Workers} = State) ->
    case maps:take(Ref, Monitors) of
        {Worker, NewMonitors} ->
            %% Borrower died: return worker to pool
            {noreply, State#state{
                workers  = queue:in(Worker, Workers),
                monitors = NewMonitors
            }};
        error ->
            {noreply, State}
    end;
handle_info({'EXIT', Worker, Reason}, #state{module=Mod} = State) ->
    logger:warning("Pool worker ~p died: ~p, restarting", [Worker, Reason]),
    NewWorker = start_worker(Mod),
    #state{workers=Workers} = State,
    {noreply, State#state{workers = queue:in(NewWorker, Workers)}}.

start_worker(Module) ->
    {ok, Pid} = Module:start_link(),
    link(Pid),
    Pid.
```

---

## 4. Backpressure with GenStage

```erlang
%% demand_producer.erl — demand-driven production
-module(demand_producer).
-behaviour(gen_server).
-export([start_link/1, subscribe/2]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

-record(state, {
    source    :: fun((pos_integer()) -> [term()]),
    consumers :: #{pid() => {reference(), non_neg_integer()}},
    buffer    :: queue:queue()
}).

start_link(Source) ->
    gen_server:start_link(?MODULE, Source, []).

subscribe(Producer, ConsumerPid) ->
    gen_server:cast(Producer, {subscribe, ConsumerPid}).

init(Source) ->
    {ok, #state{source=Source, consumers=#{}, buffer=queue:new()}}.

handle_cast({subscribe, Pid}, #state{consumers=Consumers} = State) ->
    Ref = monitor(process, Pid),
    Pid ! {subscribed, self()},
    {noreply, State#state{consumers=Consumers#{Pid => {Ref, 0}}}};

handle_cast({demand, Pid, N}, State) ->
    handle_demand(Pid, N, State).

handle_demand(Pid, N, #state{source=Source, buffer=Buf} = State) ->
    %% Try buffer first, then ask source
    {Delivered, NewBuf} = take_from_buffer(Buf, N, []),
    Remaining = N - length(Delivered),
    {NewItems, FinalItems} = case Remaining > 0 of
        true  ->
            Items = Source(Remaining),
            {Items, Delivered ++ Items};
        false ->
            {[], Delivered}
    end,
    case FinalItems of
        [] -> {noreply, State};
        _  ->
            Pid ! {items, self(), FinalItems},
            RefillBuf = case length(NewItems) > Remaining of
                true  ->
                    Extra = lists:nthtail(Remaining, NewItems),
                    lists:foldl(fun queue:in/2, NewBuf, Extra);
                false -> NewBuf
            end,
            {noreply, State#state{buffer=RefillBuf}}
    end.

take_from_buffer(Buf, 0, Acc)  -> {lists:reverse(Acc), Buf};
take_from_buffer(Buf, N, Acc) ->
    case queue:out(Buf) of
        {{value, Item}, NewBuf} -> take_from_buffer(NewBuf, N-1, [Item | Acc]);
        {empty, _}              -> {lists:reverse(Acc), Buf}
    end.

handle_info({'DOWN', Ref, process, Pid, _},
            #state{consumers=Consumers} = State) ->
    NewConsumers = maps:filter(fun(P, {R, _}) ->
        not (P =:= Pid andalso R =:= Ref)
    end, Consumers),
    {noreply, State#state{consumers=NewConsumers}}.

handle_call(_, _, State) -> {reply, ok, State}.
```

---

## 5. Circuit Breaker Actor

```erlang
%% circuit_breaker.erl — actor-based circuit breaker
-module(circuit_breaker).
-behaviour(gen_statem).
-export([start_link/2, call/3]).
-export([callback_mode/0, init/1, closed/3, open/3, half_open/3]).

-define(FAILURE_THRESHOLD, 5).
-define(RESET_TIMEOUT,     60000).  %% 1 minute

callback_mode() -> state_functions.

start_link(Name, Opts) ->
    gen_statem:start_link({local, Name}, ?MODULE, Opts, []).

call(Name, Fun, Timeout) ->
    gen_statem:call(Name, {call, Fun}, Timeout).

init(Opts) ->
    Data = #{
        failures    => 0,
        threshold   => maps:get(threshold, Opts, ?FAILURE_THRESHOLD),
        reset_after => maps:get(reset_after, Opts, ?RESET_TIMEOUT),
        last_error  => undefined
    },
    {ok, closed, Data}.

%% CLOSED: normal operation
closed({call, From}, {call, Fun}, #{failures := Fails,
                                    threshold := Thresh} = Data) ->
    Result = try Fun()
             catch C:E -> {error, {C, E}}
             end,
    case Result of
        {error, Reason} ->
            NewFails = Fails + 1,
            NewData  = Data#{failures => NewFails, last_error => Reason},
            case NewFails >= Thresh of
                true  ->
                    logger:warning("Circuit opened after ~p failures", [NewFails]),
                    TimerRef = erlang:send_after(
                        maps:get(reset_after, Data), self(), reset),
                    {next_state, open,
                     NewData#{timer => TimerRef},
                     [{reply, From, {error, Reason}}]};
                false ->
                    {keep_state, NewData, [{reply, From, {error, Reason}}]}
            end;
        _ ->
            {keep_state, Data#{failures => 0}, [{reply, From, Result}]}
    end.

%% OPEN: reject all calls
open({call, From}, {call, _Fun}, Data) ->
    {keep_state, Data,
     [{reply, From, {error, circuit_open}}]};
open(info, reset, Data) ->
    logger:info("Circuit half-open: testing..."),
    {next_state, half_open, Data#{failures => 0}}.

%% HALF_OPEN: allow one test call
half_open({call, From}, {call, Fun}, Data) ->
    Result = try Fun()
             catch C:E -> {error, {C, E}}
             end,
    case Result of
        {error, Reason} ->
            %% Test failed: go back to open
            logger:warning("Half-open test failed: ~p", [Reason]),
            TimerRef = erlang:send_after(
                maps:get(reset_after, Data), self(), reset),
            {next_state, open,
             Data#{failures => 1, timer => TimerRef, last_error => Reason},
             [{reply, From, {error, Reason}}]};
        _ ->
            %% Test passed: close circuit
            logger:info("Circuit closed after successful test"),
            {next_state, closed, Data#{failures => 0},
             [{reply, From, Result}]}
    end.
```

---

## 6. Actor Supervision Trees

```erlang
%% advanced_sup.erl — sophisticated supervision strategies
-module(advanced_sup).
-behaviour(supervisor).
-export([start_link/0, init/1]).

start_link() ->
    supervisor:start_link({local, ?MODULE}, ?MODULE, []).

init([]) ->
    %% rest_for_one: if child N fails, restart N and all children after N
    %% Use when later children depend on earlier children
    {ok, {
        #{strategy  => rest_for_one,
          intensity => 5,
          period    => 60},
        [
            %% Database pool must start first
            #{id       => db_pool,
              start    => {db_pool_sup, start_link, []},
              type     => supervisor,
              shutdown => infinity},

            %% Cache depends on DB
            #{id       => cache,
              start    => {cache_server, start_link, []},
              shutdown => 5000},

            %% HTTP depends on cache and DB
            #{id       => http,
              start    => {http_sup, start_link, []},
              type     => supervisor,
              shutdown => infinity}
        ]
    }}.

%% Dynamic supervision: add/remove children at runtime
-module(dynamic_sup).
-behaviour(supervisor).
-export([start_link/0, start_worker/1, stop_worker/1, workers/0, init/1]).

start_link() ->
    supervisor:start_link({local, ?MODULE}, ?MODULE, []).

start_worker(WorkerId) ->
    ChildSpec = #{
        id       => WorkerId,
        start    => {dynamic_worker, start_link, [WorkerId]},
        restart  => temporary,   %% don't restart: caller decides
        shutdown => 5000
    },
    supervisor:start_child(?MODULE, ChildSpec).

stop_worker(WorkerId) ->
    supervisor:terminate_child(?MODULE, WorkerId),
    supervisor:delete_child(?MODULE, WorkerId).

workers() ->
    [{Id, Pid, Type, Mods}
     || {Id, Pid, Type, Mods} <- supervisor:which_children(?MODULE),
        is_pid(Pid)].

init([]) ->
    {ok, {#{strategy => one_for_one, intensity => 0, period => 1}, []}}.
```

---

## 7. แบบฝึกหัด

1. สร้าง `consistent_hash_router`: route ตาม consistent hash ของ key
2. Implement `priority_pool`: worker pool ที่ให้ high-priority requests ก่อน
3. เพิ่ม bulkhead pattern: แยก pool สำหรับ critical vs. non-critical operations
4. เขียน `adaptive_circuit_breaker`: ปรับ threshold อัตโนมัติตาม error rate

---

## สรุป Part 65

✅ Actor hierarchies: layered supervision trees  
✅ Content-based และ round-robin routing actors  
✅ Dynamic worker pool พร้อม checkout/checkin  
✅ Demand-driven backpressure ด้วย producer-consumer  
✅ Circuit breaker ด้วย gen_statem 3 states  
✅ Advanced supervision: rest_for_one, dynamic children  

---

*Part 65/100 | [← ก่อนหน้า](../part64/README.md) | [ถัดไป →](../part66/README.md)*
