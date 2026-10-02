# Part 64: Workflow Engine

> **"Complex business processes need a reliable, observable execution engine"**  
> กระบวนการทางธุรกิจที่ซับซ้อนต้องการ execution engine ที่เชื่อถือได้และตรวจสอบได้

---

## สารบัญ

1. [Workflow Definition](#1-workflow-definition)
2. [Step Execution Engine](#2-step-execution-engine)
3. [State Machine Integration](#3-state-machine-integration)
4. [Compensating Transactions](#4-compensating-transactions)
5. [Parallel Workflows](#5-parallel-workflows)
6. [Workflow Monitoring](#6-workflow-monitoring)
7. [แบบฝึกหัด](#7-แบบฝึกหัด)

---

## 1. Workflow Definition

```erlang
%% workflow.erl — workflow definition and data structures
-module(workflow).
-export([define/2, get_step/2, next_step/3]).

%% Workflow: ordered list of steps with conditions
%% Step: {name, module, timeout, retries, compensate}

-type step_name() :: atom().
-type step_def()  :: #{
    name       := step_name(),
    module     := module(),
    timeout    := pos_integer(),
    retries    := non_neg_integer(),
    on_failure := fail | skip | compensate,
    depends_on => [step_name()]
}.

-type workflow_def() :: #{
    name  := atom(),
    steps := [step_def()],
    on_complete => fun(),
    on_fail     => fun()
}.

%% Example: order fulfillment workflow
order_workflow() ->
    #{
        name  => order_fulfillment,
        steps => [
            #{name       => validate_order,
              module     => order_validator,
              timeout    => 5000,
              retries    => 0,
              on_failure => fail},
            #{name       => reserve_inventory,
              module     => inventory_reserver,
              timeout    => 10000,
              retries    => 2,
              on_failure => compensate},
            #{name       => charge_payment,
              module     => payment_charger,
              timeout    => 30000,
              retries    => 1,
              on_failure => compensate},
            #{name       => create_shipment,
              module     => shipment_creator,
              timeout    => 15000,
              retries    => 3,
              on_failure => compensate},
            #{name       => send_confirmation,
              module     => email_sender,
              timeout    => 10000,
              retries    => 3,
              on_failure => skip}   %% email failure is OK
        ],
        on_complete => fun(Context) ->
            logger:info("Order ~p completed", [maps:get(order_id, Context)])
        end,
        on_fail => fun(Context, Step, Error) ->
            logger:error("Order ~p failed at ~p: ~p",
                         [maps:get(order_id, Context), Step, Error])
        end
    }.

define(Name, Steps) ->
    #{name => Name, steps => Steps}.

get_step(#{steps := Steps}, StepName) ->
    case [S || S <- Steps, maps:get(name, S) =:= StepName] of
        [Step | _] -> {ok, Step};
        []         -> {error, not_found}
    end.

next_step(#{steps := Steps}, CurrentStep, Result) ->
    Names = [maps:get(name, S) || S <- Steps],
    case {lists:member(CurrentStep, Names), Result} of
        {false, _} -> {error, unknown_step};
        {true, ok} ->
            Idx = index_of(CurrentStep, Names),
            case Idx < length(Names) of
                true  -> {ok, lists:nth(Idx + 1, Names)};
                false -> done
            end;
        {true, {error, _}} ->
            {ok, Step} = get_step(#{steps => Steps}, CurrentStep),
            maps:get(on_failure, Step, fail)
    end.

index_of(Elem, List) ->
    index_of(Elem, List, 1).
index_of(Elem, [Elem | _], N) -> N;
index_of(Elem, [_ | Rest], N) -> index_of(Elem, Rest, N + 1).
```

---

## 2. Step Execution Engine

```erlang
%% workflow_engine.erl — execute workflow steps
-module(workflow_engine).
-behaviour(gen_server).
-export([start_link/0, execute/3, get_status/1, cancel/1]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

-record(execution, {
    id         :: binary(),
    workflow   :: workflow:workflow_def(),
    context    :: map(),
    status     :: pending | running | completed | failed | cancelled,
    current    :: atom() | undefined,
    history    :: [step_result()],
    started_at :: integer(),
    timer      :: reference() | undefined
}).

-type step_result() :: #{
    step      := atom(),
    status    := ok | error | skipped,
    result    => term(),
    error     => term(),
    started   := integer(),
    finished  := integer()
}.

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

execute(WorkflowDef, Context, Opts) ->
    ExecId = generate_id(),
    gen_server:call(?MODULE, {execute, ExecId, WorkflowDef, Context, Opts}).

get_status(ExecId) ->
    gen_server:call(?MODULE, {get_status, ExecId}).

cancel(ExecId) ->
    gen_server:cast(?MODULE, {cancel, ExecId}).

init([]) ->
    {ok, #{executions => #{}}}.

handle_call({execute, ExecId, WorkflowDef, Context, _Opts}, _From, State) ->
    Execution = #execution{
        id         = ExecId,
        workflow   = WorkflowDef,
        context    = Context,
        status     = running,
        current    = undefined,
        history    = [],
        started_at = os:system_time(millisecond),
        timer      = undefined
    },
    self() ! {run_next_step, ExecId},
    NewExecs = maps:put(ExecId, Execution, maps:get(executions, State)),
    {reply, {ok, ExecId}, State#{executions => NewExecs}};

handle_call({get_status, ExecId}, _From, State) ->
    case maps:find(ExecId, maps:get(executions, State)) of
        {ok, Exec} ->
            {reply, {ok, execution_to_map(Exec)}, State};
        error ->
            {reply, {error, not_found}, State}
    end.

handle_cast({cancel, ExecId}, State) ->
    case maps:find(ExecId, maps:get(executions, State)) of
        {ok, Exec = #execution{timer=Timer}} ->
            case Timer of
                undefined -> ok;
                Ref       -> erlang:cancel_timer(Ref)
            end,
            Updated = Exec#execution{status=cancelled},
            Execs = maps:put(ExecId, Updated, maps:get(executions, State)),
            {noreply, State#{executions => Execs}};
        error ->
            {noreply, State}
    end.

handle_info({run_next_step, ExecId}, State) ->
    case maps:find(ExecId, maps:get(executions, State)) of
        {ok, #execution{status=running} = Exec} ->
            NewState = run_next_step(ExecId, Exec, State),
            {noreply, NewState};
        _ ->
            {noreply, State}
    end;

handle_info({step_timeout, ExecId, StepName}, State) ->
    logger:warning("Step ~p timed out in execution ~p", [StepName, ExecId]),
    State1 = handle_step_result(ExecId, StepName, {error, timeout}, State),
    {noreply, State1};

handle_info({step_done, ExecId, StepName, Result}, State) ->
    State1 = handle_step_result(ExecId, StepName, Result, State),
    {noreply, State1}.

run_next_step(ExecId, #execution{workflow=WF, history=History, context=Ctx} = Exec, State) ->
    Steps = maps:get(steps, WF),
    CompletedSteps = [maps:get(step, H) || H <- History,
                                            maps:get(status, H) =/= skipped],
    case next_pending_step(Steps, CompletedSteps) of
        none ->
            complete_execution(ExecId, Exec, State);
        StepDef ->
            execute_step(ExecId, StepDef, Ctx, Exec, State)
    end.

next_pending_step(Steps, Completed) ->
    case [S || S <- Steps, not lists:member(maps:get(name, S), Completed)] of
        []           -> none;
        [Next | _]   -> Next
    end.

execute_step(ExecId, StepDef, Context, Exec, State) ->
    #{name := StepName, module := Mod,
      timeout := Timeout, retries := _Retries} = StepDef,

    %% Set timeout
    TimerRef = erlang:send_after(Timeout, self(),
                                 {step_timeout, ExecId, StepName}),

    %% Execute step asynchronously
    Self = self(),
    spawn_link(fun() ->
        Result = try
            Mod:execute(StepName, Context)
        catch
            C:E:ST ->
                logger:error("Step ~p crashed: ~p:~p~n~p", [StepName, C, E, ST]),
                {error, {crashed, C, E}}
        end,
        Self ! {step_done, ExecId, StepName, Result}
    end),

    Updated = Exec#execution{current=StepName, timer=TimerRef},
    Execs = maps:put(ExecId, Updated, maps:get(executions, State)),
    State#{executions => Execs}.

handle_step_result(ExecId, StepName, Result, State) ->
    case maps:find(ExecId, maps:get(executions, State)) of
        {ok, #execution{timer=Timer, history=History, context=Ctx} = Exec} ->
            case Timer of
                undefined -> ok;
                Ref       -> erlang:cancel_timer(Ref)
            end,
            StepRecord = #{
                step     => StepName,
                status   => result_status(Result),
                result   => Result,
                finished => os:system_time(millisecond)
            },
            NewHistory = [StepRecord | History],
            NewCtx     = update_context(Ctx, StepName, Result),
            Updated = Exec#execution{history=NewHistory, context=NewCtx,
                                     current=undefined, timer=undefined},
            Execs = maps:put(ExecId, Updated, maps:get(executions, State)),
            State1 = State#{executions => Execs},
            self() ! {run_next_step, ExecId},
            State1;
        error ->
            State
    end.

result_status({error, _}) -> error;
result_status(ok)          -> ok;
result_status({ok, _})     -> ok.

update_context(Ctx, StepName, {ok, Value}) ->
    Ctx#{StepName => Value};
update_context(Ctx, _, _) -> Ctx.

complete_execution(ExecId, Exec, State) ->
    #execution{workflow=WF, context=Ctx} = Exec,
    case maps:find(on_complete, WF) of
        {ok, Fun} -> catch Fun(Ctx);
        error     -> ok
    end,
    Updated = Exec#execution{status=completed},
    Execs = maps:put(ExecId, Updated, maps:get(executions, State)),
    State#{executions => Execs}.

execution_to_map(#execution{id=Id, status=S, current=C, history=H}) ->
    #{id => Id, status => S, current_step => C, history => H}.

generate_id() ->
    <<A:32, B:16>> = crypto:strong_rand_bytes(6),
    iolist_to_binary(io_lib:format("wf_~8.16.0b~4.16.0b", [A, B])).
```

---

## 3. State Machine Integration

```erlang
%% workflow_fsm.erl — workflow as gen_statem
-module(workflow_fsm).
-behaviour(gen_statem).
-export([start_link/2, advance/2, get_state/1]).
-export([callback_mode/0, init/1, pending/3, running/3,
         completed/3, failed/3, compensating/3]).

callback_mode() -> state_functions.

start_link(WorkflowId, Def) ->
    gen_statem:start_link(?MODULE, {WorkflowId, Def}, []).

advance(Pid, Event) ->
    gen_statem:call(Pid, {advance, Event}).

get_state(Pid) ->
    gen_statem:call(Pid, get_state).

init({WorkflowId, Def}) ->
    {ok, pending, #{id => WorkflowId, def => Def, results => #{}}}.

pending({call, From}, start, Data) ->
    {next_state, running, Data, [{reply, From, ok},
                                  {next_event, internal, run_first_step}]};
pending({call, From}, get_state, Data) ->
    {keep_state, Data, [{reply, From, pending}]}.

running(internal, run_first_step, #{def := Def} = Data) ->
    [FirstStep | _] = maps:get(steps, Def),
    {keep_state, Data#{current => FirstStep},
     [{next_event, internal, {execute_step, FirstStep}}]};
running(internal, {execute_step, Step}, Data) ->
    #{name := Name, module := Mod} = Step,
    Ctx = maps:get(context, Data, #{}),
    Result = Mod:execute(Name, Ctx),
    {keep_state, Data, [{next_event, internal, {step_done, Name, Result}}]};
running(internal, {step_done, StepName, ok}, Data) ->
    {next_state, completed, Data#{last_step => StepName}};
running(internal, {step_done, StepName, {ok, _} = R}, Data) ->
    {next_state, completed, Data#{last_step => StepName, result => R}};
running(internal, {step_done, StepName, {error, _} = E}, Data) ->
    {next_state, failed, Data#{failed_step => StepName, error => E}};
running({call, From}, get_state, Data) ->
    {keep_state, Data, [{reply, From, running}]}.

completed({call, From}, get_state, Data) ->
    {keep_state, Data, [{reply, From, {completed, Data}}]}.

failed({call, From}, get_state, Data) ->
    {keep_state, Data, [{reply, From, {failed, maps:get(error, Data)}}]};
failed(internal, compensate, Data) ->
    {next_state, compensating, Data}.

compensating({call, From}, get_state, Data) ->
    {keep_state, Data, [{reply, From, compensating}]}.
```

---

## 4. Compensating Transactions

```erlang
%% compensator.erl — undo completed steps on failure (Saga pattern)
-module(compensator).
-export([run/2]).

run(CompletedSteps, Context) ->
    %% Compensate in reverse order
    ReversedSteps = lists:reverse(CompletedSteps),
    run_compensations(ReversedSteps, Context, []).

run_compensations([], _Ctx, Errors) ->
    case Errors of
        [] -> ok;
        _  -> {partial_failure, Errors}
    end;
run_compensations([Step | Rest], Ctx, Errors) ->
    #{step := StepName, module := Mod} = Step,
    Result = try
        Mod:compensate(StepName, Ctx)
    catch
        C:E ->
            logger:error("Compensation failed for ~p: ~p:~p", [StepName, C, E]),
            {error, {C, E}}
    end,
    case Result of
        ok        -> run_compensations(Rest, Ctx, Errors);
        {error, E} ->
            logger:warning("Compensation error at ~p: ~p", [StepName, E]),
            run_compensations(Rest, Ctx, [{StepName, E} | Errors])
    end.

%% Example compensatable step modules

%% inventory_reserver.erl
%% -export([execute/2, compensate/2]).
%% execute(reserve_inventory, #{order_id := OId}) ->
%%     case inventory:reserve(OId) of
%%         {ok, ReservationId} -> {ok, #{reservation_id => ReservationId}};
%%         {error, E} -> {error, E}
%%     end.
%% compensate(reserve_inventory, #{reserve_inventory := #{reservation_id := RId}}) ->
%%     inventory:cancel_reservation(RId).

%% payment_charger.erl
%% execute(charge_payment, #{order_id := OId, amount := Amount}) ->
%%     case payment:charge(OId, Amount) of
%%         {ok, ChargeId} -> {ok, #{charge_id => ChargeId}};
%%         {error, E} -> {error, E}
%%     end.
%% compensate(charge_payment, #{charge_payment := #{charge_id := CId}}) ->
%%     payment:refund(CId).
```

---

## 5. Parallel Workflows

```erlang
%% parallel_workflow.erl — execute independent steps concurrently
-module(parallel_workflow).
-export([execute_parallel/2]).

execute_parallel(Steps, Context) ->
    %% Group steps by dependency level
    Levels = topological_sort(Steps),
    execute_levels(Levels, Context, #{}).

execute_levels([], _Context, Results) ->
    {ok, Results};
execute_levels([Level | Rest], Context, Results) ->
    %% Execute all steps in this level concurrently
    Pids = [{StepDef, spawn_step(StepDef, Context)} || StepDef <- Level],
    LevelResults = collect_results(Pids, #{}),
    case check_failures(LevelResults) of
        ok ->
            NewResults = maps:merge(Results, LevelResults),
            NewContext = maps:merge(Context, LevelResults),
            execute_levels(Rest, NewContext, NewResults);
        {error, Failures} ->
            {error, {step_failures, Failures}}
    end.

spawn_step(#{name := Name, module := Mod}, Context) ->
    Self = self(),
    spawn_link(fun() ->
        Result = try
            Mod:execute(Name, Context)
        catch C:E -> {error, {C, E}}
        end,
        Self ! {step_result, Name, Result}
    end).

collect_results([], Acc) -> Acc;
collect_results(Pids, Acc) ->
    receive
        {step_result, Name, Result} ->
            case [{S, P} || {S, P} <- Pids, element(1, maps:get(name, S)) =/= Name] of
                Remaining ->
                    collect_results(Remaining, Acc#{Name => Result})
            end
    after 60000 ->
        {error, timeout}
    end.

check_failures(Results) ->
    Failures = [{N, E} || {N, {error, E}} <- maps:to_list(Results)],
    case Failures of
        [] -> ok;
        _  -> {error, Failures}
    end.

%% Topological sort: group steps by dependency level
topological_sort(Steps) ->
    topological_sort(Steps, [], []).

topological_sort([], _Done, Levels) ->
    lists:reverse(Levels);
topological_sort(Steps, Done, Levels) ->
    Ready = [S || S <- Steps,
                  lists:all(fun(D) -> lists:member(D, Done) end,
                            maps:get(depends_on, S, []))],
    case Ready of
        [] ->
            error(circular_dependency);
        _ ->
            ReadyNames = [maps:get(name, S) || S <- Ready],
            Remaining  = [S || S <- Steps, not lists:member(maps:get(name, S), ReadyNames)],
            topological_sort(Remaining, Done ++ ReadyNames, [Ready | Levels])
    end.
```

---

## 6. Workflow Monitoring

```erlang
%% workflow_monitor.erl — observe running workflows
-module(workflow_monitor).
-behaviour(gen_server).
-export([start_link/0, register/2, on_step_start/3, on_step_end/3,
         list_running/0, get_history/1]).
-export([init/1, handle_call/3, handle_cast/2]).

-record(state, {
    workflows :: ets:tab(),
    history   :: ets:tab()
}).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

register(ExecId, WorkflowName) ->
    gen_server:cast(?MODULE, {register, ExecId, WorkflowName}).

on_step_start(ExecId, StepName, Context) ->
    gen_server:cast(?MODULE, {step_start, ExecId, StepName, Context}).

on_step_end(ExecId, StepName, Result) ->
    gen_server:cast(?MODULE, {step_end, ExecId, StepName, Result}).

list_running() ->
    gen_server:call(?MODULE, list_running).

get_history(ExecId) ->
    gen_server:call(?MODULE, {get_history, ExecId}).

init([]) ->
    WF   = ets:new(wf_running, [set, named_table, public]),
    Hist = ets:new(wf_history, [bag, named_table, public]),
    {ok, #state{workflows=WF, history=Hist}}.

handle_call(list_running, _From, #state{workflows=WF} = State) ->
    Running = ets:tab2list(WF),
    {reply, Running, State};

handle_call({get_history, ExecId}, _From, #state{history=Hist} = State) ->
    Events = ets:lookup(Hist, ExecId),
    {reply, {ok, Events}, State}.

handle_cast({register, ExecId, WFName}, #state{workflows=WF} = State) ->
    ets:insert(WF, {ExecId, WFName, running, os:system_time(millisecond)}),
    telemetry:execute([workflow, started], #{count => 1},
                      #{name => WFName, id => ExecId}),
    {noreply, State};

handle_cast({step_start, ExecId, StepName, _Ctx}, #state{history=Hist} = State) ->
    ets:insert(Hist, {ExecId, #{step => StepName, event => start,
                                ts => os:system_time(millisecond)}}),
    {noreply, State};

handle_cast({step_end, ExecId, StepName, Result}, #state{history=Hist} = State) ->
    Status = case Result of
        ok        -> success;
        {ok, _}   -> success;
        {error, _} -> failure
    end,
    ets:insert(Hist, {ExecId, #{step => StepName, event => finish,
                                status => Status, ts => os:system_time(millisecond)}}),
    telemetry:execute([workflow, step, finished], #{count => 1},
                      #{step => StepName, status => Status}),
    {noreply, State}.
```

---

## 7. แบบฝึกหัด

1. เพิ่ม retry logic ด้วย exponential backoff สำหรับแต่ละ step
2. Implement `wait_for_approval` step: pause workflow จนกว่า human จะ approve
3. สร้าง workflow persistence: เก็บ state ลง PostgreSQL เพื่อ survive restarts
4. เพิ่ม cron-based workflow scheduling: run workflow ทุกวันเวลา 02:00

---

## สรุป Part 64

✅ Workflow definition ด้วย steps, timeout, retry  
✅ Step execution engine ด้วย GenServer  
✅ gen_statem integration สำหรับ FSM-based workflows  
✅ Compensating transactions (Saga pattern)  
✅ Parallel step execution ด้วย topological sort  
✅ Workflow monitoring ด้วย ETS + telemetry  

---

*Part 64/100 | [← ก่อนหน้า](../part63/README.md) | [ถัดไป →](../part65/README.md)*
