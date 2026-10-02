# Part 83: API Design at Scale

> **"An API is a contract with the world. Break it and you break trust"**  
> API คือสัญญากับโลก — ทำลายมันแล้วคุณทำลายความไว้วางใจ

---

## สารบัญ

1. [API Versioning Strategies](#1-api-versioning-strategies)
2. [RESTful API Design Patterns](#2-restful-api-design-patterns)
3. [API Rate Limiting at Scale](#3-api-rate-limiting-at-scale)
4. [Pagination Patterns](#4-pagination-patterns)
5. [API Documentation Generation](#5-api-documentation-generation)
6. [Deprecation Strategy](#6-deprecation-strategy)
7. [แบบฝึกหัด](#7-แบบฝึกหัด)

---

## 1. API Versioning Strategies

```erlang
%% api_router.erl — version-aware request routing
-module(api_router).
-export([routes/0]).

routes() ->
    [
        %% URL-path versioning (recommended)
        {"/api/v1/[...]", api_v1_dispatch, []},
        {"/api/v2/[...]", api_v2_dispatch, []},

        %% Latest alias — always points to newest stable
        {"/api/latest/[...]", api_v2_dispatch, []},

        %% Health and metadata don't need versioning
        {"/health", health_handler, []},
        {"/api/versions", versions_handler, []}
    ].

%% versions_handler.erl — list available API versions
-module(versions_handler).
-behaviour(cowboy_handler).
-export([init/2]).

init(Req, State) ->
    Versions = #{
        versions => [
            #{version => <<"v1">>,
              status  => <<"deprecated">>,
              sunset  => <<"2025-06-01">>,
              docs    => <<"/api/v1/docs">>},
            #{version => <<"v2">>,
              status  => <<"stable">>,
              current => true,
              docs    => <<"/api/v2/docs">>}
        ]
    },
    Resp = cowboy_req:reply(200,
        #{<<"content-type">> => <<"application/json">>},
        json:encode(Versions),
        Req),
    {ok, Resp, State}.
```

---

## 2. RESTful API Design Patterns

```erlang
%% base_handler.erl — shared handler logic for all REST resources
-module(base_handler).
-export([reply_ok/2, reply_created/3, reply_no_content/1,
         reply_error/3, paginate/3]).

reply_ok(Req, Data) ->
    cowboy_req:reply(200,
        #{<<"content-type">> => <<"application/json">>},
        json:encode(Data), Req).

reply_created(Req, Location, Data) ->
    cowboy_req:reply(201,
        #{<<"content-type">>  => <<"application/json">>,
          <<"location">>      => Location},
        json:encode(Data), Req).

reply_no_content(Req) ->
    cowboy_req:reply(204, #{}, <<>>, Req).

reply_error(Req, Code, Message) ->
    cowboy_req:reply(Code,
        #{<<"content-type">> => <<"application/json">>},
        json:encode(#{error => Message,
                      request_id => get_request_id(Req)}),
        Req).

paginate(Items, Total, Opts) ->
    Page  = maps:get(page, Opts, 1),
    Limit = maps:get(limit, Opts, 20),
    #{
        data  => Items,
        meta  => #{
            total       => Total,
            page        => Page,
            per_page    => Limit,
            total_pages => ceil(Total / Limit)
        },
        links => #{
            self  => build_url(Opts, Page),
            first => build_url(Opts, 1),
            last  => build_url(Opts, ceil(Total / Limit)),
            next  => case Page * Limit < Total of
                         true  -> build_url(Opts, Page + 1);
                         false -> null
                     end,
            prev  => case Page > 1 of
                         true  -> build_url(Opts, Page - 1);
                         false -> null
                     end
        }
    }.

get_request_id(Req) ->
    cowboy_req:header(<<"x-request-id">>, Req, <<"unknown">>).

build_url(_Opts, _Page) -> <<"/">>.   % simplified
```

```erlang
%% orders_handler.erl — example REST resource handler
-module(orders_handler).
-behaviour(cowboy_handler).
-export([init/2, allowed_methods/2, content_types_provided/2,
         content_types_accepted/2, resource_exists/2]).

init(Req, State) ->
    {cowboy_rest, Req, State}.

allowed_methods(Req, State) ->
    {[<<"GET">>, <<"POST">>, <<"PUT">>, <<"DELETE">>], Req, State}.

content_types_provided(Req, State) ->
    {[{{<<"application">>, <<"json">>, '*'}, handle_get}], Req, State}.

content_types_accepted(Req, State) ->
    {[{{<<"application">>, <<"json">>, '*'}, handle_write}], Req, State}.

resource_exists(Req, State) ->
    case cowboy_req:binding(order_id, Req) of
        undefined -> {true, Req, State};  % collection
        OrderId ->
            case orders:get(OrderId) of
                {ok, Order} -> {true, Req, State#{order => Order}};
                {error, not_found} -> {false, Req, State}
            end
    end.

handle_get(Req, #{order := Order} = State) ->
    {json:encode(Order), Req, State};

handle_get(Req, State) ->
    %% List orders with pagination
    Qs   = cowboy_req:parse_qs(Req),
    Page = binary_to_integer(proplists:get_value(<<"page">>, Qs, <<"1">>)),
    Limit = min(100, binary_to_integer(
        proplists:get_value(<<"per_page">>, Qs, <<"20">>))),
    {ok, Items, Total} = orders:list(#{page => Page, limit => Limit}),
    Response = base_handler:paginate(Items, Total, #{page => Page,
                                                      limit => Limit}),
    {json:encode(Response), Req, State}.

handle_write(Req, State) ->
    {ok, Body, Req1} = cowboy_req:read_body(Req),
    case json:decode(Body) of
        {ok, Params} ->
            case orders:create(Params) of
                {ok, Order} ->
                    Location = <<"/api/v2/orders/",
                                 (maps:get(id, Order))/binary>>,
                    Req2 = base_handler:reply_created(Req1, Location, Order),
                    {true, Req2, State};
                {error, Reason} ->
                    Req2 = base_handler:reply_error(Req1, 422, Reason),
                    {false, Req2, State}
            end;
        {error, _} ->
            Req2 = base_handler:reply_error(Req1, 400, invalid_json),
            {false, Req2, State}
    end.
```

---

## 3. API Rate Limiting at Scale

```erlang
%% rate_limiter.erl — sliding window rate limiter with ETS
-module(rate_limiter).
-behaviour(gen_server).

-export([start_link/0, check/3, reset/2]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

-define(CLEANUP_INTERVAL, 60000).

%% Rate limit tiers
-define(TIERS, #{
    free    => #{requests => 100,  window_seconds => 3600},
    basic   => #{requests => 1000, window_seconds => 3600},
    pro     => #{requests => 10000,window_seconds => 3600},
    enterprise => #{requests => 100000, window_seconds => 3600}
}).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

%% Returns: ok | {error, {rate_limited, RetryAfter}}
check(Identity, Tier, Resource) ->
    Limits = maps:get(Tier, ?TIERS, maps:get(free, ?TIERS)),
    Max    = maps:get(requests, Limits),
    Window = maps:get(window_seconds, Limits),
    Key    = {Identity, Resource},
    Now    = erlang:system_time(second),

    %% Use sliding window log in ETS
    gen_server:call(?MODULE, {check, Key, Max, Window, Now}).

reset(Identity, Resource) ->
    gen_server:call(?MODULE, {reset, {Identity, Resource}}).

init([]) ->
    ets:new(rate_windows, [named_table, public, {write_concurrency, true}]),
    erlang:send_after(?CLEANUP_INTERVAL, self(), cleanup),
    {ok, #{}}.

handle_call({check, Key, Max, Window, Now}, _From, State) ->
    Cutoff = Now - Window,
    %% Get existing timestamps for this key
    Timestamps = case ets:lookup(rate_windows, Key) of
        [{_, Ts}] -> Ts;
        []        -> []
    end,
    %% Remove expired timestamps
    Recent = [T || T <- Timestamps, T > Cutoff],
    case length(Recent) < Max of
        true ->
            %% Allow: add current timestamp
            ets:insert(rate_windows, {Key, [Now | Recent]}),
            Remaining = Max - length(Recent) - 1,
            {reply, {ok, Remaining}, State};
        false ->
            %% Blocked: return retry-after
            OldestInWindow = lists:min(Recent),
            RetryAfter = OldestInWindow + Window - Now,
            {reply, {error, {rate_limited, RetryAfter}}, State}
    end;

handle_call({reset, Key}, _From, State) ->
    ets:delete(rate_windows, Key),
    {reply, ok, State}.

handle_info(cleanup, State) ->
    Now    = erlang:system_time(second),
    Cutoff = Now - 3600,
    %% Remove entries with all timestamps expired
    ets:select_delete(rate_windows,
        [{ {'_', '$1'}, [{'==', '$1', []}], [true] }]),
    erlang:send_after(?CLEANUP_INTERVAL, self(), cleanup),
    {noreply, State}.

handle_cast(_Msg, State) -> {noreply, State}.
```

---

## 4. Pagination Patterns

```erlang
%% pagination.erl — cursor-based and offset-based pagination
-module(pagination).
-export([cursor_page/3, offset_page/3, encode_cursor/1, decode_cursor/1]).

%% Cursor-based pagination (recommended for large datasets)
cursor_page(Query, Cursor, Limit) ->
    {WhereClause, Params, ParamIdx} = case decode_cursor(Cursor) of
        {ok, #{id := LastId, created_at := LastTs}} ->
            Clause = " AND (created_at, id) < ($1, $2)",
            {Clause, [LastTs, LastId], 3};
        {error, _} ->
            {"", [], 1}
    end,
    Sql = iolist_to_binary([
        Query, WhereClause,
        " ORDER BY created_at DESC, id DESC LIMIT $",
        integer_to_binary(ParamIdx)
    ]),
    {ok, Rows} = db:query(Sql, Params ++ [Limit + 1]),

    %% Request one extra to detect if there's a next page
    {HasMore, Items} = case length(Rows) > Limit of
        true  -> {true,  lists:sublist(Rows, Limit)};
        false -> {false, Rows}
    end,

    NextCursor = case {HasMore, Items} of
        {true, [_|_]} ->
            Last = lists:last(Items),
            {ok, encode_cursor(#{id => element(1, Last),
                                  created_at => element(2, Last)})};
        _ ->
            undefined
    end,

    #{items => Items, has_more => HasMore, next_cursor => NextCursor}.

%% Offset-based pagination (simpler but doesn't scale)
offset_page(Query, Page, Limit) ->
    Offset = (Page - 1) * Limit,
    CountSql = iolist_to_binary(["SELECT COUNT(*) FROM (", Query, ") sq"]),
    {ok, [{Total}]} = db:query(CountSql, []),
    Sql = iolist_to_binary([Query, " LIMIT $1 OFFSET $2"]),
    {ok, Items} = db:query(Sql, [Limit, Offset]),
    #{items => Items, total => Total, page => Page, per_page => Limit,
      total_pages => ceil(Total / Limit)}.

encode_cursor(Data) ->
    Json    = json:encode(Data),
    Encoded = base64:encode(Json),
    Encoded.

decode_cursor(undefined) -> {error, no_cursor};
decode_cursor(Cursor) ->
    try
        Json = base64:decode(Cursor),
        {ok, json:decode(Json)}
    catch _:_ ->
        {error, invalid_cursor}
    end.
```

---

## 5. API Documentation Generation

```erlang
%% openapi_gen.erl — generate OpenAPI 3.0 spec from route annotations
-module(openapi_gen).
-export([generate/0]).

%% Annotate handlers with -api_spec attribute
%% Example:
%%   -api_spec #{
%%     method => get,
%%     path => "/api/v2/orders/{id}",
%%     summary => "Get order by ID",
%%     parameters => [#{name => id, in => path, required => true}],
%%     responses => #{200 => #{schema => order_schema}}
%%   }.

generate() ->
    %% Collect all modules with -api_spec attribute
    Modules  = get_api_modules(),
    Paths    = build_paths(Modules),
    Schemas  = collect_schemas(Modules),

    #{
        openapi  => <<"3.0.3">>,
        info     => #{
            title   => <<"MyApp API">>,
            version => <<"2.0.0">>,
            contact => #{email => <<"api@myapp.com">>}
        },
        servers  => [
            #{url => <<"https://api.myapp.com">>, description => <<"Production">>},
            #{url => <<"http://localhost:4000">>, description => <<"Local">>}
        ],
        paths    => Paths,
        components => #{schemas => Schemas}
    }.

get_api_modules() ->
    [M || M <- erlang:loaded(),
          lists:keymember(api_spec, 1, M:module_info(attributes))].

build_paths(Modules) ->
    lists:foldl(fun(Module, Acc) ->
        Specs = proplists:get_all_values(api_spec, Module:module_info(attributes)),
        lists:foldl(fun(Spec, PathAcc) ->
            Path   = maps:get(path, Spec),
            Method = maps:get(method, Spec),
            PathAcc#{Path => maps:put(Method, spec_to_operation(Spec), maps:get(Path, PathAcc, #{}))}
        end, Acc, Specs)
    end, #{}, Modules).

spec_to_operation(Spec) ->
    #{
        summary    => maps:get(summary, Spec, <<"">>),
        parameters => maps:get(parameters, Spec, []),
        responses  => maps:get(responses, Spec, #{})
    }.

collect_schemas(_Modules) -> #{}.
```

---

## 6. Deprecation Strategy

```erlang
%% deprecation.erl — manage API endpoint deprecation
-module(deprecation).
-export([deprecated_middleware/3, sunset_response/3]).

%% Middleware that adds deprecation warnings to responses
deprecated_middleware(Req, Env, DeprecatedPaths) ->
    Path = cowboy_req:path(Req),
    case maps:get(Path, DeprecatedPaths, undefined) of
        undefined ->
            {ok, Req, Env};
        #{sunset := SunsetDate, successor := Successor} ->
            Req1 = cowboy_req:set_resp_headers(#{
                %% RFC 8594: Sunset header
                <<"sunset">> => format_date(SunsetDate),
                %% Link to replacement
                <<"link">> => <<"<", Successor/binary,
                                ">; rel=\"successor-version\"">>,
                %% Custom deprecation notice
                <<"deprecation">> => <<"true">>,
                <<"warning">> =>
                    <<"299 - \"This endpoint is deprecated and will be "
                      "removed on ", SunsetDate/binary, ". "
                      "Please migrate to ", Successor/binary, "\"">>
            }, Req),
            {ok, Req1, Env}
    end.

%% After sunset date: return 410 Gone
sunset_response(Req, SunsetDate, Successor) ->
    cowboy_req:reply(410,
        #{<<"content-type">> => <<"application/json">>,
          <<"link">> => <<"<", Successor/binary, ">; rel=\"successor\"">>},
        json:encode(#{
            error   => <<"endpoint_removed">>,
            message => <<"This endpoint was removed on ", SunsetDate/binary>>,
            migrate => Successor
        }),
        Req).

format_date({Year, Month, Day}) ->
    iolist_to_binary(io_lib:format("~4..0B-~2..0B-~2..0B",
                                   [Year, Month, Day])).

%% Track deprecation usage for migration monitoring
log_deprecated_usage(Path, UserId) ->
    telemetry:execute([api, deprecated_endpoint_used], #{count => 1}, #{
        path    => Path,
        user_id => UserId
    }).
```

---

## 7. แบบฝึกหัด

1. Implement GraphQL endpoint ที่ allow clients เลือก fields ที่ต้องการ
2. สร้าง API changelog generator จาก git commits
3. เพิ่ม response caching ด้วย `Cache-Control` และ `ETag` headers
4. Implement API key authentication เป็น alternative ให้ JWT

---

## สรุป Part 83

✅ API versioning: URL-path versioning, version listing endpoint  
✅ RESTful patterns: cowboy_rest, content negotiation, proper HTTP codes  
✅ Rate limiting: sliding window log per identity+resource  
✅ Pagination: cursor-based (scalable) and offset-based (simple)  
✅ OpenAPI generation: collect -api_spec attributes from modules  
✅ Deprecation: Sunset headers, 410 Gone after sunset, usage tracking  

---

*Part 83/100 | [← ก่อนหน้า](../part82/README.md) | [ถัดไป →](../part84/README.md)*
