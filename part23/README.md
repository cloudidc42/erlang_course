# Part 23: WebSockets

> **"WebSockets enable real-time, full-duplex communication between client and server"**  
> WebSockets ช่วยให้ client และ server สื่อสารแบบ real-time, full-duplex ได้

---

## สารบัญ

1. [WebSocket คืออะไร?](#1-websocket-คืออะไร)
2. [Cowboy WebSocket Handler](#2-cowboy-websocket-handler)
3. [WebSocket Lifecycle](#3-websocket-lifecycle)
4. [Sending/Receiving Messages](#4-sendingreceiving-messages)
5. [Broadcast Pattern](#5-broadcast-pattern)
6. [Chat Application](#6-chat-application)
7. [Heartbeat/Ping-Pong](#7-heartbeatping-pong)
8. [Authentication](#8-authentication)
9. [แบบฝึกหัด](#9-แบบฝึกหัด)

---

## 1. WebSocket คืออะไร?

```
WebSocket:
├── Protocol เหนือ TCP
├── Upgrade จาก HTTP
├── Full-duplex: ส่งได้ทั้ง client และ server
├── Persistent connection
├── Low latency
└── ใช้สำหรับ: chat, live updates, games, dashboards

vs HTTP:
HTTP:      Client → Request → Server → Response (one direction, close)
WebSocket: Client ←→ Server (bi-directional, persistent)
```

---

## 2. Cowboy WebSocket Handler

```erlang
%% src/ws_handler.erl
-module(ws_handler).
-export([init/2, websocket_init/1, websocket_handle/2,
         websocket_info/2, terminate/3]).

%% Route
%% {"/ws", ws_handler, []}

%% HTTP → WebSocket upgrade
init(Req, State) ->
    {cowboy_websocket, Req, State,
     #{idle_timeout => 60000,    %% 60 second idle timeout
       compress => true}}.        %% per-message deflate

%% เรียกหลัง upgrade สำเร็จ
websocket_init(State) ->
    io:format("Client connected: ~p~n", [self()]),
    {ok, State}.

%% รับ message จาก client
websocket_handle({text, Data}, State) ->
    io:format("Received text: ~s~n", [Data]),
    Reply = <<"Echo: ", Data/binary>>,
    {[{text, Reply}], State};

websocket_handle({binary, Data}, State) ->
    %% handle binary data
    {[{binary, Data}], State};

websocket_handle(ping, State) ->
    {[pong], State};    %% reply to ping

websocket_handle(_Frame, State) ->
    {ok, State}.

%% รับ Erlang messages (จาก processes อื่น)
websocket_info({message, Text}, State) ->
    {[{text, Text}], State};

websocket_info(stop, State) ->
    {[{close, 1000, <<"Goodbye">>}], State};

websocket_info(_Info, State) ->
    {ok, State}.

%% Cleanup เมื่อ disconnect
terminate(Reason, _Req, _State) ->
    io:format("Client disconnected: ~p~n", [Reason]),
    ok.
```

---

## 3. WebSocket Lifecycle

```erlang
%% Full lifecycle:

%% 1. init/2 — HTTP request comes in
init(Req, State) ->
    %% Validate, setup initial state
    Params = cowboy_req:parse_qs(Req),
    UserId = get_user_id(Req),  %% auth check
    {cowboy_websocket, Req, #{user_id => UserId},
     #{idle_timeout => 30000}}.

%% 2. websocket_init/1 — After upgrade
websocket_init(State = #{user_id := UserId}) ->
    %% Register this connection
    ws_registry:register(UserId, self()),
    %% Send welcome message
    Welcome = jsx:encode(#{type => <<"connected">>,
                           user_id => UserId}),
    {[{text, Welcome}], State}.

%% 3. websocket_handle/2 — Client sends data
websocket_handle({text, Data}, State) ->
    handle_message(jsx:decode(Data, [return_maps]), State);
websocket_handle({ping, _Data}, State) ->
    {[pong], State}.

%% 4. websocket_info/2 — Process messages
websocket_info({broadcast, Msg}, State) ->
    {[{text, jsx:encode(Msg)}], State};
websocket_info(timeout, State) ->
    %% Handle idle timeout
    {[{close, 1000, <<"Idle timeout">>}], State}.

%% 5. terminate/3 — Connection closed
terminate(_Reason, _Req, #{user_id := UserId}) ->
    ws_registry:unregister(UserId),
    ok.
```

---

## 4. Sending/Receiving Messages

```erlang
%% Frame types
%% {text, Binary}   — UTF-8 text
%% {binary, Binary} — binary data
%% {close, Code, Reason} — close connection
%% {ping, Data}     — ping frame
%% {pong, Data}     — pong frame
%% close            — close without reason

%% Send single frame
{[{text, <<"hello">>}], State}

%% Send multiple frames
{[{text, <<"msg1">>}, {text, <<"msg2">>}], State}

%% Close connection
{[{close, 1000, <<"Normal closure">>}], State}

%% Close codes:
%% 1000 — Normal closure
%% 1001 — Going away
%% 1002 — Protocol error
%% 1003 — Unsupported data
%% 1008 — Policy violation
%% 1011 — Internal server error

%% ส่งจาก external process
Pid ! {message, <<"Hello from server">>}.
%% websocket_info รับ แล้วส่ง {[{text, ...}], State}

%% Handle text message
handle_message(#{<<"type">> := <<"ping">>}, State) ->
    {[{text, jsx:encode(#{type => <<"pong">>})}], State};

handle_message(#{<<"type">> := <<"chat">>,
                 <<"text">> := Text}, State) ->
    chat_room:broadcast(Text, maps:get(user_id, State)),
    {ok, State};

handle_message(Msg, State) ->
    logger:warning("Unknown message: ~p", [Msg]),
    {ok, State}.
```

---

## 5. Broadcast Pattern

```erlang
%% WebSocket Registry สำหรับ broadcast
-module(ws_registry).
-behaviour(gen_server).
-export([start_link/0, register/2, unregister/1, broadcast/1, send_to/2]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2, terminate/2]).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

register(UserId, Pid) ->
    gen_server:cast(?MODULE, {register, UserId, Pid}).

unregister(UserId) ->
    gen_server:cast(?MODULE, {unregister, UserId}).

broadcast(Message) ->
    gen_server:cast(?MODULE, {broadcast, Message}).

send_to(UserId, Message) ->
    gen_server:cast(?MODULE, {send_to, UserId, Message}).

init([]) ->
    {ok, #{}}.   %% #{UserId => Pid}

handle_call(_Req, _From, State) ->
    {reply, ok, State}.

handle_cast({register, UserId, Pid}, State) ->
    monitor(process, Pid),
    {noreply, State#{UserId => Pid}};

handle_cast({unregister, UserId}, State) ->
    {noreply, maps:remove(UserId, State)};

handle_cast({broadcast, Message}, State) ->
    Encoded = jsx:encode(Message),
    maps:foreach(fun(_, Pid) ->
        Pid ! {broadcast, Encoded}
    end, State),
    {noreply, State};

handle_cast({send_to, UserId, Message}, State) ->
    case maps:find(UserId, State) of
        {ok, Pid} ->
            Pid ! {message, jsx:encode(Message)};
        error ->
            ok
    end,
    {noreply, State}.

handle_info({'DOWN', _Ref, process, Pid, _}, State) ->
    %% Remove dead connections
    NewState = maps:filter(fun(_, V) -> V =/= Pid end, State),
    {noreply, NewState}.

terminate(_Reason, _State) ->
    ok.
```

---

## 6. Chat Application

```erlang
%% Chat room ที่สมบูรณ์

%% chat_room.erl — manages rooms
-module(chat_room).
-behaviour(gen_server).
-export([start_link/1, join/2, leave/2, send/3, get_history/1]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2, terminate/2]).

start_link(RoomId) ->
    gen_server:start_link(?MODULE, RoomId, []).

join(Room, UserId) ->
    gen_server:call(Room, {join, UserId, self()}).

leave(Room, UserId) ->
    gen_server:cast(Room, {leave, UserId}).

send(Room, UserId, Text) ->
    gen_server:cast(Room, {send, UserId, Text}).

get_history(Room) ->
    gen_server:call(Room, get_history).

init(RoomId) ->
    {ok, #{
        id      => RoomId,
        members => #{},     %% #{UserId => Pid}
        history => []       %% [{UserId, Text, Timestamp}]
    }}.

handle_call({join, UserId, Pid}, _From,
            #{members := M, history := H} = State) ->
    monitor(process, Pid),
    %% Notify existing members
    Msg = #{type => joined, user_id => UserId},
    broadcast_to(maps:values(M), Msg),
    {reply, {ok, H}, State#{members => M#{UserId => Pid}}};

handle_call(get_history, _From, #{history := H} = State) ->
    {reply, {ok, H}, State}.

handle_cast({leave, UserId}, #{members := M} = State) ->
    Pid = maps:get(UserId, M, undefined),
    case Pid of
        undefined -> {noreply, State};
        _ ->
            NewM = maps:remove(UserId, M),
            broadcast_to(maps:values(NewM), #{type => left, user_id => UserId}),
            {noreply, State#{members => NewM}}
    end;

handle_cast({send, UserId, Text}, #{members := M, history := H} = State) ->
    Ts  = erlang:system_time(second),
    Msg = #{type => message, user_id => UserId,
            text => Text, timestamp => Ts},
    broadcast_to(maps:values(M), Msg),
    NewH = lists:sublist([{UserId, Text, Ts} | H], 100),  %% keep last 100
    {noreply, State#{history => NewH}}.

handle_info({'DOWN', _Ref, process, Pid, _},
            #{members := M} = State) ->
    %% Find user by Pid and remove
    UserId = maps:fold(fun(K, V, Acc) ->
        case V of Pid -> K; _ -> Acc end
    end, undefined, M),
    case UserId of
        undefined -> {noreply, State};
        _ ->
            NewM = maps:remove(UserId, M),
            broadcast_to(maps:values(NewM), #{type => left, user_id => UserId}),
            {noreply, State#{members => NewM}}
    end.

terminate(_Reason, _State) ->
    ok.

broadcast_to(Pids, Msg) ->
    Encoded = jsx:encode(Msg),
    lists:foreach(fun(Pid) ->
        Pid ! {broadcast, Encoded}
    end, Pids).

%% chat_ws_handler.erl — WebSocket handler สำหรับ chat
-module(chat_ws_handler).
-export([init/2, websocket_init/1, websocket_handle/2,
         websocket_info/2, terminate/3]).

init(Req, _State) ->
    RoomId = cowboy_req:binding(room_id, Req),
    UserId = authenticate(Req),
    {cowboy_websocket, Req, #{room_id => RoomId, user_id => UserId},
     #{idle_timeout => 300000}}.

websocket_init(#{room_id := RoomId, user_id := UserId} = State) ->
    Room = chat_room_registry:get_or_create(RoomId),
    {ok, History} = chat_room:join(Room, UserId),
    %% Send history
    HistoryMsg = jsx:encode(#{type => history, messages => History}),
    {[{text, HistoryMsg}], State#{room => Room}}.

websocket_handle({text, Data}, #{user_id := UserId, room := Room} = State) ->
    case jsx:decode(Data, [return_maps]) of
        #{<<"type">> := <<"message">>, <<"text">> := Text} ->
            chat_room:send(Room, UserId, Text),
            {ok, State};
        _ ->
            {ok, State}
    end.

websocket_info({broadcast, Encoded}, State) ->
    {[{text, Encoded}], State};
websocket_info({message, Msg}, State) ->
    {[{text, Msg}], State}.

terminate(_Reason, _Req, #{user_id := UserId, room := Room}) ->
    chat_room:leave(Room, UserId),
    ok.

authenticate(Req) ->
    Token = cowboy_req:header(<<"authorization">>, Req, <<>>),
    %% JWT validation...
    <<"user_", (integer_to_binary(rand:uniform(1000)))/binary>>.
```

---

## 7. Heartbeat/Ping-Pong

```erlang
%% Heartbeat ป้องกัน connection หลุดจาก idle timeout

websocket_init(State) ->
    erlang:send_after(30000, self(), heartbeat),
    {ok, State}.

websocket_handle({ping, _}, State) ->
    %% Browser sends ping, reply with pong
    {[pong], State};

websocket_handle({pong, _}, State) ->
    %% Received pong from client (response to our ping)
    {ok, State#{last_pong => erlang:monotonic_time(second)}}.

websocket_info(heartbeat, State) ->
    %% Send ping to client
    erlang:send_after(30000, self(), heartbeat),
    LastPong = maps:get(last_pong, State, 0),
    Now = erlang:monotonic_time(second),
    case Now - LastPong > 90 of
        true ->
            %% No pong for 90 seconds, disconnect
            {[{close, 1001, <<"Heartbeat timeout">>}], State};
        false ->
            {[{ping, <<"heartbeat">>}], State#{last_pong => Now}}
    end.
```

---

## 8. Authentication

```erlang
%% Token-based auth สำหรับ WebSocket
%% WebSocket ไม่รองรับ custom headers ใน browser
%% ใช้ query parameter แทน

init(Req, State) ->
    Qs = cowboy_req:parse_qs(Req),
    case lists:keyfind(<<"token">>, 1, Qs) of
        {<<"token">>, Token} ->
            case jwt:verify(Token) of
                {ok, Claims} ->
                    UserId = maps:get(<<"sub">>, Claims),
                    {cowboy_websocket, Req, #{user_id => UserId}};
                {error, _} ->
                    Req2 = cowboy_req:reply(401, #{}, <<"Unauthorized">>, Req),
                    {ok, Req2, State}
            end;
        false ->
            Req2 = cowboy_req:reply(401, #{}, <<"Token required">>, Req),
            {ok, Req2, State}
    end.

%% JavaScript ฝั่ง client:
%% const ws = new WebSocket('wss://api.example.com/ws?token=YOUR_JWT');
```

---

## 9. แบบฝึกหัด

### Exercise: Notification Service

```erlang
%% WebSocket notification service:
%% - User connects ด้วย auth token
%% - Server ส่ง notifications แบบ real-time
%% - User สามารถ subscribe/unsubscribe topics ได้

-module(notification_ws_handler).
-export([init/2, websocket_init/1, websocket_handle/2,
         websocket_info/2, terminate/3]).

init(Req, _State) ->
    case authenticate_ws(Req) of
        {ok, UserId} ->
            {cowboy_websocket, Req, #{user_id => UserId, topics => []},
             #{idle_timeout => 120000}};
        {error, _} ->
            Req2 = cowboy_req:reply(401, #{}, <<"Unauthorized">>, Req),
            {ok, Req2, #{}}
    end.

websocket_init(#{user_id := UserId} = State) ->
    notification_registry:register(UserId, self()),
    Welcome = jsx:encode(#{type => <<"welcome">>, user_id => UserId}),
    {[{text, Welcome}], State}.

websocket_handle({text, Data}, State) ->
    case safe_decode(Data) of
        {ok, #{<<"type">> := <<"subscribe">>, <<"topic">> := Topic}} ->
            Topics = [Topic | maps:get(topics, State, [])],
            notification_registry:subscribe(maps:get(user_id, State), Topic),
            Reply = jsx:encode(#{type => <<"subscribed">>, topic => Topic}),
            {[{text, Reply}], State#{topics => Topics}};

        {ok, #{<<"type">> := <<"unsubscribe">>, <<"topic">> := Topic}} ->
            Topics = lists:delete(Topic, maps:get(topics, State, [])),
            notification_registry:unsubscribe(maps:get(user_id, State), Topic),
            Reply = jsx:encode(#{type => <<"unsubscribed">>, topic => Topic}),
            {[{text, Reply}], State#{topics => Topics}};

        _ ->
            {ok, State}
    end;
websocket_handle(_Frame, State) ->
    {ok, State}.

websocket_info({notify, Notification}, State) ->
    Encoded = jsx:encode(Notification),
    {[{text, Encoded}], State}.

terminate(_Reason, _Req, #{user_id := UserId}) ->
    notification_registry:unregister(UserId),
    ok.

authenticate_ws(Req) ->
    Qs = cowboy_req:parse_qs(Req),
    case lists:keyfind(<<"token">>, 1, Qs) of
        {<<"token">>, Token} ->
            jwt_auth:verify(Token);
        false ->
            {error, missing_token}
    end.

safe_decode(Data) ->
    try {ok, jsx:decode(Data, [return_maps])}
    catch _:_ -> {error, invalid_json}
    end.
```

---

## สรุป Part 23

✅ WebSocket คืออะไรและ use cases  
✅ Cowboy WebSocket handler  
✅ Lifecycle: init, websocket_init, handle, info, terminate  
✅ Frame types: text, binary, ping, pong, close  
✅ Broadcast pattern ด้วย registry  
✅ Chat application ครบถ้วน  
✅ Heartbeat/Ping-Pong  
✅ Authentication ผ่าน query parameter

---

*Part 23/100 | [← ก่อนหน้า](../part22/README.md) | [ถัดไป →](../part24/README.md)*
