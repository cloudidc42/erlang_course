# Part 17: gen_statem — State Machine Behaviour

> **"gen_statem models stateful protocols naturally"**  
> gen_statem จำลอง stateful protocols ได้อย่างเป็นธรรมชาติ

---

## สารบัญ

1. [State Machine คืออะไร?](#1-state-machine-คืออะไร)
2. [gen_statem Callbacks](#2-gen_statem-callbacks)
3. [Callback Mode](#3-callback-mode)
4. [State Functions (state_functions)](#4-state-functions-state_functions)
5. [handle_event_function](#5-handle_event_function)
6. [Event Types](#6-event-types)
7. [Actions](#7-actions)
8. [Timers ใน gen_statem](#8-timers-ใน-gen_statem)
9. [ตัวอย่างจริง](#9-ตัวอย่างจริง)
10. [แบบฝึกหัด](#10-แบบฝึกหัด)

---

## 1. State Machine คืออะไร?

```
State Machine:
├── States: สถานะที่เป็นไปได้
├── Events: สิ่งที่ trigger state change
├── Transitions: rules ว่า state ไหนไปสถานะไหนได้
└── Actions: สิ่งที่ทำเมื่อ transition เกิด

ตัวอย่าง: Traffic Light
States:  red, green, yellow
Events:  timer_expired
Transitions:
  red    + timer → green
  green  + timer → yellow
  yellow + timer → red
```

---

## 2. gen_statem Callbacks

```erlang
-module(my_statem).
-behaviour(gen_statem).

%% Required callbacks
-export([init/1, callback_mode/0]).

%% State functions (ถ้า callback_mode = state_functions)
-export([idle/3, active/3]).

%% หรือ handle_event (ถ้า callback_mode = handle_event_function)
-export([handle_event/4]).

%% Optional
-export([terminate/3, code_change/4, format_status/1]).

%% start
gen_statem:start_link({local, ?MODULE}, ?MODULE, Args, Opts).
```

---

## 3. Callback Mode

```erlang
%% 2 modes:
%% 1. state_functions  — แต่ละ state มีฟังก์ชันของตัวเอง
%% 2. handle_event_function — ฟังก์ชันเดียว handle ทุก state

callback_mode() -> state_functions.
%% หรือ
callback_mode() -> handle_event_function.

%% State+Enter actions: บอกว่าให้เรียก callback เมื่อ enter state
callback_mode() -> [state_functions, state_enter].
callback_mode() -> [handle_event_function, state_enter].
```

---

## 4. State Functions (state_functions)

```erlang
%% ชื่อฟังก์ชัน = ชื่อ state (atom)
%% State(EventType, EventContent, Data)

idle(enter, _OldState, Data) ->
    %% เรียกเมื่อ enter state idle (ต้องใช้ state_enter)
    {keep_state, Data};

idle({call, From}, connect, Data) ->
    {next_state, connecting, Data#{caller => From}};

idle({call, From}, _Req, Data) ->
    {keep_state, Data, [{reply, From, {error, not_connected}}]};

idle(cast, _Msg, Data) ->
    {keep_state, Data};

idle(info, _Info, Data) ->
    {keep_state, Data}.

%% connecting state
connecting(enter, _OldState, Data) ->
    %% start connection attempt
    self() ! try_connect,
    {keep_state, Data};

connecting(info, try_connect, Data) ->
    case attempt_connect() of
        {ok, Conn} ->
            Caller = maps:get(caller, Data),
            gen_statem:reply(Caller, {ok, connected}),
            {next_state, connected, Data#{conn => Conn}};
        {error, Reason} ->
            {next_state, idle, maps:remove(caller, Data),
             [{reply, maps:get(caller, Data), {error, Reason}}]}
    end;

%% connected state
connected(enter, _OldState, _Data) ->
    {keep_state_and_data};

connected({call, From}, {send, Msg}, #{conn := Conn} = Data) ->
    Result = send_message(Conn, Msg),
    {keep_state, Data, [{reply, From, Result}]};

connected({call, From}, disconnect, #{conn := Conn} = Data) ->
    close_connection(Conn),
    {next_state, idle, maps:remove(conn, Data),
     [{reply, From, ok}]}.
```

---

## 5. handle_event_function

```erlang
%% ฟังก์ชันเดียวรับทุก event
handle_event(EventType, EventContent, State, Data) ->
    handle(State, EventType, EventContent, Data).

%% pattern match บน State
handle(idle, {call, From}, connect, Data) ->
    {next_state, connecting, Data#{caller => From}};

handle(connecting, info, connected, Data) ->
    Caller = maps:get(caller, Data),
    {next_state, connected, Data,
     [{reply, Caller, ok}]};

handle(connected, {call, From}, {send, Msg}, Data) ->
    do_send(Msg, maps:get(conn, Data)),
    {keep_state, Data, [{reply, From, ok}]};

handle(_State, _EventType, _EventContent, Data) ->
    %% catch-all
    {keep_state, Data}.
```

---

## 6. Event Types

```erlang
%% Event Types:
%% {call, From}    — synchronous call (ต้อง reply)
%% cast            — async cast
%% info            — erlang message (! operator)
%% internal        — generated internally
%% enter           — entering state (ถ้า state_enter)
%% timeout         — state timeout
%% {timeout, Name} — named timeout
%% state_timeout   — timeout ใน state ปัจจุบัน

%% ตัวอย่าง
my_state({call, From}, Request, Data) ->
    Reply = process(Request),
    {keep_state, Data, [{reply, From, Reply}]};

my_state(cast, Message, Data) ->
    {keep_state, handle_async(Message, Data)};

my_state(info, {'DOWN', Ref, process, _Pid, Reason}, Data) ->
    handle_down(Ref, Reason, Data);

my_state(timeout, Timeout, Data) ->
    handle_timeout(Timeout, Data);

my_state(state_timeout, _Content, Data) ->
    %% state timeout ถูก trigger
    {next_state, idle, Data}.
```

---

## 7. Actions

```erlang
%% Actions คือ list ที่ส่งกลับใน transition

%% Reply to call
{keep_state, Data, [{reply, From, Reply}]}

%% Postpone event (เก็บไว้ handle ใน state ต่อไป)
{keep_state, Data, [postpone]}

%% Next event (inject event เข้า queue)
{keep_state, Data, [{next_event, internal, my_event}]}

%% State timeout (reset ทุกครั้งที่ enter state)
{keep_state, Data, [{state_timeout, 5000, timeout_event}]}

%% Named timeout
{keep_state, Data, [{{timeout, my_timer}, 3000, my_event}]}

%% Cancel timeout
{keep_state, Data, [{{timeout, my_timer}, cancel}]}

%% Hibernate
{keep_state, Data, [hibernate]}

%% Return values:
%% {next_state, NewState, NewData}
%% {next_state, NewState, NewData, Actions}
%% {keep_state, NewData}
%% {keep_state, NewData, Actions}
%% {keep_state_and_data}
%% {keep_state_and_data, Actions}
%% {repeat_state, NewData}
%% {repeat_state, NewData, Actions}
%% {repeat_state_and_data}
%% {stop, Reason}
%% {stop, Reason, Actions}
```

---

## 8. Timers ใน gen_statem

```erlang
%% State timeout: reset ทุกครั้งที่ enter state
my_state(enter, _OldState, Data) ->
    {keep_state, Data, [{state_timeout, 30000, idle_timeout}]};

my_state(state_timeout, idle_timeout, Data) ->
    logger:info("State timeout in my_state"),
    {next_state, idle, Data};

%% Named timeout: ควบคุมได้อิสระ
my_state(enter, _OldState, Data) ->
    {keep_state, Data, [{{timeout, heartbeat}, 5000, ping}]};

my_state({timeout, heartbeat}, ping, Data) ->
    send_heartbeat(),
    {keep_state, Data, [{{timeout, heartbeat}, 5000, ping}]};

%% Cancel timer
my_state({call, From}, stop_heartbeat, Data) ->
    {keep_state, Data, [
        {{timeout, heartbeat}, cancel},
        {reply, From, ok}
    ]};

%% Generic timeout (same as erlang:send_after)
my_state(enter, _OldState, Data) ->
    {keep_state, Data, [{timeout, 1000, check}]};

my_state(timeout, check, Data) ->
    do_check(),
    {keep_state, Data}.
```

---

## 9. ตัวอย่างจริง

### Traffic Light

```erlang
-module(traffic_light).
-behaviour(gen_statem).

-export([start_link/0, get_state/0]).
-export([init/1, callback_mode/0, red/3, green/3, yellow/3, terminate/3]).

-define(RED_TIME,    10000).
-define(GREEN_TIME,   8000).
-define(YELLOW_TIME,  2000).

start_link() ->
    gen_statem:start_link({local, ?MODULE}, ?MODULE, [], []).

get_state() ->
    gen_statem:call(?MODULE, get_state).

callback_mode() -> [state_functions, state_enter].

init([]) ->
    {ok, red, #{}}.

%% Red state
red(enter, _OldState, Data) ->
    io:format("🔴 RED~n"),
    {keep_state, Data, [{state_timeout, ?RED_TIME, next}]};
red({call, From}, get_state, Data) ->
    {keep_state, Data, [{reply, From, red}]};
red(state_timeout, next, Data) ->
    {next_state, green, Data}.

%% Green state
green(enter, _OldState, Data) ->
    io:format("🟢 GREEN~n"),
    {keep_state, Data, [{state_timeout, ?GREEN_TIME, next}]};
green({call, From}, get_state, Data) ->
    {keep_state, Data, [{reply, From, green}]};
green(state_timeout, next, Data) ->
    {next_state, yellow, Data}.

%% Yellow state
yellow(enter, _OldState, Data) ->
    io:format("🟡 YELLOW~n"),
    {keep_state, Data, [{state_timeout, ?YELLOW_TIME, next}]};
yellow({call, From}, get_state, Data) ->
    {keep_state, Data, [{reply, From, yellow}]};
yellow(state_timeout, next, Data) ->
    {next_state, red, Data}.

terminate(_Reason, _State, _Data) -> ok.
```

### TCP Connection Manager

```erlang
-module(tcp_conn).
-behaviour(gen_statem).

-export([start_link/2, send/2, disconnect/1]).
-export([init/1, callback_mode/0, disconnected/3, connecting/3, connected/3]).

start_link(Host, Port) ->
    gen_statem:start_link(?MODULE, {Host, Port}, []).

send(Pid, Data) ->
    gen_statem:call(Pid, {send, Data}).

disconnect(Pid) ->
    gen_statem:call(Pid, disconnect).

callback_mode() -> [state_functions, state_enter].

init({Host, Port}) ->
    {ok, disconnected, #{host => Host, port => Port, retries => 0}}.

%% Disconnected
disconnected(enter, _, Data) ->
    self() ! connect,
    {keep_state, Data};

disconnected(info, connect, #{host := H, port := P} = Data) ->
    case gen_tcp:connect(H, P, [binary, {packet, 0}], 5000) of
        {ok, Socket} ->
            {next_state, connected, Data#{socket => Socket, retries => 0}};
        {error, Reason} ->
            logger:warning("Connect failed: ~p", [Reason]),
            Retries = maps:get(retries, Data),
            Delay = min(30000, 1000 * (1 bsl Retries)),
            {keep_state, Data#{retries => Retries+1},
             [{state_timeout, Delay, reconnect}]}
    end;

disconnected(state_timeout, reconnect, Data) ->
    self() ! connect,
    {keep_state, Data};

disconnected({call, From}, _Req, Data) ->
    {keep_state, Data, [{reply, From, {error, disconnected}}]}.

%% Connected
connected(enter, _, _Data) ->
    keep_state_and_data;

connected({call, From}, {send, Payload}, #{socket := S} = Data) ->
    case gen_tcp:send(S, Payload) of
        ok    -> {keep_state, Data, [{reply, From, ok}]};
        Error -> {next_state, disconnected, maps:remove(socket, Data),
                  [{reply, From, Error}]}
    end;

connected({call, From}, disconnect, #{socket := S} = Data) ->
    gen_tcp:close(S),
    {next_state, disconnected, maps:remove(socket, Data),
     [{reply, From, ok}]};

connected(info, {tcp, _S, Data}, State) ->
    handle_data(Data),
    {keep_state, State};

connected(info, {tcp_closed, _S}, Data) ->
    logger:warning("Connection closed"),
    {next_state, disconnected, maps:remove(socket, Data)};

connected(info, {tcp_error, _S, Reason}, Data) ->
    logger:error("TCP error: ~p", [Reason]),
    {next_state, disconnected, maps:remove(socket, Data)}.

handle_data(Data) ->
    io:format("Received: ~p~n", [Data]).
```

---

## 10. แบบฝึกหัด

### Exercise: Order State Machine

```erlang
%% States: pending → confirmed → shipped → delivered
%%         pending → cancelled
%%         confirmed → cancelled

-module(order_fsm).
-behaviour(gen_statem).

-export([new/1, confirm/1, ship/1, deliver/1, cancel/1, status/1]).
-export([init/1, callback_mode/0]).
-export([pending/3, confirmed/3, shipped/3, delivered/3, cancelled/3]).

new(OrderId) ->
    gen_statem:start(?MODULE, OrderId, []).

confirm(Pid)  -> gen_statem:call(Pid, confirm).
ship(Pid)     -> gen_statem:call(Pid, ship).
deliver(Pid)  -> gen_statem:call(Pid, deliver).
cancel(Pid)   -> gen_statem:call(Pid, cancel).
status(Pid)   -> gen_statem:call(Pid, status).

callback_mode() -> [state_functions].

init(OrderId) ->
    {ok, pending, #{id => OrderId, history => []}}.

pending({call, From}, confirm, Data) ->
    {next_state, confirmed, add_event(confirmed, Data),
     [{reply, From, ok}]};
pending({call, From}, cancel, Data) ->
    {next_state, cancelled, add_event(cancelled, Data),
     [{reply, From, ok}]};
pending({call, From}, status, Data) ->
    {keep_state, Data, [{reply, From, pending}]};
pending({call, From}, _, Data) ->
    {keep_state, Data, [{reply, From, {error, invalid_transition}}]}.

confirmed({call, From}, ship, Data) ->
    {next_state, shipped, add_event(shipped, Data),
     [{reply, From, ok}]};
confirmed({call, From}, cancel, Data) ->
    {next_state, cancelled, add_event(cancelled, Data),
     [{reply, From, ok}]};
confirmed({call, From}, status, Data) ->
    {keep_state, Data, [{reply, From, confirmed}]};
confirmed({call, From}, _, Data) ->
    {keep_state, Data, [{reply, From, {error, invalid_transition}}]}.

shipped({call, From}, deliver, Data) ->
    {next_state, delivered, add_event(delivered, Data),
     [{reply, From, ok}]};
shipped({call, From}, status, Data) ->
    {keep_state, Data, [{reply, From, shipped}]};
shipped({call, From}, _, Data) ->
    {keep_state, Data, [{reply, From, {error, invalid_transition}}]}.

delivered({call, From}, status, Data) ->
    {keep_state, Data, [{reply, From, delivered}]};
delivered({call, From}, _, Data) ->
    {keep_state, Data, [{reply, From, {error, order_completed}}]}.

cancelled({call, From}, status, Data) ->
    {keep_state, Data, [{reply, From, cancelled}]};
cancelled({call, From}, _, Data) ->
    {keep_state, Data, [{reply, From, {error, order_cancelled}}]}.

add_event(Event, #{history := H} = Data) ->
    Data#{history => [{Event, erlang:timestamp()} | H]}.
```

---

## สรุป Part 17

✅ State machine concept  
✅ gen_statem behaviour  
✅ Callback modes: state_functions vs handle_event_function  
✅ State functions syntax  
✅ Event types: call, cast, info, internal, timeout  
✅ Actions: reply, postpone, next_event, timeouts  
✅ State timers vs named timers  
✅ ตัวอย่าง: Traffic light, TCP connection, Order FSM

---

*Part 17/100 | [← ก่อนหน้า](../part16/README.md) | [ถัดไป →](../part18/README.md)*
