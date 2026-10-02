# Part 12: GenServer — Generic Server Behaviour

> **"OTP's GenServer is the cornerstone of Erlang applications"**  
> GenServer ของ OTP คือรากฐานของ Erlang applications

---

## สารบัญ

1. [GenServer คืออะไร?](#1-genserver-คืออะไร)
2. [Callbacks ทั้งหมด](#2-callbacks-ทั้งหมด)
3. [start_link และ init](#3-start_link-และ-init)
4. [handle_call — Synchronous](#4-handle_call--synchronous)
5. [handle_cast — Asynchronous](#5-handle_cast--asynchronous)
6. [handle_info](#6-handle_info)
7. [State Management](#7-state-management)
8. [Named GenServer](#8-named-genserver)
9. [Timeouts และ Timers](#9-timeouts-และ-timers)
10. [GenServer ใน Production](#10-genserver-ใน-production)
11. [แบบฝึกหัด](#11-แบบฝึกหัด)

---

## 1. GenServer คืออะไร?

```
GenServer = Generic Server
- OTP behaviour (framework)
- แยก business logic จาก concurrency code
- มี standard callbacks: init, handle_call, handle_cast, handle_info, terminate
- ใช้กับ Supervisor ได้ง่าย
- Hot code swapping
- Tracing, statistics ฟรี
```

### ทำไมต้องใช้ GenServer?

```erlang
%% WITHOUT GenServer — ต้องเขียน boilerplate เอง
start() ->
    Pid = spawn_link(fun() -> loop(initial_state()) end),
    register(my_server, Pid),
    {ok, Pid}.

loop(State) ->
    receive
        {call, From, Ref, Req} ->
            {Reply, NewState} = handle(Req, State),
            From ! {Ref, Reply},
            loop(NewState);
        stop -> ok
    end.

%% WITH GenServer — clean, standardized
-behaviour(gen_server).
%% GenServer จัดการ boilerplate ให้ทั้งหมด
```

---

## 2. Callbacks ทั้งหมด

```erlang
%% Mandatory
init(Args) ->
    {ok, State}.

handle_call(Request, From, State) ->
    {reply, Reply, NewState}.

handle_cast(Request, State) ->
    {noreply, NewState}.

%% Optional
handle_info(Info, State) ->
    {noreply, NewState}.

terminate(Reason, State) ->
    ok.

code_change(OldVsn, State, Extra) ->
    {ok, NewState}.
```

---

## 3. start_link และ init

```erlang
-module(my_server).
-behaviour(gen_server).

-export([start_link/0, start_link/1]).
-export([init/1, handle_call/3, handle_cast/2,
         handle_info/2, terminate/2, code_change/3]).

%% API
start_link() ->
    start_link(#{}).

start_link(Opts) ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, Opts, []).
%%                        ^Name             ^Module ^Args  ^Options

%% init — ถูกเรียกครั้งเดียวตอน start
init(Opts) ->
    process_flag(trap_exit, true),
    State = #{
        count => maps:get(initial_count, Opts, 0),
        opts  => Opts
    },
    {ok, State}.

%% init return values:
%% {ok, State}              — start สำเร็จ
%% {ok, State, Timeout}     — start สำเร็จ, ตั้ง timeout
%% {ok, State, hibernate}   — start สำเร็จ, hibernate
%% {stop, Reason}           — start ล้มเหลว
%% ignore                   — ไม่ start (supervisor ignores)
```

---

## 4. handle_call — Synchronous

```erlang
%% Client side
gen_server:call(ServerRef, Request).
gen_server:call(ServerRef, Request, Timeout).  %% default: 5000ms

%% Server side
handle_call(Request, From, State) ->
    %% From = {Pid, Tag}
    Reply = process(Request),
    {reply, Reply, NewState}.

%% handle_call return values:
%% {reply, Reply, NewState}
%% {reply, Reply, NewState, Timeout}
%% {reply, Reply, NewState, hibernate}
%% {noreply, NewState}          — reply เองทีหลังด้วย gen_server:reply/2
%% {noreply, NewState, Timeout}
%% {stop, Reason, Reply, NewState}  — stop server

%% Deferred reply
handle_call({long_operation, Data}, From, State) ->
    spawn(fun() ->
        Result = do_long_operation(Data),
        gen_server:reply(From, Result)
    end),
    {noreply, State}.  %% return ทันที, reply ทีหลัง
```

### ตัวอย่าง

```erlang
%% Cache GenServer
handle_call({get, Key}, _From, #{cache := Cache} = State) ->
    Reply = maps:get(Key, Cache, not_found),
    {reply, Reply, State};

handle_call({put, Key, Value}, _From, #{cache := Cache} = State) ->
    NewState = State#{cache => Cache#{Key => Value}},
    {reply, ok, NewState};

handle_call({delete, Key}, _From, #{cache := Cache} = State) ->
    NewState = State#{cache => maps:remove(Key, Cache)},
    {reply, ok, NewState};

handle_call(clear, _From, State) ->
    {reply, ok, State#{cache => #{}}};

handle_call(Request, _From, State) ->
    {reply, {error, {unknown_request, Request}}, State}.
```

---

## 5. handle_cast — Asynchronous

```erlang
%% Client side (fire and forget)
gen_server:cast(ServerRef, Message).

%% Server side
handle_cast(Message, State) ->
    NewState = process_async(Message, State),
    {noreply, NewState}.

%% handle_cast return values:
%% {noreply, NewState}
%% {noreply, NewState, Timeout}
%% {noreply, NewState, hibernate}
%% {stop, Reason, NewState}

%% ตัวอย่าง
handle_cast({increment, N}, #{count := C} = State) ->
    {noreply, State#{count => C + N}};

handle_cast({log, Level, Msg}, State) ->
    logger:log(Level, Msg),
    {noreply, State};

handle_cast(stop, State) ->
    {stop, normal, State}.
```

---

## 6. handle_info

```erlang
%% รับ messages ที่ไม่ใช่ call หรือ cast
%% เช่น: timer, monitor, EXIT, custom messages

handle_info(Message, State) ->
    NewState = handle_message(Message, State),
    {noreply, NewState}.

%% รับ timer message
handle_info(tick, State) ->
    do_periodic_work(),
    schedule_tick(),
    {noreply, State};

%% รับ DOWN message (process monitor)
handle_info({'DOWN', Ref, process, Pid, Reason},
            #{monitors := Mons} = State) ->
    case maps:find(Ref, Mons) of
        {ok, WorkerId} ->
            handle_worker_down(WorkerId, Reason),
            {noreply, State#{monitors => maps:remove(Ref, Mons)}};
        error ->
            {noreply, State}
    end;

%% รับ EXIT signal (ถ้า trap_exit = true)
handle_info({'EXIT', Pid, Reason}, State) ->
    io:format("Linked process ~p exited: ~p~n", [Pid, Reason]),
    {noreply, State};

%% Catch-all
handle_info(Info, State) ->
    logger:warning("Unexpected message: ~p", [Info]),
    {noreply, State}.
```

---

## 7. State Management

```erlang
%% State ควรเป็น map หรือ record
%% ตัวอย่าง: User Session Server

-record(session_state, {
    sessions :: map(),
    max_sessions :: pos_integer(),
    timeout :: pos_integer()
}).

init(Opts) ->
    State = #session_state{
        sessions     = #{},
        max_sessions = maps:get(max_sessions, Opts, 1000),
        timeout      = maps:get(timeout, Opts, 3600)
    },
    {ok, State}.

handle_call({create_session, UserId}, _From,
            #session_state{sessions = S, max_sessions = Max} = State) ->
    case map_size(S) >= Max of
        true ->
            {reply, {error, too_many_sessions}, State};
        false ->
            Token = generate_token(),
            Expires = now_plus_seconds(State#session_state.timeout),
            Session = #{user_id => UserId, token => Token, expires => Expires},
            NewState = State#session_state{
                sessions = S#{Token => Session}
            },
            {reply, {ok, Token}, NewState}
    end;

handle_call({validate_session, Token}, _From,
            #session_state{sessions = S} = State) ->
    case maps:find(Token, S) of
        {ok, Session} ->
            case is_expired(Session) of
                true  -> {reply, {error, expired}, State};
                false -> {reply, {ok, Session}, State}
            end;
        error ->
            {reply, {error, not_found}, State}
    end.

generate_token() ->
    binary:encode_hex(crypto:strong_rand_bytes(16)).

now_plus_seconds(Seconds) ->
    erlang:system_time(second) + Seconds.

is_expired(#{expires := Expires}) ->
    erlang:system_time(second) > Expires.
```

---

## 8. Named GenServer

```erlang
%% Local name registration
gen_server:start_link({local, my_server}, ?MODULE, [], []).

%% Global name (distributed)
gen_server:start_link({global, my_server}, ?MODULE, [], []).

%% via (custom registry)
gen_server:start_link({via, gproc, {n, l, my_key}}, ?MODULE, [], []).

%% ชื่อ atom = module name
-define(SERVER, ?MODULE).

start_link() ->
    gen_server:start_link({local, ?SERVER}, ?MODULE, [], []).

%% เรียกด้วยชื่อ
gen_server:call(?SERVER, request).
gen_server:cast(?SERVER, message).

%% หรือด้วย PID
Pid = whereis(?SERVER),
gen_server:call(Pid, request).
```

---

## 9. Timeouts และ Timers

```erlang
%% Process-level timeout
handle_call(Request, _From, State) ->
    {reply, ok, State, 5000}.  %% ถ้าไม่มี message ใน 5s → timeout

handle_info(timeout, State) ->
    do_idle_work(),
    {noreply, State}.

%% erlang:send_after
init(Args) ->
    erlang:send_after(1000, self(), tick),
    {ok, #{}}.

handle_info(tick, State) ->
    do_tick_work(State),
    erlang:send_after(1000, self(), tick),
    {noreply, State};

%% timer:send_interval (ส่ง message ทุก N ms)
init(Args) ->
    {ok, TRef} = timer:send_interval(5000, self(), heartbeat),
    {ok, #{timer_ref => TRef}}.

terminate(_Reason, #{timer_ref := TRef}) ->
    timer:cancel(TRef),
    ok.

%% Cancellable timer
-define(IDLE_TIMEOUT, 30000).  %% 30 seconds

schedule_idle_check(State) ->
    Ref = erlang:send_after(?IDLE_TIMEOUT, self(), idle_timeout),
    State#{idle_timer => Ref}.

cancel_idle_check(#{idle_timer := Ref} = State) ->
    erlang:cancel_timer(Ref),
    State#{idle_timer => undefined}.

handle_call(activity, _From, State) ->
    S1 = cancel_idle_check(State),
    S2 = schedule_idle_check(S1),
    {reply, ok, S2}.

handle_info(idle_timeout, State) ->
    logger:info("Server idle, shutting down"),
    {stop, normal, State}.
```

---

## 10. GenServer ใน Production

```erlang
%% terminate — cleanup เมื่อ stop
terminate(normal, State) ->
    save_state(State),
    ok;
terminate(shutdown, State) ->
    save_state(State),
    ok;
terminate({shutdown, _Reason}, State) ->
    save_state(State),
    ok;
terminate(Reason, State) ->
    logger:error("GenServer terminating: ~p, state: ~p", [Reason, State]),
    ok.

%% code_change — hot code upgrade
code_change(OldVsn, State, _Extra) when OldVsn =:= "1.0" ->
    %% migrate state from 1.0 to current version
    NewState = migrate_state_v1_to_v2(State),
    {ok, NewState};
code_change(_OldVsn, State, _Extra) ->
    {ok, State}.

%% Introspection
sys:get_state(ServerRef).     %% ดู state ปัจจุบัน
sys:get_status(ServerRef).    %% ดู status ละเอียด
sys:statistics(ServerRef, true).  %% เปิด statistics
sys:trace(ServerRef, true).   %% เปิด tracing

%% Format status (แสดงใน crash report)
format_status(_Opts, [_PDict, State]) ->
    %% ซ่อน sensitive data
    SafeState = State#{password => <<"***">>},
    [{data, [{"State", SafeState}]}].
```

### Complete Example: Rate Limiter

```erlang
-module(rate_limiter).
-behaviour(gen_server).

-export([start_link/2, check/2]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2, terminate/2]).

%% API
start_link(Name, MaxPerSecond) ->
    gen_server:start_link({local, Name}, ?MODULE,
                          #{max => MaxPerSecond}, []).

check(Name, Key) ->
    gen_server:call(Name, {check, Key}).

%% Callbacks
init(#{max := Max}) ->
    {ok, #{max => Max, buckets => #{}, last_reset => erlang:monotonic_time(second)}}.

handle_call({check, Key}, _From, State) ->
    {Result, NewState} = do_check(Key, State),
    {reply, Result, NewState}.

handle_cast(_Msg, State) ->
    {noreply, State}.

handle_info(_Info, State) ->
    {noreply, State}.

terminate(_Reason, _State) ->
    ok.

%% Internal
do_check(Key, #{max := Max, buckets := Buckets, last_reset := LastReset} = State) ->
    Now = erlang:monotonic_time(second),
    {CleanBuckets, NewLastReset} =
        case Now > LastReset of
            true  -> {#{}, Now};
            false -> {Buckets, LastReset}
        end,
    Count = maps:get(Key, CleanBuckets, 0),
    case Count < Max of
        true ->
            NewBuckets = CleanBuckets#{Key => Count + 1},
            NewState = State#{buckets => NewBuckets, last_reset => NewLastReset},
            {allowed, NewState};
        false ->
            NewState = State#{buckets => CleanBuckets, last_reset => NewLastReset},
            {denied, NewState}
    end.
```

---

## 11. แบบฝึกหัด

### Exercise: Todo List Server

```erlang
-module(todo_server).
-behaviour(gen_server).

-export([start_link/0, add/2, complete/2, delete/2, list/1, stats/1]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2, terminate/2]).

-record(todo, {
    id      :: pos_integer(),
    title   :: binary(),
    done    :: boolean(),
    created :: integer()
}).

%% API
start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

add(Server, Title) ->
    gen_server:call(Server, {add, Title}).

complete(Server, Id) ->
    gen_server:call(Server, {complete, Id}).

delete(Server, Id) ->
    gen_server:call(Server, {delete, Id}).

list(Server) ->
    gen_server:call(Server, list).

stats(Server) ->
    gen_server:call(Server, stats).

%% Callbacks
init([]) ->
    {ok, #{todos => #{}, next_id => 1}}.

handle_call({add, Title}, _From, #{todos := T, next_id := Id} = State) ->
    Todo = #todo{id=Id, title=iolist_to_binary(Title),
                 done=false, created=erlang:system_time(second)},
    {reply, {ok, Id}, State#{todos => T#{Id => Todo}, next_id => Id+1}};

handle_call({complete, Id}, _From, #{todos := T} = State) ->
    case maps:find(Id, T) of
        {ok, Todo} ->
            NewT = T#{Id => Todo#todo{done=true}},
            {reply, ok, State#{todos => NewT}};
        error ->
            {reply, {error, not_found}, State}
    end;

handle_call({delete, Id}, _From, #{todos := T} = State) ->
    case maps:is_key(Id, T) of
        true  -> {reply, ok, State#{todos => maps:remove(Id, T)}};
        false -> {reply, {error, not_found}, State}
    end;

handle_call(list, _From, #{todos := T} = State) ->
    Todos = [V || {_, V} <- maps:to_list(T)],
    Sorted = lists:sort(fun(A, B) -> A#todo.id =< B#todo.id end, Todos),
    {reply, {ok, Sorted}, State};

handle_call(stats, _From, #{todos := T} = State) ->
    All = map_size(T),
    Done = length([1 || #todo{done=true} <- maps:values(T)]),
    {reply, #{total => All, done => Done, pending => All - Done}, State}.

handle_cast(_Msg, State) ->
    {noreply, State}.

handle_info(_Info, State) ->
    {noreply, State}.

terminate(_Reason, _State) ->
    ok.
```

---

## สรุป Part 12

✅ GenServer คืออะไรและทำไมต้องใช้  
✅ Callbacks: init, handle_call, handle_cast, handle_info, terminate  
✅ Synchronous call vs Asynchronous cast  
✅ State management ด้วย map และ record  
✅ Named GenServer  
✅ Timeouts และ timers  
✅ Production considerations: terminate, code_change  
✅ ตัวอย่างจริง: Rate Limiter, Todo Server

---

*Part 12/100 | [← ก่อนหน้า](../part11/README.md) | [ถัดไป →](../part13/README.md)*
