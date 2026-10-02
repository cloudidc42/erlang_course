# Part 90: Building Your Own OTP Behaviour

> **"A behaviour is a contract — define it precisely and the compiler enforces it for you"**  
> Behaviour คือสัญญา — กำหนดมันให้ชัดเจน และ compiler จะบังคับใช้แทนคุณ

---

## สารบัญ

1. [What Is a Behaviour](#1-what-is-a-behaviour)
2. [Defining Callbacks with -callback](#2-defining-callbacks-with--callback)
3. [The Behaviour Module Template](#3-the-behaviour-module-template)
4. [HTTP Handler Behaviour](#4-http-handler-behaviour)
5. [Pipeline Stage Behaviour](#5-pipeline-stage-behaviour)
6. [Compile-Time Validation](#6-compile-time-validation)
7. [แบบฝึกหัด](#7-แบบฝึกหัด)

---

## 1. What Is a Behaviour

```
Erlang Behaviour Pattern
════════════════════════════════════════════════════

A behaviour splits a module into:

  GENERIC part (behaviour module):
    - The loop, supervision hooks, protocol handling
    - Written once, reused everywhere
    - Calls the callback module at customisation points

  SPECIFIC part (callback module):
    - Business logic unique to this instance
    - Declares -behaviour(my_behaviour)
    - Implements required callbacks; optional ones have defaults

Built-in Erlang behaviours:
  gen_server   request-reply server loop
  gen_statem   finite state machine
  gen_event    event dispatcher
  supervisor   process hierarchy management
  application  OTP application lifecycle

When to write your own behaviour:
  - Multiple modules implement the same protocol
  - You want compiler warnings for missing callbacks
  - You want to encapsulate a complex generic loop
  - Creating a plugin system with well-defined extension points

Compiler integration:
  -behaviour(my_behaviour) triggers compile-time check:
    - All required callbacks implemented?
    - Correct arities?
    - Missing optional callbacks get defaults from behaviour
```

---

## 2. Defining Callbacks with -callback

```erlang
%% Callback attribute syntax and options
-module(example_spec).

%% Required callback: implementation MUST provide this
-callback init(Args :: term()) ->
    {ok, State :: term()} | {error, Reason :: term()}.

%% Required callback with multiple valid return types
-callback handle_message(Message :: term(), State :: term()) ->
    {ok, NewState :: term()} |
    {reply, Reply :: term(), NewState :: term()} |
    {stop, Reason :: term(), NewState :: term()}.

%% Optional callback: behaviour provides a default if not implemented
-optional_callbacks([format_status/1, describe/0]).

-callback format_status(State :: term()) -> map().
-callback describe() -> binary().

%% Type specs in callbacks use -type declarations from the module
-type handler_state() :: term().

-callback terminate(Reason :: term(), State :: handler_state()) -> ok.
```

---

## 3. The Behaviour Module Template

```erlang
%% worker_behaviour.erl — generic worker pool behaviour
-module(worker_behaviour).

%% Callbacks the implementing module MUST define
-callback init(Args :: term()) ->
    {ok, State :: term()} | {error, Reason :: term()}.

-callback handle_job(Job :: term(), State :: term()) ->
    {ok, Result :: term(), NewState :: term()} |
    {error, Reason :: term(), NewState :: term()} |
    {stop, Reason :: term()}.

-callback terminate(Reason :: term(), State :: term()) -> ok.

%% Optional callbacks — have defaults
-optional_callbacks([job_timeout/0, max_jobs/0]).

-callback job_timeout() -> pos_integer().  %% default: 30000ms
-callback max_jobs() -> pos_integer().     %% default: 100

%% Generic API that wraps callback module
-export([start_link/2, submit_job/2]).

start_link(Module, Args) ->
    gen_server:start_link(?MODULE, {Module, Args}, []).

submit_job(Worker, Job) ->
    gen_server:call(Worker, {job, Job}).

%% Generic implementation
-record(state, {
    module,
    callback_state,
    active_jobs = 0
}).

init({Module, Args}) ->
    case Module:init(Args) of
        {ok, CbState} ->
            {ok, #state{module = Module, callback_state = CbState}};
        {error, Reason} ->
            {stop, Reason}
    end.

handle_call({job, Job}, _From, #state{module = Mod, callback_state = CbState,
                                       active_jobs = N} = State) ->
    MaxJobs = case erlang:function_exported(Mod, max_jobs, 0) of
        true  -> Mod:max_jobs();
        false -> 100
    end,
    case N >= MaxJobs of
        true ->
            {reply, {error, overloaded}, State};
        false ->
            Timeout = case erlang:function_exported(Mod, job_timeout, 0) of
                true  -> Mod:job_timeout();
                false -> 30000
            end,
            case execute_with_timeout(fun() -> Mod:handle_job(Job, CbState) end, Timeout) of
                {ok, Result, NewCbState} ->
                    NewState = State#state{callback_state = NewCbState,
                                          active_jobs = N - 1},
                    {reply, {ok, Result}, NewState};
                {error, Reason, NewCbState} ->
                    NewState = State#state{callback_state = NewCbState,
                                          active_jobs = N - 1},
                    {reply, {error, Reason}, NewState};
                {stop, Reason} ->
                    {stop, Reason, {error, stopped}, State}
            end
    end.

execute_with_timeout(Fun, TimeoutMs) ->
    Ref = make_ref(),
    Self = self(),
    Pid = spawn(fun() -> Self ! {Ref, Fun()} end),
    receive
        {Ref, Result} -> Result
    after TimeoutMs ->
        exit(Pid, kill),
        {error, timeout, undefined}
    end.

handle_cast(_Msg, State) -> {noreply, State}.
handle_info(_Info, State) -> {noreply, State}.
terminate(_Reason, #state{module = Mod, callback_state = CbState}) ->
    Mod:terminate(normal, CbState).
```

---

## 4. HTTP Handler Behaviour

```erlang
%% http_handler.erl — behaviour for HTTP request handlers
-module(http_handler).

%% Implementing module must define:
-callback init(Req :: cowboy_req:req(), Opts :: any()) ->
    {ok, State :: term()} |
    {error, Status :: 400..599, Body :: binary()}.

-callback handle(Method :: binary(), Path :: [binary()],
                 Req :: cowboy_req:req(), State :: term()) ->
    {ok, Response :: map(), NewState :: term()} |
    {error, Status :: 400..599, Body :: binary()}.

-callback terminate(Req :: cowboy_req:req(), State :: term()) -> ok.

%% Optional: restrict allowed methods
-optional_callbacks([allowed_methods/0]).
-callback allowed_methods() -> [binary()].

%% Generic Cowboy dispatch wrapper
-export([dispatch/3]).

dispatch(Handler, Req, Opts) ->
    Method = cowboy_req:method(Req),

    %% Check allowed methods
    Allowed = case erlang:function_exported(Handler, allowed_methods, 0) of
        true  -> Handler:allowed_methods();
        false -> [<<"GET">>, <<"POST">>, <<"PUT">>, <<"DELETE">>]
    end,
    case lists:member(Method, Allowed) of
        false ->
            cowboy_req:reply(405, #{<<"allow">> => join_methods(Allowed)}, <<>>, Req);
        true ->
            case Handler:init(Req, Opts) of
                {error, Status, Body} ->
                    cowboy_req:reply(Status, #{}, Body, Req);
                {ok, State} ->
                    PathInfo = cowboy_req:path_info(Req),
                    case Handler:handle(Method, PathInfo, Req, State) of
                        {ok, Response, _NewState} ->
                            Status  = maps:get(status, Response, 200),
                            Headers = maps:get(headers, Response, #{}),
                            Body    = maps:get(body, Response, <<>>),
                            cowboy_req:reply(Status, Headers, Body, Req);
                        {error, Status, Body} ->
                            cowboy_req:reply(Status, #{}, Body, Req)
                    end
            end
    end.

join_methods(Methods) ->
    list_to_binary(lists:join(<<", ">>, Methods)).
```

---

## 5. Pipeline Stage Behaviour

```erlang
%% pipeline_stage.erl — composable pipeline stages
-module(pipeline_stage).

-callback name() -> atom().
-callback process(Input :: term(), Config :: map()) ->
    {ok, Output :: term()} |
    {skip, Reason :: term()} |
    {error, Reason :: term()}.

-optional_callbacks([validate_config/1, describe/0]).
-callback validate_config(Config :: map()) -> ok | {error, Reason :: term()}.
-callback describe() -> binary().

%% Pipeline executor
-export([run/3]).

run(Stages, Input, Config) ->
    run_stages(Stages, Input, Config, []).

run_stages([], Final, _Config, _History) ->
    {ok, Final};
run_stages([{Stage, StageConfig} | Rest], Input, Config, History) ->
    FullConfig = maps:merge(Config, StageConfig),
    case Stage:process(Input, FullConfig) of
        {ok, Output} ->
            run_stages(Rest, Output, Config, [{Stage, ok} | History]);
        {skip, Reason} ->
            %% Stage skipped — pass input unchanged to next stage
            logger:debug("Stage ~p skipped: ~p", [Stage, Reason]),
            run_stages(Rest, Input, Config, [{Stage, {skipped, Reason}} | History]);
        {error, Reason} ->
            {error, #{stage => Stage, reason => Reason, history => History}}
    end.

%% Validate all stage configs before running
validate_pipeline(Stages, Config) ->
    Errors = lists:filtermap(fun({Stage, StageConfig}) ->
        case erlang:function_exported(Stage, validate_config, 1) of
            true ->
                FullConfig = maps:merge(Config, StageConfig),
                case Stage:validate_config(FullConfig) of
                    ok              -> false;
                    {error, Reason} -> {true, {Stage, Reason}}
                end;
            false -> false
        end
    end, Stages),
    case Errors of
        [] -> ok;
        _  -> {error, {invalid_config, Errors}}
    end.
```

---

## 6. Compile-Time Validation

```erlang
%% Implementing a behaviour: compiler checks callbacks
-module(my_image_processor).
-behaviour(pipeline_stage).

%% If any of these are missing, compiler warns:
%% "Warning: undefined callback function process/2 (behaviour 'pipeline_stage')"

-export([name/0, process/2, validate_config/1, describe/0]).

name() -> image_resize.

process(#{path := Path, format := Fmt}, #{width := W, height := H}) ->
    %% Actual image resizing logic
    case resize_image(Path, W, H, Fmt) of
        {ok, OutPath} -> {ok, #{path => OutPath, width => W, height => H}};
        {error, E}    -> {error, E}
    end;
process(_Input, _Config) ->
    {error, invalid_input}.

validate_config(#{width := W, height := H})
  when is_integer(W), W > 0,
       is_integer(H), H > 0 -> ok;
validate_config(_) ->
    {error, "width and height must be positive integers"}.

describe() ->
    <<"Resize images to specified dimensions">>.

resize_image(_Path, _W, _H, _Fmt) -> {ok, "/tmp/resized.jpg"}.  %% stub
```

---

## 7. แบบฝึกหัด

1. สร้าง `cache_backend` behaviour ที่รองรับ ETS, Redis, Memcached backends
2. Implement `auth_provider` behaviour สำหรับ JWT, API key, OAuth
3. เพิ่ม behaviour ที่ตรวจสอบว่า callback module มี module-level doc ครบ
4. สร้าง test helper ที่ generate mock module สำหรับ behaviour ใดก็ได้

---

## สรุป Part 90

✅ Behaviour concept: generic loop + specific callbacks = reusable protocol  
✅ `-callback` syntax: types, optional callbacks, arities  
✅ Generic worker template: init, handle_job, timeout, max_jobs defaults  
✅ HTTP handler behaviour: cowboy dispatch wrapper with method check  
✅ Pipeline stage behaviour: process, skip, error handling with history  
✅ Compile-time validation: compiler enforces callback contract  

---

*Part 90/100 | [← ก่อนหน้า](../part89/README.md) | [ถัดไป →](../part91/README.md)*
