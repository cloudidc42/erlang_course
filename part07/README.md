# Part 07: Control Flow

> **"Control flow in Erlang is elegant — pattern matching replaces most conditionals"**  
> การไหลของการควบคุมใน Erlang สง่างาม — pattern matching แทนที่ conditional ส่วนใหญ่

---

## สารบัญ

1. [if Expression](#1-if-expression)
2. [case Expression ขั้นสูง](#2-case-expression-ขั้นสูง)
3. [Guards ขั้นสูง](#3-guards-ขั้นสูง)
4. [receive Expression](#4-receive-expression)
5. [try/catch/after](#5-trycatchafter)
6. [Exceptions: throw, error, exit](#6-exceptions-throw-error-exit)
7. [Exception Handling Patterns](#7-exception-handling-patterns)
8. [begin/end Block](#8-beginend-block)
9. [Control Flow Best Practices](#9-control-flow-best-practices)
10. [แบบฝึกหัด](#10-แบบฝึกหัด)

---

## 1. if Expression

`if` ใน Erlang แตกต่างจากภาษาอื่น — ต้องมี guard ที่เป็น true เสมอ!

### Syntax

```erlang
if
    Guard1 -> Expr1;
    Guard2 -> Expr2;
    ...
    true   -> DefaultExpr  %% catch-all (สำคัญ!)
end
```

### ตัวอย่างพื้นฐาน

```erlang
%% if พื้นฐาน
classify(N) ->
    if
        N > 0  -> positive;
        N < 0  -> negative;
        N =:= 0 -> zero
    end.

1> classify(5).
positive
2> classify(-3).
negative
3> classify(0).
zero

%% if ต้องมี guard ที่ match เสมอ!
bad_if(N) ->
    if N > 0 -> positive end.  %% ถ้า N =< 0 จะ crash!

%% ใช้ true เป็น catch-all
safe_if(N) ->
    if
        N > 0 -> positive;
        true  -> "not positive"  %% default
    end.
```

### if vs case

```erlang
%% if เหมาะสำหรับ guards เท่านั้น
%% case เหมาะสำหรับ pattern matching

%% ใช้ if:
%% - เมื่อต้องการ compare numbers/booleans
%% - Guards ง่ายๆ

%% ใช้ case:
%% - เมื่อต้องการ pattern match
%% - Complex conditions

%% if (ดี)
max(A, B) ->
    if A >= B -> A;
       true   -> B
    end.

%% case (ดีกว่าสำหรับ pattern match)
describe({ok, V}) ->
    case V of
        N when N > 0 -> "positive";
        0            -> "zero";
        _            -> "negative"
    end.
```

---

## 2. case Expression ขั้นสูง

### Pattern + Guard

```erlang
%% รวม pattern matching กับ guards
classify_input(Input) ->
    case Input of
        N when is_integer(N), N > 0 ->
            {positive_integer, N};
        N when is_integer(N), N < 0 ->
            {negative_integer, N};
        0 ->
            zero;
        F when is_float(F) ->
            {float, F};
        B when is_binary(B), byte_size(B) > 0 ->
            {non_empty_binary, B};
        <<>> ->
            empty_binary;
        L when is_list(L) ->
            {list, length(L)};
        _ ->
            unknown
    end.
```

### Nested case

```erlang
%% ซ้อน case (หลีกเลี่ยงถ้าเป็นไปได้)
process_request(#{method := Method, path := Path, body := Body}) ->
    case authenticate(Method, Path) of
        {ok, User} ->
            case authorize(User, Method, Path) of
                allowed ->
                    case validate_body(Body) of
                        {ok, ValidBody} ->
                            handle(User, Method, Path, ValidBody);
                        {error, Reason} ->
                            {400, #{error => Reason}}
                    end;
                denied ->
                    {403, #{error => <<"Forbidden">>}}
            end;
        {error, _} ->
            {401, #{error => <<"Unauthorized">>}}
    end.

%% Better: ใช้ helper functions เพื่อลด nesting
process_request2(Request) ->
    with_auth(Request, fun(User, Req) ->
        with_authz(User, Req, fun(ValidReq) ->
            handle_validated(User, ValidReq)
        end)
    end).
```

### Case กับ As-Pattern

```erlang
%% ใช้ = ใน case pattern เพื่อ bind ทั้ง value
handle_response(Response) ->
    case Response of
        {ok, Body} = OkResponse when is_binary(Body) ->
            log_success(OkResponse),
            process_body(Body);
        {error, _} = ErrorResponse ->
            log_error(ErrorResponse),
            {error, failed}
    end.
```

---

## 3. Guards ขั้นสูง

### Custom Guard Macros

```erlang
%% ใช้ macro เพื่อ guard ซับซ้อน
-define(IS_VALID_PORT(P), 
    (is_integer(P) andalso P >= 0 andalso P =< 65535)).

-define(IS_VALID_IP(IP),
    (is_tuple(IP) andalso
     tuple_size(IP) =:= 4 andalso
     element(1, IP) >= 0 andalso element(1, IP) =< 255 andalso
     element(2, IP) >= 0 andalso element(2, IP) =< 255 andalso
     element(3, IP) >= 0 andalso element(3, IP) =< 255 andalso
     element(4, IP) >= 0 andalso element(4, IP) =< 255)).

connect(IP, Port) when ?IS_VALID_IP(IP), ?IS_VALID_PORT(Port) ->
    do_connect(IP, Port);
connect(_, _) ->
    {error, invalid_address}.
```

### Guards ใน Function Specs

```erlang
%% Guards ทำให้ Dialyzer ทำงานดีขึ้น

-spec safe_divide(number(), number()) -> float().
safe_divide(A, B) when is_number(A), is_number(B), B =/= 0 ->
    A / B.

-spec process_list(list()) -> list().
process_list(List) when is_list(List) ->
    lists:map(fun process/1, List).

%% Guard กับ is_record/2
-spec process_user(#user{}) -> ok.
process_user(User) when is_record(User, user) ->
    %% safe to access fields
    ok.
```

### Guard Sequences

```erlang
%% Multiple guards ด้วย ; (OR)
is_valid_name(Name) when is_binary(Name); is_list(Name) ->
    byte_size(iolist_to_binary([Name])) > 0.

%% Multiple guards ด้วย , (AND)
is_adult_user(#{age := Age, active := Active})
    when is_integer(Age), Age >= 18, Active =:= true ->
    true;
is_adult_user(_) ->
    false.

%% Complex guard
validate_range(Value, Min, Max) when
    is_number(Value),
    is_number(Min),
    is_number(Max),
    Min =< Max,
    Value >= Min,
    Value =< Max ->
    ok;
validate_range(_, _, _) ->
    error.
```

---

## 4. receive Expression

`receive` ใช้สำหรับรับ messages จาก process

### Syntax

```erlang
receive
    Pattern1 [when Guard1] -> Body1;
    Pattern2 [when Guard2] -> Body2;
    ...
after
    Timeout -> TimeoutBody  %% optional timeout
end
```

### ตัวอย่างพื้นฐาน

```erlang
%% Simple receive
wait_for_message() ->
    receive
        hello ->
            io:format("Got hello!~n");
        {msg, Text} ->
            io:format("Got: ~s~n", [Text]);
        _ ->
            io:format("Unknown message~n")
    end.

%% ใช้งาน
Pid = spawn(fun wait_for_message/0).
Pid ! hello.

%% Receive กับ timeout
wait_with_timeout(Timeout) ->
    receive
        {result, V} -> {ok, V}
    after Timeout ->
        {error, timeout}
    end.

%% receive loop
server_loop(State) ->
    receive
        {request, From, Data} ->
            {NewState, Response} = handle(State, Data),
            From ! {response, Response},
            server_loop(NewState);
        stop ->
            io:format("Stopping server~n");
        Unexpected ->
            io:format("Unexpected: ~p~n", [Unexpected]),
            server_loop(State)
    end.
```

### Selective Receive

```erlang
%% Erlang mailbox ทำ selective receive ได้
%% Messages ที่ไม่ match จะอยู่ใน mailbox ต่อไป

selective_example() ->
    self() ! message1,
    self() ! message2,
    self() ! message3,
    
    receive message2 -> io:format("Got message2~n") end,
    receive message1 -> io:format("Got message1~n") end,
    receive message3 -> io:format("Got message3~n") end.

%% หลีกเลี่ยง selective receive ในรูปแบบนี้ (inefficient):
wait_for_specific(Ref) ->
    receive
        {Ref, Result} ->   %% รอ message เฉพาะ Ref นี้
            Result
        %% messages อื่นๆ อยู่ใน mailbox ต่อไป (อาจ leak!)
    after 5000 ->
        timeout
    end.

%% ดีกว่า: จัดการทุก message
wait_with_drain(Ref) ->
    receive
        {Ref, Result} -> Result;
        _Other        -> wait_with_drain(Ref)  %% discard other messages
    after 5000 ->
        timeout
    end.
```

### Receive และ Pattern Matching

```erlang
%% Complex pattern ใน receive
handle_messages() ->
    receive
        %% Tagged messages
        {get, Key, From} ->
            From ! {key, Key, get_value(Key)},
            handle_messages();
        
        %% Binary messages
        <<Type:8, Data/binary>> when Type =:= 1 ->
            process_type1(Data),
            handle_messages();
        
        %% List messages
        [Action | Args] when is_atom(Action) ->
            apply(?MODULE, Action, Args),
            handle_messages();
        
        %% Map messages (Erlang 17+)
        #{type := subscribe, topic := Topic, pid := Pid} ->
            add_subscriber(Topic, Pid),
            handle_messages();
        
        stop ->
            stopped
    end.
```

---

## 5. try/catch/after

```erlang
%% Syntax:
try
    Expressions
catch
    ExceptionClass:Pattern [when Guard] -> Handler;
    ...
after
    Cleanup  %% เสมอ execute ไม่ว่าจะ exception หรือไม่
end
```

### ตัวอย่างพื้นฐาน

```erlang
%% try/catch พื้นฐาน
safe_divide(X, Y) ->
    try X / Y
    catch
        error:badarith -> {error, division_by_zero}
    end.

1> safe_divide(10, 2).
5.0
2> safe_divide(10, 0).
{error, division_by_zero}

%% Catch ทุก exception
catch_all(F) ->
    try F()
    catch
        error:Error   -> {error, Error};
        exit:Reason   -> {exit, Reason};
        throw:Value   -> {throw, Value}
    end.
%% หรือ
catch_all2(F) ->
    try F()
    catch
        Class:Reason -> {exception, Class, Reason}
    end.
```

### try/catch/after

```erlang
%% after เสมอ execute
with_file(Path, Fun) ->
    {ok, File} = file:open(Path, [read]),
    try
        Fun(File)
    after
        file:close(File)  %% เสมอปิดไฟล์!
    end.

%% Database connection example
with_db_connection(Fun) ->
    Conn = db:connect(),
    try
        db:transaction(Conn, Fun)
    catch
        error:Reason ->
            db:rollback(Conn),
            {error, Reason}
    after
        db:close(Conn)  %% เสมอปิด connection
    end.
```

### try กับ Value

```erlang
%% try สามารถ return value ได้
Result = try
    do_risky_operation()
catch
    error:Reason -> {error, Reason}
end,
use_result(Result).

%% ตัวอย่าง: parse with fallback
parse_number(Input) ->
    try
        {ok, list_to_integer(Input)}
    catch
        error:badarg ->
            try
                {ok, list_to_float(Input)}
            catch
                error:badarg ->
                    {error, not_a_number}
            end
    end.
```

---

## 6. Exceptions: throw, error, exit

Erlang มี 3 ประเภทของ exceptions:

### 1. error — programming errors

```erlang
%% error สำหรับ programming errors
%% เกิดเมื่อ:
%% - Pattern match fails
%% - Function call with wrong args  
%% - Division by zero
%% - เรียก erlang:error/1 หรือ erlang:error/2

%% สร้าง error
erlang:error(badarg).
erlang:error({invalid_input, X}).
erlang:error(badarg, [Arg1, Arg2]).  % กับ args info

%% Catch error
try dangerous()
catch
    error:badarg -> handle_bad_arg();
    error:{invalid, Reason} -> handle_invalid(Reason)
end.

%% Stack trace
try dangerous()
catch
    error:Reason:Stacktrace ->
        io:format("Error: ~p~nStack: ~p~n", [Reason, Stacktrace])
end.
```

### 2. throw — intentional non-local jump

```erlang
%% throw สำหรับ non-local return
%% เหมือน "early exit" หรือ "break"

find_first(Pred, List) ->
    try
        lists:foreach(fun(E) ->
            case Pred(E) of
                true  -> throw({found, E});
                false -> ok
            end
        end, List),
        not_found
    catch
        throw:{found, E} -> {ok, E}
    end.

1> find_first(fun(X) -> X > 3 end, [1,2,3,4,5]).
{ok, 4}

%% throw ใน nested computation
deep_search(Tree, Target) ->
    try
        search_node(Tree, Target),
        not_found
    catch
        throw:{found, Value} -> {ok, Value}
    end.
```

### 3. exit — process termination

```erlang
%% exit สำหรับ terminate process

%% terminate ตัวเอง
exit(normal).        % normal exit
exit(killed).        % killed
exit({error, Msg}).  % error exit

%% terminate process อื่น
exit(Pid, kill).     % force kill
exit(Pid, shutdown). % graceful shutdown
exit(Pid, Reason).   % exit with reason

%% Catch exit (ยกเว้น kill ที่ไม่สามารถ catch ได้)
try
    spawn_and_wait()
catch
    exit:normal   -> ok;
    exit:shutdown -> ok;
    exit:Reason   -> {error, {unexpected_exit, Reason}}
end.
```

### เปรียบเทียบ Exception Types

```
Exception Type  ใช้เมื่อ                        Can Catch?
────────────────────────────────────────────────────────
error           Programming error (bug)          Yes
throw           Intentional non-local return     Yes
exit(normal)    Normal process termination       Yes (catch)
exit(kill)      Force kill                       No! (ไม่สามารถ catch)
exit(Reason)    Process termination with reason  Yes (catch)
```

---

## 7. Exception Handling Patterns

### Safe Wrappers

```erlang
%% safe_apply: catch ทุก exception
safe_apply(F, Args) ->
    try erlang:apply(F, Args)
    catch
        Class:Reason ->
            {error, {Class, Reason}}
    end.

%% safe_call: เรียก function ที่อาจ crash
safe_call(Fun) ->
    try {ok, Fun()}
    catch
        error:Reason -> {error, Reason};
        throw:Value  -> {throw, Value}
    end.

%% with_error_handling
with_error_handling(Fun, ErrorHandler) ->
    try Fun()
    catch
        error:Reason ->
            ErrorHandler({error, Reason})
    end.
```

### Error Classification

```erlang
%% จัดหมวดหมู่ errors
classify_error(Reason) ->
    case Reason of
        badarg            -> user_error;
        badarith          -> user_error;
        {badmatch, _}     -> programming_error;
        function_clause   -> programming_error;
        {case_clause, _}  -> programming_error;
        noproc            -> system_error;
        noconnection      -> network_error;
        _                 -> unknown_error
    end.

%% Retry สำหรับ transient errors
retry_on_transient(Fun, MaxRetries) ->
    retry_on_transient(Fun, MaxRetries, 1).

retry_on_transient(Fun, MaxRetries, Attempt) ->
    case safe_call(Fun) of
        {ok, Result} ->
            {ok, Result};
        {error, Reason} when is_transient(Reason), Attempt < MaxRetries ->
            timer:sleep(backoff(Attempt)),
            retry_on_transient(Fun, MaxRetries, Attempt + 1);
        {error, Reason} ->
            {error, Reason}
    end.

is_transient(timeout) -> true;
is_transient(noproc) -> true;
is_transient(noconnection) -> true;
is_transient(_) -> false.

backoff(N) -> min(1000 * (1 bsl (N-1)), 30000).  % exponential backoff, max 30s
```

### Error Propagation

```erlang
%% ส่ง error ไปยัง caller
%% Pattern 1: Return error tuple
read_config(Path) ->
    case file:read_file(Path) of
        {ok, Content} -> parse_config(Content);
        {error, enoent} -> {error, config_not_found};
        {error, Reason} -> {error, {io_error, Reason}}
    end.

%% Pattern 2: Throw exception
read_config_throw(Path) ->
    case file:read_file(Path) of
        {ok, Content} ->
            case parse_config(Content) of
                {ok, Config} -> Config;
                {error, R}   -> throw({config_error, R})
            end;
        {error, enoent} -> throw(config_not_found)
    end.

%% Pattern 3: Exit process
read_config_exit(Path) ->
    case file:read_file(Path) of
        {ok, Content} -> parse_config(Content);
        {error, Reason} -> exit({config_error, Reason})
    end.
```

---

## 8. begin/end Block

```erlang
%% begin/end ใช้เพื่อ group expressions

%% ใน case branch
process(Input) ->
    case Input of
        {complex, Data} ->
            begin
                Parsed = parse(Data),
                Validated = validate(Parsed),
                transform(Validated)
            end;
        Simple ->
            Simple
    end.

%% ใน if
show_info(Level) ->
    if Level >= 2 ->
        begin
            io:format("Detail 1~n"),
            io:format("Detail 2~n"),
            io:format("Detail 3~n")
        end;
    true ->
        io:format("Basic info~n")
    end.

%% begin/end ใน receive
server() ->
    receive
        {compute, From, Args} ->
            Result = begin
                Step1 = preprocess(Args),
                Step2 = compute(Step1),
                postprocess(Step2)
            end,
            From ! {result, Result},
            server()
    end.
```

---

## 9. Control Flow Best Practices

### Prefer Pattern Matching over if

```erlang
%% AVOID
bad_example(Type) ->
    if
        Type =:= circle    -> circle_area();
        Type =:= rectangle -> rect_area();
        Type =:= triangle  -> tri_area()
    end.

%% PREFER
good_example(Type) ->
    case Type of
        circle    -> circle_area();
        rectangle -> rect_area();
        triangle  -> tri_area()
    end.

%% BEST: function clauses
area(circle)    -> circle_area();
area(rectangle) -> rect_area();
area(triangle)  -> tri_area().
```

### Avoid Deep Nesting

```erlang
%% BAD: deep nesting
bad_process(Input) ->
    case validate(Input) of
        ok ->
            case fetch_data(Input) of
                {ok, Data} ->
                    case transform(Data) of
                        {ok, Result} ->
                            {ok, Result};
                        Error -> Error
                    end;
                Error -> Error
            end;
        Error -> Error
    end.

%% GOOD: early returns + helper functions
good_process(Input) ->
    case validate(Input) of
        ok    -> fetch_and_transform(Input);
        Error -> Error
    end.

fetch_and_transform(Input) ->
    case fetch_data(Input) of
        {ok, Data} -> transform(Data);
        Error      -> Error
    end.

%% BEST: error monad pattern
process_pipeline(Input) ->
    Ops = [
        fun validate/1,
        fun fetch_data/1,
        fun transform/1
    ],
    lists:foldl(fun
        (Op, {ok, Data}) -> Op(Data);
        (_, Error)       -> Error
    end, {ok, Input}, Ops).
```

### Handle All Cases

```erlang
%% ALWAYS handle the default case!

%% BAD: might crash on unexpected input
bad_match(Status) ->
    case Status of
        active   -> do_active();
        inactive -> do_inactive()
    end.  %% crashes if Status = pending!

%% GOOD: explicit catch-all
good_match(Status) ->
    case Status of
        active   -> do_active();
        inactive -> do_inactive();
        Other    -> {error, {unexpected_status, Other}}
    end.

%% Or: use function clauses กับ catch-all
handle_status(active)   -> do_active();
handle_status(inactive) -> do_inactive();
handle_status(Other)    -> error({unexpected_status, Other}).
```

### เปรียบเทียบ try/catch vs {ok,error} tuples

```erlang
%% Exceptions สำหรับ exceptional conditions (ไม่ใช่ business logic)
%% Return tuples สำหรับ expected errors

%% WRONG: ใช้ exception สำหรับ expected conditions
bad_find_user(Id) ->
    case find_in_db(Id) of
        undefined -> throw(user_not_found);
        User      -> User
    end.

%% RIGHT: ใช้ return tuple
good_find_user(Id) ->
    case find_in_db(Id) of
        undefined -> {error, not_found};
        User      -> {ok, User}
    end.

%% CORRECT ใช้ exception: unexpected system failures
risky_db_query(Query) ->
    try execute_query(Query)
    catch
        error:{db_error, Reason} ->
            log_error(Reason),
            {error, db_unavailable}
    end.
```

---

## 10. แบบฝึกหัด

### Exercise 1: Safe Math Operations

```erlang
%% สร้าง safe_math.erl ที่จัดการทุก error
-module(safe_math).
-export([divide/2, sqrt/1, log/1, power/2]).

divide(_, 0) -> {error, division_by_zero};
divide(X, Y) when is_number(X), is_number(Y) ->
    {ok, X / Y};
divide(_, _) -> {error, not_numbers}.

sqrt(X) when is_number(X), X >= 0 ->
    {ok, math:sqrt(X)};
sqrt(X) when is_number(X) ->
    {error, negative_input};
sqrt(_) ->
    {error, not_a_number}.

log(X) when is_number(X), X > 0 ->
    {ok, math:log(X)};
log(0) -> {error, log_of_zero};
log(X) when is_number(X) ->
    {error, {negative_input, X}};
log(_) ->
    {error, not_a_number}.

power(_, 0) -> {ok, 1};
power(0, _) -> {ok, 0};
power(X, N) when is_number(X), is_integer(N), N > 0 ->
    {ok, math:pow(X, N)};
power(X, N) when is_number(X), is_integer(N), N < 0 ->
    {ok, 1 / math:pow(X, -N)};
power(_, _) -> {error, invalid_arguments}.
```

### Exercise 2: State Machine ด้วย receive

```erlang
%% สร้าง traffic_light.erl ที่ simulate traffic light
%% States: red -> green -> yellow -> red (loop)
%% Messages: {change, manual} | {status, Pid} | stop

-module(traffic_light).
-export([start/0, change/1, status/1, stop/1]).

start() ->
    spawn(fun() -> loop(red, #{red => 3000, green => 2000, yellow => 1000}) end).

change(Pid) ->
    Pid ! {change, manual}.

status(Pid) ->
    Pid ! {status, self()},
    receive {status, State} -> State after 1000 -> timeout end.

stop(Pid) ->
    Pid ! stop.

loop(State, Timings) ->
    receive
        {change, manual} ->
            Next = next_state(State),
            io:format("Manual change: ~p -> ~p~n", [State, Next]),
            loop(Next, Timings);
        {status, From} ->
            From ! {status, State},
            loop(State, Timings);
        stop ->
            io:format("Traffic light stopped at: ~p~n", [State])
    after maps:get(State, Timings) ->
        Next = next_state(State),
        io:format("Auto change: ~p -> ~p~n", [State, Next]),
        loop(Next, Timings)
    end.

next_state(red)    -> green;
next_state(green)  -> yellow;
next_state(yellow) -> red.
```

### Exercise 3: Error Handling Pipeline

```erlang
%% สร้าง pipeline.erl ที่ process data ผ่าน steps
%% แต่ละ step อาจ fail และควรส่ง error ที่มี context

-module(pipeline).
-export([run/2]).

%% Run list ของ steps, stop ที่ first error
run(Data, Steps) ->
    run_steps(Data, Steps, []).

run_steps(Data, [], _Completed) ->
    {ok, Data};
run_steps(Data, [Step|Rest], Completed) ->
    StepName = element(1, Step),
    StepFun  = element(2, Step),
    try
        case StepFun(Data) of
            {ok, Result} ->
                run_steps(Result, Rest, [StepName|Completed]);
            {error, Reason} ->
                {error, #{
                    step       => StepName,
                    reason     => Reason,
                    completed  => lists:reverse(Completed),
                    input      => Data
                }}
        end
    catch
        Class:Reason:Stack ->
            {error, #{
                step       => StepName,
                class      => Class,
                reason     => Reason,
                stacktrace => Stack,
                completed  => lists:reverse(Completed)
            }}
    end.

%% ตัวอย่าง steps
validate_name(Data = #{name := Name}) when byte_size(Name) > 0 ->
    {ok, Data};
validate_name(#{name := _}) ->
    {error, empty_name};
validate_name(_) ->
    {error, missing_name}.

normalize_email(Data = #{email := Email}) ->
    {ok, Data#{email => string:lowercase(Email)}}.

enrich_data(Data) ->
    {ok, Data#{processed_at => erlang:system_time(second)}}.

%% ใช้งาน
test() ->
    Steps = [
        {validate_name, fun validate_name/1},
        {normalize_email, fun normalize_email/1},
        {enrich, fun enrich_data/1}
    ],
    Input = #{name => <<"Alice">>, email => <<"Alice@Example.COM">>},
    run(Input, Steps).
%% {ok, #{name => <<"Alice">>, email => <<"alice@example.com">>,
%%        processed_at => 1706198400}}
```

### Exercise 4: Complex Control Flow

```erlang
%% สร้าง order_processor.erl
%% ที่จัดการ order processing ด้วย complex control flow

-module(order_processor).
-export([process_order/1]).

process_order(Order = #{items := Items, customer := Customer}) ->
    try
        %% Step 1: Validate
        ok = validate_order(Order),
        
        %% Step 2: Check inventory
        ok = check_inventory(Items),
        
        %% Step 3: Calculate price
        {ok, TotalPrice} = calculate_price(Items),
        
        %% Step 4: Process payment
        {ok, PaymentId} = process_payment(Customer, TotalPrice),
        
        %% Step 5: Reserve inventory
        ok = reserve_inventory(Items),
        
        %% Step 6: Create shipment
        {ok, ShipmentId} = create_shipment(Order),
        
        {ok, #{
            order_id    => generate_id(),
            payment_id  => PaymentId,
            shipment_id => ShipmentId,
            total       => TotalPrice,
            status      => confirmed
        }}
    catch
        throw:{validation_error, Reason} ->
            {error, {invalid_order, Reason}};
        throw:{inventory_error, Items2} ->
            {error, {out_of_stock, Items2}};
        throw:{payment_error, Reason} ->
            {error, {payment_failed, Reason}};
        error:Reason:Stack ->
            logger:error("Order processing failed: ~p~n~p", [Reason, Stack]),
            {error, internal_error}
    end.

validate_order(#{items := []}) ->
    throw({validation_error, empty_order});
validate_order(#{customer := Customer}) when not is_map(Customer) ->
    throw({validation_error, invalid_customer});
validate_order(_) ->
    ok.

check_inventory(Items) ->
    OutOfStock = [I || I <- Items, not in_stock(I)],
    case OutOfStock of
        [] -> ok;
        _  -> throw({inventory_error, OutOfStock})
    end.

calculate_price(Items) ->
    Total = lists:sum([item_price(I) || I <- Items]),
    {ok, Total}.

process_payment(_Customer, _Amount) ->
    %% ในระบบจริงจะเชื่อมต่อ payment gateway
    {ok, generate_payment_id()}.

reserve_inventory(_Items) -> ok.
create_shipment(_Order) -> {ok, generate_shipment_id()}.

in_stock(_) -> true.  %% mock
item_price(_) -> 10.0.  %% mock
generate_id() -> erlang:unique_integer([positive]).
generate_payment_id() -> <<"pay_", (integer_to_binary(erlang:unique_integer([positive])))/binary>>.
generate_shipment_id() -> <<"ship_", (integer_to_binary(erlang:unique_integer([positive])))/binary>>.
```

---

## สรุป Part 07

ใน Part นี้คุณได้เรียนรู้:

✅ if expression และ catch-all pattern  
✅ case expression ขั้นสูงกับ guards  
✅ Guards — custom macros, function specs  
✅ receive — selective receive, patterns  
✅ try/catch/after — exception handling  
✅ throw, error, exit — 3 ประเภท exceptions  
✅ Exception handling patterns  
✅ begin/end blocks  
✅ Control flow best practices  

---

## ต่อไป: [Part 08 — String และ Binary](../part08/README.md)

ใน Part ถัดไปเราจะเรียนรู้:
- String operations อย่างละเอียด
- Binary manipulation
- Regular expressions
- Unicode handling
- iolist และ formatting
- Binary protocols

---

*Part 07/100 | [← ก่อนหน้า](../part06/README.md) | [ถัดไป →](../part08/README.md)*
