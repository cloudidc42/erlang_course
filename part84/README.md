# Part 84: Testing Strategies

> **"A test suite you trust is worth more than a test suite that passes"**  
> test suite ที่คุณเชื่อถือมีค่ามากกว่า test suite ที่ผ่าน

---

## สารบัญ

1. [Testing Pyramid](#1-testing-pyramid)
2. [Property-Based Testing with PropEr](#2-property-based-testing-with-proper)
3. [Integration Testing](#3-integration-testing)
4. [Load Testing](#4-load-testing)
5. [Contract Testing](#5-contract-testing)
6. [Chaos Engineering](#6-chaos-engineering)
7. [แบบฝึกหัด](#7-แบบฝึกหัด)

---

## 1. Testing Pyramid

```
Erlang Testing Pyramid
════════════════════════════════════════════════════════

         /\
        /  \
       / E2E \    ← Few, slow, expensive
      /  (5%)  \     Browser/API integration
     /──────────\
    /            \
   / Integration  \ ← Some, medium speed
  /    (20%)       \   DB, Cowboy, external services
 /──────────────────\
/                    \
/    Unit / EUnit     \ ← Many, fast, cheap
/       (75%)          \   Pure functions, gen_server state
/──────────────────────\

KEY TOOLS:
  • EUnit        — unit tests, simple assertions
  • Common Test  — integration, complex setups
  • PropEr       — property-based testing
  • Meck         — mocking gen_server, modules
  • tsung/k6     — load testing
```

---

## 2. Property-Based Testing with PropEr

```erlang
%% prop_account.erl — property-based tests for account aggregate
-module(prop_account).
-include_lib("proper/include/proper.hrl").
-include_lib("eunit/include/eunit.hrl").

%% Generator for valid amounts
amount() -> pos_integer().

%% Generator for a sequence of valid operations
operations() ->
    list(oneof([
        {deposit,  amount()},
        {withdraw, amount()},
        {deposit,  amount()}
    ])).

%% Property: balance is always >= 0 after any sequence of operations
prop_balance_never_negative() ->
    ?FORALL(Ops, operations(),
        begin
            InitBalance = 1000,
            FinalBalance = simulate_ops(InitBalance, Ops),
            FinalBalance >= 0
        end
    ).

%% Property: deposit then withdraw leaves balance unchanged
prop_deposit_withdraw_inverse() ->
    ?FORALL(Amount, amount(),
        begin
            Balance0 = 500,
            Balance1 = Balance0 + Amount,
            Balance2 = max(0, Balance1 - Amount),
            Balance2 =:= Balance0
        end
    ).

%% Property: ordering of deposits doesn't matter (commutativity)
prop_deposit_order_independent() ->
    ?FORALL({A, B}, {amount(), amount()},
        begin
            B1 = 0 + A + B,
            B2 = 0 + B + A,
            B1 =:= B2
        end
    ).

simulate_ops(Balance, []) -> Balance;
simulate_ops(Balance, [{deposit, Amount} | Rest]) ->
    simulate_ops(Balance + Amount, Rest);
simulate_ops(Balance, [{withdraw, Amount} | Rest]) ->
    NewBalance = Balance - Amount,
    case NewBalance >= 0 of
        true  -> simulate_ops(NewBalance, Rest);
        false -> simulate_ops(Balance, Rest)  % Reject overdraft
    end.

%% Run all properties
prop_test_() ->
    [
        {timeout, 60,
         ?_assert(proper:quickcheck(prop_balance_never_negative(),
                                    [{numtests, 1000}]))},
        ?_assert(proper:quickcheck(prop_deposit_withdraw_inverse())),
        ?_assert(proper:quickcheck(prop_deposit_order_independent()))
    ].
```

```erlang
%% prop_routing.erl — property tests for route optimizer
-module(prop_routing).
-include_lib("proper/include/proper.hrl").

%% Generator for geographic coordinates
coord() ->
    {float(-90, 90), float(-180, 180)}.

stop() ->
    ?LET({Lat, Lng}, coord(),
         #{lat => Lat, lng => Lng, package_id => binary()}).

stops() -> non_empty(list(stop())).

%% Property: optimized route visits all stops exactly once
prop_all_stops_visited() ->
    ?FORALL(Stops, stops(),
        begin
            Depot   = #{lat => 0.0, lng => 0.0},
            Route   = route_optimizer:optimize_route(Depot, Stops),
            Visited = [maps:get(package_id, S) || S <- Route],
            Expected = [maps:get(package_id, S) || S <- Stops],
            lists:sort(Visited) =:= lists:sort(Expected)
        end
    ).

%% Property: route length is reasonable (not exponentially longer than naive)
prop_route_not_exponentially_bad() ->
    ?FORALL(Stops, vector(5, stop()),
        begin
            Depot     = #{lat => 0.0, lng => 0.0},
            Route     = route_optimizer:optimize_route(Depot, Stops),
            OptLen    = total_distance(Depot, Route),
            NaiveLen  = total_distance(Depot, Stops),
            OptLen =< NaiveLen * 3   % Within 3x optimal (heuristic bound)
        end
    ).

total_distance(_Prev, []) -> 0;
total_distance(Prev, [Stop | Rest]) ->
    route_optimizer:haversine(Prev, Stop) + total_distance(Stop, Rest).

binary() -> ?LET(N, pos_integer(), integer_to_binary(N)).
```

---

## 3. Integration Testing

```erlang
%% test/integration/api_SUITE.erl
-module(api_SUITE).
-include_lib("common_test/include/ct.hrl").
-export([all/0, init_per_suite/1, end_per_suite/1,
         init_per_testcase/2, end_per_testcase/2]).
-export([test_create_order/1, test_order_lifecycle/1,
         test_rate_limiting/1, test_auth_required/1]).

all() ->
    [test_create_order, test_order_lifecycle,
     test_rate_limiting, test_auth_required].

init_per_suite(Config) ->
    %% Start the application with test config
    application:set_env(myapp, db_pool_size, 3),
    application:set_env(myapp, port, 8099),
    {ok, _} = application:ensure_all_started(myapp),
    %% Create test DB
    setup_test_db(),
    [{base_url, <<"http://localhost:8099">>} | Config].

end_per_suite(_Config) ->
    application:stop(myapp),
    cleanup_test_db().

init_per_testcase(_Test, Config) ->
    %% Create test user and get token
    UserId = create_test_user(Config),
    Token  = create_test_token(UserId),
    [{user_id, UserId}, {token, Token} | Config].

end_per_testcase(_Test, Config) ->
    delete_test_user(proplists:get_value(user_id, Config)).

test_create_order(Config) ->
    BaseUrl = proplists:get_value(base_url, Config),
    Token   = proplists:get_value(token, Config),
    Body    = #{items => [#{product_id => <<"p1">>, qty => 2}]},

    {ok, {{_Vsn, 201, _}, Headers, RespBody}} =
        httpc:request(post,
            {<<BaseUrl/binary, "/api/v2/orders">>,
             [{"authorization", "Bearer " ++ binary_to_list(Token)}],
             "application/json",
             json:encode(Body)},
            [], []),

    Order = json:decode(list_to_binary(RespBody)),
    ct:comment("Created order: ~p", [maps:get(<<"id">>, Order)]),

    %% Assertions
    ?assertMatch(<<"order_", _/binary>>, maps:get(<<"id">>, Order)),
    ?assert(lists:keymember("location", 1, Headers)),
    ok.

test_order_lifecycle(Config) ->
    BaseUrl = proplists:get_value(base_url, Config),
    Token   = proplists:get_value(token, Config),

    %% Create
    {ok, OrderId} = api_create_order(BaseUrl, Token, test_order()),

    %% Verify created state
    {ok, Order1} = api_get_order(BaseUrl, Token, OrderId),
    ?assertEqual(<<"pending">>, maps:get(<<"status">>, Order1)),

    %% Pay for it
    ok = api_pay_order(BaseUrl, Token, OrderId),

    %% Verify paid state
    {ok, Order2} = api_get_order(BaseUrl, Token, OrderId),
    ?assertEqual(<<"processing">>, maps:get(<<"status">>, Order2)),
    ok.

test_rate_limiting(Config) ->
    BaseUrl = proplists:get_value(base_url, Config),
    Token   = proplists:get_value(token, Config),

    %% Make requests until rate limited
    Results = [api_get_order(BaseUrl, Token, <<"x">>) || _ <- lists:seq(1, 150)],
    RateLimited = [R || R <- Results, R =:= {error, rate_limited}],
    ?assert(length(RateLimited) > 0),
    ok.

test_auth_required(Config) ->
    BaseUrl = proplists:get_value(base_url, Config),
    {ok, {{_, Code, _}, _, _}} = httpc:request(get,
        {<<BaseUrl/binary, "/api/v2/orders">>, []}, [], []),
    ?assertEqual(401, Code),
    ok.

%% Helpers
api_create_order(_, _, _) -> {ok, <<"order_test">>}.
api_get_order(_, _, _)    -> {ok, #{<<"status">> => <<"pending">>}}.
api_pay_order(_, _, _)    -> ok.
test_order()               -> #{items => []}.
setup_test_db()            -> ok.
cleanup_test_db()          -> ok.
create_test_user(_)        -> <<"user_test">>.
create_test_token(_)       -> <<"test_token">>.
delete_test_user(_)        -> ok.
```

---

## 4. Load Testing

```erlang
%% load_test.erl — in-process load testing harness
-module(load_test).
-export([run/1, ramp_up/3]).

-record(result, {
    total       = 0 :: integer(),
    success     = 0 :: integer(),
    error       = 0 :: integer(),
    latencies   = [] :: list(),
    start_time  :: integer(),
    end_time    :: integer()
}).

run(Opts) ->
    Concurrency = maps:get(concurrency, Opts, 10),
    Duration    = maps:get(duration_ms, Opts, 10000),
    Fun         = maps:get(fun_, Opts),

    StartTime = erlang:monotonic_time(millisecond),
    EndTime   = StartTime + Duration,

    %% Spawn workers
    Workers = [spawn_monitor(fun() ->
        worker_loop(Fun, EndTime, [])
    end) || _ <- lists:seq(1, Concurrency)],

    %% Collect results
    Results = collect_results(Workers, #result{start_time = StartTime}),
    print_report(Results#{end_time => erlang:monotonic_time(millisecond)}).

ramp_up(Fun, MaxConcurrency, RampSeconds) ->
    Step = MaxConcurrency div RampSeconds,
    lists:foreach(fun(C) ->
        run(#{concurrency => C, duration_ms => 1000, fun_ => Fun}),
        io:format("Ramp: ~p concurrent workers OK~n", [C])
    end, lists:seq(Step, MaxConcurrency, Step)).

worker_loop(Fun, EndTime, Latencies) ->
    case erlang:monotonic_time(millisecond) >= EndTime of
        true ->
            exit({done, Latencies});
        false ->
            T0 = erlang:monotonic_time(microsecond),
            Result = try Fun() catch _:_ -> error end,
            T1 = erlang:monotonic_time(microsecond),
            Lat = {T1 - T0, Result},
            worker_loop(Fun, EndTime, [Lat | Latencies])
    end.

collect_results([], Acc) -> Acc;
collect_results([{Pid, Ref} | Rest], Acc) ->
    receive
        {'DOWN', Ref, process, Pid, {done, Latencies}} ->
            Total   = length(Latencies),
            Success = length([1 || {_, ok} <- Latencies]),
            Errors  = Total - Success,
            Lats    = [L || {L, _} <- Latencies],
            NewAcc = Acc#result{
                total     = Acc#result.total + Total,
                success   = Acc#result.success + Success,
                error     = Acc#result.error + Errors,
                latencies = Acc#result.latencies ++ Lats
            },
            collect_results(Rest, NewAcc)
    after 60000 -> Acc
    end.

print_report(Result) ->
    Sorted   = lists:sort(Result#result.latencies),
    N        = length(Sorted),
    DurationS = (Result#result.end_time - Result#result.start_time) / 1000,
    RPS       = Result#result.total / DurationS,
    io:format("~n=== Load Test Results ===~n"),
    io:format("Total requests:  ~p~n", [Result#result.total]),
    io:format("Success:         ~p (~.1f%)~n", [Result#result.success,
              Result#result.success / Result#result.total * 100]),
    io:format("Errors:          ~p~n", [Result#result.error]),
    io:format("RPS:             ~.1f~n", [RPS]),
    case N > 0 of
        false -> ok;
        true ->
            io:format("Latency (us):~n"),
            io:format("  p50:  ~p~n", [lists:nth(N div 2, Sorted)]),
            io:format("  p95:  ~p~n", [lists:nth(trunc(N * 0.95), Sorted)]),
            io:format("  p99:  ~p~n", [lists:nth(trunc(N * 0.99), Sorted)]),
            io:format("  max:  ~p~n", [lists:last(Sorted)])
    end.
```

---

## 5. Contract Testing

```erlang
%% contract_test.erl — verify API contracts between services
-module(contract_test).
-include_lib("eunit/include/eunit.hrl").

%% Define the contract between producer and consumer
-define(ORDER_CONTRACT, #{
    required_fields => [id, status, user_id, items, created_at],
    field_types => #{
        id         => binary,
        status     => atom,
        user_id    => binary,
        items      => list,
        created_at => integer
    }
}).

verify_contract_test_() ->
    [
        {"Order response matches contract",
         fun test_order_contract/0},
        {"Event payload matches contract",
         fun test_event_contract/0}
    ].

test_order_contract() ->
    {ok, Order} = orders:create(#{user_id => <<"u1">>,
                                   items   => []}),
    verify_schema(Order, ?ORDER_CONTRACT).

test_event_contract() ->
    %% Subscribe and capture the event
    event_bus:subscribe(order_events, self()),
    orders:create(#{user_id => <<"u1">>, items => []}),
    receive
        {event, order_events, Event} ->
            verify_schema(Event, #{
                required_fields => [type, order_id, timestamp],
                field_types => #{
                    type      => atom,
                    order_id  => binary,
                    timestamp => integer
                }
            })
    after 1000 -> ?assert(false)
    end.

verify_schema(Data, #{required_fields := Required, field_types := Types}) ->
    %% Check all required fields present
    lists:foreach(fun(Field) ->
        ?assert(maps:is_key(Field, Data),
                io_lib:format("Missing field: ~p", [Field]))
    end, Required),
    %% Check field types
    maps:foreach(fun(Field, ExpectedType) ->
        case maps:get(Field, Data, undefined) of
            undefined -> ok;  % Optional field
            Value ->
                ?assert(check_type(Value, ExpectedType),
                        io_lib:format("~p: expected ~p, got ~p",
                                      [Field, ExpectedType, Value]))
        end
    end, Types).

check_type(V, binary)  -> is_binary(V);
check_type(V, atom)    -> is_atom(V);
check_type(V, integer) -> is_integer(V);
check_type(V, list)    -> is_list(V);
check_type(V, map)     -> is_map(V);
check_type(_, _)       -> true.
```

---

## 6. Chaos Engineering

```erlang
%% chaos.erl — inject failures for resilience testing
-module(chaos).
-export([inject_latency/2, inject_error/2, kill_random_worker/1,
         network_partition/2]).

%% Wrap a function with artificial latency
inject_latency(Fun, MaxLatencyMs) ->
    fun() ->
        case application:get_env(myapp, chaos_enabled, false) of
            true ->
                Delay = rand:uniform(MaxLatencyMs),
                timer:sleep(Delay),
                Fun();
            false ->
                Fun()
        end
    end.

%% Inject random errors at a given rate
inject_error(Fun, ErrorRate) ->
    fun() ->
        case application:get_env(myapp, chaos_enabled, false) of
            true ->
                case rand:uniform(100) =< ErrorRate of
                    true  -> error(chaos_injected_error);
                    false -> Fun()
                end;
            false ->
                Fun()
        end
    end.

%% Kill a random worker in a pool to test recovery
kill_random_worker(PoolName) ->
    Workers = supervisor:which_children(PoolName),
    case Workers of
        [] -> no_workers;
        _ ->
            {_, Pid, _, _} = lists:nth(rand:uniform(length(Workers)), Workers),
            exit(Pid, kill),
            {killed, Pid}
    end.

%% Simulate network partition between two processes
network_partition(ProcessA, ProcessB) ->
    %% In real implementation, use traffic control (tc) to block packets
    %% For in-process simulation, we intercept and drop messages
    spawn(fun() ->
        erlang:monitor(process, ProcessA),
        erlang:monitor(process, ProcessB),
        partition_loop(ProcessA, ProcessB)
    end).

partition_loop(A, B) ->
    receive
        {'DOWN', _, _, _, _} -> ok;
        restore -> ok
    after 5000 ->
        %% Restore after 5 seconds
        io:format("Partition healed between ~p and ~p~n", [A, B])
    end.
```

---

## 7. แบบฝึกหัด

1. เขียน property test สำหรับ `text_analyzer:tokenize/1` ที่ verify ว่า tokenize → join ไม่สูญเสียข้อมูล
2. สร้าง Common Test suite ที่ test full HTTP lifecycle ตั้งแต่ login → create order → pay
3. Implement test data factory ที่ generate realistic test data ด้วย PropEr generators
4. สร้าง chaos test ที่รัน application และ inject failures แบบ continuous

---

## สรุป Part 84

✅ Testing pyramid: EUnit/Common Test/PropEr/load testing hierarchy  
✅ Property-based testing: PropEr generators, invariant properties  
✅ Integration testing: Common Test suite with test DB setup/teardown  
✅ Load testing: in-process harness with p50/p95/p99 latency reporting  
✅ Contract testing: schema verification between producers and consumers  
✅ Chaos engineering: latency injection, error rates, random worker kills  

---

*Part 84/100 | [← ก่อนหน้า](../part83/README.md) | [ถัดไป →](../part85/README.md)*
