# Part 37: Microservices ด้วย Erlang

> **"Small services, clear contracts, independent deployments"**  
> Services เล็ก, contracts ชัดเจน, deploy อิสระ

---

## สารบัญ

1. [Microservices vs Monolith](#1-microservices-vs-monolith)
2. [Service Communication](#2-service-communication)
3. [Service Discovery](#3-service-discovery)
4. [API Gateway Pattern](#4-api-gateway-pattern)
5. [Event-Driven Microservices](#5-event-driven-microservices)
6. [Service Health Checks](#6-service-health-checks)
7. [Tracing และ Observability](#7-tracing-และ-observability)
8. [Inter-Service Authentication](#8-inter-service-authentication)
9. [แบบฝึกหัด](#9-แบบฝึกหัด)

---

## 1. Microservices vs Monolith

```
Erlang Perspective:

Monolith in Erlang (OTP Application):
  ├── Everything in one release
  ├── Communicate via function calls / messages
  ├── Shared ETS, Mnesia
  └── Simple, fast, ปกติเหมาะกว่า

Microservices (Multiple Erlang nodes):
  ├── ใช้ Erlang distribution (ง่ายที่สุด)
  ├── หรือ REST/gRPC between services
  ├── Independent scaling
  └── ต้องการ: service discovery, circuit breaker

ข้อดี Erlang สำหรับ Microservices:
  - Lightweight processes ≈ micro-services ภายใน
  - OTP supervision tree = resilient services
  - Hot code upgrade = no-downtime deploy
  - Distribution = built-in service mesh

คำแนะนำ: เริ่มด้วย monolith Erlang app
(ชัดเจน, BEAM actor = microservice ภายใน)
Microservices เมื่อจำเป็นจริงๆ
```

---

## 2. Service Communication

```erlang
%% Option A: Erlang Distribution (best for Erlang services)
%% ไม่ต้องทำอะไรพิเศษ — ใช้ rpc:call โดยตรง

rpc:call('user_service@host1', user_svc, get_user, [UserId]).
rpc:call('order_service@host2', order_svc, create_order, [OrderData]).

%% Option B: HTTP REST (เหมาะเมื่อ service ไม่ใช่ Erlang)
-module(user_service_client).
-export([get_user/1, create_user/1]).

-define(BASE_URL, <<"http://user-service:8080">>).

get_user(UserId) ->
    Url = <<(base_url())/binary, "/users/",
            (integer_to_binary(UserId))/binary>>,
    case hackney:request(get, Url, auth_headers(), <<>>, [with_body]) of
        {ok, 200, _, Body} -> {ok, jsx:decode(Body, [return_maps])};
        {ok, 404, _, _}    -> {error, not_found};
        {ok, S, _, B}      -> {error, {S, B}};
        {error, R}         -> {error, R}
    end.

base_url() ->
    %% Read from environment or service registry
    application:get_env(myapp, user_service_url, ?BASE_URL).

auth_headers() ->
    Token = get_service_token(),
    [{<<"authorization">>, <<"Bearer ", Token/binary>>},
     {<<"x-service-name">>, <<"order-service">>}].

get_service_token() ->
    application:get_env(myapp, service_token, <<"dev_token">>).
```

---

## 3. Service Discovery

```erlang
%% service_registry.erl — Simple in-process registry
-module(service_registry).
-behaviour(gen_server).

-export([start_link/0, register/2, discover/1, health_check/0]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

-record(service, {
    name    :: binary(),
    url     :: binary(),
    health  :: up | down,
    last_check :: integer()
}).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

register(Name, Url) ->
    gen_server:call(?MODULE, {register, Name, Url}).

discover(Name) ->
    gen_server:call(?MODULE, {discover, Name}).

health_check() ->
    gen_server:cast(?MODULE, health_check).

init([]) ->
    erlang:send_after(30000, self(), health_check),
    %% Load services from config
    Services = load_services_config(),
    {ok, Services}.

handle_call({register, Name, Url}, _From, Services) ->
    S = #service{name=Name, url=Url, health=up,
                 last_check=erlang:system_time(second)},
    {reply, ok, Services#{Name => S}};

handle_call({discover, Name}, _From, Services) ->
    case maps:find(Name, Services) of
        {ok, #service{health=up, url=Url}} -> {reply, {ok, Url}, Services};
        {ok, #service{health=down}}        -> {reply, {error, service_down}, Services};
        error                               -> {reply, {error, not_found}, Services}
    end;

handle_info(health_check, Services) ->
    Services2 = check_all_services(Services),
    erlang:send_after(30000, self(), health_check),
    {noreply, Services2};
handle_info(_, S) -> {noreply, S}.

handle_cast(health_check, Services) ->
    {noreply, check_all_services(Services)}.

check_all_services(Services) ->
    maps:map(fun(_, S) ->
        Health = do_health_check(S#service.url),
        S#service{health=Health, last_check=erlang:system_time(second)}
    end, Services).

do_health_check(Url) ->
    HealthUrl = <<Url/binary, "/health">>,
    case hackney:request(get, HealthUrl, [], <<>>, [with_body]) of
        {ok, 200, _, _} -> up;
        _               -> down
    end.

load_services_config() ->
    Services = application:get_env(myapp, services, []),
    maps:from_list([{Name, #service{name=Name, url=Url,
                                    health=up, last_check=0}}
                    || {Name, Url} <- Services]).
```

---

## 4. API Gateway Pattern

```erlang
%% api_gateway.erl — Single entry point for all services
-module(api_gateway).
-export([init/2]).

-define(ROUTES, #{
    <<"/api/users">>   => <<"user-service">>,
    <<"/api/orders">>  => <<"order-service">>,
    <<"/api/products">> => <<"product-service">>
}).

init(Req, State) ->
    Path    = cowboy_req:path(Req),
    Method  = cowboy_req:method(Req),
    Headers = cowboy_req:headers(Req),

    %% Rate limiting
    {Ip, _} = cowboy_req:peer(Req),
    Key = iolist_to_binary(inet:ntoa(Ip)),
    case rate_limiter:check(Key) of
        deny ->
            Req2 = cowboy_req:reply(429, #{}, <<"Rate limit exceeded">>, Req),
            {ok, Req2, State};
        allow ->
            proxy_to_service(Method, Path, Headers, Req, State)
    end.

proxy_to_service(Method, Path, Headers, Req, State) ->
    ServiceName = find_service(Path),
    case service_registry:discover(ServiceName) of
        {ok, ServiceUrl} ->
            TargetUrl = <<ServiceUrl/binary, Path/binary>>,
            {ok, Body, Req2} = cowboy_req:read_body(Req),
            %% Forward request
            case hackney:request(method_atom(Method), TargetUrl,
                                  forward_headers(Headers), Body, [with_body]) of
                {ok, Status, RespHdrs, RespBody} ->
                    Req3 = cowboy_req:reply(Status,
                        filter_headers(RespHdrs),
                        RespBody, Req2),
                    {ok, Req3, State};
                {error, Reason} ->
                    Req3 = cowboy_req:reply(502, #{},
                        jsx:encode(#{error => <<"upstream error">>,
                                     reason => list_to_binary(atom_to_list(Reason))}),
                        Req2),
                    {ok, Req3, State}
            end;
        {error, _} ->
            Req2 = cowboy_req:reply(503, #{}, <<"Service unavailable">>, Req),
            {ok, Req2, State}
    end.

find_service(Path) ->
    maps:fold(fun(Prefix, ServiceName, Acc) ->
        case binary:match(Path, Prefix) of
            {0, _} -> ServiceName;
            _      -> Acc
        end
    end, <<"default">>, ?ROUTES).

method_atom(<<"GET">>)    -> get;
method_atom(<<"POST">>)   -> post;
method_atom(<<"PUT">>)    -> put;
method_atom(<<"DELETE">>) -> delete;
method_atom(M)            -> binary_to_atom(string:lowercase(M), utf8).

forward_headers(Headers) ->
    %% Remove hop-by-hop headers, add service headers
    Remove = [<<"host">>, <<"connection">>, <<"keep-alive">>],
    Filtered = maps:without(Remove, Headers),
    maps:to_list(Filtered#{<<"x-forwarded-by">> => <<"api-gateway">>}).

filter_headers(Headers) ->
    %% Only pass safe headers downstream
    Safe = [<<"content-type">>, <<"cache-control">>, <<"etag">>],
    maps:with(Safe, maps:from_list(Headers)).
```

---

## 5. Event-Driven Microservices

```erlang
%% event_bus.erl — Simple in-process event bus
%% สำหรับ Erlang-to-Erlang communication

-module(event_bus).
-behaviour(gen_server).

-export([start_link/0, publish/2, subscribe/2, unsubscribe/2]).
-export([init/1, handle_call/3, handle_cast/2]).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

publish(Topic, Event) ->
    gen_server:cast(?MODULE, {publish, Topic, Event}).

subscribe(Topic, Pid) ->
    gen_server:call(?MODULE, {subscribe, Topic, Pid}).

unsubscribe(Topic, Pid) ->
    gen_server:call(?MODULE, {unsubscribe, Topic, Pid}).

init([]) ->
    {ok, #{}}.  %% #{topic => [pid]}

handle_call({subscribe, Topic, Pid}, _From, Subs) ->
    Pids = maps:get(Topic, Subs, []),
    {reply, ok, Subs#{Topic => [Pid | lists:delete(Pid, Pids)]}};

handle_call({unsubscribe, Topic, Pid}, _From, Subs) ->
    Pids = maps:get(Topic, Subs, []),
    {reply, ok, Subs#{Topic => lists:delete(Pid, Pids)}};

handle_cast({publish, Topic, Event}, Subs) ->
    Pids = maps:get(Topic, Subs, []),
    lists:foreach(fun(Pid) ->
        Pid ! {event, Topic, Event}
    end, Pids),
    {noreply, Subs}.

%% ตัวอย่าง: order service publishes, email service subscribes
setup() ->
    event_bus:subscribe(<<"order.created">>, email_service),
    event_bus:subscribe(<<"order.created">>, inventory_service).

%% เมื่อ order ถูกสร้าง
on_order_created(Order) ->
    event_bus:publish(<<"order.created">>, #{
        order_id => Order#order.id,
        user_id  => Order#order.user_id,
        total    => Order#order.total
    }).

%% email_service รับ event
handle_info({event, <<"order.created">>, #{user_id:=UserId}}, State) ->
    email:send_confirmation(UserId),
    {noreply, State}.
```

---

## 6. Service Health Checks

```erlang
%% health_handler.erl — Standard health endpoint
-module(health_handler).
-export([init/2]).

init(Req, State) ->
    Health = check_health(),
    Status = case Health of
        #{status := <<"ok">>} -> 200;
        _                     -> 503
    end,
    Req2 = cowboy_req:reply(Status,
        #{<<"content-type">> => <<"application/json">>},
        jsx:encode(Health), Req),
    {ok, Req2, State}.

check_health() ->
    Checks = [
        {<<"database">>,  check_db()},
        {<<"redis">>,     check_redis()},
        {<<"queue">>,     check_queue()}
    ],
    Statuses = [S || {_, S} <- Checks],
    OverallStatus = case lists:all(fun(S) -> S =:= <<"ok">> end, Statuses) of
        true  -> <<"ok">>;
        false -> <<"degraded">>
    end,
    #{
        status   => OverallStatus,
        version  => app_version(),
        checks   => maps:from_list(Checks),
        uptime   => uptime_seconds()
    }.

check_db() ->
    case catch db:query("SELECT 1", []) of
        {ok, _} -> <<"ok">>;
        _       -> <<"down">>
    end.

check_redis() ->
    case catch redis:ping() of
        <<"PONG">> -> <<"ok">>;
        _          -> <<"down">>
    end.

check_queue() ->
    Stats = job_queue_server:stats(),
    Dead  = maps:get(dead, Stats, 0),
    if Dead > 100 -> <<"degraded">>; true -> <<"ok">> end.

app_version() ->
    {ok, Vsn} = application:get_key(myapp, vsn),
    list_to_binary(Vsn).

uptime_seconds() ->
    {S, _} = erlang:statistics(wall_clock),
    S div 1000.
```

---

## 7. Tracing และ Observability

```erlang
%% trace_context.erl — Distributed tracing (simplified OpenTelemetry)
-module(trace_context).
-export([new_span/1, finish_span/2, current_span/0, with_span/2]).

new_span(Name) ->
    TraceId = case get(trace_id) of
        undefined -> make_trace_id();
        TId -> TId
    end,
    SpanId = make_span_id(),
    put(trace_id, TraceId),
    put(current_span_id, SpanId),
    #{
        trace_id => TraceId,
        span_id  => SpanId,
        name     => Name,
        start    => erlang:monotonic_time(microsecond)
    }.

finish_span(#{start:=Start, name:=Name} = Span, Attrs) ->
    Duration = erlang:monotonic_time(microsecond) - Start,
    Event = Span#{duration => Duration, attributes => Attrs},
    telemetry:execute([span, finished], #{duration => Duration},
                      #{name => Name, span => Event}),
    Event.

current_span() ->
    #{trace_id => get(trace_id), span_id => get(current_span_id)}.

with_span(Name, Fun) ->
    Span = new_span(Name),
    try
        Result = Fun(),
        finish_span(Span, #{status => ok}),
        Result
    catch
        Class:Reason ->
            finish_span(Span, #{status => error, class => Class, reason => Reason}),
            erlang:raise(Class, Reason, erlang:get_stacktrace())
    end.

%% Propagate trace context in HTTP headers
trace_headers() ->
    #{trace_id:=TraceId, span_id:=SpanId} = current_span(),
    [{<<"x-trace-id">>, TraceId},
     {<<"x-span-id">>,  SpanId}].

extract_trace(Headers) ->
    TraceId = maps:get(<<"x-trace-id">>, Headers, make_trace_id()),
    put(trace_id, TraceId).

make_trace_id() -> binary:encode_hex(crypto:strong_rand_bytes(16)).
make_span_id()  -> binary:encode_hex(crypto:strong_rand_bytes(8)).
```

---

## 8. Inter-Service Authentication

```erlang
%% service_auth.erl — Mutual service authentication
-module(service_auth).
-export([create_service_token/1, verify_service_token/1]).

%% Each service has a shared secret
-define(SERVICE_SECRETS, #{
    <<"user-service">>    => <<"secret1">>,
    <<"order-service">>   => <<"secret2">>,
    <<"product-service">> => <<"secret3">>
}).

create_service_token(ServiceName) ->
    Secret = maps:get(ServiceName, ?SERVICE_SECRETS),
    Now    = erlang:system_time(second),
    Claims = #{
        iss => ServiceName,
        iat => Now,
        exp => Now + 300,   %% 5 minute token
        aud => <<"internal">>
    },
    Payload = jsx:encode(Claims),
    H = base64url(jsx:encode(#{alg => <<"HS256">>})),
    P = base64url(Payload),
    Msg = <<H/binary, ".", P/binary>>,
    Sig = base64url(crypto:mac(hmac, sha256, Secret, Msg)),
    <<Msg/binary, ".", Sig/binary>>.

verify_service_token(Token) ->
    case binary:split(Token, <<".">>, [global]) of
        [H, P, S] ->
            Payload = jsx:decode(base64url_dec(P), [return_maps]),
            #{<<"iss">> := Issuer} = Payload,
            case maps:find(Issuer, ?SERVICE_SECRETS) of
                {ok, Secret} ->
                    Msg = <<H/binary, ".", P/binary>>,
                    ExpSig = base64url(crypto:mac(hmac, sha256, Secret, Msg)),
                    case crypto:hash_equals(S, ExpSig) of
                        true ->
                            Now = erlang:system_time(second),
                            case maps:get(<<"exp">>, Payload, 0) > Now of
                                true  -> {ok, Issuer};
                                false -> {error, expired}
                            end;
                        false -> {error, invalid_signature}
                    end;
                error -> {error, unknown_service}
            end;
        _ -> {error, invalid_token}
    end.

base64url(Data) ->
    B = base64:encode(Data),
    binary:replace(
        binary:replace(
            binary:replace(B, <<"+">>, <<"-">>, [global]),
            <<"/">>, <<"_">>, [global]),
        <<"=">>, <<>>, [global]).

base64url_dec(Data) ->
    Padded = case byte_size(Data) rem 4 of
        0 -> Data; 2 -> <<Data/binary, "==">>; 3 -> <<Data/binary, "=">>; _ -> Data
    end,
    B1 = binary:replace(Padded, <<"-">>, <<"+">>, [global]),
    B2 = binary:replace(B1, <<"_">>, <<"/">>, [global]),
    base64:decode(B2).
```

---

## 9. แบบฝึกหัด

### Exercise: Two Services + Gateway

```erlang
%% สร้าง 2 services:
%% 1. user_service: CRUD users
%% 2. order_service: create orders (calls user_service)
%% 3. api_gateway: route requests

%% user_service/src/user_service_app.erl
%% start on port 8081

%% order_service/src/order_service_app.erl
%% start on port 8082
%% สร้าง order: GET user info from user_service ก่อน

%% api_gateway/src/api_gateway_app.erl
%% start on port 8080
%% /api/users/* → user_service:8081
%% /api/orders/* → order_service:8082

%% Test:
%% curl http://localhost:8080/api/users
%% curl http://localhost:8080/api/orders
```

---

## สรุป Part 37

✅ Microservices vs Monolith trade-offs  
✅ Service communication (Erlang dist, REST)  
✅ Service discovery registry  
✅ API Gateway with proxy  
✅ Event-driven communication  
✅ Health check endpoint  
✅ Distributed tracing  
✅ Inter-service authentication

---

*Part 37/100 | [← ก่อนหน้า](../part36/README.md) | [ถัดไป →](../part38/README.md)*
