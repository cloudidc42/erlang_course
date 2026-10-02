# Part 29: Rate Limiting

> **"Protect your system — don't let any single client overwhelm the rest"**  
> ปกป้อง system — อย่าให้ client คนใดคนหนึ่ง overwhelm คนอื่น

---

## สารบัญ

1. [Rate Limiting Algorithms](#1-rate-limiting-algorithms)
2. [Token Bucket](#2-token-bucket)
3. [Sliding Window Counter](#3-sliding-window-counter)
4. [Fixed Window Counter](#4-fixed-window-counter)
5. [Leaky Bucket](#5-leaky-bucket)
6. [Cowboy Middleware Rate Limiter](#6-cowboy-middleware-rate-limiter)
7. [Distributed Rate Limiting](#7-distributed-rate-limiting)
8. [แบบฝึกหัด](#9-แบบฝึกหัด)

---

## 1. Rate Limiting Algorithms

```
Algorithms:

Token Bucket:
  - Tokens refill at rate R per second
  - Each request consumes 1 token
  - Burst allowed up to bucket capacity
  - เหมาะ: API rate limiting ที่ยืดหยุ่น

Sliding Window:
  - Track timestamps of requests in time window
  - Count = requests in last N seconds
  - More accurate than fixed window
  - เหมาะ: strict per-second limits

Fixed Window:
  - Count reset every N seconds
  - Simple, but double-burst at window boundary
  - เหมาะ: simple use cases

Leaky Bucket:
  - Requests enter queue
  - Process at fixed rate
  - Overflow = rejected
  - เหมาะ: smoothing traffic
```

---

## 2. Token Bucket

```erlang
%% token_bucket.erl — Per-key token bucket rate limiter
-module(token_bucket).
-behaviour(gen_server).

-export([start_link/0, check/1, check/3]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

%% Default: 100 req/s, burst 200
-define(DEFAULT_RATE,     100).
-define(DEFAULT_CAPACITY, 200).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

check(Key) ->
    check(Key, ?DEFAULT_RATE, ?DEFAULT_CAPACITY).

check(Key, Rate, Capacity) ->
    gen_server:call(?MODULE, {check, Key, Rate, Capacity}).

init([]) ->
    ets:new(token_buckets, [named_table, set, public,
                             {write_concurrency, true}]),
    {ok, #{}}.

handle_call({check, Key, Rate, Capacity}, _From, S) ->
    Now = erlang:system_time(millisecond),
    Result = case ets:lookup(token_buckets, Key) of
        [] ->
            %% New key: start with full bucket
            ets:insert(token_buckets, {Key, Capacity - 1, Now}),
            allow;
        [{Key, Tokens, LastRefill}] ->
            %% Refill tokens based on time elapsed
            Elapsed = (Now - LastRefill) / 1000.0,  %% seconds
            NewTokens = min(Capacity, Tokens + Elapsed * Rate),
            if
                NewTokens >= 1 ->
                    ets:insert(token_buckets, {Key, NewTokens - 1, Now}),
                    allow;
                true ->
                    deny
            end
    end,
    {reply, Result, S};

handle_cast(_, S) -> {noreply, S}.
handle_info(_, S) -> {noreply, S}.

%% ใช้งาน
example() ->
    token_bucket:start_link(),
    Key = <<"user:123">>,
    case token_bucket:check(Key) of
        allow -> process_request();
        deny  -> reply_429()
    end.

process_request() -> ok.
reply_429() -> {error, rate_limited}.
```

---

## 3. Sliding Window Counter

```erlang
%% sliding_window.erl
-module(sliding_window).
-behaviour(gen_server).

-export([start_link/0, check/1, check/3]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

-define(DEFAULT_LIMIT,  100).
-define(DEFAULT_WINDOW, 60).   %% 60 seconds

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

check(Key) -> check(Key, ?DEFAULT_LIMIT, ?DEFAULT_WINDOW).
check(Key, Limit, Window) ->
    gen_server:call(?MODULE, {check, Key, Limit, Window}).

init([]) ->
    ets:new(sw_requests, [named_table, bag, public,
                           {write_concurrency, true}]),
    erlang:send_after(5000, self(), cleanup),
    {ok, #{}}.

handle_call({check, Key, Limit, Window}, _From, S) ->
    Now = erlang:system_time(second),
    Cutoff = Now - Window,

    %% Count requests in window
    All = ets:lookup(sw_requests, Key),
    InWindow = [T || {_, T} <- All, T > Cutoff],
    Count = length(InWindow),

    Result = if
        Count < Limit ->
            ets:insert(sw_requests, {Key, Now}),
            {allow, Limit - Count - 1};
        true ->
            {deny, 0}
    end,
    {reply, Result, S};

handle_info(cleanup, S) ->
    Now = erlang:system_time(second),
    Cutoff = Now - 3600,  %% Remove entries older than 1 hour
    ets:select_delete(sw_requests,
        [{{'_', '$1'}, [{'<', '$1', Cutoff}], [true]}]),
    erlang:send_after(5000, self(), cleanup),
    {noreply, S};
handle_info(_, S) -> {noreply, S}.

handle_cast(_, S) -> {noreply, S}.
```

---

## 4. Fixed Window Counter

```erlang
%% fixed_window.erl — Simple, fast fixed window counter
-module(fixed_window).
-export([check/2, check/3]).

%% Key format: {user_id, window_start}
%% window_start = timestamp rounded to window size

check(Key, Limit) ->
    check(Key, Limit, 60).

check(Key, Limit, Window) ->
    Now = erlang:system_time(second),
    WindowStart = (Now div Window) * Window,
    CounterKey  = {Key, WindowStart},

    %% atomic increment and check
    Count = ets:update_counter(rate_counters, CounterKey, 1, {CounterKey, 0}),

    if
        Count =:= 1 ->
            %% First request in window: set expiry
            ExpireAt = WindowStart + Window + 1,
            spawn(fun() ->
                Delay = (ExpireAt - erlang:system_time(second)) * 1000,
                timer:sleep(max(Delay, 0)),
                ets:delete(rate_counters, CounterKey)
            end);
        true -> ok
    end,

    if
        Count =< Limit -> {allow, Limit - Count};
        true           -> {deny, 0}
    end.

%% Initialize ETS table
init() ->
    ets:new(rate_counters, [
        named_table, set, public,
        {write_concurrency, true}
    ]).
```

---

## 5. Leaky Bucket

```erlang
%% leaky_bucket.erl — Process requests at fixed rate
-module(leaky_bucket).
-behaviour(gen_server).

-export([start_link/2, request/1]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

-record(state, {
    rate       :: pos_integer(),   %% requests per second
    queue      :: queue:queue(),
    max_queue  :: pos_integer(),
    processing :: boolean()
}).

start_link(Rate, MaxQueue) ->
    gen_server:start_link(?MODULE, [Rate, MaxQueue], []).

request(Pid) ->
    gen_server:call(Pid, request, 10000).

init([Rate, MaxQueue]) ->
    {ok, #state{
        rate=Rate, queue=queue:new(),
        max_queue=MaxQueue, processing=false
    }}.

handle_call(request, From, #state{max_queue=Max}=S) ->
    QSize = queue:len(S#state.queue),
    if
        QSize >= Max ->
            {reply, {error, queue_full}, S};
        true ->
            Queue2 = queue:in(From, S#state.queue),
            S2 = S#state{queue=Queue2},
            maybe_start_processing(S2),
            {noreply, S2}
    end.

handle_cast(_, S) -> {noreply, S}.

handle_info(process_next, #state{rate=Rate}=S) ->
    case queue:out(S#state.queue) of
        {empty, _} ->
            {noreply, S#state{processing=false}};
        {{value, From}, Queue2} ->
            gen_server:reply(From, ok),
            %% Schedule next processing
            Delay = 1000 div Rate,
            erlang:send_after(Delay, self(), process_next),
            {noreply, S#state{queue=Queue2, processing=true}}
    end;
handle_info(_, S) -> {noreply, S}.

maybe_start_processing(#state{processing=false}) ->
    self() ! process_next;
maybe_start_processing(_) ->
    ok.
```

---

## 6. Cowboy Middleware Rate Limiter

```erlang
%% rate_limit_middleware.erl — Cowboy middleware
-module(rate_limit_middleware).
-behaviour(cowboy_middleware).

-export([execute/2]).

execute(Req, Env) ->
    Key = get_limit_key(Req),
    case token_bucket:check(Key, 100, 200) of
        allow ->
            {ok, Req, Env};
        deny ->
            Req2 = cowboy_req:reply(429,
                #{
                    <<"content-type">>  => <<"application/json">>,
                    <<"retry-after">>   => <<"1">>,
                    <<"x-rate-limit-limit">> => <<"100">>,
                    <<"x-rate-limit-remaining">> => <<"0">>
                },
                <<"{\"error\":\"rate_limit_exceeded\"}">>,
                Req
            ),
            {stop, Req2}
    end.

get_limit_key(Req) ->
    %% Rate limit per IP
    {Ip, _Port} = cowboy_req:peer(Req),
    IpBin = list_to_binary(inet:ntoa(Ip)),

    %% Or per authenticated user
    case cowboy_req:header(<<"authorization">>, Req) of
        undefined ->
            <<"ip:", IpBin/binary>>;
        Token ->
            case auth:verify_token(Token) of
                {ok, UserId} ->
                    <<"user:", (integer_to_binary(UserId))/binary>>;
                _ ->
                    <<"ip:", IpBin/binary>>
            end
    end.

%% Register middleware:
%% cowboy:start_clear(http, [{port, 8080}], #{
%%     middlewares => [
%%         rate_limit_middleware,
%%         cowboy_router,
%%         cowboy_handler
%%     ],
%%     env => #{dispatch => Dispatch}
%% })
```

---

## 7. Distributed Rate Limiting

```erlang
%% distributed_rate_limiter.erl
%% ใช้ global counter across all nodes

-module(distributed_rate_limiter).
-behaviour(gen_server).

-export([start_link/0, check/2]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

start_link() ->
    gen_server:start_link({global, ?MODULE}, ?MODULE, [], []).

check(Key, Limit) ->
    case global:whereis_name(?MODULE) of
        undefined ->
            %% Fallback: use local rate limiter if global unavailable
            token_bucket:check(Key);
        Pid ->
            gen_server:call(Pid, {check, Key, Limit})
    end.

init([]) ->
    ets:new(global_rate, [named_table, set, public,
                           {write_concurrency, true}]),
    {ok, #{}}.

handle_call({check, Key, Limit}, _From, S) ->
    Window = 60,  %% 1 minute window
    Now = erlang:system_time(second),
    WindowKey = {Key, (Now div Window) * Window},
    Count = ets:update_counter(global_rate, WindowKey, 1, {WindowKey, 0}),
    Result = if
        Count =< Limit -> {allow, Limit - Count};
        true           -> {deny, 0}
    end,
    {reply, Result, S};

handle_cast(_, S) -> {noreply, S}.
handle_info(_, S) -> {noreply, S}.

%% พัฒนาต่อ: ใช้ Mnesia transaction สำหรับ true distributed counter
distributed_check(Key, Limit) ->
    mnesia:transaction(fun() ->
        Current = case mnesia:read(rate_counters, Key) of
            [{rate_counters, Key, C, _}] -> C;
            [] -> 0
        end,
        if
            Current < Limit ->
                mnesia:write({rate_counters, Key, Current+1,
                              erlang:system_time(second)}),
                allow;
            true ->
                deny
        end
    end).
```

---

## 8. แบบฝึกหัด

### Exercise: Tiered Rate Limiting

```erlang
%% Rate limit ต่างกันตาม user tier
%% Free: 100 req/min
%% Pro: 1000 req/min
%% Enterprise: 10000 req/min

-module(tiered_rate_limiter).
-export([check/2]).

-define(TIERS, #{
    free       => 100,
    pro        => 1000,
    enterprise => 10000
}).

check(UserId, Tier) ->
    Limit = maps:get(Tier, ?TIERS, 100),
    Window = 60,
    Now = erlang:system_time(second),
    WindowStart = (Now div Window) * Window,
    Key = {UserId, WindowStart},

    Count = ets:update_counter(tier_rate, Key, 1, {Key, 0}),

    Remaining = max(0, Limit - Count),
    ResetAt   = WindowStart + Window,

    if
        Count =< Limit ->
            {allow, #{
                limit     => Limit,
                remaining => Remaining,
                reset     => ResetAt
            }};
        true ->
            {deny, #{
                limit     => Limit,
                remaining => 0,
                reset     => ResetAt,
                retry_after => ResetAt - Now
            }}
    end.
```

---

## สรุป Part 29

✅ Rate limiting algorithms: Token Bucket, Sliding Window, Fixed Window, Leaky Bucket  
✅ Token bucket implementation ด้วย ETS  
✅ Sliding window counter  
✅ Fixed window counter  
✅ Leaky bucket for smooth traffic  
✅ Cowboy middleware integration  
✅ Distributed rate limiting  
✅ Tiered rate limiting

---

*Part 29/100 | [← ก่อนหน้า](../part28/README.md) | [ถัดไป →](../part30/README.md)*
