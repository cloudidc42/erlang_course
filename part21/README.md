# Part 21: Cowboy HTTP Server พื้นฐาน

> **"Cowboy is the Swiss Army knife of Erlang web development"**  
> Cowboy คือ Swiss Army knife ของ Erlang web development

---

## สารบัญ

1. [Cowboy คืออะไร?](#1-cowboy-คืออะไร)
2. [Setup และ Installation](#2-setup-และ-installation)
3. [Basic HTTP Server](#3-basic-http-server)
4. [Routing](#4-routing)
5. [Request Handling](#5-request-handling)
6. [Response Building](#6-response-building)
7. [Middleware](#7-middleware)
8. [Static Files](#8-static-files)
9. [HTTPS Setup](#9-https-setup)
10. [แบบฝึกหัด](#10-แบบฝึกหัด)

---

## 1. Cowboy คืออะไร?

```
Cowboy:
├── HTTP server สำหรับ Erlang/OTP
├── รองรับ HTTP/1.1, HTTP/2, WebSocket
├── เร็วมาก — ใช้ใน RabbitMQ, WhatsApp
├── Based on Ranch (TCP acceptor pool)
├── Clean request/response API
└── Middleware pipeline

ทางเลือกอื่น:
- Elli: minimal, simple
- Inets/httpd: built-in OTP (limited)
- Yaws: full-featured web server
```

---

## 2. Setup และ Installation

```erlang
%% rebar.config
{erl_opts, [debug_info]}.

{deps, [
    {cowboy, "2.10.0"}
]}.

{shell, [
    {config, "config/sys.config"},
    {apps, [myapp]}
]}.
```

```erlang
%% src/myapp.app.src
{application, myapp, [
    {description, "Web Application"},
    {vsn, "1.0.0"},
    {applications, [
        kernel,
        stdlib,
        cowboy
    ]},
    {mod, {myapp_app, []}},
    {env, [{port, 8080}]}
]}.
```

---

## 3. Basic HTTP Server

```erlang
%% src/myapp_app.erl
-module(myapp_app).
-behaviour(application).
-export([start/2, stop/1]).

start(_Type, _Args) ->
    Port = application:get_env(myapp, port, 8080),
    Routes = myapp_router:routes(),
    Dispatch = cowboy_router:compile(Routes),
    
    {ok, _} = cowboy:start_clear(http,
        [{port, Port}],
        #{env => #{dispatch => Dispatch}}
    ),
    
    io:format("Server started on port ~p~n", [Port]),
    myapp_sup:start_link().

stop(_State) ->
    cowboy:stop_listener(http),
    ok.
```

```erlang
%% src/myapp_router.erl
-module(myapp_router).
-export([routes/0]).

routes() ->
    [{'_', [
        {"/",           hello_handler, []},
        {"/api/users",  users_handler, []},
        {"/api/users/:id", user_handler, []},
        {"/static/[...]", cowboy_static, {priv_dir, myapp, "static"}}
    ]}].
```

---

## 4. Routing

```erlang
%% Route patterns
%% /              — exact path
%% /users/:id     — :id ตัวแปร
%% /files/[...]   — rest ของ path (ทุกอย่างหลัง /files/)
%% /users/[:id]   — optional :id

%% Host-based routing
Routes = [
    {"api.example.com", [
        {"/v1/[...]", api_v1_handler, []}
    ]},
    {"www.example.com", [
        {"/", web_handler, []}
    ]},
    {'_', [  %% default host
        {"/", default_handler, []}
    ]}
].

%% ดึงค่าจาก path
cowboy_req:binding(id, Req).          %% binary
cowboy_req:binding(id, Req, default). %% with default

%% ดึง remaining path (ต้องใช้ [...])
cowboy_req:path_info(Req).  %% [<<"files">>, <<"a">>, <<"b">>]

%% Query string
cowboy_req:qs(Req).         %% <<"page=1&limit=20">>
cowboy_req:parse_qs(Req).   %% [{<<"page">>, <<"1">>}, ...]
```

---

## 5. Request Handling

```erlang
%% Handler module: ต้องมี init/2 callback
-module(users_handler).
-export([init/2]).

init(Req, State) ->
    Method = cowboy_req:method(Req),
    handle(Method, Req, State).

handle(<<"GET">>, Req, State) ->
    Users = users:list(),
    Body = json:encode(Users),
    Req2 = cowboy_req:reply(200,
        #{<<"content-type">> => <<"application/json">>},
        Body,
        Req
    ),
    {ok, Req2, State};

handle(<<"POST">>, Req, State) ->
    {ok, Body, Req2} = cowboy_req:read_body(Req),
    case json:decode(Body) of
        {ok, Data} ->
            case users:create(Data) of
                {ok, User} ->
                    Req3 = cowboy_req:reply(201,
                        #{<<"content-type">> => <<"application/json">>},
                        json:encode(User),
                        Req2
                    ),
                    {ok, Req3, State};
                {error, Reason} ->
                    Req3 = cowboy_req:reply(400,
                        #{},
                        json:encode(#{error => Reason}),
                        Req2
                    ),
                    {ok, Req3, State}
            end;
        {error, _} ->
            Req3 = cowboy_req:reply(400, #{}, <<"Invalid JSON">>, Req2),
            {ok, Req3, State}
    end;

handle(Method, Req, State) ->
    Req2 = cowboy_req:reply(405,
        #{<<"allow">> => <<"GET, POST">>},
        <<"Method Not Allowed">>,
        Req
    ),
    {ok, Req2, State}.
```

---

## 6. Response Building

```erlang
%% Reply: ตอบ response
cowboy_req:reply(StatusCode, Headers, Body, Req).
cowboy_req:reply(200, #{}, <<"Hello">>, Req).

%% Headers เป็น map binary -> binary
cowboy_req:reply(200,
    #{
        <<"content-type">>  => <<"application/json; charset=utf-8">>,
        <<"cache-control">> => <<"no-cache">>,
        <<"x-request-id">>  => generate_request_id()
    },
    Body,
    Req
).

%% Streaming response
Req2 = cowboy_req:stream_reply(200,
    #{<<"content-type">> => <<"text/plain">>},
    Req
),
cowboy_req:stream_body(<<"chunk 1\n">>, nofin, Req2),
cowboy_req:stream_body(<<"chunk 2\n">>, nofin, Req2),
cowboy_req:stream_body(<<"final chunk\n">>, fin, Req2).

%% Read request info
cowboy_req:method(Req).          %% <<"GET">>, <<"POST">>, etc.
cowboy_req:path(Req).            %% <<"/api/users">>
cowboy_req:host(Req).            %% <<"example.com">>
cowboy_req:port(Req).            %% 8080
cowboy_req:qs(Req).              %% <<"page=1">>
cowboy_req:header(<<"accept">>, Req).          %% single header
cowboy_req:headers(Req).         %% all headers as map
cowboy_req:peer(Req).            %% {{127,0,0,1}, 54321}
```

---

## 7. Middleware

```erlang
%% Cowboy middleware via OnResponse callback

%% Logging middleware
-module(logging_middleware).
-behaviour(cowboy_middleware).
-export([execute/2]).

execute(Req, Env) ->
    T1 = erlang:monotonic_time(millisecond),
    Method = cowboy_req:method(Req),
    Path = cowboy_req:path(Req),
    
    %% เพิ่ม request ID
    ReqId = binary:encode_hex(crypto:strong_rand_bytes(8)),
    Req2 = cowboy_req:set_resp_header(<<"x-request-id">>, ReqId, Req),
    
    %% ส่งต่อไปยัง next middleware
    Result = {ok, Req3, Env2} = next_middleware(Req2, Env),
    
    T2 = erlang:monotonic_time(millisecond),
    Status = cowboy_req:resp_status(Req3),
    logger:info("~s ~s ~p ~pms [~s]",
                [Method, Path, Status, T2-T1, ReqId]),
    Result.

%% กำหนด middleware stack
cowboy:start_clear(http, [{port, 8080}], #{
    middlewares => [
        logging_middleware,
        cowboy_router,
        cowboy_handler
    ],
    env => #{dispatch => Dispatch}
}).
```

---

## 8. Static Files

```erlang
%% Serve static files
Routes = [{'_', [
    %% serve entire directory
    {"/static/[...]", cowboy_static,
     {priv_dir, myapp, "static"}},

    %% serve single file
    {"/favicon.ico", cowboy_static,
     {priv_file, myapp, "static/favicon.ico"}},

    %% serve from absolute path
    {"/assets/[...]", cowboy_static,
     {dir, "/var/www/myapp/assets"}}
]}].

%% priv/static/ structure:
%% priv/static/
%% ├── css/
%% │   └── style.css
%% ├── js/
%% │   └── app.js
%% └── index.html

%% MIME types (อัตโนมัติจาก extension)
%% .html → text/html
%% .css  → text/css
%% .js   → application/javascript
%% .png  → image/png
%% .json → application/json
```

---

## 9. HTTPS Setup

```erlang
%% ต้องมี SSL certificate

%% development: self-signed certificate
%% openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
%%   -keyout priv/ssl/key.pem -out priv/ssl/cert.pem

start_https() ->
    Port = application:get_env(myapp, https_port, 8443),
    CertFile = priv_file("ssl/cert.pem"),
    KeyFile  = priv_file("ssl/key.pem"),
    
    cowboy:start_tls(https,
        [{port, Port},
         {certfile, CertFile},
         {keyfile, KeyFile}],
        #{env => #{dispatch => Dispatch}}
    ).

priv_file(File) ->
    filename:join(code:priv_dir(myapp), File).

%% HTTP → HTTPS redirect
redirect_handler:init(Req, State) ->
    Host = cowboy_req:host(Req),
    Path = cowboy_req:path(Req),
    Qs   = cowboy_req:qs(Req),
    
    Url = case Qs of
        <<>> -> <<"https://", Host/binary, Path/binary>>;
        _    -> <<"https://", Host/binary, Path/binary, "?", Qs/binary>>
    end,
    
    Req2 = cowboy_req:reply(301,
        #{<<"location">> => Url},
        <<>>,
        Req
    ),
    {ok, Req2, State}.
```

---

## 10. แบบฝึกหัด

### Exercise: Simple REST API

```erlang
%% สร้าง REST API สำหรับ Todo List

%% src/todo_api.erl — handler สำหรับ /api/todos
-module(todo_api).
-export([init/2]).

init(Req, State) ->
    Method = cowboy_req:method(Req),
    handle(Method, Req, State).

handle(<<"GET">>, Req, State) ->
    Todos = todo_server:list(),
    reply_json(200, Todos, Req, State);

handle(<<"POST">>, Req, State) ->
    {ok, Body, Req2} = cowboy_req:read_body(Req),
    case jsx:decode(Body, [return_maps]) of
        #{<<"title">> := Title} ->
            case todo_server:add(Title) of
                {ok, Id} ->
                    reply_json(201, #{id => Id, title => Title}, Req2, State);
                {error, Reason} ->
                    reply_json(400, #{error => Reason}, Req2, State)
            end;
        _ ->
            reply_json(400, #{error => <<"Missing title">>}, Req2, State)
    end.

%% src/todo_item_api.erl — handler สำหรับ /api/todos/:id
-module(todo_item_api).
-export([init/2]).

init(Req, State) ->
    Id = binary_to_integer(cowboy_req:binding(id, Req)),
    Method = cowboy_req:method(Req),
    handle(Method, Id, Req, State).

handle(<<"GET">>, Id, Req, State) ->
    case todo_server:get(Id) of
        {ok, Todo}         -> reply_json(200, Todo, Req, State);
        {error, not_found} -> reply_json(404, #{error => <<"Not found">>}, Req, State)
    end;

handle(<<"DELETE">>, Id, Req, State) ->
    case todo_server:delete(Id) of
        ok                 -> reply_json(204, #{}, Req, State);
        {error, not_found} -> reply_json(404, #{error => <<"Not found">>}, Req, State)
    end.

%% Helper
reply_json(Status, Data, Req, State) ->
    Body = jsx:encode(Data),
    Req2 = cowboy_req:reply(Status,
        #{<<"content-type">> => <<"application/json">>},
        Body, Req
    ),
    {ok, Req2, State}.
```

---

## สรุป Part 21

✅ Cowboy setup ใน rebar3  
✅ start_clear สำหรับ HTTP  
✅ Router configuration  
✅ Handler callbacks: init/2  
✅ Reading request: method, path, headers, body  
✅ Building responses: reply/4  
✅ Streaming responses  
✅ Middleware  
✅ Static file serving  
✅ HTTPS setup

---

*Part 21/100 | [← ก่อนหน้า](../part20/README.md) | [ถัดไป →](../part22/README.md)*
