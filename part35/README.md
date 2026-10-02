# Part 35: Real-World Project — Chat Application

> **"Build real things — theory without practice is empty"**  
> สร้างของจริง — ทฤษฎีที่ไม่ได้ปฏิบัติก็เปล่าประโยชน์

---

## สารบัญ

1. [Architecture Overview](#1-architecture-overview)
2. [Project Structure](#2-project-structure)
3. [Application Supervision Tree](#3-application-supervision-tree)
4. [Room Manager](#4-room-manager)
5. [WebSocket Handler](#5-websocket-handler)
6. [Message Storage](#6-message-storage)
7. [REST API](#7-rest-api)
8. [Frontend Integration](#8-frontend-integration)
9. [Testing](#9-testing)

---

## 1. Architecture Overview

```
Chat Application Architecture:

Client (Browser)
    │
    │  WebSocket (ws://localhost:8080/ws?token=JWT)
    │  REST API  (http://localhost:8080/api/*)
    ▼
[Cowboy HTTP Server]
    │
    ├── [Auth Middleware]
    ├── [Rate Limit Middleware]
    │
    ├── REST Handlers:
    │   ├── /api/rooms    ← list/create rooms
    │   ├── /api/rooms/:id/messages ← message history
    │   └── /api/users    ← user management
    │
    └── WebSocket Handler:
        └── /ws ← real-time connection
            │
            └── [chat_room process per room]
                    │
                    ├── broadcast messages to members
                    ├── track online users
                    └── persist messages to DB
```

---

## 2. Project Structure

```
chat_app/
├── rebar.config
├── config/
│   ├── sys.config
│   └── vm.args
├── src/
│   ├── chat_app.app.src
│   ├── chat_app.erl          ← application callback
│   ├── chat_sup.erl          ← top supervisor
│   ├── room/
│   │   ├── room_manager.erl  ← create/lookup rooms
│   │   ├── room_server.erl   ← per-room GenServer
│   │   └── room_sup.erl      ← dynamic room supervisor
│   ├── ws/
│   │   └── chat_ws_handler.erl ← WebSocket handler
│   ├── api/
│   │   ├── rooms_handler.erl
│   │   ├── messages_handler.erl
│   │   └── users_handler.erl
│   ├── auth/
│   │   ├── auth_middleware.erl
│   │   └── jwt.erl
│   └── db/
│       ├── db.erl             ← pool wrapper
│       ├── message_repo.erl
│       └── user_repo.erl
└── priv/
    └── static/
        └── index.html
```

---

## 3. Application Supervision Tree

```erlang
%% chat_app.erl
-module(chat_app).
-behaviour(application).
-export([start/2, stop/1]).

start(_Type, _Args) ->
    Port = application:get_env(chat_app, port, 8080),
    Routes = chat_router:routes(),
    Dispatch = cowboy_router:compile(Routes),
    {ok, _} = cowboy:start_clear(http,
        [{port, Port}],
        #{
            env => #{dispatch => Dispatch},
            middlewares => [
                auth_middleware,
                cowboy_router,
                cowboy_handler
            ]
        }),
    chat_sup:start_link().

stop(_State) ->
    cowboy:stop_listener(http).

%% chat_sup.erl
-module(chat_sup).
-behaviour(supervisor).
-export([start_link/0, init/1]).

start_link() ->
    supervisor:start_link({local, ?MODULE}, ?MODULE, []).

init([]) ->
    Children = [
        %% Database connection pool
        #{id => db_pool, start => {db, start_link, []},
          restart => permanent, type => supervisor},

        %% Room supervisor (dynamic)
        #{id => room_sup, start => {room_sup, start_link, []},
          restart => permanent, type => supervisor},

        %% Room manager (registry)
        #{id => room_manager, start => {room_manager, start_link, []},
          restart => permanent, type => worker}
    ],
    {ok, {{one_for_one, 5, 10}, Children}}.
```

---

## 4. Room Manager

```erlang
%% room_manager.erl
-module(room_manager).
-behaviour(gen_server).

-export([start_link/0, get_or_create/1, list_rooms/0,
         join/2, leave/2, get_members/1]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

get_or_create(RoomId) ->
    gen_server:call(?MODULE, {get_or_create, RoomId}).

list_rooms() ->
    gen_server:call(?MODULE, list_rooms).

join(RoomId, UserId) ->
    {ok, Pid} = get_or_create(RoomId),
    room_server:join(Pid, UserId).

leave(RoomId, UserId) ->
    case find_room(RoomId) of
        {ok, Pid} -> room_server:leave(Pid, UserId);
        error     -> ok
    end.

get_members(RoomId) ->
    case find_room(RoomId) of
        {ok, Pid} -> room_server:get_members(Pid);
        error     -> []
    end.

init([]) ->
    ets:new(rooms, [named_table, set, protected]),
    {ok, #{}}.

handle_call({get_or_create, RoomId}, _From, S) ->
    case ets:lookup(rooms, RoomId) of
        [{RoomId, Pid}] when is_process_alive(Pid) ->
            {reply, {ok, Pid}, S};
        _ ->
            {ok, Pid} = room_sup:start_room(RoomId),
            ets:insert(rooms, {RoomId, Pid}),
            {reply, {ok, Pid}, S}
    end;

handle_call(list_rooms, _From, S) ->
    Rooms = [{Id, Pid} || [{Id, Pid}] <- ets:match(rooms, '$1'),
             is_process_alive(Pid)],
    {reply, Rooms, S};

handle_info({'DOWN', _Ref, process, Pid, _}, S) ->
    %% Clean up dead room
    ets:match_delete(rooms, {'_', Pid}),
    {noreply, S};
handle_info(_, S) -> {noreply, S}.
handle_cast(_, S) -> {noreply, S}.

find_room(RoomId) ->
    case ets:lookup(rooms, RoomId) of
        [{RoomId, Pid}] -> {ok, Pid};
        []              -> error
    end.
```

---

## 5. WebSocket Handler

```erlang
%% chat_ws_handler.erl
-module(chat_ws_handler).
-export([init/2, websocket_init/1, websocket_handle/2,
         websocket_info/2, terminate/3]).

init(Req, _Opts) ->
    %% Auth check
    Qs = cowboy_req:parse_qs(Req),
    case lists:keyfind(<<"token">>, 1, Qs) of
        {_, Token} ->
            case jwt:verify(Token) of
                {ok, #{<<"sub">> := UserId}} ->
                    State = #{user_id => UserId, rooms => []},
                    {cowboy_websocket, Req, State,
                     #{idle_timeout => 60000}};
                _ ->
                    Req2 = cowboy_req:reply(401, #{}, <<"Unauthorized">>, Req),
                    {ok, Req2, #{}}
            end;
        false ->
            Req2 = cowboy_req:reply(401, #{}, <<"Token required">>, Req),
            {ok, Req2, #{}}
    end.

websocket_init(#{user_id := UserId} = State) ->
    logger:info("WS connected: user=~p", [UserId]),
    {ok, State}.

%% Handle incoming messages from client
websocket_handle({text, Json}, State) ->
    case jsx:decode(Json, [return_maps]) of
        #{<<"type">> := <<"join">>, <<"room">> := RoomId} ->
            handle_join(RoomId, State);
        #{<<"type">> := <<"leave">>, <<"room">> := RoomId} ->
            handle_leave(RoomId, State);
        #{<<"type">> := <<"message">>,
          <<"room">> := RoomId,
          <<"text">> := Text} ->
            handle_message(RoomId, Text, State);
        #{<<"type">> := <<"ping">>} ->
            Reply = jsx:encode(#{type => <<"pong">>}),
            {reply, {text, Reply}, State};
        _ ->
            {ok, State}
    end;
websocket_handle({ping, _}, State) ->
    {reply, pong, State};
websocket_handle(_, State) ->
    {ok, State}.

%% Handle messages from Erlang processes
websocket_info({chat_msg, RoomId, SenderId, Text, Ts}, State) ->
    Msg = jsx:encode(#{
        type    => <<"message">>,
        room    => RoomId,
        user    => SenderId,
        text    => Text,
        time    => Ts
    }),
    {reply, {text, Msg}, State};

websocket_info({user_joined, RoomId, UserId}, State) ->
    Msg = jsx:encode(#{type => <<"user_joined">>,
                       room => RoomId, user => UserId}),
    {reply, {text, Msg}, State};

websocket_info({user_left, RoomId, UserId}, State) ->
    Msg = jsx:encode(#{type => <<"user_left">>,
                       room => RoomId, user => UserId}),
    {reply, {text, Msg}, State};

websocket_info(_, State) ->
    {ok, State}.

terminate(_Reason, _Req, #{user_id := UserId, rooms := Rooms}) ->
    %% Leave all rooms on disconnect
    lists:foreach(fun(RoomId) ->
        room_manager:leave(RoomId, UserId)
    end, Rooms).

handle_join(RoomId, #{user_id := UserId, rooms := Rooms} = State) ->
    room_manager:join(RoomId, UserId),
    %% Send recent message history
    History = message_repo:recent(RoomId, 50),
    HistMsg = jsx:encode(#{type => <<"history">>,
                           room => RoomId, messages => History}),
    State2 = State#{rooms => [RoomId | Rooms]},
    {reply, {text, HistMsg}, State2}.

handle_leave(RoomId, #{user_id := UserId, rooms := Rooms} = State) ->
    room_manager:leave(RoomId, UserId),
    State2 = State#{rooms => lists:delete(RoomId, Rooms)},
    {ok, State2}.

handle_message(RoomId, Text, #{user_id := UserId} = State) ->
    Ts = erlang:system_time(second),
    %% Persist message
    message_repo:save(#{room_id => RoomId, user_id => UserId,
                        text => Text, created => Ts}),
    %% Broadcast to room
    room_manager:broadcast(RoomId, UserId, Text, Ts),
    {ok, State}.
```

---

## 6. Message Storage

```erlang
%% message_repo.erl
-module(message_repo).
-export([save/1, recent/2, by_id/1]).

save(#{room_id:=RoomId, user_id:=UserId, text:=Text, created:=Ts}) ->
    db:query(
        "INSERT INTO messages (room_id, user_id, text, created_at) "
        "VALUES ($1, $2, $3, to_timestamp($4))",
        [RoomId, UserId, Text, Ts]).

recent(RoomId, Limit) ->
    {ok, Rows} = db:query(
        "SELECT m.id, m.text, m.created_at, "
        "       u.username, u.avatar_url "
        "FROM messages m "
        "JOIN users u ON u.id = m.user_id "
        "WHERE m.room_id = $1 "
        "ORDER BY m.created_at DESC "
        "LIMIT $2",
        [RoomId, Limit]),
    lists:reverse(db:rows_to_maps(Rows)).

by_id(Id) ->
    case db:query("SELECT * FROM messages WHERE id = $1", [Id]) of
        {ok, [Row]} -> {ok, db:row_to_map(Row)};
        {ok, []}    -> {error, not_found}
    end.
```

---

## 7. REST API

```erlang
%% rooms_handler.erl
-module(rooms_handler).
-export([init/2]).

init(Req, State) ->
    Method = cowboy_req:method(Req),
    handle(Method, Req, State).

handle(<<"GET">>, Req, State) ->
    Rooms = room_manager:list_rooms(),
    RoomData = [#{id => Id, members => length(room_manager:get_members(Id))}
                || {Id, _Pid} <- Rooms],
    reply_json(200, RoomData, Req, State);

handle(<<"POST">>, Req, State) ->
    {ok, Body, Req2} = cowboy_req:read_body(Req),
    case jsx:decode(Body, [return_maps]) of
        #{<<"name">> := Name} ->
            RoomId = base64:encode(crypto:strong_rand_bytes(8)),
            room_manager:get_or_create(RoomId),
            reply_json(201, #{id => RoomId, name => Name}, Req2, State);
        _ ->
            reply_json(400, #{error => <<"name required">>}, Req2, State)
    end.

reply_json(Status, Data, Req, State) ->
    Req2 = cowboy_req:reply(Status,
        #{<<"content-type">> => <<"application/json">>},
        jsx:encode(Data), Req),
    {ok, Req2, State}.
```

---

## 8. Frontend Integration

```html
<!-- priv/static/index.html -->
<!DOCTYPE html>
<html>
<head>
    <title>Erlang Chat</title>
    <style>
        body { font-family: sans-serif; margin: 20px; }
        #messages { height: 400px; overflow-y: scroll; border: 1px solid #ccc; padding: 10px; }
        .msg { margin: 5px 0; }
        .msg .user { font-weight: bold; color: #333; }
    </style>
</head>
<body>
    <div id="messages"></div>
    <input id="input" type="text" placeholder="Type a message..." style="width:400px">
    <button onclick="sendMsg()">Send</button>

    <script>
    const token = localStorage.getItem('token') || 'demo_token';
    const ws = new WebSocket(`ws://localhost:8080/ws?token=${token}`);
    const currentRoom = 'general';

    ws.onopen = () => {
        ws.send(JSON.stringify({type: 'join', room: currentRoom}));
    };

    ws.onmessage = (e) => {
        const msg = JSON.parse(e.data);
        if (msg.type === 'message') {
            addMessage(msg.user, msg.text);
        } else if (msg.type === 'history') {
            msg.messages.forEach(m => addMessage(m.username, m.text));
        }
    };

    function sendMsg() {
        const text = document.getElementById('input').value;
        if (!text) return;
        ws.send(JSON.stringify({type: 'message', room: currentRoom, text}));
        document.getElementById('input').value = '';
    }

    function addMessage(user, text) {
        const div = document.createElement('div');
        div.className = 'msg';
        div.innerHTML = `<span class="user">${user}:</span> ${text}`;
        const msgs = document.getElementById('messages');
        msgs.appendChild(div);
        msgs.scrollTop = msgs.scrollHeight;
    }

    document.getElementById('input').addEventListener('keypress', e => {
        if (e.key === 'Enter') sendMsg();
    });
    </script>
</body>
</html>
```

---

## 9. Testing

```erlang
%% chat_SUITE.erl — Common Test suite
-module(chat_SUITE).
-include_lib("common_test/include/ct.hrl").
-export([all/0, init_per_suite/1, end_per_suite/1]).
-export([test_join_room/1, test_send_message/1, test_history/1]).

all() -> [test_join_room, test_send_message, test_history].

init_per_suite(Config) ->
    {ok, _} = application:ensure_all_started(chat_app),
    Config.

end_per_suite(_Config) ->
    application:stop(chat_app).

test_join_room(_Config) ->
    RoomId = <<"test_room">>,
    UserId = <<"user1">>,
    ok = room_manager:join(RoomId, UserId),
    Members = room_manager:get_members(RoomId),
    true = lists:member(UserId, Members).

test_send_message(_Config) ->
    RoomId = <<"test_room2">>,
    UserId = <<"user2">>,
    Text   = <<"Hello World">>,
    room_manager:join(RoomId, UserId),
    {ok, _} = message_repo:save(#{
        room_id => RoomId, user_id => UserId,
        text => Text, created => erlang:system_time(second)
    }),
    Messages = message_repo:recent(RoomId, 10),
    [#{text := Text}|_] = Messages.

test_history(_Config) ->
    RoomId = <<"history_room">>,
    Messages = message_repo:recent(RoomId, 50),
    true = is_list(Messages).
```

---

## สรุป Part 35

✅ Chat application architecture  
✅ Supervision tree design  
✅ Room manager (ETS registry + dynamic supervisor)  
✅ WebSocket handler (join/leave/message)  
✅ Message persistence  
✅ REST API for rooms  
✅ Frontend HTML/JS  
✅ Common Test suite

---

*Part 35/100 | [← ก่อนหน้า](../part34/README.md) | [ถัดไป →](../part36/README.md)*
