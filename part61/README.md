# Part 61: Advanced Cowboy — Middleware, Streaming, Uploads

> **"A web framework is only as good as its handler pipeline"**  
> Web framework ดีแค่ไหนขึ้นอยู่กับ handler pipeline ของมัน

---

## สารบัญ

1. [Cowboy Router and Dispatch](#1-cowboy-router-and-dispatch)
2. [Middleware Pipeline](#2-middleware-pipeline)
3. [Streaming Responses](#3-streaming-responses)
4. [File Uploads](#4-file-uploads)
5. [WebSocket Advanced Patterns](#5-websocket-advanced-patterns)
6. [Long Polling and SSE](#6-long-polling-and-sse)
7. [แบบฝึกหัด](#7-แบบฝึกหัด)

---

## 1. Cowboy Router and Dispatch

```erlang
%% router.erl — advanced Cowboy routing
-module(router).
-export([routes/0, start/0]).

routes() ->
    [
        %% Static: exact match
        {"/", index_handler, []},
        {"/health", health_handler, []},

        %% Path parameters
        {"/users/:id", user_handler, []},
        {"/users/:user_id/posts/:post_id", post_handler, []},

        %% Wildcard: match anything under path
        {"/static/[...]", cowboy_static, {priv_dir, myapp, "static"}},

        %% Host-based routing (multi-tenant)
        %% Route dispatch from start_listener:
        {'_', "/api/[...]", api_handler, []}
    ].

start() ->
    Dispatch = cowboy_router:compile([
        %% Host pattern, then route list
        {'_', routes()}  %% '_' = any hostname
    ]),
    {ok, _} = cowboy:start_clear(http_listener,
        [{port, 8080}],
        #{
            env         => #{dispatch => Dispatch},
            middlewares => [
                cowboy_router,
                request_id_middleware,
                auth_middleware,
                rate_limit_middleware,
                cowboy_handler
            ],
            stream_handlers => [cowboy_compress_h, cowboy_stream_h]
        }
    ).
```

---

## 2. Middleware Pipeline

```erlang
%% request_id_middleware.erl
-module(request_id_middleware).
-behaviour(cowboy_middleware).
-export([execute/2]).

execute(Req, Env) ->
    RequestId = case cowboy_req:header(<<"x-request-id">>, Req) of
        undefined -> generate_request_id();
        Id        -> Id
    end,
    Req1 = cowboy_req:set_resp_header(<<"x-request-id">>, RequestId, Req),
    Req2 = cowboy_req:set_resp_header(<<"x-correlation-id">>, RequestId, Req1),
    %% Store in process dictionary for logging
    put(request_id, RequestId),
    {ok, Req2, Env}.

generate_request_id() ->
    <<A:32, B:16, C:16, D:16, E:48>> = crypto:strong_rand_bytes(16),
    iolist_to_binary(io_lib:format(
        "~8.16.0b-~4.16.0b-~4.16.0b-~4.16.0b-~12.16.0b",
        [A, B, C, D, E])).
```

```erlang
%% auth_middleware.erl
-module(auth_middleware).
-behaviour(cowboy_middleware).
-export([execute/2]).

-define(PUBLIC_PATHS, [<<"/health">>, <<"/login">>, <<"/register">>]).

execute(Req, Env) ->
    Path = cowboy_req:path(Req),
    case lists:member(Path, ?PUBLIC_PATHS) of
        true  ->
            {ok, Req, Env};
        false ->
            case authenticate(Req) of
                {ok, User} ->
                    Req1 = cowboy_req:set_resp_header(<<"x-user-id">>,
                               integer_to_binary(maps:get(id, User)), Req),
                    {ok, Req1, Env#{current_user => User}};
                {error, Reason} ->
                    Body = jsx:encode(#{error => Reason}),
                    Req1 = cowboy_req:reply(401,
                               #{<<"content-type">> => <<"application/json">>},
                               Body, Req),
                    {stop, Req1}
            end
    end.

authenticate(Req) ->
    case cowboy_req:header(<<"authorization">>, Req) of
        <<"Bearer ", Token/binary>> ->
            jwt_auth:verify(Token);
        _ ->
            {error, <<"missing_token">>}
    end.
```

---

## 3. Streaming Responses

```erlang
%% stream_handler.erl — stream large responses
-module(stream_handler).
-behaviour(cowboy_handler).
-export([init/2]).

%% Stream a large CSV export
init(Req, State) ->
    Req1 = cowboy_req:stream_reply(200,
               #{<<"content-type">> => <<"text/csv">>,
                 <<"content-disposition">> => <<"attachment; filename=export.csv">>},
               Req),
    stream_csv(Req1),
    {ok, Req1, State}.

stream_csv(Req) ->
    %% Write header
    cowboy_req:stream_body(<<"id,name,email\n">>, nofin, Req),
    %% Stream rows in batches
    stream_rows(1, 10000, Req).

stream_rows(Offset, Total, Req) when Offset > Total ->
    cowboy_req:stream_body(<<>>, fin, Req);
stream_rows(Offset, Total, Req) ->
    BatchSize = 500,
    Rows = fetch_rows(Offset, BatchSize),
    Csv  = rows_to_csv(Rows),
    cowboy_req:stream_body(Csv, nofin, Req),
    stream_rows(Offset + BatchSize, Total, Req).

fetch_rows(Offset, Limit) ->
    %% Paginate DB query
    {ok, Rows} = db:query(
        "SELECT id, name, email FROM users ORDER BY id LIMIT $1 OFFSET $2",
        [Limit, Offset]),
    Rows.

rows_to_csv(Rows) ->
    iolist_to_binary([
        [integer_to_binary(maps:get(<<"id">>, R)), $,,
         maps:get(<<"name">>, R), $,,
         maps:get(<<"email">>, R), $\n]
        || R <- Rows
    ]).
```

---

## 4. File Uploads

```erlang
%% upload_handler.erl — multipart file upload
-module(upload_handler).
-behaviour(cowboy_handler).
-export([init/2]).

-define(MAX_FILE_SIZE, 10 * 1024 * 1024).  %% 10 MB

init(Req0, State) ->
    case cowboy_req:method(Req0) of
        <<"POST">> -> handle_upload(Req0, State);
        _          -> {ok, cowboy_req:reply(405, Req0), State}
    end.

handle_upload(Req0, State) ->
    %% Validate content-type
    case cowboy_req:parse_header(<<"content-type">>, Req0) of
        {<<"multipart">>, <<"form-data">>, _} ->
            process_multipart(Req0, State, []);
        _ ->
            Req1 = cowboy_req:reply(415,
                       #{<<"content-type">> => <<"application/json">>},
                       jsx:encode(#{error => <<"must be multipart/form-data">>}),
                       Req0),
            {ok, Req1, State}
    end.

process_multipart(Req0, State, Files) ->
    case cowboy_req:read_part(Req0) of
        {ok, Headers, Req1} ->
            {ok, File, Req2} = read_part_body(Req1, <<>>),
            case parse_content_disposition(Headers) of
                {file, FieldName, FileName} ->
                    case byte_size(File) > ?MAX_FILE_SIZE of
                        true ->
                            Req3 = cowboy_req:reply(413,
                                       #{<<"content-type">> => <<"application/json">>},
                                       jsx:encode(#{error => <<"file too large">>}),
                                       Req2),
                            {ok, Req3, State};
                        false ->
                            SavedPath = save_file(FileName, File),
                            process_multipart(Req2, State,
                                [{FieldName, FileName, SavedPath} | Files])
                    end;
                {field, FieldName} ->
                    logger:info("Field: ~s = ~s", [FieldName, File]),
                    process_multipart(Req2, State, Files)
            end;
        {done, Req1} ->
            respond_upload_done(Req1, State, Files)
    end.

read_part_body(Req0, Acc) ->
    case cowboy_req:read_part_body(Req0, #{length => 65536}) of
        {ok, Data, Req1}   -> {ok, <<Acc/binary, Data/binary>>, Req1};
        {more, Data, Req1} -> read_part_body(Req1, <<Acc/binary, Data/binary>>)
    end.

parse_content_disposition(Headers) ->
    CD = maps:get(<<"content-disposition">>, maps:from_list(Headers), <<>>),
    case cowboy_http:params(CD) of
        Params ->
            Name     = proplists:get_value(<<"name">>, Params, <<"file">>),
            FileName = proplists:get_value(<<"filename">>, Params),
            case FileName of
                undefined -> {field, Name};
                FN        -> {file, Name, FN}
            end
    end.

save_file(FileName, Data) ->
    SafeName = sanitize_filename(FileName),
    Path = filename:join(["/uploads", SafeName]),
    ok   = file:write_file(Path, Data),
    Path.

sanitize_filename(Name) ->
    %% Remove path traversal, keep only basename
    Base = filename:basename(Name),
    re:replace(Base, "[^a-zA-Z0-9._-]", "_", [global, {return, binary}]).

respond_upload_done(Req, State, Files) ->
    Body = jsx:encode(#{
        success => true,
        files   => [#{name => FN, path => P} || {_, FN, P} <- Files]
    }),
    Req1 = cowboy_req:reply(201,
               #{<<"content-type">> => <<"application/json">>},
               Body, Req),
    {ok, Req1, State}.
```

---

## 5. WebSocket Advanced Patterns

```erlang
%% ws_handler.erl — advanced WebSocket handler
-module(ws_handler).
-behaviour(cowboy_websocket).
-export([init/2, websocket_init/1, websocket_handle/2,
         websocket_info/2, terminate/3]).

-record(state, {
    user_id   :: integer(),
    rooms     :: [binary()],
    ping_ref  :: reference()
}).

init(Req, _Opts) ->
    case authenticate_ws(Req) of
        {ok, UserId} ->
            {cowboy_websocket, Req, #{user_id => UserId},
             #{
                 idle_timeout    => 60000,   %% 1 min idle disconnect
                 max_frame_size  => 1024 * 1024,  %% 1MB max frame
                 compress        => true     %% per-message deflate
             }};
        error ->
            Req1 = cowboy_req:reply(401, Req),
            {ok, Req1, #{}}
    end.

websocket_init(#{user_id := UserId}) ->
    %% Subscribe to user's notification channel
    pubsub:subscribe({user, UserId}),
    %% Start ping timer
    PingRef = erlang:send_after(30000, self(), ping),
    State = #state{user_id=UserId, rooms=[], ping_ref=PingRef},
    {[], State}.

websocket_handle({text, Data}, State) ->
    case jsx:decode(Data, [return_maps]) of
        #{<<"type">> := <<"join">>, <<"room">> := Room} ->
            handle_join(Room, State);
        #{<<"type">> := <<"message">>, <<"room">> := Room, <<"text">> := Msg} ->
            handle_message(Room, Msg, State);
        #{<<"type">> := <<"pong">>} ->
            {[], State};
        _ ->
            {[{text, jsx:encode(#{error => <<"unknown_message_type">>})}], State}
    end;
websocket_handle({binary, _Data}, State) ->
    %% Ignore binary frames
    {[], State};
websocket_handle(ping, State) ->
    {[pong], State}.

handle_join(Room, #state{rooms=Rooms, user_id=UserId} = State) ->
    case lists:member(Room, Rooms) of
        true  ->
            {[{text, jsx:encode(#{error => <<"already_in_room">>})}], State};
        false ->
            pubsub:subscribe({room, Room}),
            pubsub:broadcast({room, Room}, #{type => joined, user => UserId}),
            State1 = State#state{rooms=[Room | Rooms]},
            Ack = jsx:encode(#{type => <<"joined">>, room => Room}),
            {[{text, Ack}], State1}
    end.

handle_message(Room, Msg, #state{rooms=Rooms, user_id=UserId} = State) ->
    case lists:member(Room, Rooms) of
        false ->
            {[{text, jsx:encode(#{error => <<"not_in_room">>})}], State};
        true  ->
            pubsub:broadcast({room, Room}, #{
                type => <<"message">>,
                room => Room,
                from => UserId,
                text => Msg,
                ts   => os:system_time(millisecond)
            }),
            {[], State}
    end.

websocket_info(ping, #state{ping_ref=OldRef} = State) ->
    erlang:cancel_timer(OldRef),
    NewRef = erlang:send_after(30000, self(), ping),
    {[{text, jsx:encode(#{type => <<"ping">>})}],
     State#state{ping_ref=NewRef}};
websocket_info({pubsub, Msg}, State) ->
    {[{text, jsx:encode(Msg)}], State};
websocket_info(_Info, State) ->
    {[], State}.

terminate(Reason, _Req, #state{user_id=UserId, rooms=Rooms}) ->
    logger:info("WS disconnected user=~p reason=~p", [UserId, Reason]),
    [pubsub:unsubscribe({room, R}) || R <- Rooms],
    pubsub:unsubscribe({user, UserId}),
    ok.

authenticate_ws(Req) ->
    case cowboy_req:parse_qs(Req) of
        QS ->
            case proplists:get_value(<<"token">>, QS) of
                undefined -> error;
                Token     ->
                    case jwt_auth:verify(Token) of
                        {ok, #{<<"sub">> := UserId}} -> {ok, UserId};
                        _                            -> error
                    end
            end
    end.
```

---

## 6. Long Polling and SSE

```erlang
%% sse_handler.erl — Server-Sent Events handler
-module(sse_handler).
-behaviour(cowboy_handler).
-export([init/2]).

init(Req0, State) ->
    %% SSE headers
    Req1 = cowboy_req:stream_reply(200,
               #{<<"content-type">>  => <<"text/event-stream">>,
                 <<"cache-control">> => <<"no-cache">>,
                 <<"connection">>    => <<"keep-alive">>,
                 <<"x-accel-buffering">> => <<"no">>},  %% disable nginx buffering
               Req0),

    %% Register for events
    EventBus = list_to_atom(
        "events_" ++ binary_to_list(cowboy_req:binding(channel, Req1))),
    pg:join(EventBus, self()),

    %% Send initial "connected" event
    send_sse_event(Req1, <<"connected">>, #{ts => os:system_time(millisecond)}),

    %% Start heartbeat
    HeartbeatRef = erlang:send_after(15000, self(), heartbeat),
    event_loop(Req1, State, EventBus, HeartbeatRef).

event_loop(Req, State, EventBus, HBRef) ->
    receive
        heartbeat ->
            %% SSE comment as keepalive
            cowboy_req:stream_body(<<": heartbeat\n\n">>, nofin, Req),
            NewRef = erlang:send_after(15000, self(), heartbeat),
            event_loop(Req, State, EventBus, NewRef);
        {event, Type, Data} ->
            send_sse_event(Req, Type, Data),
            event_loop(Req, State, EventBus, HBRef);
        {disconnect} ->
            erlang:cancel_timer(HBRef),
            pg:leave(EventBus, self()),
            cowboy_req:stream_body(<<>>, fin, Req),
            {ok, Req, State}
    after 300000 ->
        %% 5 minute max connection
        erlang:cancel_timer(HBRef),
        cowboy_req:stream_body(<<>>, fin, Req),
        {ok, Req, State}
    end.

send_sse_event(Req, Type, Data) ->
    Json = jsx:encode(Data),
    Event = iolist_to_binary([
        <<"event: ">>, Type, <<"\n">>,
        <<"data: ">>,  Json, <<"\n\n">>
    ]),
    cowboy_req:stream_body(Event, nofin, Req).
```

---

## 7. แบบฝึกหัด

1. เพิ่ม `cors_middleware` ที่ handle preflight OPTIONS requests
2. สร้าง `rate_limit_middleware` ที่ limit ตาม IP address
3. เพิ่ม chunked transfer encoding สำหรับ JSON stream endpoint
4. เพิ่ม resume support สำหรับ file uploads ขนาดใหญ่

---

## สรุป Part 61

✅ Advanced Cowboy routing ด้วย path parameters  
✅ Middleware pipeline (request ID, auth, rate limit)  
✅ Streaming responses สำหรับ large data export  
✅ Multipart file uploads พร้อม validation  
✅ WebSocket advanced: rooms, ping/pong, pub/sub  
✅ Server-Sent Events (SSE) สำหรับ real-time streams  

---

*Part 61/100 | [← ก่อนหน้า](../part60/README.md) | [ถัดไป →](../part62/README.md)*
