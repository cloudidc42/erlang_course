# Part 39: Advanced Testing Strategies

> **"Untested code is broken code — you just don't know it yet"**  
> Code ที่ไม่ได้ test คือ code ที่พัง — แค่ยังไม่รู้เท่านั้น

---

## สารบัญ

1. [Testing Pyramid ใน Erlang](#1-testing-pyramid-ใน-erlang)
2. [Property-Based Testing ขั้นสูง](#2-property-based-testing-ขั้นสูง)
3. [Integration Tests ด้วย Common Test](#3-integration-tests-ด้วย-common-test)
4. [Contract Testing](#4-contract-testing)
5. [Chaos Engineering](#5-chaos-engineering)
6. [Performance Testing](#6-performance-testing)
7. [Test Doubles (meck)](#7-test-doubles-meck)
8. [Test Coverage](#8-test-coverage)
9. [แบบฝึกหัด](#9-แบบฝึกหัด)

---

## 1. Testing Pyramid ใน Erlang

```
Testing Pyramid:

        /\
       /  \   E2E Tests (Common Test, few)
      /----\
     /      \  Integration Tests (Common Test + meck)
    /--------\
   /          \ Unit Tests (EUnit, many)
  /------------\

Unit Tests (EUnit):
  - Pure functions, pure logic
  - Fast: milliseconds
  - No I/O, no DB

Integration Tests (Common Test):
  - With DB, with Cowboy server
  - Test multiple modules together
  - Slower: seconds

E2E Tests:
  - Full system with real dependencies
  - Slowest but most realistic
  - Minimal set

Property Tests (PropEr):
  - Generate random inputs
  - Find edge cases automatically
  - Middle ground: unit + properties
```

---

## 2. Property-Based Testing ขั้นสูง

```erlang
%% math_prop.erl — PropEr property tests
-module(math_prop).
-include_lib("proper/include/proper.hrl").
-include_lib("eunit/include/eunit.hrl").

%% Property: reverse ของ reverse = original
prop_reverse_involution() ->
    ?FORALL(L, list(integer()),
        lists:reverse(lists:reverse(L)) =:= L).

%% Property: sort idempotent
prop_sort_idempotent() ->
    ?FORALL(L, list(integer()),
        begin
            Sorted = lists:sort(L),
            lists:sort(Sorted) =:= Sorted
        end).

%% Property: sort preserves length
prop_sort_length() ->
    ?FORALL(L, list(integer()),
        length(lists:sort(L)) =:= length(L)).

%% Custom generator: non-empty list
non_empty_list() ->
    ?SUCHTHAT(L, list(integer()), L =/= []).

%% Property: max is in list
prop_max_in_list() ->
    ?FORALL(L, non_empty_list(),
        begin
            Max = lists:max(L),
            lists:member(Max, L)
        end).

%% Property: encode/decode roundtrip
prop_json_roundtrip() ->
    ?FORALL(Map, json_map(),
        begin
            Encoded = jsx:encode(Map),
            Decoded = jsx:decode(Encoded, [return_maps]),
            Decoded =:= Map
        end).

json_map() ->
    ?LET(Keys,   list(json_key()),
    ?LET(Values, list(json_value()),
        maps:from_list(lists:zip(Keys, Values)))).

json_key()   -> ?LET(K, non_empty(binary()), K).
json_value() -> oneof([integer(), float(), binary(), boolean()]).

%% Run all properties
proper_test_() ->
    {timeout, 60, fun() ->
        [] = proper:module(?MODULE, [quiet, {numtests, 100}])
    end}.
```

---

## 3. Integration Tests ด้วย Common Test

```erlang
%% api_SUITE.erl — Integration tests for REST API
-module(api_SUITE).
-include_lib("common_test/include/ct.hrl").

-export([all/0, groups/0,
         init_per_suite/1, end_per_suite/1,
         init_per_group/2, end_per_group/2,
         init_per_testcase/2, end_per_testcase/2]).

-export([
    test_create_user/1,
    test_get_user/1,
    test_update_user/1,
    test_delete_user/1,
    test_auth_required/1,
    test_rate_limiting/1
]).

all() ->
    [{group, users_api}, {group, security}].

groups() ->
    [
        {users_api, [sequence], [
            test_create_user,
            test_get_user,
            test_update_user,
            test_delete_user
        ]},
        {security, [parallel], [
            test_auth_required,
            test_rate_limiting
        ]}
    ].

init_per_suite(Config) ->
    %% Start application
    {ok, _} = application:ensure_all_started(myapp),
    %% Create test DB
    db:query("CREATE TABLE IF NOT EXISTS users_test "
             "(id SERIAL PRIMARY KEY, name TEXT, email TEXT)", []),
    [{base_url, "http://localhost:8080"} | Config].

end_per_suite(Config) ->
    db:query("DROP TABLE IF EXISTS users_test", []),
    application:stop(myapp),
    Config.

init_per_group(users_api, Config) ->
    %% Create test user for auth
    Token = create_test_token(admin),
    [{auth_token, Token} | Config];
init_per_group(_, Config) ->
    Config.

end_per_group(_, _Config) -> ok.

init_per_testcase(_TestCase, Config) -> Config.
end_per_testcase(_TestCase, _Config) -> ok.

%% Test cases
test_create_user(Config) ->
    BaseUrl  = ?config(base_url, Config),
    Token    = ?config(auth_token, Config),
    Url      = BaseUrl ++ "/api/users",
    Body     = jsx:encode(#{name => <<"Alice">>, email => <<"alice@test.com">>}),
    {ok, 201, _Headers, RespBody} = http_test:post(Url, Token, Body),
    #{<<"id">> := UserId} = jsx:decode(RespBody, [return_maps]),
    ct:pal("Created user id: ~p", [UserId]),
    [{user_id, UserId} | Config].

test_get_user(Config) ->
    BaseUrl = ?config(base_url, Config),
    Token   = ?config(auth_token, Config),
    UserId  = ?config(user_id, Config),
    Url     = BaseUrl ++ "/api/users/" ++ integer_to_list(UserId),
    {ok, 200, _, Body} = http_test:get(Url, Token),
    #{<<"name">> := <<"Alice">>} = jsx:decode(Body, [return_maps]).

test_update_user(Config) ->
    BaseUrl = ?config(base_url, Config),
    Token   = ?config(auth_token, Config),
    UserId  = ?config(user_id, Config),
    Url     = BaseUrl ++ "/api/users/" ++ integer_to_list(UserId),
    Body    = jsx:encode(#{name => <<"Alice Updated">>}),
    {ok, 200, _, RespBody} = http_test:put(Url, Token, Body),
    #{<<"name">> := <<"Alice Updated">>} = jsx:decode(RespBody, [return_maps]).

test_delete_user(Config) ->
    BaseUrl = ?config(base_url, Config),
    Token   = ?config(auth_token, Config),
    UserId  = ?config(user_id, Config),
    Url     = BaseUrl ++ "/api/users/" ++ integer_to_list(UserId),
    {ok, 204, _, _} = http_test:delete(Url, Token),
    %% Verify deleted
    {ok, 404, _, _} = http_test:get(Url, Token).

test_auth_required(Config) ->
    BaseUrl = ?config(base_url, Config),
    {ok, 401, _, _} = http_test:get(BaseUrl ++ "/api/users", no_token).

test_rate_limiting(Config) ->
    BaseUrl = ?config(base_url, Config),
    Token   = ?config(auth_token, Config),
    %% Make 200 requests quickly
    Results = [http_test:get(BaseUrl ++ "/api/users", Token)
               || _ <- lists:seq(1, 200)],
    Codes = [S || {ok, S, _, _} <- Results],
    TooMany = length([C || C <- Codes, C =:= 429]),
    ct:pal("429 responses: ~p", [TooMany]),
    true = TooMany > 0.

create_test_token(Role) ->
    jwt:sign(#{sub => 1, role => Role}).
```

---

## 4. Contract Testing

```erlang
%% contract_test.erl — Ensure API contracts are maintained
-module(contract_test).
-include_lib("eunit/include/eunit.hrl").

%% Define API contracts
user_schema() ->
    #{
        <<"id">>    => {required, integer},
        <<"name">>  => {required, binary},
        <<"email">> => {required, binary},
        <<"created_at">> => {required, integer}
    }.

%% Validate response against contract
validate_contract(Data, Schema) ->
    maps:fold(fun(Field, {required, Type}, Acc) ->
        case maps:find(Field, Data) of
            {ok, Value} ->
                case check_type(Value, Type) of
                    true  -> Acc;
                    false -> [#{field => Field, error => wrong_type} | Acc]
                end;
            error ->
                [#{field => Field, error => missing} | Acc]
        end
    end, [], Schema).

check_type(V, integer) -> is_integer(V);
check_type(V, binary)  -> is_binary(V);
check_type(V, list)    -> is_list(V);
check_type(V, map)     -> is_map(V).

%% Test: user API response matches contract
user_api_contract_test() ->
    %% Call actual API
    {ok, Body} = http_test:get("http://localhost:8080/api/users/1", token),
    User = jsx:decode(Body, [return_maps]),
    Errors = validate_contract(User, user_schema()),
    ?assertEqual([], Errors, "User API response violates contract").
```

---

## 5. Chaos Engineering

```erlang
%% chaos.erl — Fault injection for resilience testing
-module(chaos).
-export([start/1, stop/0, inject_fault/2]).

-define(CHAOS_TABLE, chaos_faults).

start(Mode) ->
    ets:new(?CHAOS_TABLE, [named_table, set, public]),
    logger:warning("CHAOS MODE: ~p enabled!", [Mode]),
    case Mode of
        {latency, Ms}     -> inject_fault(all, {latency, Ms});
        {error_rate, Rate} -> inject_fault(all, {error_rate, Rate});
        {kill_workers, N}  -> kill_random_workers(N)
    end.

stop() ->
    ets:delete(?CHAOS_TABLE),
    logger:info("Chaos mode stopped").

inject_fault(Target, Fault) ->
    ets:insert(?CHAOS_TABLE, {Target, Fault}).

%% Chaos wrapper: use around external calls
with_chaos(Target, Fun) ->
    case ets:lookup(?CHAOS_TABLE, Target) of
        [] ->
            Fun();
        [{_, {latency, Ms}}] ->
            timer:sleep(Ms),
            Fun();
        [{_, {error_rate, Rate}}] ->
            case rand:uniform() < Rate of
                true  -> {error, chaos_fault};
                false -> Fun()
            end;
        _ ->
            Fun()
    end.

kill_random_workers(N) ->
    Workers = [P || P <- processes(),
               {current_function, {job_worker, _, _}} <-
                   [process_info(P, current_function)]],
    ToKill = lists:sublist(shuffle(Workers), N),
    lists:foreach(fun(P) ->
        logger:warning("Chaos: killing worker ~p", [P]),
        exit(P, chaos_kill)
    end, ToKill).

shuffle(List) ->
    [X || {_, X} <- lists:sort([{rand:uniform(), E} || E <- List])].
```

---

## 6. Performance Testing

```erlang
%% load_test.erl — HTTP load testing
-module(load_test).
-export([run/3]).

%% run/3: Url, Concurrency, TotalRequests
run(Url, Concurrency, Total) ->
    Self = self(),
    PerWorker = Total div Concurrency,

    T1 = erlang:monotonic_time(millisecond),

    Workers = [spawn_link(fun() ->
        Results = [time_request(Url) || _ <- lists:seq(1, PerWorker)],
        Self ! {results, Results}
    end) || _ <- lists:seq(1, Concurrency)],

    AllResults = collect_results(length(Workers), []),
    T2 = erlang:monotonic_time(millisecond),

    analyze(AllResults, T2 - T1, Total).

time_request(Url) ->
    T1 = erlang:monotonic_time(microsecond),
    Result = hackney:request(get, Url, [], <<>>, [with_body]),
    T2 = erlang:monotonic_time(microsecond),
    Status = case Result of
        {ok, S, _, _} -> S;
        {error, _}    -> 0
    end,
    {Status, T2 - T1}.

collect_results(0, Acc) -> Acc;
collect_results(N, Acc) ->
    receive
        {results, R} -> collect_results(N-1, R ++ Acc)
    after 60000 ->
        Acc
    end.

analyze(Results, TotalMs, Total) ->
    Times   = [T || {_, T} <- Results],
    Success = length([S || {S, _} <- Results, S >= 200, S < 300]),
    Errors  = Total - Success,
    Sorted  = lists:sort(Times),
    Mean    = lists:sum(Times) div length(Times),
    P50     = percentile(Sorted, 50),
    P95     = percentile(Sorted, 95),
    P99     = percentile(Sorted, 99),
    TPS     = round(Total * 1000 / TotalMs),

    io:format("~n=== Load Test Results ===~n"),
    io:format("Total requests: ~p~n", [Total]),
    io:format("Success: ~p (~.1f%)~n",
              [Success, Success * 100 / Total]),
    io:format("Errors: ~p~n", [Errors]),
    io:format("Duration: ~pms~n", [TotalMs]),
    io:format("TPS: ~p req/s~n", [TPS]),
    io:format("Mean: ~pµs~n", [Mean]),
    io:format("P50:  ~pµs~n", [P50]),
    io:format("P95:  ~pµs~n", [P95]),
    io:format("P99:  ~pµs~n", [P99]).

percentile(Sorted, P) ->
    Idx = max(1, round(length(Sorted) * P / 100)),
    lists:nth(Idx, Sorted).
```

---

## 7. Test Doubles (meck)

```erlang
%% ใช้ meck สำหรับ mock/stub ใน tests
%% deps: {meck, "0.9.2"}

order_service_test_() ->
    {setup,
     fun() ->
         %% Mock external dependencies
         meck:new(payment_gateway, [non_strict]),
         meck:new(email_service, [non_strict]),
         meck:new(inventory, [non_strict])
     end,
     fun(_) ->
         meck:unload(payment_gateway),
         meck:unload(email_service),
         meck:unload(inventory)
     end,
     [
         {"successful order", fun test_successful_order/0},
         {"payment failure",  fun test_payment_failure/0},
         {"out of stock",     fun test_out_of_stock/0}
     ]}.

test_successful_order() ->
    meck:expect(inventory,        check_stock,  fun(_, _) -> ok end),
    meck:expect(payment_gateway,  charge,       fun(_, _) -> {ok, <<"ch_123">>} end),
    meck:expect(inventory,        decrement,    fun(_, _) -> ok end),
    meck:expect(email_service,    send_confirm, fun(_) -> ok end),

    Order = #{user_id => 1, item_id => 42, quantity => 1, amount => 100},
    {ok, _} = order_service:place(Order),

    %% Verify interactions
    ?assert(meck:called(payment_gateway, charge, [100, '_'])),
    ?assert(meck:called(email_service, send_confirm, ['_'])).

test_payment_failure() ->
    meck:expect(inventory,       check_stock, fun(_, _) -> ok end),
    meck:expect(payment_gateway, charge,      fun(_, _) -> {error, declined} end),

    Order = #{user_id => 1, item_id => 42, quantity => 1, amount => 100},
    {error, payment_declined} = order_service:place(Order),

    %% Inventory should NOT be decremented
    ?assertNot(meck:called(inventory, decrement, '_')).

test_out_of_stock() ->
    meck:expect(inventory, check_stock, fun(_, _) -> {error, out_of_stock} end),

    Order = #{user_id => 1, item_id => 99, quantity => 5, amount => 500},
    {error, out_of_stock} = order_service:place(Order),

    ?assertNot(meck:called(payment_gateway, charge, '_')).
```

---

## 8. Test Coverage

```bash
# rebar3 cover — generate coverage report
rebar3 eunit --cover
rebar3 cover --verbose

# Generate HTML report
rebar3 cover --report html

# View coverage
open _build/test/cover/index.html

# With Common Test
rebar3 ct --cover
rebar3 cover

# Coverage in rebar.config
{cover_enabled, true}.
{cover_opts, [verbose]}.
{cover_excl_mods, [
    myapp_app,    %% app callback, too simple to cover
    myapp_sup     %% supervisor, hard to test line-by-line
]}.
```

```erlang
%% Force coverage of important paths

%% ตัวอย่าง: 100% coverage สำหรับ validator
validator_coverage_test_() ->
    [
        %% Happy paths
        ?_assertEqual({ok, #{name => <<"Alice">>}},
                      validator:validate(#{name => <<"Alice">>},
                                        [{name, {required}}])),

        %% Error cases
        ?_assertMatch({error, #{name := _}},
                      validator:validate(#{},
                                        [{name, {required}}])),
        ?_assertMatch({error, #{email := _}},
                      validator:validate(#{email => <<"notanemail">>},
                                        [{email, {email}}])),
        ?_assertMatch({error, #{age := _}},
                      validator:validate(#{age => 17},
                                        [{age, {range, 18, 120}}]))
    ].
```

---

## 9. แบบฝึกหัด

### Exercise: Complete Test Suite

```erlang
%% เขียน test suite สมบูรณ์สำหรับ todo_service

%% todo_service_tests.erl
-module(todo_service_tests).
-include_lib("eunit/include/eunit.hrl").
-include_lib("proper/include/proper.hrl").

%% Unit Tests
create_todo_test() ->
    {ok, Todo} = todo_service:create(<<"Buy milk">>),
    ?assertMatch(#{title := <<"Buy milk">>, done := false}, Todo).

complete_todo_test() ->
    {ok, Todo} = todo_service:create(<<"Task">>),
    Id = maps:get(id, Todo),
    {ok, Done} = todo_service:complete(Id),
    ?assert(maps:get(done, Done)).

delete_nonexistent_test() ->
    ?assertEqual({error, not_found},
                 todo_service:delete(999999)).

%% Property Tests
prop_create_then_find() ->
    ?FORALL(Title, binary(),
        begin
            {ok, #{id := Id}} = todo_service:create(Title),
            {ok, Found} = todo_service:find(Id),
            maps:get(title, Found) =:= Title
        end).

prop_all_todos_findable() ->
    ?FORALL(Titles, list(non_empty(binary())),
        begin
            Ids = [begin {ok, #{id:=Id}} = todo_service:create(T), Id
                   end || T <- Titles],
            All = todo_service:list(),
            AllIds = [maps:get(id, T) || T <- All],
            lists:all(fun(Id) -> lists:member(Id, AllIds) end, Ids)
        end).
```

---

## สรุป Part 39

✅ Testing pyramid ใน Erlang  
✅ Property-based testing ขั้นสูง (PropEr)  
✅ Integration tests ด้วย Common Test  
✅ Contract testing  
✅ Chaos engineering  
✅ Load testing  
✅ Test doubles ด้วย meck  
✅ Test coverage

---

*Part 39/100 | [← ก่อนหน้า](../part38/README.md) | [ถัดไป →](../part40/README.md)*
