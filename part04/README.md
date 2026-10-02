# Part 04: Functions และ Modules

> **"Modules are the basic unit of code organization in Erlang"**  
> Module คือหน่วยพื้นฐานของการจัดระเบียบโค้ดใน Erlang

---

## สารบัญ

1. [Modules คืออะไร?](#1-modules-คืออะไร)
2. [Module Declarations](#2-module-declarations)
3. [Function Basics](#3-function-basics)
4. [Export และ Import](#4-export-และ-import)
5. [Function Clauses และ Arity](#5-function-clauses-และ-arity)
6. [Tail Recursion](#6-tail-recursion)
7. [Higher-Order Functions](#7-higher-order-functions)
8. [Closures](#8-closures)
9. [Function References](#9-function-references)
10. [Apply Functions](#10-apply-functions)
11. [Module Attributes](#11-module-attributes)
12. [Module Info](#12-module-info)
13. [Code Organization Best Practices](#13-code-organization-best-practices)
14. [แบบฝึกหัด](#14-แบบฝึกหัด)

---

## 1. Modules คืออะไร?

Module ใน Erlang คือ **namespace สำหรับ functions** ที่รวบรวม functions ที่เกี่ยวข้องกันไว้ด้วยกัน

```
Module Structure
════════════════
my_module.erl
├── Module Declaration  (-module(my_module).)
├── Attributes          (-export([...]), -include([...]), etc.)
├── Functions           (name(Args) -> Body.)
└── End of File
```

### หลักการของ Module

```erlang
%% ใน Erlang ทุกอย่างอยู่ใน module
%% เรียกใช้: ModuleName:FunctionName(Args)

lists:sort([3,1,2]).
io:format("Hello~n").
erlang:now().

%% Module name ต้องตรงกับชื่อไฟล์ (ไม่มี .erl)
%% my_module.erl -> -module(my_module).
```

---

## 2. Module Declarations

### Minimal Module

```erlang
%% File: hello.erl
-module(hello).          %% REQUIRED: ต้องอยู่บรรทัดแรก
-export([greet/0]).      %% Export list

greet() ->
    io:format("Hello, World!~n").
```

### Module Attributes ทั้งหมด

```erlang
%% File: full_module.erl

%%% ============================================================
%%% Module Declaration
%%% ============================================================
-module(full_module).

%%% ============================================================  
%%% Compiler Options
%%% ============================================================
-compile([debug_info, {parse_transform, my_transform}]).

%%% ============================================================
%%% Version
%%% ============================================================
-vsn("1.0.0").

%%% ============================================================
%%% Exports
%%% ============================================================
-export([
    public_function/0,
    public_function/1,
    another_public/2
]).

%% Export สำหรับ test เท่านั้น
-ifdef(TEST).
-export([private_helper/1]).
-endif.

%%% ============================================================
%%% Type declarations (Dialyzer)
%%% ============================================================
-type name() :: binary().
-type age()  :: non_neg_integer().
-type user() :: #{name := name(), age := age()}.

%%% ============================================================
%%% Record declarations
%%% ============================================================
-record(state, {
    name    :: name(),
    age     :: age(),
    active  = true :: boolean()
}).

%%% ============================================================
%%% Macros
%%% ============================================================
-define(MAX_SIZE, 1000).
-define(DEFAULT_TIMEOUT, 5000).
-define(LOG(Msg), io:format("[LOG] ~p~n", [Msg])).

%%% ============================================================
%%% Includes
%%% ============================================================
-include("include/my_app.hrl").
-include_lib("stdlib/include/assert.hrl").

%%% ============================================================
%%% Functions
%%% ============================================================

public_function() ->
    ?LOG("called"),
    ok.

public_function(Name) ->
    io:format("Hello, ~s~n", [Name]).

another_public(X, Y) ->
    X + Y.

private_helper(X) ->
    X * 2.
```

---

## 3. Function Basics

### Function Syntax

```erlang
%% Syntax:
%% FunctionName(Pattern1, Pattern2, ...) [when Guard] -> Body.

%% Simple function
add(X, Y) -> X + Y.

%% Multiple expressions (ใช้ , คั่น, ส่ง value ของ expression สุดท้าย)
complex(X, Y) ->
    Z = X + Y,
    W = Z * 2,
    W - 1.

%% Function with guard
safe_div(X, Y) when Y =/= 0 ->
    X / Y;
safe_div(_, 0) ->
    {error, division_by_zero}.
```

### Function Arity

```erlang
%% Arity คือจำนวน arguments
%% Function ชื่อเดียวกันแต่ Arity ต่างกัน = คนละ function!

greet/0 หมายถึง function greet ที่ไม่มี argument
greet/1 หมายถึง function greet ที่มี 1 argument
greet/2 หมายถึง function greet ที่มี 2 arguments

-export([greet/0, greet/1, greet/2]).

greet() ->
    greet("World").

greet(Name) ->
    greet("Hello", Name).

greet(Greeting, Name) ->
    io:format("~s, ~s!~n", [Greeting, Name]).

%% เรียกใช้
1> greet().          % Hello, World!
2> greet("Alice").   % Hello, Alice!
3> greet("Hi", "Bob"). % Hi, Bob!
```

### Function Body

```erlang
%% Body คือ sequence ของ expressions คั่นด้วย ,
%% ค่า return คือ value ของ expression สุดท้าย

calculate(X, Y) ->
    Sum = X + Y,          %% expression 1
    Product = X * Y,      %% expression 2
    Diff = abs(X - Y),    %% expression 3
    {Sum, Product, Diff}. %% expression 4 (return value)

%% ตัวอย่าง
1> calculate(3, 4).
{7, 12, 1}
```

---

## 4. Export และ Import

### Export

```erlang
%% Export ทำให้ function เรียกได้จากภายนอก module

%% วิธีที่ 1: รวม exports ไว้ที่เดียว
-export([
    func1/0,
    func2/1,
    func3/2
]).

%% วิธีที่ 2: กระจาย exports (ไม่นิยม)
-export([func1/0]).
-export([func2/1]).

%% Export ทั้งหมด (อย่าใช้ใน production)
-compile(export_all).

%% Private functions ไม่ต้อง export
private_helper(X) -> X * 2.
```

### Import

```erlang
%% Import ทำให้เรียก function โดยไม่ต้องใส่ module name
%% (ไม่ค่อยนิยมใช้ เพราะทำให้ code อ่านยาก)

-import(lists, [sort/1, reverse/1, map/2]).
-import(io, [format/1, format/2]).

use_imported() ->
    Sorted = sort([3,1,2]),    %% แทน lists:sort(...)
    Reversed = reverse(Sorted), %% แทน lists:reverse(...)
    format("Result: ~p~n", [Reversed]).  %% แทน io:format(...)
```

### Visibility Rules

```erlang
%% Functions ที่ export ได้ = public
%% Functions ที่ไม่ export = private (เรียกได้แค่ใน module เดิม)

-module(bank_account).
-export([deposit/2, withdraw/2, balance/1]).  %% public API

deposit(Account, Amount) ->
    NewBalance = get_balance(Account) + Amount,  %% เรียก private
    store_balance(Account, NewBalance).

withdraw(Account, Amount) ->
    case get_balance(Account) of
        Balance when Balance >= Amount ->
            NewBalance = Balance - Amount,
            store_balance(Account, NewBalance),
            {ok, NewBalance};
        Balance ->
            {error, {insufficient_funds, Balance}}
    end.

balance(Account) ->
    get_balance(Account).

%% Private functions
get_balance(Account) ->
    ets:lookup_element(accounts, Account, 2).

store_balance(Account, Balance) ->
    ets:insert(accounts, {Account, Balance}).
```

---

## 5. Function Clauses และ Arity

### Multiple Clauses

```erlang
%% Function สามารถมีหลาย clause
%% Erlang ลอง clause ตามลำดับ หยุดเมื่อ match

%% Factorial
fact(0) -> 1;
fact(N) when N > 0 -> N * fact(N-1).

%% fibonacci
fib(0) -> 0;
fib(1) -> 1;
fib(N) when N > 1 -> fib(N-1) + fib(N-2).

%% List processing
describe([])       -> "empty list";
describe([_])      -> "singleton list";
describe([_, _])   -> "two-element list";
describe([_|_])    -> "multi-element list".
```

### Clause Ordering (สำคัญ!)

```erlang
%% Clauses ถูก match จากบนลงล่าง
%% specific patterns ต้องอยู่ก่อน general patterns

%% CORRECT order (specific first)
process({error, timeout})  -> retry();
process({error, Reason})   -> log_error(Reason);  %% more general
process({ok, Value})       -> handle(Value).

%% WRONG order (general before specific)
process({error, Reason})   -> log_error(Reason);  %% catches timeout too!
process({error, timeout})  -> retry().             %% never reached!

%% Compiler warning: เตือนเรื่อง unreachable clauses
```

### Default/Catch-all Clauses

```erlang
%% Catch-all clause อยู่ท้ายสุด
handle_message({ping, From}) ->
    From ! pong;
handle_message({request, From, Data}) ->
    Response = process(Data),
    From ! {response, Response};
handle_message(Unknown) ->
    io:format("Unknown message: ~p~n", [Unknown]).
    %% อย่าลืม catch-all! มิฉะนั้น process จะ crash
```

---

## 6. Tail Recursion

### ทำไมต้อง Tail Recursion?

```erlang
%% Non-tail recursive (BAD for large inputs)
bad_sum([]) -> 0;
bad_sum([H|T]) -> 
    H + bad_sum(T).  %% Stack grows with each call!
    %% [1,2,3,4,5] -> 1 + (2 + (3 + (4 + (5 + 0))))
    %% Stack: 5 frames deep

%% Tail recursive (GOOD - constant stack space)
good_sum(List) -> good_sum(List, 0).

good_sum([], Acc) -> Acc;
good_sum([H|T], Acc) -> 
    good_sum(T, H + Acc).  %% Tail call! Stack doesn't grow
    %% [1,2,3,4,5] -> good_sum([2,3,4,5], 1)
    %%             -> good_sum([3,4,5], 3)
    %%             -> good_sum([4,5], 6)
    %%             -> good_sum([5], 10)
    %%             -> good_sum([], 15)
    %%             -> 15
```

### Tail Call Optimization (TCO)

```erlang
%% BEAM VM ทำ TCO โดยอัตโนมัติ
%% Tail call = call เป็น operation สุดท้ายใน function

%% Tail recursive reverse
reverse(List) -> reverse(List, []).
reverse([], Acc) -> Acc;                     %% base case
reverse([H|T], Acc) -> reverse(T, [H|Acc]). %% tail call

%% Non-tail: H ++ reverse(T, Acc) ไม่ใช่ tail call
%% เพราะต้องรอ reverse(T, Acc) ก่อนแล้วค่อย ++

%% Tail recursive flatten
flatten(List) -> flatten(List, []).

flatten([], Acc) ->
    lists:reverse(Acc);
flatten([[H|T]|Rest], Acc) ->
    flatten([H|T] ++ Rest, Acc);
flatten([[]|Rest], Acc) ->
    flatten(Rest, Acc);
flatten([H|T], Acc) ->
    flatten(T, [H|Acc]).
```

### ตัวอย่าง Tail Recursive Functions

```erlang
%% Map
my_map(Fun, List) -> my_map(Fun, List, []).
my_map(_, [], Acc) -> lists:reverse(Acc);
my_map(Fun, [H|T], Acc) -> my_map(Fun, T, [Fun(H)|Acc]).

%% Filter
my_filter(Pred, List) -> my_filter(Pred, List, []).
my_filter(_, [], Acc) -> lists:reverse(Acc);
my_filter(Pred, [H|T], Acc) ->
    case Pred(H) of
        true  -> my_filter(Pred, T, [H|Acc]);
        false -> my_filter(Pred, T, Acc)
    end.

%% Take N elements
take(_, 0, _) -> [];
take(List, N, Acc) -> take_inner(List, N, Acc).

take_inner(_, 0, Acc) -> lists:reverse(Acc);
take_inner([], _, Acc) -> lists:reverse(Acc);
take_inner([H|T], N, Acc) -> take_inner(T, N-1, [H|Acc]).

take(List, N) -> take(List, N, []).

%% Zip ด้วย accumulator
zip(L1, L2) -> zip(L1, L2, []).
zip([], _, Acc)         -> lists:reverse(Acc);
zip(_, [], Acc)         -> lists:reverse(Acc);
zip([H1|T1], [H2|T2], Acc) -> zip(T1, T2, [{H1,H2}|Acc]).
```

---

## 7. Higher-Order Functions

### Functions as Arguments

```erlang
%% Higher-order function รับ function เป็น argument

apply_twice(F, X) -> F(F(X)).

1> Double = fun(X) -> X * 2 end.
2> apply_twice(Double, 3).
12  % Double(Double(3)) = Double(6) = 12

%% compose: f(g(x))
compose(F, G) ->
    fun(X) -> F(G(X)) end.

3> AddOne = fun(X) -> X + 1 end.
4> Double = fun(X) -> X * 2 end.
5> DoubleThenAdd = compose(AddOne, Double).
6> DoubleThenAdd(5).
11  % AddOne(Double(5)) = AddOne(10) = 11
```

### Built-in Higher-Order Functions

```erlang
%% lists:map/2
1> lists:map(fun(X) -> X * X end, [1,2,3,4,5]).
[1,4,9,16,25]

%% lists:filter/2
2> lists:filter(fun(X) -> X > 3 end, [1,2,3,4,5]).
[4,5]

%% lists:foldl/3
3> lists:foldl(fun(X, Acc) -> X + Acc end, 0, [1,2,3,4,5]).
15

%% lists:foldr/3
4> lists:foldr(fun(X, Acc) -> [X*2|Acc] end, [], [1,2,3]).
[2,4,6]

%% lists:any/2
5> lists:any(fun(X) -> X > 3 end, [1,2,3,4,5]).
true

%% lists:all/2
6> lists:all(fun(X) -> X > 0 end, [1,2,3,4,5]).
true

%% lists:foreach/2 (ไม่ return value ที่มีประโยชน์)
7> lists:foreach(fun(X) -> io:format("~p~n", [X]) end, [1,2,3]).
1
2
3
ok

%% lists:sort/2 กับ custom comparator
8> lists:sort(fun(A, B) -> A > B end, [3,1,4,1,5,9]).
[9,5,4,3,1,1]
```

### Currying และ Partial Application

```erlang
%% Erlang ไม่ support currying โดยตรง แต่ทำได้ผ่าน closures

%% Partial application ด้วย fun
add(X, Y) -> X + Y.

add5 = fun(Y) -> add(5, Y) end.

1> add5(3).
8

%% Curry function manually
curry(Fun) ->
    fun(X) -> fun(Y) -> Fun(X, Y) end end.

2> CurriedAdd = curry(fun(X, Y) -> X + Y end).
3> Add5 = CurriedAdd(5).
4> Add5(3).
8

%% Partial application helper
partial(Fun, X) ->
    fun(Y) -> Fun(X, Y) end.

5> Multiply = fun(X, Y) -> X * Y end.
6> Double = partial(Multiply, 2).
7> Double(7).
14
```

---

## 8. Closures

Closure คือ function ที่ "จดจำ" environment ที่มันถูกสร้าง

```erlang
%% Closure จับตัวแปร จาก outer scope
make_counter() ->
    Count = 0,  % จะถูก capture โดย fun
    fun() ->
        Count + 1  % Count = 0 เสมอ! (immutable)
    end.

%% เนื่องจาก immutability ต้องใช้ process สำหรับ mutable state
make_counter_process() ->
    spawn(fun() -> counter_loop(0) end).

counter_loop(N) ->
    receive
        {get, From} ->
            From ! {count, N},
            counter_loop(N);
        increment ->
            counter_loop(N + 1);
        reset ->
            counter_loop(0)
    end.

%% ใช้งาน
1> C = make_counter_process().
2> C ! increment.
3> C ! increment.
4> C ! increment.
5> C ! {get, self()}.
6> receive {count, N} -> N end.
3
```

### Practical Closure Examples

```erlang
%% สร้าง validator ด้วย closure
make_range_validator(Min, Max) ->
    fun(Value) ->
        if
            Value >= Min, Value =< Max -> {ok, Value};
            true -> {error, {out_of_range, Value, Min, Max}}
        end
    end.

1> ValidAge = make_range_validator(0, 150).
2> ValidAge(25).
{ok, 25}
3> ValidAge(200).
{error, {out_of_range, 200, 0, 150}}

%% Memoization ด้วย ETS (จะเรียนละเอียดใน Part 19)
make_memoized(Fun) ->
    Table = ets:new(memo_table, [set]),
    fun(Arg) ->
        case ets:lookup(Table, Arg) of
            [{_, Cached}] -> Cached;
            [] ->
                Result = Fun(Arg),
                ets:insert(Table, {Arg, Result}),
                Result
        end
    end.

%% Timer closure
make_timer(Delay, Action) ->
    fun() ->
        timer:sleep(Delay),
        Action()
    end.
```

---

## 9. Function References

```erlang
%% อ้างถึง function ด้วย fun Module:Name/Arity
%% หรือ fun Name/Arity (สำหรับ function ใน module เดิม)

%% อ้างถึง module function
1> Sort = fun lists:sort/1.
2> Sort([3,1,2]).
[1,2,3]

3> Format = fun io:format/2.
4> Format("Hello ~s~n", ["World"]).
Hello World
ok

%% อ้างถึง local function
-module(my_module).
-export([apply_to_list/2]).

double(X) -> X * 2.

apply_to_list(Fun, List) ->
    lists:map(Fun, List).

use_local() ->
    apply_to_list(fun double/1, [1,2,3,4,5]).

%% ใช้ map/filter/fold กับ function reference
5> lists:map(fun erlang:abs/1, [-1, -2, 3, -4, 5]).
[1,2,3,4,5]

6> lists:filter(fun erlang:is_integer/1, [1, 1.5, 2, 2.5, 3]).
[1,2,3]
```

### Dynamic Function Calls

```erlang
%% เรียก function แบบ dynamic ด้วย erlang:apply/3
Module = lists.
Function = sort.
Args = [[3,1,2]].

erlang:apply(Module, Function, Args).
% [1,2,3]

%% หรือ apply/2 กับ fun
Fun = fun lists:sort/1.
erlang:apply(Fun, [[3,1,2]]).
% [1,2,3]

%% ใช้ใน plugin system
call_plugin(PluginModule, Function, Args) ->
    case erlang:function_exported(PluginModule, Function, length(Args)) of
        true  -> erlang:apply(PluginModule, Function, Args);
        false -> {error, {function_not_exported, PluginModule, Function}}
    end.
```

---

## 10. Apply Functions

```erlang
%% erlang:apply/2 และ erlang:apply/3

%% apply/2: ใช้ fun
Fun = fun(X, Y) -> X + Y end.
erlang:apply(Fun, [3, 4]).
% 7

%% apply/3: ใช้ module:function
erlang:apply(lists, sort, [[3,1,2]]).
% [1,2,3]

%% Dynamic dispatch
dispatch(Commands, State) ->
    lists:foldl(
        fun({Module, Func, Args}, CurrentState) ->
            erlang:apply(Module, Func, [CurrentState | Args])
        end,
        State,
        Commands
    ).

%% ตัวอย่าง: plugin system
-module(plugin_manager).

run_plugin(Plugin, Event, Data) ->
    Module = Plugin#plugin.module,
    case erlang:function_exported(Module, handle_event, 2) of
        true  -> Module:handle_event(Event, Data);
        false -> {error, {no_handler, Module}}
    end.
```

---

## 11. Module Attributes

### Custom Attributes

```erlang
-module(my_module).

%% Custom attributes (accessible at runtime)
-author("Alice Smith").
-version("2.1.0").
-description("My useful module").
-created("2024-01-15").

%% ดู custom attributes
1> my_module:module_info(attributes).
[{vsn,[...hash...]},
 {author,"Alice Smith"},
 {version,"2.1.0"},
 ...
```

### Conditional Compilation

```erlang
%% -ifdef สำหรับ conditional compilation
-ifdef(DEBUG).
-define(LOG(X), io:format("[DEBUG] ~p~n", [X])).
-else.
-define(LOG(_), ok).
-endif.

%% ใช้ใน code
process(Data) ->
    ?LOG(Data),
    % ... processing ...
    ok.

%% Compile ด้วย DEBUG flag
%% erlc -DDEBUG my_module.erl

%% -ifndef
-ifndef(TEST).
-define(ASSERT(X), ok).
-else.
-define(ASSERT(X), true = (X)).
-endif.
```

### Type Declarations

```erlang
%% สำหรับ Dialyzer static analysis

%% Basic types
-type name()  :: binary().
-type age()   :: 0..150.
-type email() :: binary().

%% Custom types
-type user_id() :: pos_integer().
-type status()  :: active | inactive | banned.

%% Record types
-type user() :: #{
    id     := user_id(),
    name   := name(),
    age    := age(),
    email  := email(),
    status := status()
}.

%% Function specs
-spec create_user(name(), age(), email()) -> {ok, user()} | {error, term()}.
create_user(Name, Age, Email) ->
    validate_and_create(Name, Age, Email).

-spec get_user(user_id()) -> {ok, user()} | {error, not_found}.
get_user(Id) ->
    db:lookup(users, Id).
```

---

## 12. Module Info

```erlang
%% ดู module information ที่ runtime
1> lists:module_info().
[{module,lists},
 {exports,[{all,2},{any,2},{append,1},{append,2},...
 {attributes,[{vsn,[...]}]},
 {compile,[{options,[...]},{version,...},...}
 {native,false},
 {md5,...}]

%% ดู specific key
2> lists:module_info(exports).
[{all,2},{any,2},{append,1},...

3> lists:module_info(attributes).
[{vsn,[...]}]

%% ตรวจสอบว่า function exist
4> erlang:function_exported(lists, sort, 1).
true
5> erlang:function_exported(lists, nonexistent, 0).
false

%% ดู loaded modules
6> code:all_loaded().
[{gen_server,...},{supervisor,...},...]

%% Load module
7> code:load_file(my_module).
{module, my_module}

%% Reload module (hot code swap!)
8> code:purge(my_module).
9> code:load_file(my_module).
```

---

## 13. Code Organization Best Practices

### Single Responsibility

```erlang
%% แต่ละ module ควรมีหน้าที่เดียว

%% BAD: module ทำหลายอย่างเกินไป
-module(everything).
-export([validate_user/1, store_user/1, send_email/2, 
         format_report/1, parse_csv/1]).

%% GOOD: แยก module ตาม responsibility
-module(user_validator).    %% แค่ validate
-module(user_store).        %% แค่ store/retrieve
-module(email_service).     %% แค่ send email
-module(report_formatter).  %% แค่ format
-module(csv_parser).        %% แค่ parse CSV
```

### API Design

```erlang
%% API module ควร expose ฟังก์ชันที่ user-friendly
-module(bank).

%% Public API
-export([open_account/1, deposit/2, withdraw/2, 
         transfer/3, balance/1, close_account/1]).

%% Implementation แยกใน internal modules
open_account(Name) ->
    bank_account:create(Name).

deposit(AccountId, Amount) ->
    bank_account:add_funds(AccountId, Amount).

% ... etc

%% Internal functions (private)
validate_amount(Amount) when Amount > 0 -> ok;
validate_amount(_) -> {error, invalid_amount}.

generate_transaction_id() ->
    erlang:unique_integer([positive]).
```

### Error Handling Consistency

```erlang
%% ใช้ {ok, Value} | {error, Reason} consistently

-module(file_reader).

%% ทุก function return {ok, ...} หรือ {error, ...}
read_file(Path) ->
    case file:read_file(Path) of
        {ok, Content}    -> {ok, Content};
        {error, enoent}  -> {error, file_not_found};
        {error, eacces}  -> {error, permission_denied};
        {error, Reason}  -> {error, {unexpected, Reason}}
    end.

parse_content(Content) ->
    try {ok, do_parse(Content)}
    catch
        error:badarg -> {error, invalid_content};
        _:Reason     -> {error, {parse_error, Reason}}
    end.
```

### Module Hierarchy

```
my_app/
├── my_app.erl           %% Public API
├── my_app_core.erl      %% Core business logic  
├── my_app_store.erl     %% Data storage
├── my_app_validator.erl %% Input validation
├── my_app_formatter.erl %% Output formatting
└── my_app_utils.erl     %% Utility functions
```

---

## 14. แบบฝึกหัด

### Exercise 1: Math Library

```erlang
%% สร้าง math_lib.erl ที่มีฟังก์ชัน:
%% - gcd(A, B)           : Greatest Common Divisor
%% - lcm(A, B)           : Least Common Multiple
%% - is_prime(N)         : ตรวจสอบว่าเป็น prime
%% - primes_up_to(N)     : list ของ primes ถึง N (Sieve of Eratosthenes)
%% - fibonacci(N)        : N-th fibonacci number
%% - combinations(N, K)  : C(n,k)

-module(math_lib).
-export([gcd/2, lcm/2, is_prime/1, primes_up_to/1, 
         fibonacci/1, combinations/2]).

gcd(A, 0) -> A;
gcd(A, B) -> gcd(B, A rem B).

lcm(A, B) -> A * B div gcd(A, B).

is_prime(N) when N < 2 -> false;
is_prime(2) -> true;
is_prime(N) when N rem 2 =:= 0 -> false;
is_prime(N) -> is_prime(N, 3).

is_prime(N, I) when I * I > N -> true;
is_prime(N, I) when N rem I =:= 0 -> false;
is_prime(N, I) -> is_prime(N, I + 2).

primes_up_to(N) ->
    Sieve = array:new(N+1, {default, true}),
    sieve(Sieve, 2, N, []).

sieve(Sieve, I, N, Primes) when I > N ->
    lists:reverse(Primes);
sieve(Sieve, I, N, Primes) ->
    case array:get(I, Sieve) of
        true ->
            NewSieve = mark_multiples(Sieve, I*I, I, N),
            sieve(NewSieve, I+1, N, [I|Primes]);
        false ->
            sieve(Sieve, I+1, N, Primes)
    end.

mark_multiples(Sieve, J, _, N) when J > N -> Sieve;
mark_multiples(Sieve, J, I, N) ->
    mark_multiples(array:set(J, false, Sieve), J+I, I, N).

fibonacci(N) -> fibonacci(N, 0, 1).
fibonacci(0, A, _) -> A;
fibonacci(N, A, B) -> fibonacci(N-1, B, A+B).

combinations(N, K) when K < 0; K > N -> 0;
combinations(N, K) when K =:= 0; K =:= N -> 1;
combinations(N, K) ->
    factorial(N) div (factorial(K) * factorial(N-K)).

factorial(0) -> 1;
factorial(N) -> N * factorial(N-1).
```

### Exercise 2: Functional Utilities

```erlang
%% สร้าง func_utils.erl ที่มี:
%% - pipe(Value, Funs) : pipe value ผ่าน list of functions
%% - memoize(Fun)      : return memoized version ของ fun
%% - retry(Fun, N)     : ลอง N ครั้งถ้า fail
%% - timed(Fun)        : return {Time, Result}

-module(func_utils).
-export([pipe/2, retry/2, timed/1]).

pipe(Value, []) -> Value;
pipe(Value, [Fun|Funs]) ->
    pipe(Fun(Value), Funs).

retry(Fun, MaxTries) ->
    retry(Fun, MaxTries, 1).

retry(Fun, MaxTries, Attempt) when Attempt > MaxTries ->
    {error, max_retries_exceeded};
retry(Fun, MaxTries, Attempt) ->
    case Fun() of
        {ok, Result} -> {ok, Result};
        {error, _} -> 
            timer:sleep(100 * Attempt),  % exponential backoff
            retry(Fun, MaxTries, Attempt + 1)
    end.

timed(Fun) ->
    Start = erlang:monotonic_time(microsecond),
    Result = Fun(),
    End = erlang:monotonic_time(microsecond),
    {End - Start, Result}.

%% ทดสอบ
%% 1> func_utils:pipe(5, [
%%      fun(X) -> X * 2 end,
%%      fun(X) -> X + 1 end,
%%      fun(X) -> X * X end
%%    ]).
%% 121  (((5*2)+1)^2 = 11^2 = 121)
%%
%% 2> {Time, _} = func_utils:timed(fun() -> timer:sleep(100) end).
%% {~100000, ok}
```

### Exercise 3: String Processing Module

```erlang
%% สร้าง str_utils.erl ที่มี:
%% - capitalize(Str)  : "hello world" -> "Hello World"
%% - camel_to_snake(Str) : "helloWorld" -> "hello_world"
%% - snake_to_camel(Str) : "hello_world" -> "helloWorld"
%% - truncate(Str, N) : ตัด string ให้ยาว N chars + "..."
%% - count_words(Str) : นับจำนวนคำ

-module(str_utils).
-export([capitalize/1, truncate/2, count_words/1]).

capitalize(Str) ->
    Words = string:tokens(Str, " "),
    Capitalized = lists:map(fun capitalize_word/1, Words),
    string:join(Capitalized, " ").

capitalize_word([]) -> [];
capitalize_word([H|T]) ->
    [string:to_upper([H]) ++ T].

%% ต้อง handle list of chars properly
capitalize_word2([H|T]) when H >= $a, H =< $z ->
    [H - 32 | T];
capitalize_word2(Word) ->
    Word.

truncate(Str, N) when length(Str) > N ->
    string:substr(Str, 1, N - 3) ++ "...";
truncate(Str, _) ->
    Str.

count_words(Str) ->
    Words = string:tokens(Str, " \t\n"),
    length(Words).

%% ทดสอบ
%% 1> str_utils:capitalize("hello world foo").
%% "Hello World Foo"
%% 2> str_utils:truncate("This is a long string", 10).
%% "This is..."
%% 3> str_utils:count_words("hello world foo bar").
%% 4
```

---

## สรุป Part 04

ใน Part นี้คุณได้เรียนรู้:

✅ Module คืออะไรและโครงสร้าง  
✅ Module declarations และ attributes  
✅ Function syntax, arity, multiple clauses  
✅ Export และ Import  
✅ Tail recursion และ TCO  
✅ Higher-order functions  
✅ Closures  
✅ Function references  
✅ Apply functions  
✅ Module info  
✅ Code organization best practices  

---

## ต่อไป: [Part 05 — Lists และ Recursion](../part05/README.md)

ใน Part ถัดไปเราจะเรียนรู้:
- List operations ลึกซึ้ง
- Recursion patterns ขั้นสูง
- List Comprehensions
- lists module ทั้งหมด
- Algorithms ด้วย Lists
- Performance considerations

---

*Part 04/100 | [← ก่อนหน้า](../part03/README.md) | [ถัดไป →](../part05/README.md)*
