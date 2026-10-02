# Part 69: Advanced OTP Design Patterns

> **"OTP behaviours are the vocabulary — mastering them makes you fluent in Erlang"**  
> OTP behaviours คือ vocabulary — การ master พวกมันทำให้คุณพูด Erlang ได้คล่อง

---

## สารบัญ

1. [Custom Behaviours](#1-custom-behaviours)
2. [gen_server Deep Dive](#2-gen_server-deep-dive)
3. [gen_statem Advanced](#3-gen_statem-advanced)
4. [Application Lifecycle](#4-application-lifecycle)
5. [Supervision Strategies Deep Dive](#5-supervision-strategies-deep-dive)
6. [OTP Release Upgrades](#6-otp-release-upgrades)
7. [แบบฝึกหัด](#7-แบบฝึกหัด)

---

## 1. Custom Behaviours

```erlang
%% Define a custom behaviour for plugins
%% plugin_behaviour.erl — define contract for plugins

-module(plugin_behaviour).

%% Any module implementing this behaviour must export:
-callback init(Config :: map()) ->
    {ok, State :: term()} | {error, Reason :: term()}.

-callback handle_event(Event :: term(), State :: term()) ->
    {ok, NewState :: term()} |
    {emit, Events :: [term()], NewState :: term()} |
    {stop, Reason :: term()}.

-callback terminate(Reason :: term(), State :: term()) -> ok.

%% Optional callbacks (not required)
-callback name() -> atom().
-optional_callbacks([name/0]).

%% Helper to check if module implements behaviour
implements(Module) ->
    Exports = Module:module_info(exports),
    Required = [{init, 1}, {handle_event, 2}, {terminate, 2}],
    lists:all(fun(F) -> lists:member(F, Exports) end, Required).

%% Plugin runner — calls into behaviour callbacks
-module(plugin_runner).
-export([start/2, process/2, stop/2]).

start(Module, Config) ->
    case Module:init(Config) of
        {ok, State}    -> {ok, {Module, State}};
        {error, Reason} -> {error, Reason}
    end.

process({Module, State}, Event) ->
    case Module:handle_event(Event, State) of
        {ok, NewState}           -> {{Module, NewState}, []};
        {emit, Events, NewState} -> {{Module, NewState}, Events};
        {stop, Reason}           -> throw({plugin_stopped, Reason})
    end.

stop({Module, State}, Reason) ->
    Module:terminate(Reason, State).
```

```erlang
%% Example plugin implementing the behaviour
-module(logger_plugin).
-behaviour(plugin_behaviour).
-export([init/1, handle_event/2, terminate/2, name/0]).

name() -> logger_plugin.

init(#{level := Level}) ->
    {ok, #{level => Level, count => 0}}.

handle_event({log, Severity, Msg}, #{level := MinLevel, count := N} = State) ->
    case severity_level(Severity) >= severity_level(MinLevel) of
        true  ->
            io:format("[~s] ~s~n", [Severity, Msg]),
            {ok, State#{count => N + 1}};
        false ->
            {ok, State}
    end.

terminate(Reason, #{count := N}) ->
    io:format("Logger plugin stopped (~p). Logged ~p events.~n", [Reason, N]),
    ok.

severity_level(debug)   -> 0;
severity_level(info)    -> 1;
severity_level(warning) -> 2;
severity_level(error)   -> 3;
severity_level(critical) -> 4.
```

---

## 2. gen_server Deep Dive

```erlang
%% advanced_genserver.erl — gen_server advanced patterns
-module(advanced_genserver).
-behaviour(gen_server).
-export([start_link/1, get/2, put/3, delete/2, stats/1]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2,
         code_change/3, terminate/2, format_status/2]).

-record(state, {
    data    :: map(),
    stats   :: #{atom() => non_neg_integer()},
    options :: map()
}).

start_link(Options) ->
    gen_server:start_link(?MODULE, Options, []).

get(Pid, Key)       -> gen_server:call(Pid, {get, Key}).
put(Pid, Key, Value) -> gen_server:call(Pid, {put, Key, Value}).
delete(Pid, Key)    -> gen_server:call(Pid, {delete, Key}).
stats(Pid)          -> gen_server:call(Pid, stats).

init(Options) ->
    %% Trap exits to ensure terminate/2 is called
    process_flag(trap_exit, true),
    %% Set up hibernation timer
    Schedule = maps:get(hibernate_after, Options, 5000),
    erlang:send_after(Schedule, self(), maybe_hibernate),
    {ok, #state{data=#{}, stats=new_stats(), options=Options}}.

handle_call({get, Key}, _From, #state{data=D, stats=S} = State) ->
    Result = maps:find(Key, D),
    {reply, Result, State#state{stats=increment(S, get)}};

handle_call({put, Key, Value}, _From, #state{data=D, stats=S} = State) ->
    {reply, ok, State#state{data=D#{Key => Value},
                            stats=increment(S, put)}};

handle_call({delete, Key}, _From, #state{data=D, stats=S} = State) ->
    {reply, ok, State#state{data=maps:remove(Key, D),
                            stats=increment(S, delete)}};

handle_call(stats, _From, #state{stats=S, data=D} = State) ->
    {reply, S#{size => map_size(D)}, State}.

handle_cast(_Msg, State) -> {noreply, State}.

handle_info(maybe_hibernate, #state{options=Opts} = State) ->
    Schedule = maps:get(hibernate_after, Opts, 5000),
    erlang:send_after(Schedule, self(), maybe_hibernate),
    {noreply, State, hibernate};  %% hibernate: release heap until next message

handle_info({'EXIT', _Pid, Reason}, State) ->
    logger:warning("Linked process died: ~p", [Reason]),
    {noreply, State};

handle_info(_Info, State) -> {noreply, State}.

%% Called for hot code upgrade
code_change(OldVsn, State, Extra) ->
    logger:info("code_change from ~p, extra=~p", [OldVsn, Extra]),
    %% Migrate state if needed
    {ok, migrate_state(State)}.

terminate(Reason, #state{stats=S, data=D}) ->
    logger:info("Shutting down: reason=~p, stats=~p, size=~p",
                [Reason, S, map_size(D)]),
    ok.

%% Custom process info (shown in sys:get_status/1)
format_status(Opt, [_PDict, State]) ->
    case Opt of
        terminate ->
            %% Called when about to crash — strip sensitive data
            State#state{data=#{}};
        normal ->
            #{state_size => map_size(State#state.data),
              stats      => State#state.stats}
    end.

new_stats() -> #{get => 0, put => 0, delete => 0}.
increment(Stats, Key) ->
    maps:update_with(Key, fun(N) -> N+1 end, 1, Stats).

migrate_state(State) -> State.
```

---

## 3. gen_statem Advanced

```erlang
%% advanced_statem.erl — advanced gen_statem patterns
-module(advanced_statem).
-behaviour(gen_statem).
-export([start_link/0]).
-export([callback_mode/0, init/1]).

%% Callback modes:
%% state_functions: one function per state
%% handle_event_function: one function handles all states

callback_mode() ->
    %% Use handle_event_function for shared logic across states
    [handle_event_function, state_enter].

init([]) ->
    {ok, idle, #{}, []}.

%% state_enter: called when entering each state
handle_event(enter, OldState, NewState, Data) ->
    logger:debug("State transition: ~p -> ~p", [OldState, NewState]),
    telemetry:execute([fsm, state_change], #{count => 1},
                      #{from => OldState, to => NewState}),
    {keep_state, Data};

%% Event with postpone: process later
handle_event({call, From}, event_while_busy, busy, Data) ->
    {keep_state, Data, [postpone]};  %% defer until state changes

%% Internal events (generated by the FSM itself)
handle_event(internal, start_timer, waiting, Data) ->
    {keep_state, Data,
     [{timeout, 5000, check_expired}]};

%% Named timers: multiple independent timers per state
handle_event(info, start_ping, active, Data) ->
    {keep_state, Data,
     [{{timeout, ping_timer}, 1000, ping}]};

handle_event({timeout, ping_timer}, ping, active, Data) ->
    send_ping(),
    {keep_state, Data,
     [{{timeout, ping_timer}, 1000, ping}]};

%% Cancel a named timer
handle_event(cast, stop_ping, active, Data) ->
    {keep_state, Data,
     [{{timeout, ping_timer}, cancel}]};

%% Reply to multiple callers at once
handle_event({call, From1}, request, State, #{pending := [From2 | _]} = Data) ->
    Result = compute_result(),
    {keep_state, Data,
     [{reply, From1, Result},
      {reply, From2, Result}]}.

%% Catch-all
handle_event(EventType, EventContent, State, Data) ->
    logger:debug("Unhandled ~p/~p in state ~p", [EventType, EventContent, State]),
    {keep_state, Data}.

send_ping() -> ok.
compute_result() -> ok.
```

---

## 4. Application Lifecycle

```erlang
%% myapp.erl — OTP Application callback
-module(myapp).
-behaviour(application).
-export([start/2, stop/1, prep_stop/1]).

start(normal, _Args) ->
    logger:info("Starting ~p application", [myapp]),
    case myapp_sup:start_link() of
        {ok, Pid} ->
            %% Post-start initialization
            ok = init_database(),
            ok = load_config(),
            ok = register_metrics(),
            logger:info("~p started successfully", [myapp]),
            {ok, Pid};
        {error, {already_started, Pid}} ->
            {ok, Pid};
        {error, Reason} ->
            logger:error("Failed to start ~p: ~p", [myapp, Reason]),
            {error, Reason}
    end;
start(takeover, _Args) ->
    %% Hot takeover during failover
    logger:warning("Application takeover"),
    start(normal, []);
start(failover, _Args) ->
    logger:warning("Application failover"),
    start(normal, []).

%% prep_stop: called BEFORE supervisor terminates children
%% Use for graceful shutdown: drain connections, flush buffers
prep_stop(State) ->
    logger:info("Preparing to stop, draining connections"),
    http_acceptor:drain(),
    logger:info("All connections drained"),
    State.

stop(_State) ->
    logger:info("Application stopped"),
    ok.

init_database() ->
    case db_pool:init() of
        ok         -> ok;
        {error, E} -> throw({db_init_failed, E})
    end.

load_config() ->
    Env = os:getenv("APP_ENV", "prod"),
    config:load_file(Env).

register_metrics() ->
    telemetry:attach_many(myapp_metrics, [
        [http, request, stop],
        [db, query, stop]
    ], fun metrics_handler:handle/4, #{}).
```

```erlang
%% sys module: send messages to OTP processes without gen_server calls
sys_examples() ->
    %% Suspend a gen_server (useful for debugging)
    sys:suspend(my_server),
    %% ... inspect state ...
    sys:resume(my_server),

    %% Get internal state (calls format_status)
    {status, _Pid, {module, _}, [_PDict, _SysState, _Parent, _Debug, Info]} =
        sys:get_status(my_server),
    io:format("State: ~p~n", [Info]),

    %% Install a debug function (trace calls temporarily)
    sys:install(my_server, {fun debug_fn/3, #{}}),
    sys:remove(my_server, fun debug_fn/3),

    %% Send a code_change event
    sys:change_code(my_server, my_module, "1.0.0", undefined).

debug_fn(func, Event, State) ->
    io:format("Event: ~p~n", [Event]),
    State.
```

---

## 5. Supervision Strategies Deep Dive

```erlang
%% supervision_strategies.erl — choosing the right strategy

%% one_for_one: most common
%% - each child restarts independently
%% - use when children are unrelated
%% Example: DB pool workers, cache workers

%% one_for_all: restart everything together
%% - if one fails, restart all
%% - use when children share state or all depend on each other
%% Example: a cluster of synchronized workers

%% rest_for_one: restart failed child and all after it
%% - dependency chain: child N depends on N-1
%% - use for pipeline-style processes
%% Example: DB connection → connection pool → query worker

%% simple_one_for_one (now just use one_for_one with dynamic)
%% - same child spec cloned for each start_child call
%% - efficient for many identical workers

-module(pipeline_sup).
-behaviour(supervisor).
-export([start_link/0, init/1]).

start_link() ->
    supervisor:start_link({local, ?MODULE}, ?MODULE, []).

init([]) ->
    %% rest_for_one: if broker dies, restart broker AND all workers after it
    {ok, {
        #{strategy  => rest_for_one,
          intensity => 5,
          period    => 10},
        [
            %% Step 1: message broker (everything else depends on this)
            #{id    => broker,
              start => {message_broker, start_link, []},
              type  => worker},
            %% Step 2: processor (depends on broker)
            #{id    => processor,
              start => {message_processor, start_link, []},
              type  => worker},
            %% Step 3: persister (depends on processor)
            #{id    => persister,
              start => {message_persister, start_link, []},
              type  => worker}
        ]
    }}.

%% Dynamic child spec: same spec, different args
-module(worker_sup).
-behaviour(supervisor).
-export([start_link/0, start_worker/1, init/1]).

start_link() ->
    supervisor:start_link({local, ?MODULE}, ?MODULE, []).

start_worker(WorkerArgs) ->
    supervisor:start_child(?MODULE, [WorkerArgs]).

init([]) ->
    %% Template spec: no id needed for dynamic
    {ok, {
        #{strategy => simple_one_for_one},
        [#{id       => worker,
           start    => {dynamic_worker, start_link, []},
           restart  => temporary}]
    }}.

%% Intensity/period: max restarts before supervisor gives up
%% intensity=5, period=60: 5 restarts within 60 seconds → supervisor crashes
%% This propagates the crash UP the tree to the parent supervisor
intensity_example() ->
    %% For critical services: high tolerance
    %% #{intensity => 100, period => 10} — 100 per 10s

    %% For unstable services: low tolerance (fail fast up the tree)
    %% #{intensity => 1, period => 10} — 1 per 10s
    ok.
```

---

## 6. OTP Release Upgrades

```erlang
%% appup_example.erl — hot code upgrade support

%% myapp.appup file format:
%% {"2.0.0",
%%  [{"1.0.0", [{load_module, mymodule},
%%              {update, mygenserver, {advanced, []}}]}],
%%  [{"1.0.0", [{load_module, mymodule}]}]}.

%% Module supporting hot upgrade:
-module(mygenserver).
-behaviour(gen_server).
-export([code_change/3]).

code_change("1.0.0", OldState, _Extra) ->
    %% Migrate state from v1 to v2
    NewState = case OldState of
        #{data := D} ->
            %% v1 had just 'data', v2 adds 'version'
            #{data => D, version => 2, migrated_at => os:system_time()};
        _ ->
            OldState#{version => 2}
    end,
    {ok, NewState};
code_change(_OldVsn, State, _Extra) ->
    {ok, State}.

%% Relup file: instructions for releasing the upgrade
%% Generated by systools:make_relup/3
relup_notes() ->
    %% Steps during hot upgrade:
    %% 1. Load new beam files
    %% 2. Suspend affected gen_servers
    %% 3. Call code_change/3 on each
    %% 4. Resume gen_servers
    %% 5. Purge old code
    %%
    %% No downtime! But:
    %% - code_change must handle state migration
    %% - avoid breaking changes to external interfaces during upgrade
    %% - test upgrade path before applying in production
    ok.

%% Checking if upgrade is safe
pre_upgrade_checks() ->
    %% Check no pending messages in critical queues
    {message_queue_len, Q} = erlang:process_info(whereis(mygenserver), message_queue_len),
    case Q > 0 of
        true  -> {warn, {messages_pending, Q}};
        false -> ok
    end.
```

---

## 7. แบบฝึกหัด

1. สร้าง custom behaviour สำหรับ `rate_limiter`: `check/2` และ `reset/1` callbacks
2. Implement `gen_pipeline` behaviour: แต่ละ stage เป็น gen_server
3. เขียน appup file สำหรับ upgrade gen_server state จาก tuple → map
4. ทดสอบ sys:suspend/resume บน running gen_server ใน Erlang shell

---

## สรุป Part 69

✅ Custom behaviours ด้วย -callback และ -optional_callbacks  
✅ gen_server deep dive: hibernate, code_change, format_status  
✅ gen_statem advanced: state_enter, named timers, postpone  
✅ Application lifecycle: start/stop/prep_stop  
✅ Supervision strategies: one_for_one/all/rest  
✅ Hot code upgrades: appup, relup, code_change migration  

---

*Part 69/100 | [← ก่อนหน้า](../part68/README.md) | [ถัดไป →](../part70/README.md)*
