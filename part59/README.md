# Part 59: Parse Transforms and Macros

> **"Erlang lets you extend the compiler — use this power carefully"**  
> Erlang ให้คุณขยาย compiler ได้ — ใช้ power นี้ด้วยความระมัดระวัง

---

## สารบัญ

1. [Compile-Time Macros](#1-compile-time-macros)
2. [Parse Transforms](#2-parse-transforms)
3. [Attribute-Based Code Generation](#3-attribute-based-code-generation)
4. [Custom Compile Checks](#4-custom-compile-checks)
5. [Practical Examples](#5-practical-examples)
6. [แบบฝึกหัด](#6-แบบฝึกหัด)

---

## 1. Compile-Time Macros

```erlang
%% macros.erl — using -define for constants and utilities
-module(macros).

%% Constants
-define(MAX_RETRIES, 3).
-define(TIMEOUT_MS,  5000).
-define(APP_NAME,    myapp).

%% Macro functions (compile-time substitution, not runtime calls)
-define(NOW(), os:system_time(millisecond)).
-define(LOG(Level, Msg, Args), logger:Level(Msg, Args)).
-define(INFO(Msg),  ?LOG(info, Msg, [])).
-define(WARN(Msg),  ?LOG(warning, Msg, [])).
-define(ERROR(Msg), ?LOG(error, Msg, [])).

%% Guard macros
-define(IS_STRING(X), (is_binary(X) orelse is_list(X))).
-define(IS_POS(X),    (is_integer(X) andalso X > 0)).

%% Assert macro (only active in test profile)
-ifdef(TEST).
-define(ASSERT(Expr),
    case Expr of
        true  -> ok;
        false -> error({assertion_failed, ??Expr, ?MODULE, ?LINE})
    end).
-else.
-define(ASSERT(_Expr), ok).
-endif.

%% Conditional compilation
-ifdef(PROD).
-define(DEBUG_LOG(Msg), ok).
-else.
-define(DEBUG_LOG(Msg), io:format("[DEBUG] ~s~n", [Msg])).
-endif.

%% Usage
example() ->
    ?INFO("Starting operation"),
    timer:sleep(?TIMEOUT_MS),
    ?ASSERT(1 + 1 =:= 2),
    ?DEBUG_LOG("Done").
```

---

## 2. Parse Transforms

```erlang
%% log_calls_transform.erl — automatically log all function calls
-module(log_calls_transform).
-export([parse_transform/2]).

parse_transform(Forms, _Options) ->
    [transform_form(Form) || Form <- Forms].

transform_form({function, Line, Name, Arity, Clauses}) ->
    {function, Line, Name, Arity,
     [transform_clause(Name, Arity, C) || C <- Clauses]};
transform_form(Form) ->
    Form.

transform_clause(FunName, Arity, {clause, Line, Patterns, Guards, Body}) ->
    LogCall = {call, Line,
               {remote, Line, {atom, Line, logger}, {atom, Line, debug}},
               [{string, Line, io_lib:format("Calling ~p/~p", [FunName, Arity])},
                {nil, Line}]},
    {clause, Line, Patterns, Guards, [LogCall | Body]}.

%% To use this transform:
%% -compile({parse_transform, log_calls_transform}).
```

---

## 3. Attribute-Based Code Generation

```erlang
%% route_transform.erl — generate router from -route attributes
-module(route_transform).
-export([parse_transform/2]).

parse_transform(Forms, _Options) ->
    Routes = extract_routes(Forms),
    RouterFunction = generate_router(Routes),
    Forms ++ [RouterFunction].

extract_routes(Forms) ->
    [Route || {attribute, _, route, Route} <- Forms].

generate_router(Routes) ->
    Clauses = [generate_clause(R) || R <- Routes] ++ [default_clause()],
    {function, 1, route, 1, Clauses}.

generate_clause({Path, Handler}) ->
    {clause, 1,
     [{bin, 1, [{bin_element, 1, {string, 1, Path}, default, default}]}],
     [],
     [{call, 1, {atom, 1, Handler}, [{var, 1, 'Req'}]}]}.

default_clause() ->
    {clause, 1,
     [{var, 1, '_'}],
     [],
     [{tuple, 1, [{atom, 1, error}, {atom, 1, not_found}]}]}.
```

```erlang
%% Usage of route_transform
-module(my_router).
-compile({parse_transform, route_transform}).

-route({"/api/users", users_handler}).
-route({"/api/posts", posts_handler}).
-route({"/health",    health_handler}).

%% This generates: route(Path) → calls the right handler
```

---

## 4. Custom Compile Checks

```erlang
%% validate_transform.erl — enforce coding standards at compile time
-module(validate_transform).
-export([parse_transform/2]).

parse_transform(Forms, _Options) ->
    Module = get_module_name(Forms),
    Warnings = check_module(Forms, Module),
    [begin
         io:format("WARNING ~p:~p - ~s~n", [Module, Line, Msg]),
         Forms
     end || {Line, Msg} <- Warnings],
    Forms.

check_module(Forms, _Module) ->
    lists:flatten([
        check_functions(Forms),
        check_no_process_dict(Forms)
    ]).

check_functions(Forms) ->
    [begin
         {Line, io_lib:format("Function ~p/~p missing -spec", [Name, Arity])}
     end
     || {function, Line, Name, Arity, _} <- Forms,
        not has_spec(Name, Arity, Forms),
        not is_hidden(Name)].

has_spec(Name, Arity, Forms) ->
    lists:any(fun
        ({attribute, _, spec, {{Name, Arity}, _}}) -> true;
        (_) -> false
    end, Forms).

is_hidden(module_info) -> true;
is_hidden(_) -> false.

check_no_process_dict(Forms) ->
    AllCalls = find_calls(Forms),
    [{Line, "Use of process dictionary (put/get) discouraged"}
     || {call, Line, put_or_get} <- AllCalls].

find_calls(Forms) ->
    lists:flatten([find_in_form(F) || F <- Forms]).

find_in_form({call, Line, {atom, _, put}, _}) -> [{call, Line, put_or_get}];
find_in_form({call, Line, {atom, _, get}, _}) -> [{call, Line, put_or_get}];
find_in_form(Tuple) when is_tuple(Tuple)      ->
    [find_in_form(E) || E <- tuple_to_list(Tuple)];
find_in_form(List) when is_list(List) ->
    [find_in_form(E) || E <- List];
find_in_form(_) -> [].

get_module_name(Forms) ->
    case [M || {attribute, _, module, M} <- Forms] of
        [M|_] -> M;
        []    -> unknown
    end.
```

---

## 5. Practical Examples

```erlang
%% Common useful macros in real Erlang codebases

%% Timing macro
-define(TIMED(Label, Expr),
    begin
        __T1 = erlang:monotonic_time(microsecond),
        __Result = Expr,
        __T2 = erlang:monotonic_time(microsecond),
        logger:info("~s took ~pµs", [Label, __T2 - __T1]),
        __Result
    end).

%% Retry macro
-define(RETRY(N, Expr),
    begin
        __RetryN = N,
        __RetryFn = fun __RetryLoop(0) -> error(max_retries_exceeded);
                        __RetryLoop(Count) ->
                            case (fun() -> Expr end)() of
                                {error, _} -> timer:sleep(100), __RetryLoop(Count-1);
                                Result     -> Result
                            end
                    end,
        __RetryFn(__RetryN)
    end).

%% Map get with default
-define(MAP_GET(Map, Key, Default),
    maps:get(Key, Map, Default)).

%% Record to map
-define(RECORD_TO_MAP(Record, RecordName, Fields),
    maps:from_list(lists:zip(Fields,
                             tl(tuple_to_list(Record))))).
```

---

## 6. แบบฝึกหัด

1. สร้าง parse_transform ที่ inject `telemetry:execute` ก่อนและหลังทุก public function
2. เพิ่ม compile-time check: warn เมื่อ atom_to_binary ถูกเรียกโดยไม่ใช้ `utf8`
3. สร้าง `-route` attribute system สำหรับ HTTP routing
4. เขียน macro `?PIPELINE(Value, Fns)` ที่ pipe value ผ่าน list of functions

---

## สรุป Part 59

✅ Compile-time macros ด้วย -define  
✅ Parse transforms สำหรับ code generation  
✅ Attribute-based routing generation  
✅ Custom compile-time checks  
✅ Practical macros (timing, retry, map access)  

---

*Part 59/100 | [← ก่อนหน้า](../part58/README.md) | [ถัดไป →](../part60/README.md)*
