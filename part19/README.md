# Part 19: Testing ด้วย EUnit และ Common Test

> **"Tests are the safety net that lets you refactor with confidence"**  
> Tests คือตาข่ายนิรภัยที่ให้คุณ refactor โดยไม่กลัว

---

## สารบัญ

1. [EUnit พื้นฐาน](#1-eunit-พื้นฐาน)
2. [EUnit Test Macros](#2-eunit-test-macros)
3. [Test Fixtures](#3-test-fixtures)
4. [Test Generators](#4-test-generators)
5. [Common Test (CT)](#5-common-test-ct)
6. [CT Suites](#6-ct-suites)
7. [CT Groups](#7-ct-groups)
8. [Mocking ด้วย meck](#8-mocking-ด้วย-meck)
9. [Property-Based Testing (PropEr)](#9-property-based-testing-proper)
10. [แบบฝึกหัด](#10-แบบฝึกหัด)

---

## 1. EUnit พื้นฐาน

```erlang
%% test/mymod_tests.erl
-module(mymod_tests).
-include_lib("eunit/include/eunit.hrl").

%% Simple test: ฟังก์ชันที่ชื่อลงท้ายด้วย _test
my_first_test() ->
    ?assert(1 + 1 =:= 2).

addition_test() ->
    ?assertEqual(4, 2 + 2).

%% Test ใน source module (ถ้า compile ด้วย TEST)
-ifdef(TEST).
-include_lib("eunit/include/eunit.hrl").

square_test() ->
    ?assertEqual(9, square(3)).
-endif.

%% Run tests
%% rebar3 eunit
%% eunit:test(mymod_tests).
%% eunit:test(mymod_tests, [verbose]).
```

---

## 2. EUnit Test Macros

```erlang
%% Assertion macros
?assert(BoolExpr).
?assertNot(BoolExpr).
?assertEqual(Expected, Actual).
?assertNotEqual(NotExpected, Actual).
?assertMatch(Pattern, Expr).        %% pattern match
?assertNotMatch(Pattern, Expr).

%% Exception macros
?assertError(Error, Expr).
?assertExit(Exit, Expr).
?assertThrow(Throw, Expr).
?assertException(Class, Term, Expr).

%% Process macros
?debugVal(Expr).   %% print value, คืน value
?debugMsg(Msg).    %% print message
?debugTime(Msg, Expr).  %% print execution time

%% ตัวอย่าง
parse_test() ->
    ?assertMatch({ok, _}, parse("valid input")),
    ?assertMatch({error, _}, parse("invalid")),
    ?assertError(badarg, parse(123)).

%% assertEqual vs assertMatch
?assertEqual({ok, 42}, Result).   %% exact match
?assertMatch({ok, _}, Result).    %% pattern match (ไม่สนค่า)
```

---

## 3. Test Fixtures

```erlang
%% Setup/Teardown
setup_test_() ->
    {setup,
     fun setup/0,
     fun teardown/1,
     fun tests/1}.

setup() ->
    application:start(myapp),
    #{db => db:connect()}.

teardown(#{db := Db}) ->
    db:disconnect(Db),
    application:stop(myapp).

tests(Context) ->
    [
        ?_test(test_create(Context)),
        ?_test(test_read(Context)),
        ?_test(test_delete(Context))
    ].

%% ?_test(...) สร้าง test function (lazy)
test_create(#{db := Db}) ->
    ?assertEqual({ok, 1}, db:insert(Db, {user, "Alice"})).

%% Setup with foreach (แต่ละ test มี setup/teardown)
setup_each_test_() ->
    {foreach,
     fun setup/0,
     fun teardown/1,
     [
         fun test_one/1,
         fun test_two/1
     ]}.
```

---

## 4. Test Generators

```erlang
%% Test Generator: ฟังก์ชันที่ชื่อลงท้ายด้วย _test_
%% คืน list ของ tests หรือ test structure

addition_tests_test_() ->
    [
        ?_assertEqual(4,   2 + 2),
        ?_assertEqual(10,  5 + 5),
        ?_assertEqual(100, 50 + 50)
    ].

%% Named tests
named_test_() ->
    [
        {"add two numbers", ?_assertEqual(4, 2 + 2)},
        {"add zero", ?_assertEqual(5, 5 + 0)},
        {"negative numbers", ?_assertEqual(-1, -3 + 2)}
    ].

%% Parameterized tests
param_test_() ->
    Cases = [
        {1, 1, 2},
        {2, 3, 5},
        {0, 0, 0},
        {-1, 1, 0}
    ],
    [{lists:concat([A, "+", B, "=", C]),
      ?_assertEqual(C, A + B)} || {A, B, C} <- Cases].

%% Timeout test
slow_test_() ->
    {timeout, 10,    %% 10 seconds timeout
     ?_test(slow_operation())}.
```

---

## 5. Common Test (CT)

```erlang
%% Common Test: สำหรับ integration testing, acceptance testing
%% ใช้สำหรับ:
%% - Application-level tests
%% - Tests ที่ต้องการ setup ซับซ้อน
%% - Tests ที่ต้องการ external resources
%% - Parallel testing
%% - HTML reports

%% รัน CT
%% rebar3 ct
%% ct:run_test([{suite, myapp_SUITE}]).
```

---

## 6. CT Suites

```erlang
%% test/myapp_SUITE.erl
-module(myapp_SUITE).
-include_lib("common_test/include/ct.hrl").

%% Required
-export([all/0]).

%% Lifecycle callbacks (optional)
-export([suite/0, init_per_suite/1, end_per_suite/1,
         init_per_testcase/2, end_per_testcase/2]).

%% Test cases
-export([test_add/1, test_subtract/1, test_error/1]).

%% ต้อง return list ของ test cases
all() ->
    [test_add, test_subtract, test_error].

suite() ->
    [{timetrap, {seconds, 30}}].  %% timeout สำหรับทุก test

init_per_suite(Config) ->
    ok = application:start(myapp),
    [{started, true} | Config].

end_per_suite(_Config) ->
    application:stop(myapp),
    ok.

init_per_testcase(test_add, Config) ->
    [{operand, 10} | Config];
init_per_testcase(_, Config) ->
    Config.

end_per_testcase(_, _Config) ->
    ok.

%% Test cases — รับ Config list
test_add(Config) ->
    Operand = proplists:get_value(operand, Config, 5),
    Result = mymath:add(Operand, 2),
    ct:pal("Add result: ~p", [Result]),     %% print ใน CT log
    true = (Result =:= Operand + 2).        %% assertion

test_subtract(_Config) ->
    Result = mymath:subtract(10, 3),
    7 = Result.

test_error(_Config) ->
    {error, division_by_zero} = mymath:divide(5, 0).
```

---

## 7. CT Groups

```erlang
%% Groups: จัดกลุ่ม test cases
all() ->
    [
        {group, arithmetic},
        {group, string_ops},
        test_standalone
    ].

groups() ->
    [
        {arithmetic, [parallel],   %% รันพร้อมกัน
         [test_add, test_sub, test_mul]},
        {string_ops, [sequence],   %% รัน sequential
         [test_concat, test_split]}
    ].

%% Group properties:
%% parallel    — tests run in parallel
%% sequence    — tests run in sequence (default)
%% shuffle     — random order
%% {shuffle, Seed}
%% repeat      — repeat all tests
%% {repeat, N}
%% {repeat_until_ok, N}
%% {repeat_until_fail, N}

init_per_group(arithmetic, Config) ->
    %% setup สำหรับ group นี้
    Config;
init_per_group(_, Config) ->
    Config.

end_per_group(_, _Config) ->
    ok.
```

---

## 8. Mocking ด้วย meck

```erlang
%% rebar.config: {deps, [{meck, "0.9.2"}]}

-module(my_service_tests).
-include_lib("eunit/include/eunit.hrl").

mock_test_() ->
    {setup,
     fun() ->
         meck:new(http_client, [non_strict]),
         meck:expect(http_client, get,
             fun("https://api.example.com/user/1") ->
                 {ok, #{<<"id">> => 1, <<"name">> => <<"Alice">>}}
             end)
     end,
     fun(_) ->
         meck:unload(http_client)
     end,
     [
         ?_test(begin
             {ok, User} = my_service:get_user(1),
             ?assertEqual(<<"Alice">>, maps:get(<<"name">>, User))
         end)
     ]}.

%% Verify calls
verify_mock_test_() ->
    {setup,
     fun() ->
         meck:new(db, [passthrough]),
         meck:expect(db, insert, fun(_) -> ok end)
     end,
     fun(_) ->
         meck:unload(db)
     end,
     [
         ?_test(begin
             my_service:create_user("Alice"),
             %% ตรวจสอบว่า db:insert ถูกเรียก 1 ครั้ง
             ?assert(meck:called(db, insert, '_')),
             ?assertEqual(1, meck:num_calls(db, insert, '_'))
         end)
     ]}.

%% Mock return sequence
sequence_mock_test() ->
    meck:new(cache, [non_strict]),
    meck:sequence(cache, get, 1, [miss, {ok, "value"}, {ok, "value"}]),
    %% first call: miss, second: {ok, "value"}, ...
    miss = cache:get(key),
    {ok, "value"} = cache:get(key),
    meck:unload(cache).
```

---

## 9. Property-Based Testing (PropEr)

```erlang
%% rebar.config: {deps, [{proper, "1.4.0"}]}

-module(prop_math).
-include_lib("proper/include/proper.hrl").
-include_lib("eunit/include/eunit.hrl").

%% Property: addition is commutative
prop_add_commutative() ->
    ?FORALL({A, B}, {integer(), integer()},
        A + B =:= B + A).

%% Property: sorting preserves length
prop_sort_length() ->
    ?FORALL(List, list(integer()),
        length(lists:sort(List)) =:= length(List)).

%% Property: reverse twice = original
prop_double_reverse() ->
    ?FORALL(List, list(integer()),
        lists:reverse(lists:reverse(List)) =:= List).

%% Custom generator
username() ->
    ?LET(Name, non_empty(list(choose($a, $z))), list_to_binary(Name)).

prop_username_length() ->
    ?FORALL(Name, username(),
        byte_size(Name) > 0).

%% Conditional property
prop_division() ->
    ?FORALL({A, B}, {integer(), pos_integer()},
        is_integer(A div B)).

%% Run properties with EUnit
proper_test_() ->
    Props = [prop_add_commutative, prop_sort_length, prop_double_reverse],
    [{atom_to_list(P),
      ?_assert(proper:quickcheck(?MODULE:P(), [quiet]))}
     || P <- Props].
```

---

## 10. แบบฝึกหัด

### Exercise: Test Calculator Module

```erlang
%% สร้าง test suite สำหรับ calculator module

%% calculator.erl
-module(calculator).
-export([add/2, subtract/2, multiply/2, divide/2, factorial/1]).

add(A, B) -> A + B.
subtract(A, B) -> A - B.
multiply(A, B) -> A * B.
divide(_, 0) -> {error, division_by_zero};
divide(A, B) -> {ok, A / B}.
factorial(0) -> 1;
factorial(N) when N > 0 -> N * factorial(N-1).

%% test/calculator_tests.erl
-module(calculator_tests).
-include_lib("eunit/include/eunit.hrl").

add_test_() ->
    [
        {"positive numbers",   ?_assertEqual(5,  calculator:add(2, 3))},
        {"negative result",    ?_assertEqual(-1, calculator:add(2, -3))},
        {"zeros",              ?_assertEqual(0,  calculator:add(0, 0))},
        {"large numbers",      ?_assertEqual(1000000, calculator:add(500000, 500000))}
    ].

divide_test_() ->
    [
        {"normal division",    ?_assertEqual({ok, 2.5}, calculator:divide(5, 2))},
        {"exact division",     ?_assertEqual({ok, 4.0}, calculator:divide(8, 2))},
        {"division by zero",   ?_assertEqual({error, division_by_zero},
                                            calculator:divide(5, 0))}
    ].

factorial_test_() ->
    Cases = [{0, 1}, {1, 1}, {5, 120}, {10, 3628800}],
    [{lists:concat(["factorial(", N, ")"]),
      ?_assertEqual(Expected, calculator:factorial(N))}
     || {N, Expected} <- Cases].

error_test_() ->
    [
        ?_assertError(badarith, calculator:add("a", 1)),
        ?_assertError(function_clause, calculator:factorial(-1))
    ].
```

---

## สรุป Part 19

✅ EUnit macros: assert, assertEqual, assertMatch  
✅ Test generators (_test_ suffix)  
✅ Setup/Teardown fixtures  
✅ Common Test suites and lifecycle  
✅ CT groups: parallel, sequence, shuffle  
✅ Mocking ด้วย meck: expect, verify, sequence  
✅ Property-based testing ด้วย PropEr  
✅ Integration ด้วย rebar3

---

*Part 19/100 | [← ก่อนหน้า](../part18/README.md) | [ถัดไป →](../part20/README.md)*
