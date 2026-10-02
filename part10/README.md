# Part 10: Error Handling พื้นฐาน

> **"A crash is just a message waiting to be handled"**  
> การ crash คือแค่ message ที่รอการจัดการ

---

## สารบัญ

1. [Philosophy: Let it Crash](#1-philosophy-let-it-crash)
2. [Error Types ใน Erlang](#2-error-types-ใน-erlang)
3. [try/catch ขั้นสูง](#3-trycatch-ขั้นสูง)
4. [Error Classification](#4-error-classification)
5. [Defensive vs Offensive Programming](#5-defensive-vs-offensive-programming)
6. [Result Tuples Pattern](#6-result-tuples-pattern)
7. [Error Propagation](#7-error-propagation)
8. [Safe Wrappers](#8-safe-wrappers)
9. [Error Logging](#9-error-logging)
10. [Best Practices](#10-best-practices)
11. [แบบฝึกหัด](#11-แบบฝึกหัด)

---

## 1. Philosophy: Let it Crash

### หลักการ "Let it Crash"

```
ใน Erlang: ถ้า process ล้มเหลว ให้มัน crash
แล้วให้ Supervisor restart มัน (จะเรียนใน Part 33)

ไม่ต้องพยายาม recover ทุกอย่าง
แค่ตรวจสอบสิ่งที่คุณสามารถ handle ได้จริงๆ
ส่วนที่เหลือ — ปล่อยให้ crash แล้วรับมือด้วย Supervisor
```

```erlang
%% BAD: พยายาม handle ทุก error (defensive)
get_user(Id) ->
    case db:find_user(Id) of
        {ok, User} -> User;
        {error, not_found} -> undefined;
        {error, timeout} -> undefined;
        {error, connection_failed} -> undefined;
        {error, _} -> undefined
    end.

%% GOOD: handle แค่ที่คาดไว้, crash ส่วนที่เหลือ
get_user(Id) ->
    case db:find_user(Id) of
        {ok, User}         -> {ok, User};
        {error, not_found} -> {error, not_found}
        %% อื่นๆ crash → supervisor restart process
    end.
```

### เปรียบเทียบกับภาษาอื่น

```erlang
%% Java/Python: try-catch ทุกที่ เพราะกลัว crash
%% → โค้ดยุ่ง, error ถูก swallow โดยไม่รู้ตัว

%% Erlang: crash is acceptable
%% → process ใหม่เริ่มต้นในสถานะที่สะอาด
%% → error ไม่สูญหาย (supervisor log มัน)
%% → ระบบโดยรวมยังทำงานต่อได้

%% คำพูด Joe Armstrong:
%% "If you have a dog and it bites a stranger,
%%  you teach the dog not to bite.
%%  In Erlang, we kill the dog and get a new puppy."
```

---

## 2. Error Types ใน Erlang

### 3 ประเภทหลัก

```erlang
%% 1. error — programming error, runtime error
erlang:error(badarg).
erlang:error({custom_error, "something went wrong"}).
1 + "string".           %% throws badarg error
element(0, {a,b,c}).    %% throws badarg error

%% 2. throw — non-local return (control flow)
throw(not_found).
throw({error, "user not found"}).

%% 3. exit — process termination
exit(normal).           %% normal termination
exit(shutdown).         %% supervised shutdown
exit({shutdown, Reason}). %% supervised with reason
exit(kill).             %% cannot be caught!
```

### การ Catch แต่ละประเภท

```erlang
try
    some_operation()
catch
    error:badarg ->
        io:format("Bad argument~n");
    error:{custom_error, Msg} ->
        io:format("Custom: ~s~n", [Msg]);
    throw:not_found ->
        io:format("Not found~n");
    throw:{error, Msg} ->
        io:format("Thrown error: ~s~n", [Msg]);
    exit:Reason ->
        io:format("Process exit: ~p~n", [Reason]);
    _Class:_Reason ->
        io:format("Unknown error~n")
end.

%% Stack trace
try
    some_operation()
catch
    Class:Reason:Stacktrace ->
        io:format("~p:~p~n~p~n", [Class, Reason, Stacktrace])
end.
```

### exit/1 vs exit/2

```erlang
%% exit/1 — ออกจาก process ตัวเอง
exit(normal).     %% normal exit
exit(Reason).     %% abnormal exit

%% exit/2 — ส่ง exit signal ไปยัง process อื่น
exit(Pid, kill).    %% cannot be caught
exit(Pid, normal).  %% ไม่ทำอะไรกับ process ปกติ
exit(Pid, Reason).  %% ส่ง exit signal
```

---

## 3. try/catch ขั้นสูง

### Syntax ทั้งหมด

```erlang
%% Full syntax
Result = try
    Expr1,
    Expr2,
    ReturnValue
catch
    error:Error:Stack ->
        handle_error(Error, Stack);
    throw:Thrown ->
        handle_thrown(Thrown);
    exit:Reason ->
        handle_exit(Reason)
after
    cleanup()
end.

%% try ส่งคืน value
Val = try
    1 + 1
catch
    _:_ -> 0
end.
%% Val = 2

%% catch expression (เก่า, หลีกเลี่ยง)
Result = catch some_function(),
%% ถ้า error: {'EXIT', {Reason, Stacktrace}}
%% ถ้า throw: Value
%% ถ้า success: Value
```

### Pattern ที่ใช้บ่อย

```erlang
%% Safe call
safe_call(Fun) ->
    try Fun()
    catch
        Class:Reason:Stack ->
            {error, {Class, Reason, Stack}}
    end.

%% With default value
try_with_default(Fun, Default) ->
    try Fun()
    catch _:_ -> Default
    end.

%% Rethrow
handle_with_logging(Fun) ->
    try Fun()
    catch
        Class:Reason:Stack ->
            log_error(Class, Reason, Stack),
            erlang:raise(Class, Reason, Stack)  %% rethrow
    end.

%% After for cleanup (always runs)
with_resource(AcquireFun, UseFun) ->
    Resource = AcquireFun(),
    try
        UseFun(Resource)
    after
        release(Resource)
    end.
```

---

## 4. Error Classification

### ประเภทของ errors ใน application

```erlang
%% 1. Expected errors — คาดการณ์ได้ ใช้ result tuple
%%    เช่น: user not found, invalid input, file not found
case find_user(Id) of
    {ok, User}         -> process(User);
    {error, not_found} -> {error, "User not found"}
end.

%% 2. Unexpected errors — bug, crash ได้เลย
%%    เช่น: null pointer, division by zero, bad pattern
do_something(List) when is_list(List) ->
    %% ถ้าไม่ใช่ list จะ crash — แค่ก็ดี
    lists:sum(List).

%% 3. System errors — infrastructure problems
%%    เช่น: network down, DB offline
%%    → ใช้ circuit breaker, retry, supervisor

%% Error hierarchy
-type reason() ::
    not_found        |
    unauthorized     |
    validation_error |
    {system_error, term()}.

-type result(T) :: {ok, T} | {error, reason()}.
```

### Error Module Pattern

```erlang
%% errors.erl — centralized error definitions
-module(errors).
-export([new/1, new/2, message/1, code/1, is_retryable/1]).

-type error_code() ::
    not_found | unauthorized | validation_failed |
    rate_limited | internal_error | timeout.

-record(app_error, {
    code    :: error_code(),
    message :: binary(),
    meta    :: map()
}).

new(Code) -> new(Code, default_message(Code)).
new(Code, Msg) ->
    #app_error{code = Code, message = to_binary(Msg), meta = #{}}.

message(#app_error{message = M}) -> M.
code(#app_error{code = C}) -> C.

is_retryable(#app_error{code = rate_limited}) -> true;
is_retryable(#app_error{code = timeout})      -> true;
is_retryable(_)                               -> false.

default_message(not_found)        -> <<"Resource not found">>;
default_message(unauthorized)     -> <<"Unauthorized">>;
default_message(validation_failed)-> <<"Validation failed">>;
default_message(rate_limited)     -> <<"Rate limit exceeded">>;
default_message(internal_error)   -> <<"Internal server error">>;
default_message(timeout)          -> <<"Request timed out">>.

to_binary(B) when is_binary(B) -> B;
to_binary(L) when is_list(L)   -> list_to_binary(L);
to_binary(A) when is_atom(A)   -> atom_to_binary(A, utf8).
```

---

## 5. Defensive vs Offensive Programming

### Defensive Programming (Java-style)

```erlang
%% Erlang ไม่แนะนำ แต่รู้ไว้
get_config(Key, Config) when is_map(Config) ->
    case maps:find(Key, Config) of
        {ok, Value} -> Value;
        error       -> undefined
    end;
get_config(_Key, _Config) ->
    undefined.  %% ซ่อน type error!
```

### Offensive Programming (Erlang-style)

```erlang
%% ปล่อยให้ crash ถ้า argument ผิดประเภท
get_config(Key, Config) ->
    maps:get(Key, Config).
%% ถ้า Config ไม่ใช่ map → crash → supervisor รู้
%% ถ้า Key ไม่มี → crash → แสดง bug ชัดเจน

%% หรือ pattern match เฉพาะ type ที่ถูกต้อง
get_config(Key, Config) when is_map(Config) ->
    maps:get(Key, Config).
```

### Guard Clauses

```erlang
%% ใช้ guard เพื่อ document preconditions
process_order(Order) when is_map(Order),
                          is_binary(maps:get(id, Order, <<>>)),
                          is_integer(maps:get(amount, Order, 0)),
                          maps:get(amount, Order, 0) > 0 ->
    do_process(Order).

%% Input validation เฉพาะที่ boundary
validate_request(#{id := Id, amount := Amount})
    when is_binary(Id), byte_size(Id) > 0,
         is_integer(Amount), Amount > 0 ->
    {ok, #{id => Id, amount => Amount}};
validate_request(_) ->
    {error, invalid_request}.
```

---

## 6. Result Tuples Pattern

### Standard Patterns

```erlang
%% {ok, Value}
%% {error, Reason}
%% {ok, Value1, Value2} (บางครั้ง)

%% Chaining result tuples
fetch_and_process(Id) ->
    case fetch_user(Id) of
        {ok, User} ->
            case validate_user(User) of
                {ok, ValidUser} ->
                    process_user(ValidUser);
                {error, Reason} ->
                    {error, {validation, Reason}}
            end;
        {error, Reason} ->
            {error, {fetch, Reason}}
    end.

%% Using andthen (monadic style)
andthen({ok, Value}, Fun) -> Fun(Value);
andthen({error, _} = Err, _Fun) -> Err.

%% Cleaner version
fetch_and_process2(Id) ->
    Result = {ok, Id},
    andthen(Result, fun fetch_user/1),
    andthen(Result, fun validate_user/1),
    andthen(Result, fun process_user/1).

%% อ่านไม่ออก — ต้องทำ properly
fetch_and_process3(Id) ->
    R1 = fetch_user(Id),
    R2 = andthen(R1, fun validate_user/1),
    andthen(R2, fun process_user/1).
```

### Pipeline Pattern

```erlang
-module(pipeline).
-export([run/2]).

%% run pipeline of steps
run(Input, Steps) ->
    lists:foldl(fun(Step, {ok, Value}) ->
                        Step(Value);
                   (_Step, {error, _} = Err) ->
                        Err
                end,
                {ok, Input},
                Steps).

%% Usage
process_order(Order) ->
    pipeline:run(Order, [
        fun validate_order/1,
        fun enrich_order/1,
        fun charge_payment/1,
        fun send_confirmation/1
    ]).

validate_order(Order) ->
    case maps:get(amount, Order, 0) > 0 of
        true  -> {ok, Order};
        false -> {error, invalid_amount}
    end.

enrich_order(Order) ->
    {ok, Order#{timestamp => erlang:timestamp()}}.

charge_payment(Order) ->
    %% simulate payment
    {ok, Order#{payment_status => charged}}.

send_confirmation(Order) ->
    io:format("Order ~p confirmed~n", [maps:get(id, Order, undefined)]),
    {ok, Order}.
```

---

## 7. Error Propagation

### ส่งต่อ Errors

```erlang
%% Unwrap หรือ propagate
get_user_name(Id) ->
    case get_user(Id) of
        {ok, #{name := Name}} -> {ok, Name};
        {error, _} = Err      -> Err  %% propagate
    end.

%% Context enrichment
get_user_name2(Id) ->
    case get_user(Id) of
        {ok, #{name := Name}} ->
            {ok, Name};
        {error, Reason} ->
            {error, {get_user_name, Reason}}  %% add context
    end.

%% Crash vs Return error
%%   Return error: เมื่อ caller สามารถ handle ได้
%%   Crash:        เมื่อนี่เป็น bug หรือ unexpected state

%% Pattern: ให้ function signature บอก intent
%%   {ok, _} | {error, _} → caller ต้อง handle
%%   crash                → caller ไม่ควรต้องรู้
```

### Error Context Stack

```erlang
%% สร้าง error stack เพื่อ debug
-type error_context() :: {module(), atom(), term()}.
-type error_stack() :: [error_context()].

wrap_error(Module, Function, Error) ->
    case Error of
        {error, {stack, Stack, Reason}} ->
            {error, {stack, [{Module, Function, []} | Stack], Reason}};
        {error, Reason} ->
            {error, {stack, [{Module, Function, []}], Reason}}
    end.

format_error_stack({error, {stack, Stack, Reason}}) ->
    StackStr = string:join(
        [io_lib:format("  ~p:~p", [M, F]) || {M, F, _} <- Stack],
        "\n"
    ),
    io:format("Error: ~p~nStack:~n~s~n", [Reason, StackStr]);
format_error_stack({error, Reason}) ->
    io:format("Error: ~p~n", [Reason]).
```

---

## 8. Safe Wrappers

### Higher-Order Safe Functions

```erlang
%% ทำให้ function ปลอดภัยจาก crash
safe(Fun) ->
    fun(Args) ->
        try apply(Fun, [Args])
        catch Class:Reason ->
            {error, {Class, Reason}}
        end
    end.

%% Safe map
safe_map(Fun, List) ->
    [case catch Fun(X) of
         {'EXIT', Reason} -> {error, Reason};
         Result           -> {ok, Result}
     end || X <- List].

%% Safe apply
safe_apply(Module, Function, Args) ->
    try apply(Module, Function, Args) of
        Result -> {ok, Result}
    catch
        error:undef ->
            {error, {function_not_found, {Module, Function, length(Args)}}};
        error:badarg ->
            {error, {bad_arguments, Args}};
        Class:Reason ->
            {error, {Class, Reason}}
    end.

%% With timeout
safe_call_timeout(Fun, Timeout) ->
    Ref = make_ref(),
    Pid = spawn(fun() ->
        Result = try {ok, Fun()}
                 catch C:R -> {error, {C, R}}
                 end,
        self() ! {Ref, Result}
    end),
    receive
        {Ref, Result} -> Result
    after Timeout ->
        exit(Pid, kill),
        {error, timeout}
    end.
```

---

## 9. Error Logging

### logger Module (OTP 21+)

```erlang
%% Setup logger (ใน application.erl)
setup_logger() ->
    logger:set_primary_config(level, debug),
    logger:add_handler(file_handler, logger_disk_log_h, #{
        config => #{
            file => "/var/log/myapp.log",
            max_no_bytes => 10 * 1024 * 1024,  %% 10MB
            max_no_files => 5
        },
        formatter => {logger_formatter, #{
            template => [time, " [", level, "] ", msg, "\n"]
        }}
    }).

%% Usage
logger:debug("Debug message ~p", [Value]).
logger:info("User ~p logged in", [UserId]).
logger:warning("High memory usage: ~p%", [Percent]).
logger:error("Failed to connect: ~p", [Reason]).
logger:critical("Database down: ~p", [Reason]).

%% Structured logging
logger:info(#{
    event => user_login,
    user_id => UserId,
    ip => IpAddress,
    timestamp => erlang:timestamp()
}).

%% With metadata
logger:info("Processing order", #{
    order_id => OrderId,
    amount   => Amount
}).
```

### Custom Error Logger

```erlang
-module(app_logger).
-export([log_error/3, log_warning/2, log_info/2]).

log_error(Module, Function, Error) ->
    logger:error(#{
        module   => Module,
        function => Function,
        error    => format_error(Error),
        timestamp => calendar:local_time()
    }).

log_warning(Tag, Message) ->
    logger:warning(#{
        tag     => Tag,
        message => Message,
        pid     => self()
    }).

log_info(Event, Data) ->
    logger:info(#{event => Event, data => Data}).

format_error({Class, Reason, Stack}) ->
    #{
        class  => Class,
        reason => Reason,
        stack  => format_stack(Stack)
    };
format_error(Reason) ->
    #{reason => Reason}.

format_stack(Stack) ->
    [#{module => M, function => F, arity => A, line => Line}
     || {M, F, A, Info} <- Stack,
        Line <- [proplists:get_value(line, Info, unknown)]].
```

---

## 10. Best Practices

```erlang
%% 1. ใช้ result tuples สำหรับ expected failures
%% 2. ปล่อย crash สำหรับ bugs
%% 3. อย่า swallow errors
%% 4. Log errors ด้วย context ที่เพียงพอ
%% 5. ใช้ guard สำหรับ preconditions
%% 6. Validate ที่ system boundary เท่านั้น

%% BAD: swallowing error
get_value(Key, Map) ->
    try maps:get(Key, Map)
    catch _:_ -> undefined  %% error ถูกซ่อน!
    end.

%% GOOD: explicit default
get_value(Key, Map) ->
    maps:get(Key, Map, undefined).

%% BAD: over-defensive
process(X) ->
    if is_integer(X) ->
        X * 2;
       true ->
        throw(not_integer)
    end.

%% GOOD: guard clause
process(X) when is_integer(X) ->
    X * 2.
%% ถ้าไม่ใช่ integer → function_clause crash → supervisor handle

%% Pattern: Error Boundary
handle_request(Request) ->
    try process_request(Request) of
        {ok, Response} -> send_response(200, Response);
        {error, not_found} -> send_response(404, "Not Found");
        {error, unauthorized} -> send_response(401, "Unauthorized")
    catch
        error:Reason:Stack ->
            logger:error("Unhandled error: ~p~n~p", [Reason, Stack]),
            send_response(500, "Internal Server Error")
    end.
```

---

## 11. แบบฝึกหัด

### Exercise 1: Error-safe Calculator

```erlang
-module(safe_calc).
-export([eval/1]).

%% รองรับ expression: {add, A, B}, {sub, A, B},
%%                    {mul, A, B}, {div, A, B},
%%                    {sqrt, X}, {log, X}
%% ต้อง return {ok, Result} หรือ {error, Reason}

eval({add, A, B}) when is_number(A), is_number(B) ->
    {ok, A + B};
eval({sub, A, B}) when is_number(A), is_number(B) ->
    {ok, A - B};
eval({mul, A, B}) when is_number(A), is_number(B) ->
    {ok, A * B};
eval({div, _, 0}) ->
    {error, division_by_zero};
eval({div, A, B}) when is_number(A), is_number(B) ->
    {ok, A / B};
eval({sqrt, X}) when is_number(X), X >= 0 ->
    {ok, math:sqrt(X)};
eval({sqrt, X}) when is_number(X) ->
    {error, {domain_error, {sqrt, X}}};
eval({log, X}) when is_number(X), X > 0 ->
    {ok, math:log(X)};
eval({log, X}) when is_number(X) ->
    {error, {domain_error, {log, X}}};
eval(Expr) ->
    {error, {invalid_expression, Expr}}.
```

### Exercise 2: Retry with Backoff

```erlang
-module(retry).
-export([run/3]).

%% run(Fun, MaxRetries, BaseDelayMs) -> {ok, Result} | {error, Reason}
run(Fun, MaxRetries, BaseDelay) ->
    run(Fun, MaxRetries, BaseDelay, 1).

run(Fun, MaxRetries, BaseDelay, Attempt) when Attempt =< MaxRetries ->
    case try_once(Fun) of
        {ok, _} = Result ->
            Result;
        {error, Reason} when Attempt < MaxRetries ->
            Delay = BaseDelay * (1 bsl (Attempt - 1)),  %% exponential
            logger:warning("Attempt ~p failed: ~p, retrying in ~pms",
                           [Attempt, Reason, Delay]),
            timer:sleep(Delay),
            run(Fun, MaxRetries, BaseDelay, Attempt + 1);
        {error, _} = Error ->
            Error
    end;
run(_Fun, _MaxRetries, _BaseDelay, _Attempt) ->
    {error, max_retries_exceeded}.

try_once(Fun) ->
    try {ok, Fun()}
    catch Class:Reason ->
        {error, {Class, Reason}}
    end.
```

---

## สรุป Part 10

✅ "Let it Crash" philosophy — crash แล้วให้ supervisor handle  
✅ 3 error types: error, throw, exit  
✅ try/catch/after syntax ครบ  
✅ Error classification: expected vs unexpected  
✅ Defensive vs Offensive programming  
✅ Result tuple pattern  
✅ Error propagation strategies  
✅ Safe wrappers  
✅ Error logging ด้วย logger module  
✅ Best practices

---

## ต่อไป: [Part 11 — Processes และ Concurrency พื้นฐาน](../part11/README.md)

---

*Part 10/100 | [← ก่อนหน้า](../part09/README.md) | [ถัดไป →](../part11/README.md)*
