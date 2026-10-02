# Part 26: HTTP Client ด้วย hackney และ gun

> **"Talking to the world from Erlang — HTTP clients for every use case"**  
> คุยกับโลกจาก Erlang — HTTP clients สำหรับทุก use case

---

## สารบัญ

1. [HTTP Client Options](#1-http-client-options)
2. [hackney — Simple HTTP Client](#2-hackney--simple-http-client)
3. [GET และ POST Requests](#3-get-และ-post-requests)
4. [Headers และ Authentication](#4-headers-และ-authentication)
5. [JSON API Client](#5-json-api-client)
6. [gun — HTTP/2 Client](#6-gun--http2-client)
7. [Connection Pooling](#7-connection-pooling)
8. [Error Handling](#8-error-handling)
9. [แบบฝึกหัด](#9-แบบฝึกหัด)

---

## 1. HTTP Client Options

```
Erlang HTTP Client Libraries:

hackney:
├── Popular, simple API
├── HTTP/1.1
├── Connection pooling ในตัว
└── เหมาะสำหรับ REST API calls

gun:
├── Modern, async
├── รองรับ HTTP/1.1, HTTP/2, WebSocket
├── Stream-based
└── เหมาะสำหรับ long-lived connections

httpc (OTP built-in):
├── ไม่ต้อง dependencies เพิ่ม
├── HTTP/1.1 only
└── API ไม่ค่อย ergonomic

ibrowse:
├── เก่า, stable
└── ใช้ใน legacy systems
```

---

## 2. hackney — Simple HTTP Client

```erlang
%% rebar.config
{deps, [
    {hackney, "1.20.1"},
    {jsx, "3.1.0"}
]}.

%% Start hackney (ถ้าไม่ได้ใช้ application)
hackney:start().

%% Basic GET
{ok, Status, Headers, ClientRef} =
    hackney:get("https://api.example.com/users", [], <<>>, []).

%% อ่าน body
{ok, Body} = hackney:body(ClientRef).

%% Basic POST
{ok, Status, Headers, ClientRef} =
    hackney:post(
        "https://api.example.com/users",
        [{<<"content-type">>, <<"application/json">>}],
        jsx:encode(#{name => <<"Alice">>}),
        []
    ).

%% One-shot request (อ่าน body อัตโนมัติ)
{ok, Status, Headers, Body} =
    hackney:request(get, Url, [], <<>>, [with_body]).
```

---

## 3. GET และ POST Requests

```erlang
-module(http_client).
-export([get/1, get/2, post/2, post/3, put/3, delete/1]).

%% GET
get(Url) -> get(Url, []).
get(Url, Headers) ->
    case hackney:request(get, Url, Headers, <<>>, [with_body]) of
        {ok, 200, _Headers, Body} ->
            {ok, Body};
        {ok, Status, _Headers, Body} ->
            {error, {Status, Body}};
        {error, Reason} ->
            {error, Reason}
    end.

%% POST JSON
post(Url, Data) -> post(Url, Data, []).
post(Url, Data, ExtraHeaders) ->
    Body = jsx:encode(Data),
    Headers = [{<<"content-type">>, <<"application/json">>} | ExtraHeaders],
    case hackney:request(post, Url, Headers, Body, [with_body]) of
        {ok, Status, _Headers, RespBody} when Status >= 200, Status < 300 ->
            {ok, jsx:decode(RespBody, [return_maps])};
        {ok, Status, _Headers, RespBody} ->
            {error, {Status, safe_decode(RespBody)}};
        {error, Reason} ->
            {error, Reason}
    end.

%% PUT JSON
put(Url, Data, ExtraHeaders) ->
    Body = jsx:encode(Data),
    Headers = [{<<"content-type">>, <<"application/json">>} | ExtraHeaders],
    case hackney:request(put, Url, Headers, Body, [with_body]) of
        {ok, Status, _Hdrs, RespBody} when Status >= 200, Status < 300 ->
            {ok, jsx:decode(RespBody, [return_maps])};
        {ok, Status, _Hdrs, RespBody} ->
            {error, {Status, safe_decode(RespBody)}}
    end.

%% DELETE
delete(Url) ->
    case hackney:request(delete, Url, [], <<>>, [with_body]) of
        {ok, 204, _, _} -> ok;
        {ok, 200, _, _} -> ok;
        {ok, Status, _, Body} -> {error, {Status, Body}};
        {error, R} -> {error, R}
    end.

safe_decode(Body) ->
    try jsx:decode(Body, [return_maps])
    catch _:_ -> Body
    end.
```

---

## 4. Headers และ Authentication

```erlang
%% Bearer Token Auth
auth_header(Token) ->
    [{<<"authorization">>, <<"Bearer ", Token/binary>>}].

%% Basic Auth
basic_auth_header(Username, Password) ->
    Cred = base64:encode(<<Username/binary, ":", Password/binary>>),
    [{<<"authorization">>, <<"Basic ", Cred/binary>>}].

%% API Key
api_key_header(Key) ->
    [{<<"x-api-key">>, Key}].

%% ใช้งาน
get_users(Token) ->
    Headers = auth_header(Token),
    http_client:get("https://api.example.com/users", Headers).

%% Custom headers
request_with_headers(Url, Data) ->
    Headers = [
        {<<"content-type">>,  <<"application/json">>},
        {<<"accept">>,        <<"application/json">>},
        {<<"user-agent">>,    <<"MyApp/1.0">>},
        {<<"x-request-id">>,  generate_id()}
    ],
    hackney:request(post, Url, Headers, jsx:encode(Data), [with_body]).

generate_id() ->
    binary:encode_hex(crypto:strong_rand_bytes(8)).

%% Read response headers
{ok, 200, RespHeaders, Body} = hackney:request(get, Url, [], <<>>, [with_body]),
ContentType = hackney:header_value(<<"content-type">>, RespHeaders),
RateLimit   = hackney:header_value(<<"x-rate-limit-remaining">>, RespHeaders).
```

---

## 5. JSON API Client

```erlang
%% Generic JSON API client module
-module(api_client).
-export([new/2, get/2, post/3, put/3, delete/2]).

-record(client, {
    base_url :: binary(),
    headers  :: list()
}).

new(BaseUrl, Token) ->
    Headers = [
        {<<"authorization">>, <<"Bearer ", Token/binary>>},
        {<<"accept">>,        <<"application/json">>},
        {<<"user-agent">>,    <<"ErlangApp/1.0">>}
    ],
    #client{base_url=BaseUrl, headers=Headers}.

get(#client{base_url=Base, headers=H}, Path) ->
    Url = <<Base/binary, Path/binary>>,
    case hackney:request(get, Url, H, <<>>, [with_body]) of
        {ok, 200, _, Body}     -> {ok, decode(Body)};
        {ok, 404, _, _}        -> {error, not_found};
        {ok, 401, _, _}        -> {error, unauthorized};
        {ok, Status, _, Body}  -> {error, {Status, decode(Body)}};
        {error, R}             -> {error, R}
    end.

post(#client{base_url=Base, headers=H}, Path, Data) ->
    Url = <<Base/binary, Path/binary>>,
    Body = jsx:encode(Data),
    Headers = [{<<"content-type">>, <<"application/json">>} | H],
    case hackney:request(post, Url, Headers, Body, [with_body]) of
        {ok, Status, _, RespBody} when Status >= 200, Status < 300 ->
            {ok, decode(RespBody)};
        {ok, 422, _, RespBody} ->
            {error, {validation, decode(RespBody)}};
        {ok, Status, _, RespBody} ->
            {error, {Status, decode(RespBody)}};
        {error, R} ->
            {error, R}
    end.

put(#client{base_url=Base, headers=H}, Path, Data) ->
    Url = <<Base/binary, Path/binary>>,
    Body = jsx:encode(Data),
    Headers = [{<<"content-type">>, <<"application/json">>} | H],
    case hackney:request(put, Url, Headers, Body, [with_body]) of
        {ok, Status, _, RespBody} when Status >= 200, Status < 300 ->
            {ok, decode(RespBody)};
        {ok, Status, _, RespBody} ->
            {error, {Status, decode(RespBody)}};
        {error, R} ->
            {error, R}
    end.

delete(#client{base_url=Base, headers=H}, Path) ->
    Url = <<Base/binary, Path/binary>>,
    case hackney:request(delete, Url, H, <<>>, [with_body]) of
        {ok, Status, _, _} when Status >= 200, Status < 300 -> ok;
        {ok, Status, _, Body} -> {error, {Status, decode(Body)}};
        {error, R} -> {error, R}
    end.

decode(Body) ->
    try jsx:decode(Body, [return_maps])
    catch _:_ -> Body
    end.

%% ตัวอย่างใช้งาน
example() ->
    C = api_client:new(<<"https://api.github.com">>, <<"MY_TOKEN">>),
    {ok, User} = api_client:get(C, <<"/users/octocat">>),
    io:format("Login: ~s~n", [maps:get(<<"login">>, User)]).
```

---

## 6. gun — HTTP/2 Client

```erlang
%% gun เหมาะสำหรับ HTTP/2 และ long-lived connections

%% rebar.config deps
{gun, "2.0.1"}

%% เปิด connection
{ok, ConnPid} = gun:open("api.example.com", 443, #{
    transport => tls,
    protocols => [http2]
}).

%% รอ connection ready
{ok, Protocol} = gun:await_up(ConnPid),
io:format("Connected via ~p~n", [Protocol]).

%% GET request
StreamRef = gun:get(ConnPid, "/users", [
    {<<"accept">>, <<"application/json">>}
]).

%% รอ response
{response, fin, Status, Headers} = gun:await(ConnPid, StreamRef).

%% หรือถ้ามี body
{response, nofin, Status, _Headers} = gun:await(ConnPid, StreamRef),
{ok, Body} = gun:await_body(ConnPid, StreamRef).

%% POST
StreamRef2 = gun:post(ConnPid, "/users", [
    {<<"content-type">>, <<"application/json">>}
], jsx:encode(#{name => <<"Bob">>})).

{response, nofin, 201, _Headers} = gun:await(ConnPid, StreamRef2),
{ok, Body2} = gun:await_body(ConnPid, StreamRef2).

%% ปิด connection
gun:close(ConnPid).
```

---

## 7. Connection Pooling

```erlang
%% hackney มี connection pool ในตัว

%% Default pool
hackney:request(get, Url, [], <<>>, [
    with_body,
    {pool, default}
]).

%% Named pool
hackney_pool:start_pool(my_pool, [
    {timeout, 30000},
    {max_connections, 50}
]).

hackney:request(get, Url, [], <<>>, [
    with_body,
    {pool, my_pool}
]).

%% ตรวจสอบ pool stats
hackney_pool:get_stats(my_pool).

%% Pool per host pattern
-module(pool_manager).
-export([start/0, request/3]).

start() ->
    Pools = [
        {github_pool,    <<"api.github.com">>,    [{max_connections, 20}]},
        {stripe_pool,    <<"api.stripe.com">>,    [{max_connections, 10}]},
        {internal_pool,  <<"internal.svc">>,      [{max_connections, 100}]}
    ],
    lists:foreach(fun({Name, _Host, Opts}) ->
        hackney_pool:start_pool(Name, Opts)
    end, Pools).

request(Pool, Url, Opts) ->
    hackney:request(get, Url, [], <<>>, [with_body, {pool, Pool} | Opts]).
```

---

## 8. Error Handling

```erlang
%% Retry with exponential backoff
-module(http_retry).
-export([get/1, get/3]).

get(Url) -> get(Url, 3, 1000).
get(Url, MaxRetries, BaseDelay) ->
    do_request(Url, MaxRetries, BaseDelay, 0).

do_request(_Url, Max, _Delay, Max) ->
    {error, max_retries_exceeded};
do_request(Url, Max, Delay, Attempt) ->
    case hackney:request(get, Url, [], <<>>, [with_body]) of
        {ok, Status, _, Body} when Status >= 200, Status < 300 ->
            {ok, Body};
        {ok, Status, _, _Body} when Status >= 500 ->
            %% Server error: retry
            Sleep = Delay * (1 bsl Attempt),
            timer:sleep(min(Sleep, 30000)),
            do_request(Url, Max, Delay, Attempt + 1);
        {ok, 429, Headers, _} ->
            %% Rate limited: respect Retry-After
            RetryAfter = get_retry_after(Headers, Delay),
            timer:sleep(RetryAfter),
            do_request(Url, Max, Delay, Attempt + 1);
        {ok, Status, _, Body} ->
            %% Client error: don't retry
            {error, {Status, Body}};
        {error, _Reason} ->
            %% Network error: retry
            Sleep = Delay * (1 bsl Attempt),
            timer:sleep(min(Sleep, 30000)),
            do_request(Url, Max, Delay, Attempt + 1)
    end.

get_retry_after(Headers, Default) ->
    case hackney:header_value(<<"retry-after">>, Headers) of
        undefined -> Default;
        V -> binary_to_integer(V) * 1000
    end.
```

---

## 9. แบบฝึกหัด

### Exercise: GitHub API Client

```erlang
%% สร้าง GitHub API client
-module(github_client).
-export([new/1, get_user/2, list_repos/2, create_repo/3]).

-record(gh_client, {token :: binary()}).

new(Token) -> #gh_client{token=Token}.

base_url() -> <<"https://api.github.com">>.

headers(#gh_client{token=T}) ->
    [
        {<<"authorization">>, <<"Bearer ", T/binary>>},
        {<<"accept">>,        <<"application/vnd.github.v3+json">>},
        {<<"user-agent">>,    <<"ErlangGHClient/1.0">>}
    ].

get_user(Client, Username) ->
    Url = <<(base_url())/binary, "/users/", Username/binary>>,
    case hackney:request(get, Url, headers(Client), <<>>, [with_body]) of
        {ok, 200, _, Body} ->
            {ok, jsx:decode(Body, [return_maps])};
        {ok, 404, _, _} ->
            {error, not_found};
        {ok, Status, _, Body} ->
            {error, {Status, Body}};
        {error, R} ->
            {error, R}
    end.

list_repos(Client, Username) ->
    Url = <<(base_url())/binary, "/users/", Username/binary, "/repos">>,
    case hackney:request(get, Url, headers(Client), <<>>, [with_body]) of
        {ok, 200, _, Body} ->
            Repos = jsx:decode(Body, [return_maps]),
            Names = [maps:get(<<"name">>, R) || R <- Repos],
            {ok, Names};
        {ok, Status, _, Body} ->
            {error, {Status, Body}};
        {error, R} ->
            {error, R}
    end.

create_repo(Client, Name, Private) ->
    Url = <<(base_url())/binary, "/user/repos">>,
    Data = #{name => Name, private => Private},
    H = [{<<"content-type">>, <<"application/json">>} | headers(Client)],
    case hackney:request(post, Url, H, jsx:encode(Data), [with_body]) of
        {ok, 201, _, Body} ->
            {ok, jsx:decode(Body, [return_maps])};
        {ok, Status, _, Body} ->
            {error, {Status, jsx:decode(Body, [return_maps])}};
        {error, R} ->
            {error, R}
    end.
```

---

## สรุป Part 26

✅ HTTP client options: hackney, gun, httpc  
✅ hackney: GET, POST, PUT, DELETE  
✅ Headers และ authentication (Bearer, Basic, API Key)  
✅ Generic JSON API client module  
✅ gun สำหรับ HTTP/2  
✅ Connection pooling  
✅ Retry with exponential backoff  
✅ Rate limit handling

---

*Part 26/100 | [← ก่อนหน้า](../part25/README.md) | [ถัดไป →](../part27/README.md)*
