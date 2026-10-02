# Part 31: Advanced OTP Patterns

> **"Master OTP patterns and you master Erlang's power"**  
> เชี่ยวชาญ OTP patterns = เชี่ยวชาญพลังของ Erlang

---

## สารบัญ

1. [Process Registry Patterns](#1-process-registry-patterns)
2. [Dynamic Supervisor Patterns](#2-dynamic-supervisor-patterns)
3. [GenServer Best Practices](#3-genserver-best-practices)
4. [Callback Timeout Patterns](#4-callback-timeout-patterns)
5. [Process Hibernation](#5-process-hibernation)
6. [Backpressure Pattern](#6-backpressure-pattern)
7. [Circuit Breaker Pattern](#7-circuit-breaker-pattern)
8. [Saga Pattern](#8-saga-pattern)
9. [แบบฝึกหัด](#9-แบบฝึกหัด)

---

## 1. Process Registry Patterns

```erlang
%% Pattern 1: ชื่อแบบ static — ใช้สำหรับ singletons
gen_server:start_link({local, my_server}, ...).
gen_server:call(my_server, request).

%% Pattern 2: gproc — dynamic process registry
%% deps: {gproc, "0.9.0"}
gproc:reg({n, l, {user_session, UserId}}).
Pid = gproc:lookup_pid({n, l, {user_session, UserId}}).

%% Pattern 3: pg — process groups (OTP 23+)
pg:join(user_sessions, self()).
Members = pg:get_members(user_sessions).

%% Pattern 4: custom registry ด้วย ETS
-module(proc_registry).
-export([register/2, lookup/1, unregister/1]).

register(Name, Pid) ->
    ets:insert(proc_registry, {Name, Pid}),
    monitor(process, Pid),
    ok.

lookup(Name) ->
    case ets:lookup(proc_registry, Name) of
        [{Name, Pid}] -> {ok, Pid};
        []            -> {error, not_found}
    end.

unregister(Name) ->
    ets:delete(proc_registry, Name).

%% Pattern 5: via tuple — ใช้ GenServer ตรงๆ
gen_server:start_link({via, gproc, {n, l, my_key}}, Mod, Args, Opts).
gen_server:call({via, gproc, {n, l, my_key}}, Request).
```

---

## 2. Dynamic Supervisor Patterns

```erlang
%% Pattern: Start workers on-demand
-module(user_session_sup).
-behaviour(supervisor).

-export([start_link/0, start_session/1, stop_session/1]).
-export([init/1]).

start_link() ->
    supervisor:start_link({local, ?MODULE}, ?MODULE, []).

start_session(UserId) ->
    supervisor:start_child(?MODULE, [UserId]).

stop_session(UserId) ->
    case gproc:lookup_pid({n, l, {session, UserId}}) of
        Pid -> supervisor:terminate_child(?MODULE, Pid);
        _   -> {error, not_found}
    end.

init([]) ->
    ChildSpec = #{
        id       => user_session,
        start    => {user_session, start_link, []},
        restart  => temporary,  %% ไม่ restart อัตโนมัติ
        shutdown => 5000,
        type     => worker
    },
    {ok, {{simple_one_for_one, 0, 1}, [ChildSpec]}}.

%% user_session.erl
-module(user_session).
-behaviour(gen_server).

start_link(UserId) ->
    gen_server:start_link(?MODULE, [UserId], []).

init([UserId]) ->
    gproc:reg({n, l, {session, UserId}}),
    {ok, #{user_id => UserId, data => #{}}}.
```

---

## 3. GenServer Best Practices

```erlang
%% 1. ใช้ handle_continue สำหรับ init ที่ต้องทำงานหนัก
init(Args) ->
    %% อย่า block ใน init นานเกิน
    {ok, initial_state(Args), {continue, load_data}}.

handle_continue(load_data, State) ->
    Data = load_from_db(),
    {noreply, State#{data => Data}}.

%% 2. ให้ callbacks เร็วเสมอ — อย่า block
%% BAD:
handle_call(request, _From, State) ->
    Result = slow_external_call(),  %% blocks everyone!
    {reply, Result, State}.

%% GOOD: ทำงานใน separate process
handle_call(request, From, State) ->
    spawn_link(fun() ->
        Result = slow_external_call(),
        gen_server:reply(From, Result)
    end),
    {noreply, State}.

%% 3. ใช้ handle_info สำหรับ monitor DOWN messages
handle_info({'DOWN', Ref, process, Pid, Reason}, State) ->
    NewState = cleanup_dead_process(Pid, Reason, State),
    {noreply, NewState};

%% 4. ใช้ State record แทน map สำหรับ performance ที่ดีกว่า
-record(state, {
    db_conn :: pid(),
    cache   :: map(),
    config  :: map()
}).

%% 5. Log important state changes
handle_cast({update, Key, Value}, State) ->
    logger:debug("Updating ~p = ~p", [Key, Value]),
    {noreply, State#{Key => Value}}.
```

---

## 4. Callback Timeout Patterns

```erlang
%% Timeout ใน GenServer callbacks

%% 1. Simple timeout: เรียก handle_info(timeout, State) หลัง N ms
handle_call(request, _From, State) ->
    {reply, ok, State, 5000}.  %% timeout หลัง 5s

handle_info(timeout, State) ->
    %% ทำ cleanup หรือ refresh
    {noreply, do_refresh(State)}.

%% 2. {continue, Action} — ทำต่อหลัง reply
handle_call(get_data, _From, State) ->
    {reply, State#state.data, State, {continue, refresh}}.

handle_continue(refresh, State) ->
    {noreply, do_refresh(State)}.

%% 3. hibernate — ลด memory เมื่อ idle นาน
handle_call(request, _From, State) ->
    {reply, ok, State, hibernate}.  %% hibernate after reply

handle_info(timeout, State) ->
    {noreply, State, hibernate}.  %% hibernate after timeout

%% 4. erlang:send_after สำหรับ repeating work
init([]) ->
    erlang:send_after(60000, self(), refresh_cache),
    {ok, #{}}.

handle_info(refresh_cache, State) ->
    NewState = do_refresh_cache(State),
    erlang:send_after(60000, self(), refresh_cache),
    {noreply, NewState}.

%% 5. cancel timer
init([]) ->
    Ref = erlang:send_after(5000, self(), check),
    {ok, #{timer => Ref}}.

handle_call(cancel_check, _From, #{timer := Ref} = State) ->
    erlang:cancel_timer(Ref),
    {reply, ok, maps:remove(timer, State)}.
```

---

## 5. Process Hibernation

```erlang
%% Hibernate reduces memory when process is idle
%% เมื่อ hibernate: GC runs, stack กลับไป minimum

%% ใช้เมื่อ:
%% - Process idle นาน (>= 1 min)
%% - Memory usage สำคัญ
%% - Process มี state ใหญ่

%% Automatic hibernation ด้วย timeout
handle_info(timeout, State) ->
    %% Hibernate until next message
    {noreply, State, hibernate}.

%% Manual hibernation
hibernate_if_idle(State) ->
    case has_pending_work(State) of
        false ->
            proc_lib:hibernate(gen_server, enter_loop,
                               [?MODULE, [], State]);
        true ->
            {noreply, State}
    end.

%% Websocket handler with hibernation
websocket_info(no_activity, State) ->
    {ok, State, hibernate};
websocket_info({send, Data}, State) ->
    {reply, {text, Data}, State}.

%% ระวัง: ค่าใช้จ่าย wake-up จาก hibernate
%% ทุก wakeup ต้องเรียก erlang:hibernate ใหม่
```

---

## 6. Backpressure Pattern

```erlang
%% backpressure_server.erl — ป้องกัน overload
-module(backpressure_server).
-behaviour(gen_server).

-export([start_link/0, submit/1]).
-export([init/1, handle_call/3, handle_cast/2]).

-define(MAX_QUEUE, 1000).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

submit(Work) ->
    gen_server:call(?MODULE, {submit, Work}, 5000).

init([]) ->
    {ok, #{queue_size => 0, processing => 0}}.

handle_call({submit, Work}, From,
            #{queue_size := QSize} = State) when QSize >= ?MAX_QUEUE ->
    %% Reject: queue full
    {reply, {error, overloaded}, State};

handle_call({submit, Work}, From, State) ->
    %% Check current load
    case maps:get(processing, State) of
        N when N >= 100 ->
            %% Slow down: return estimated wait time
            WaitMs = N * 10,
            {reply, {wait, WaitMs}, State};
        _ ->
            %% Accept work
            spawn_link(fun() ->
                Result = process_work(Work),
                gen_server:reply(From, {ok, Result})
            end),
            {noreply, State#{processing => maps:get(processing, State) + 1}}
    end.

process_work(Work) ->
    %% Do the work
    timer:sleep(10),
    done.
```

---

## 7. Circuit Breaker Pattern

```erlang
%% circuit_breaker.erl
-module(circuit_breaker).
-behaviour(gen_server).

-export([start_link/2, call/2]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

-define(FAIL_THRESHOLD,    5).     %% failures to open circuit
-define(RESET_TIMEOUT,     30000). %% ms before trying again
-define(SUCCESS_THRESHOLD, 2).     %% successes to close circuit

%% States: closed | open | half_open
-record(state, {
    name      :: atom(),
    status    :: closed | open | half_open,
    failures  :: non_neg_integer(),
    successes :: non_neg_integer(),
    last_fail :: integer()
}).

start_link(Name, _Opts) ->
    gen_server:start_link({local, Name}, ?MODULE, [Name], []).

call(Name, Fun) ->
    gen_server:call(Name, {call, Fun}).

init([Name]) ->
    {ok, #state{name=Name, status=closed,
                failures=0, successes=0, last_fail=0}}.

handle_call({call, Fun}, _From, #state{status=open}=S) ->
    Now = erlang:system_time(millisecond),
    if
        Now - S#state.last_fail >= ?RESET_TIMEOUT ->
            %% Try half-open
            execute(Fun, S#state{status=half_open});
        true ->
            {reply, {error, circuit_open}, S}
    end;

handle_call({call, Fun}, _From, S) ->
    execute(Fun, S).

execute(Fun, S) ->
    try
        Result = Fun(),
        NewS = on_success(S),
        {reply, {ok, Result}, NewS}
    catch
        _:Reason ->
            NewS = on_failure(S),
            {reply, {error, Reason}, NewS}
    end.

on_success(#state{status=half_open, successes=N}=S)
  when N + 1 >= ?SUCCESS_THRESHOLD ->
    logger:info("Circuit ~p: CLOSED", [S#state.name]),
    S#state{status=closed, failures=0, successes=0};
on_success(#state{status=half_open}=S) ->
    S#state{successes=S#state.successes+1};
on_success(S) ->
    S#state{failures=0}.

on_failure(#state{failures=N}=S) when N + 1 >= ?FAIL_THRESHOLD ->
    logger:warning("Circuit ~p: OPEN", [S#state.name]),
    S#state{status=open,
            failures=N+1,
            last_fail=erlang:system_time(millisecond)};
on_failure(S) ->
    S#state{status=closed, failures=S#state.failures+1}.

handle_cast(_, S) -> {noreply, S}.
handle_info(_, S) -> {noreply, S}.
```

---

## 8. Saga Pattern

```erlang
%% saga.erl — Distributed transaction with compensation
-module(saga).
-export([execute/1]).

%% Saga = list of {Action, Compensation} pairs
%% ถ้า step N ล้มเหลว: ทำ compensations ของ steps 1..N-1 ย้อนกลับ

execute(Steps) ->
    execute_steps(Steps, []).

execute_steps([], Completed) ->
    %% All steps done
    {ok, lists:reverse(Completed)};

execute_steps([{Action, Compensation} | Rest], Completed) ->
    case Action() of
        {ok, Result} ->
            execute_steps(Rest, [{Result, Compensation} | Completed]);
        {error, Reason} ->
            %% Compensate all completed steps
            compensate(Completed),
            {error, Reason}
    end.

compensate(Completed) ->
    lists:foreach(fun({Result, Comp}) ->
        try Comp(Result)
        catch C:R ->
            logger:error("Compensation failed: ~p:~p", [C, R])
        end
    end, Completed).

%% ตัวอย่าง: order placement saga
place_order_saga(Order) ->
    saga:execute([
        {
            fun() -> inventory:reserve(Order) end,
            fun(Reservation) -> inventory:release(Reservation) end
        },
        {
            fun() -> payment:charge(Order) end,
            fun(ChargeId) -> payment:refund(ChargeId) end
        },
        {
            fun() -> shipping:schedule(Order) end,
            fun(ShipmentId) -> shipping:cancel(ShipmentId) end
        },
        {
            fun() -> notification:send_confirmation(Order) end,
            fun(_) -> ok end  %% no compensation needed
        }
    ]).
```

---

## 9. แบบฝึกหัด

### Exercise: Circuit Breaker สำหรับ External API

```erlang
%% ใช้ circuit_breaker กับ external service

%% 1. Start circuit breaker
circuit_breaker:start_link(payment_api, [
    {fail_threshold, 3},
    {reset_timeout, 60000}
]).

%% 2. Wrap external calls
make_payment(Amount, CardToken) ->
    Fun = fun() ->
        payment_gateway:charge(Amount, CardToken)
    end,
    case circuit_breaker:call(payment_api, Fun) of
        {ok, Result} ->
            {ok, Result};
        {error, circuit_open} ->
            %% Circuit is open: queue for later or use fallback
            payment_queue:enqueue(Amount, CardToken),
            {error, service_unavailable};
        {error, Reason} ->
            {error, Reason}
    end.
```

---

## สรุป Part 31

✅ Process registry patterns: local, gproc, pg, ETS  
✅ Dynamic supervisor patterns  
✅ GenServer best practices  
✅ Callback timeout patterns  
✅ Process hibernation  
✅ Backpressure pattern  
✅ Circuit breaker pattern  
✅ Saga pattern สำหรับ distributed transactions

---

*Part 31/100 | [← ก่อนหน้า](../part30/README.md) | [ถัดไป →](../part32/README.md)*
