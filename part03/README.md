# Part 03: Pattern Matching

> **"Pattern matching is the most powerful feature in Erlang"**  
> — ความสามารถที่ทำให้ Erlang โดดเด่นและแตกต่างจากภาษาอื่น

---

## สารบัญ

1. [Pattern Matching คืออะไร?](#1-pattern-matching-คืออะไร)
2. [พื้นฐาน Pattern Matching](#2-พื้นฐาน-pattern-matching)
3. [Matching กับ Tuples](#3-matching-กับ-tuples)
4. [Matching กับ Lists](#4-matching-กับ-lists)
5. [Matching กับ Binaries](#5-matching-กับ-binaries)
6. [Matching กับ Maps](#6-matching-กับ-maps)
7. [Guards](#7-guards)
8. [Multiple Clause Functions](#8-multiple-clause-functions)
9. [Case Expression](#9-case-expression)
10. [Pattern Matching ขั้นสูง](#10-pattern-matching-ขั้นสูง)
11. [Common Patterns และ Idioms](#11-common-patterns-และ-idioms)
12. [แบบฝึกหัด](#12-แบบฝึกหัด)

---

## 1. Pattern Matching คืออะไร?

Pattern Matching คือการ **จับคู่โครงสร้างของข้อมูล** กับ pattern ที่กำหนด

```erlang
% ใน Erlang, = ไม่ใช่ assignment แต่คือ MATCH operator!

% ถ้า pattern ตรงกัน -> ตัวแปรจะถูก bind
X = 5.        % 5 matches 5, X binds to 5

% ถ้า pattern ไม่ตรงกัน -> exception!
5 = 6.        % ** exception error: no match
```

### ทำไม Pattern Matching ถึงสำคัญ?

```
ปัญหา: ต้องจัดการกับ response หลายรูปแบบ

// C/Java style (ugly)
if (result.isOk()) {
    value = result.getValue();
} else {
    error = result.getError();
    handle(error);
}

% Erlang style (elegant pattern matching)
case result of
    {ok, Value}     -> process(Value);
    {error, Reason} -> handle(Reason)
end.
```

Pattern Matching ทำให้:
- Code อ่านง่ายและ expressive
- Compiler ตรวจสอบ completeness
- ไม่ต้องเขียน null checks
- Error handling เป็น first-class citizen

---

## 2. พื้นฐาน Pattern Matching

### Match กับค่าเฉพาะ

```erlang
% Literal values
1> 42 = 42.       % ok, matches
42
2> hello = hello.  % ok, matches  
hello
3> 42 = 43.       % ** exception error
4> ok = error.    % ** exception error

% Binding ตัวแปร
5> X = 42.
42
6> X.
42

% ตัวแปรที่ถูก bind แล้ว จะ match ค่าเดิม
7> X = 42.   % ok (X already bound to 42, 42 matches 42)
42
8> X = 100.  % ** exception error (X is 42, can't match 100)
```

### Wildcard Pattern `_`

```erlang
% _ match ทุกอย่างและไม่ bind ค่า
1> _ = 42.
42
2> _ = hello.
hello
3> _ = [1,2,3].
[1,2,3]

% ใช้ _ เมื่อไม่สนใจค่า
4> {_, Y} = {1, 2}.
{1,2}
5> Y.
2

% _Name convention: match แต่แสดงว่าอาจไม่ใช้
ignore_first({_First, Second}) ->
    Second.
```

### Match ซ้อน (Nested Matching)

```erlang
% Pattern matching ทำงาน recursive
1> {A, {B, C}} = {1, {2, 3}}.
{1,{2,3}}
2> A.
1
3> B.
2
4> C.
3

% Match หลายระดับ
5> [[H|T]|Rest] = [[1,2,3],[4,5]].
6> H.
1
7> T.
[2,3]
8> Rest.
[[4,5]]
```

---

## 3. Matching กับ Tuples

### Tuple Pattern Matching

```erlang
% Match tuple ทั้งหมด
1> {X, Y} = {10, 20}.
{10,20}
2> X.
10
3> Y.
20

% Match บาง element
4> {_, Second, _} = {a, b, c}.
5> Second.
b

% Match ขนาด tuple
6> {A} = {single}.       % 1-tuple
7> {B, C} = {1, 2}.      % 2-tuple
8> {D, E, F} = {1, 2, 3}. % 3-tuple

% ขนาดไม่ตรงจะ error!
9> {X, Y} = {1, 2, 3}.
** exception error: no match of right hand side value {1,2,3}
```

### Tagged Tuple Patterns

```erlang
% Pattern ที่นิยมมาก: {Tag, Value}
handle_result({ok, Value}) ->
    process(Value);
handle_result({error, not_found}) ->
    default_value();
handle_result({error, Reason}) ->
    log_error(Reason),
    error.

% ใช้งาน
handle_result({ok, 42}).          % process(42)
handle_result({error, not_found}). % default_value()
handle_result({error, timeout}).   % log_error(timeout), error

% HTTP Status Pattern
describe_status({status, 200}) -> "OK";
describe_status({status, 201}) -> "Created";
describe_status({status, 400}) -> "Bad Request";
describe_status({status, 404}) -> "Not Found";
describe_status({status, 500}) -> "Internal Server Error";
describe_status({status, Code}) -> "Unknown: " ++ integer_to_list(Code).
```

### Match กับ Record-like Tuples

```erlang
% ก่อนจะมี Records และ Maps ใน Erlang
% นิยมใช้ tagged tuples เก็บ structured data

% Person: {person, Name, Age, Email}
person_name({person, Name, _, _}) -> Name.
person_age({person, _, Age, _}) -> Age.
person_email({person, _, _, Email}) -> Email.

% สร้าง person
alice() -> {person, "Alice", 30, "alice@example.com"}.

% ใช้งาน
1> P = alice().
{person,"Alice",30,"alice@example.com"}
2> person_name(P).
"Alice"
3> person_age(P).
30
```

---

## 4. Matching กับ Lists

### Head/Tail Matching

```erlang
% [Head | Tail] pattern
1> [H|T] = [1, 2, 3, 4, 5].
2> H.
1
3> T.
[2,3,4,5]

% Match หลาย elements
4> [A, B | Rest] = [1, 2, 3, 4, 5].
5> A.
1
6> B.
2
7> Rest.
[3,4,5]

% Match ทั้งหมด
8> [X, Y, Z] = [1, 2, 3].
9> X.
1

% Empty list
10> [] = [].  % ok
11> [H|T] = [].  % ** exception error! (empty list has no head)
```

### Recursive List Patterns

```erlang
% ฟังก์ชัน recursive กับ list patterns

% sum all elements
sum([]) -> 0;
sum([H|T]) -> H + sum(T).

% ทดสอบ
1> sum([1, 2, 3, 4, 5]).
15

% length ของ list
my_length([]) -> 0;
my_length([_|T]) -> 1 + my_length(T).

2> my_length([a, b, c, d]).
4

% reverse list
my_reverse([]) -> [];
my_reverse([H|T]) -> my_reverse(T) ++ [H].

% (เวอร์ชัน efficient กว่า)
my_reverse(List) -> my_reverse(List, []).
my_reverse([], Acc) -> Acc;
my_reverse([H|T], Acc) -> my_reverse(T, [H|Acc]).

3> my_reverse([1, 2, 3, 4, 5]).
[5,4,3,2,1]
```

### Pattern Matching กับ List Values

```erlang
% Match specific values ใน list
starts_with_one([1|_]) -> true;
starts_with_one(_) -> false.

% Match first two elements
has_two_elements([_, _]) -> true;
has_two_elements(_) -> false.

% Match specific sequence
starts_with_hello([$h,$e,$l,$l,$o|_]) -> true;
starts_with_hello(<<"hello", _/binary>>) -> true;  % binary version
starts_with_hello(_) -> false.

% Zipping two lists
zip([H1|T1], [H2|T2]) -> [{H1, H2} | zip(T1, T2)];
zip([], []) -> [];
zip(_, _) -> error.

1> zip([1,2,3], [a,b,c]).
[{1,a},{2,b},{3,c}]
```

---

## 5. Matching กับ Binaries

Binary pattern matching ทรงพลังมากสำหรับ protocol parsing

```erlang
% Basic binary matching
1> <<A, B, C>> = <<1, 2, 3>>.
2> A.
1

% Matching กับ size
3> <<First:16, Rest/binary>> = <<1, 2, 3, 4, 5>>.
4> First.
258  % 1*256 + 2 = 258
5> Rest.
<<3,4,5>>

% String matching
6> <<"hello", " ", World/binary>> = <<"hello world">>.
7> World.
<<"world">>

% Match กับ type specifiers
8> <<Float:64/float>> = <<64, 9, 30, 184, 81, 235, 133, 31>>.
9> Float.
3.14159265358979

% Match integer fields
10> <<R:8, G:8, B:8>> = <<255, 128, 0>>.
11> R.
255
12> G.
128
13> B.
0
```

### Protocol Parsing ด้วย Binary Pattern

```erlang
%% ตัวอย่าง: Parse HTTP request line
parse_request_line(<<"GET ", Path/binary>>) ->
    {get, parse_path(Path)};
parse_request_line(<<"POST ", Path/binary>>) ->
    {post, parse_path(Path)};
parse_request_line(<<"PUT ", Path/binary>>) ->
    {put, parse_path(Path)};
parse_request_line(<<"DELETE ", Path/binary>>) ->
    {delete, parse_path(Path)}.

parse_path(PathAndVersion) ->
    case binary:split(PathAndVersion, <<" ">>) of
        [Path, _Version] -> Path;
        [Path]           -> Path
    end.

%% Parse IPv4 address
parse_ipv4(<<A, B, C, D>>) ->
    {A, B, C, D}.

%% Build IPv4 binary
build_ipv4({A, B, C, D}) ->
    <<A, B, C, D>>.

%% Parse simple TLV (Type-Length-Value)
parse_tlv(<<Type:8, Length:16, Value:Length/binary, Rest/binary>>) ->
    [{Type, Value} | parse_tlv(Rest)];
parse_tlv(<<>>) ->
    [].

%% ตัวอย่าง custom protocol
parse_packet(<<Version:8, Type:8, Length:32, Payload:Length/binary>>) ->
    #{version => Version,
      type    => Type,
      payload => Payload}.
```

### Bit Syntax ขั้นสูง

```erlang
% Extracting bits
1> <<Flag1:1, Flag2:1, Flag3:1, _:5>> = <<16#80>>.
% 16#80 = 10000000
2> Flag1.
1
3> Flag2.
0
4> Flag3.
0

% UTF-8 encoding
5> <<C/utf8>> = <<"A">>.
6> C.
65

7> <<Thai/utf8>> = <<"ก">>.
8> Thai.
3585  % Unicode code point

% Building binary
9> <<"hello ", World/binary>> = <<"hello world">>.
10> World.
<<"world">>
```

---

## 6. Matching กับ Maps

```erlang
% Map pattern matching ใช้ :=
1> #{name := Name} = #{name => "Alice", age => 30}.
2> Name.
"Alice"

% Match หลาย keys
3> #{name := N, age := A} = #{name => "Bob", age => 25, city => "NYC"}.
4> N.
"Bob"
5> A.
25

% ไม่จำเป็นต้อง match ทุก key
6> #{x := X} = #{x => 1, y => 2, z => 3}.
7> X.
1

% Match ใน function
describe_user(#{name := Name, age := Age}) when Age >= 18 ->
    io:format("~s is an adult~n", [Name]);
describe_user(#{name := Name}) ->
    io:format("~s is a minor~n", [Name]).

% Nested map matching
get_city(#{address := #{city := City}}) ->
    City.

1> get_city(#{address => #{city => "Bangkok", country => "Thailand"}}).
"Bangkok"
```

### Map Update Pattern

```erlang
% Map update syntax
update_age(User = #{age := _}, NewAge) ->
    User#{age => NewAge}.

1> U = #{name => "Alice", age => 30}.
2> update_age(U, 31).
#{age => 31, name => "Alice"}
```

---

## 7. Guards

Guards คือ conditions เพิ่มเติมที่ต้องเป็น true ก่อน pattern จะ match

### Guard Syntax

```erlang
% Syntax: function(Pattern) when Guard -> Body

% Guard ใช้ when keyword
classify_number(N) when N > 0 -> positive;
classify_number(N) when N < 0 -> negative;
classify_number(0)            -> zero.

% หลาย guards ด้วย , (AND) หรือ ; (OR)
classify_age(Age) when Age >= 0, Age < 18  -> minor;
classify_age(Age) when Age >= 18, Age < 65 -> adult;
classify_age(Age) when Age >= 65           -> senior.

% Guard ด้วย ; (OR)
is_weekday(Day) when 
    Day =:= monday;
    Day =:= tuesday;
    Day =:= wednesday;
    Day =:= thursday;
    Day =:= friday ->
    true;
is_weekday(_) ->
    false.
```

### Guard Expressions ที่ใช้ได้

```erlang
% Type checks
is_integer(X)
is_float(X)
is_number(X)
is_atom(X)
is_boolean(X)
is_binary(X)
is_bitstring(X)
is_list(X)
is_tuple(X)
is_map(X)
is_pid(X)
is_reference(X)
is_function(X)
is_function(X, Arity)
is_record(X, RecordName)
is_record(X, RecordName, Size)

% Comparison
X == Y, X /= Y, X =:= Y, X =/= Y
X < Y, X > Y, X <= Y, X >= Y

% Arithmetic (pure functions only)
abs(X), round(X), trunc(X), length(X)
element(N, Tuple), map_size(Map), tuple_size(Tuple)
byte_size(Binary), bit_size(Binary)
hd(List), tl(List)
float(X), integer_to_list(X)
max(X, Y), min(X, Y)
node(), node(Pid), self()
size(X)

% Boolean
not X, X and Y, X or Y
X andalso Y, X orelse Y
```

### Guard ขั้นสูง

```erlang
% Guard ซับซ้อน
safe_divide(X, Y) when 
    is_number(X),
    is_number(Y), 
    Y =/= 0 ->
    {ok, X / Y};
safe_divide(_, 0) ->
    {error, division_by_zero};
safe_divide(X, Y) ->
    {error, {not_numbers, X, Y}}.

% Guard ใน case
process(Value) ->
    case Value of
        X when is_integer(X), X > 0 ->
            {positive_integer, X};
        X when is_float(X) ->
            {float, X};
        X when is_binary(X), byte_size(X) > 0 ->
            {non_empty_binary, X};
        _ ->
            {other, Value}
    end.

% Guard กับ tuple
validate_point({X, Y}) when 
    is_number(X), 
    is_number(Y),
    X >= 0, X =< 100,
    Y >= 0, Y =< 100 ->
    {ok, {X, Y}};
validate_point(P) ->
    {error, {invalid_point, P}}.
```

---

## 8. Multiple Clause Functions

Function ใน Erlang สามารถมีหลาย clause โดยแต่ละ clause มี pattern ต่างกัน

### Syntax พื้นฐาน

```erlang
% แต่ละ clause คั่นด้วย ;
% Clause สุดท้ายลงท้ายด้วย .

greet(alice) ->
    "Hello, Alice!";
greet(bob) ->
    "Hi, Bob!";
greet(Name) ->
    "Hey, " ++ atom_to_list(Name) ++ "!".

% การทำงาน: Erlang จะลอง clause แต่ละอันตามลำดับ
% หยุดเมื่อ pattern match สำเร็จ
```

### Recursive Functions ด้วย Multiple Clauses

```erlang
%% Fibonacci
fib(0) -> 0;
fib(1) -> 1;
fib(N) when N > 1 -> fib(N-1) + fib(N-2).

1> fib(10).
55

%% Factorial
factorial(0) -> 1;
factorial(N) when N > 0 -> N * factorial(N-1).

2> factorial(5).
120

%% Power function
power(_, 0) -> 1;
power(Base, Exp) when Exp > 0 ->
    Base * power(Base, Exp - 1).

%% Efficient power (fast exponentiation)
fast_power(_, 0) -> 1;
fast_power(Base, Exp) when Exp rem 2 =:= 0 ->
    Half = fast_power(Base, Exp div 2),
    Half * Half;
fast_power(Base, Exp) ->
    Base * fast_power(Base, Exp - 1).
```

### List Processing Functions

```erlang
%% Map
my_map(_, []) -> [];
my_map(Fun, [H|T]) -> [Fun(H) | my_map(Fun, T)].

%% Filter
my_filter(_, []) -> [];
my_filter(Pred, [H|T]) ->
    case Pred(H) of
        true  -> [H | my_filter(Pred, T)];
        false -> my_filter(Pred, T)
    end.

%% FoldLeft
my_foldl(_, Acc, []) -> Acc;
my_foldl(Fun, Acc, [H|T]) ->
    my_foldl(Fun, Fun(H, Acc), T).

%% FoldRight
my_foldr(_, Acc, []) -> Acc;
my_foldr(Fun, Acc, [H|T]) ->
    Fun(H, my_foldr(Fun, Acc, T)).

%% ทดสอบ
1> my_map(fun(X) -> X * X end, [1,2,3,4,5]).
[1,4,9,16,25]

2> my_filter(fun(X) -> X rem 2 =:= 0 end, [1,2,3,4,5,6]).
[2,4,6]

3> my_foldl(fun(X, Acc) -> X + Acc end, 0, [1,2,3,4,5]).
15
```

### Dispatch ด้วย Atom Patterns

```erlang
%% State machine ง่ายๆ
handle_event(idle, start) -> running;
handle_event(running, pause) -> paused;
handle_event(running, stop) -> idle;
handle_event(paused, resume) -> running;
handle_event(paused, stop) -> idle;
handle_event(State, Event) ->
    {error, {invalid_transition, State, Event}}.

%% ใช้งาน
1> S0 = idle.
2> S1 = handle_event(S0, start).   % running
3> S2 = handle_event(S1, pause).   % paused
4> S3 = handle_event(S2, resume).  % running
5> handle_event(S3, stop).         % idle

%% Router pattern
route(get, "/users") -> list_users();
route(get, "/users/" ++ Id) -> get_user(Id);
route(post, "/users") -> create_user();
route(put, "/users/" ++ Id) -> update_user(Id);
route(delete, "/users/" ++ Id) -> delete_user(Id);
route(Method, Path) -> {error, {not_found, Method, Path}}.
```

---

## 9. Case Expression

`case` ช่วยให้เราทำ pattern matching ใน expression

### Syntax

```erlang
case Expression of
    Pattern1 [when Guard1] -> Body1;
    Pattern2 [when Guard2] -> Body2;
    ...
    PatternN               -> BodyN
end
```

### ตัวอย่างการใช้งาน

```erlang
%% Case พื้นฐาน
describe_list(List) ->
    case List of
        []        -> "empty";
        [_]       -> "singleton";
        [_, _]    -> "pair";
        [_,_,_|_] -> "three or more"
    end.

1> describe_list([]).
"empty"
2> describe_list([1]).
"singleton"
3> describe_list([1,2]).
"pair"
4> describe_list([1,2,3,4]).
"three or more"

%% Case ซับซ้อน
process_request(Request) ->
    case authenticate(Request) of
        {ok, User} ->
            case authorize(User, Request) of
                allowed ->
                    {ok, handle(User, Request)};
                denied ->
                    {error, forbidden}
            end;
        {error, Reason} ->
            {error, {auth_failed, Reason}}
    end.
```

### Case vs Function Clauses

```erlang
%% วิธีที่ 1: Multiple function clauses
classify_temp_fc(T) when T < 0   -> freezing;
classify_temp_fc(T) when T < 10  -> cold;
classify_temp_fc(T) when T < 20  -> cool;
classify_temp_fc(T) when T < 30  -> warm;
classify_temp_fc(_)              -> hot.

%% วิธีที่ 2: Case expression
classify_temp_case(T) ->
    case T of
        _ when T < 0  -> freezing;
        _ when T < 10 -> cold;
        _ when T < 20 -> cool;
        _ when T < 30 -> warm;
        _             -> hot
    end.

%% ทั้งสองวิธีเหมือนกันทุกประการ
%% ใช้ function clauses เมื่อ function นั้นมีหลาย behavior
%% ใช้ case เมื่ออยู่ใน middle ของ logic
```

---

## 10. Pattern Matching ขั้นสูง

### Matching กับ Existing Values

```erlang
%% เมื่อตัวแปรถูก bind แล้ว จะ match ค่าเดิม
check_same_values(X, X) ->
    {same, X};
check_same_values(X, Y) ->
    {different, X, Y}.

1> check_same_values(5, 5).
{same, 5}
2> check_same_values(5, 6).
{different, 5, 6}

%% Pin operator (^) ใน Elixir มีใน Erlang แบบ implicit
check_expected_status({Status, _Body}) when Status =:= expected_status() ->
    ok;
check_expected_status({Status, Body}) ->
    {unexpected, Status, Body}.
```

### As-Patterns

```erlang
%% ใช้ = เพื่อ bind whole value และ match pattern
process_list([] = List) ->
    {empty, List};
process_list([_|_] = List) ->
    {non_empty, length(List), List}.

%% เทียบเท่ากับ
process_list2(List) ->
    case List of
        []    -> {empty, List};
        [_|_] -> {non_empty, length(List), List}
    end.

1> process_list([]).
{empty, []}
2> process_list([1,2,3]).
{non_empty, 3, [1,2,3]}

%% Useful pattern: bind และ destructure พร้อมกัน
head_and_list([H|_] = List) ->
    {H, List}.

1> head_and_list([1,2,3]).
{1, [1,2,3]}
```

### Nested Pattern Matching

```erlang
%% Pattern matching หลายระดับพร้อมกัน
process_user({user, {name, First, Last}, {age, Age}, {roles, Roles}}) 
    when Age >= 18, 
         lists:member(admin, Roles) ->
    {ok, {admin, First, Last}};
process_user({user, {name, First, Last}, _, _}) ->
    {ok, {user, First, Last}}.

%% Deep binary matching
parse_dns_name(<<Length:8, Name:Length/binary, Rest/binary>>) ->
    {Name, Rest};
parse_dns_name(<<0>>) ->
    {<<>>, <<>>}.

%% Complex list patterns
zip_with_index(List) ->
    zip_with_index(List, 0, []).

zip_with_index([], _, Acc) ->
    lists:reverse(Acc);
zip_with_index([H|T], Index, Acc) ->
    zip_with_index(T, Index + 1, [{Index, H} | Acc]).

1> zip_with_index([a, b, c, d]).
[{0,a},{1,b},{2,c},{3,d}]
```

### Matching ใน Receive

```erlang
%% Pattern matching ใน receive
server_loop() ->
    receive
        {get, Key, From} ->
            Value = get_value(Key),
            From ! {ok, Value},
            server_loop();
        {put, Key, Value} ->
            store_value(Key, Value),
            server_loop();
        {delete, Key} ->
            remove_value(Key),
            server_loop();
        stop ->
            io:format("Server stopping~n");
        Unknown ->
            io:format("Unknown message: ~p~n", [Unknown]),
            server_loop()
    end.
```

---

## 11. Common Patterns และ Idioms

### Result Pattern

```erlang
%% {ok, Value} | {error, Reason}
%% เป็น idiom มาตรฐานใน Erlang

safe_head([H|_]) -> {ok, H};
safe_head([])    -> {error, empty_list}.

safe_parse_int(S) ->
    try {ok, list_to_integer(S)}
    catch _:_ -> {error, not_an_integer}
    end.

%% Chaining results
process_pipeline(Input) ->
    case step1(Input) of
        {ok, Result1} ->
            case step2(Result1) of
                {ok, Result2} ->
                    case step3(Result2) of
                        {ok, Final} -> {ok, Final};
                        Error -> Error
                    end;
                Error -> Error
            end;
        Error -> Error
    end.
```

### Option Pattern (Maybe)

```erlang
%% undefined | {value, X}
find_user(Id, Users) ->
    case lists:keyfind(Id, 1, Users) of
        false       -> undefined;
        {_, User}   -> {value, User}
    end.

%% ใช้งาน
case find_user(123, AllUsers) of
    undefined     -> handle_not_found();
    {value, User} -> process_user(User)
end.
```

### Accumulator Pattern

```erlang
%% ใช้ accumulator สำหรับ tail recursion
sum_list(List) -> sum_list(List, 0).

sum_list([], Acc)    -> Acc;
sum_list([H|T], Acc) -> sum_list(T, H + Acc).

%% count elements ที่ match condition
count_matching(Pred, List) ->
    count_matching(Pred, List, 0).

count_matching(_, [], Acc) -> Acc;
count_matching(Pred, [H|T], Acc) ->
    case Pred(H) of
        true  -> count_matching(Pred, T, Acc + 1);
        false -> count_matching(Pred, T, Acc)
    end.

1> count_matching(fun(X) -> X rem 2 =:= 0 end, [1,2,3,4,5,6]).
3
```

### State Machine Pattern

```erlang
%% Traffic light state machine
next_light(red)    -> green;
next_light(green)  -> yellow;
next_light(yellow) -> red.

run_lights(Color, 0) ->
    io:format("Done at: ~p~n", [Color]);
run_lights(Color, N) ->
    io:format("Light: ~p~n", [Color]),
    timer:sleep(1000),
    run_lights(next_light(Color), N - 1).

%% ใช้งาน
run_lights(red, 6).
```

### Visitor Pattern

```erlang
%% Traverse tree structure
-type tree() :: {node, term(), tree(), tree()} | leaf.

tree_fold(_, Acc, leaf) ->
    Acc;
tree_fold(Fun, Acc, {node, Value, Left, Right}) ->
    Acc1 = tree_fold(Fun, Acc, Left),
    Acc2 = Fun(Value, Acc1),
    tree_fold(Fun, Acc2, Right).

tree_sum(Tree) ->
    tree_fold(fun(V, A) -> V + A end, 0, Tree).

%% สร้าง tree
T = {node, 1,
     {node, 2, leaf, leaf},
     {node, 3, leaf, leaf}}.
tree_sum(T).  % 6
```

---

## 12. แบบฝึกหัด

### Exercise 1: Pattern Matching Practice

```erlang
%% สร้าง shapes.erl ที่คำนวณพื้นที่รูปทรงต่างๆ
-module(shapes).
-export([area/1, perimeter/1]).

%% รูปทรง:
%% {circle, Radius}
%% {rectangle, Width, Height}
%% {triangle, Base, Height}
%% {square, Side}

area({circle, R}) ->
    math:pi() * R * R;
area({rectangle, W, H}) ->
    W * H;
area({triangle, B, H}) ->
    0.5 * B * H;
area({square, S}) ->
    S * S.

perimeter({circle, R}) ->
    2 * math:pi() * R;
perimeter({rectangle, W, H}) ->
    2 * (W + H);
perimeter({triangle, A, B, C}) ->  % ต้องผ่าน 3 sides
    A + B + C;
perimeter({square, S}) ->
    4 * S.

%% Test
%% 1> shapes:area({circle, 5}).
%% 78.53981633974483
%% 2> shapes:area({rectangle, 4, 6}).
%% 24
```

### Exercise 2: Expression Parser

```erlang
%% สร้าง expr_parser.erl
%% ที่ parse และ evaluate expressions แบบนี้:
%% {add, 1, 2}          -> 3
%% {mul, 3, 4}          -> 12
%% {add, {mul, 2, 3}, 4} -> 10
%% {neg, 5}             -> -5

-module(expr_parser).
-export([eval/1]).

eval({add, A, B}) -> eval(A) + eval(B);
eval({sub, A, B}) -> eval(A) - eval(B);
eval({mul, A, B}) -> eval(A) * eval(B);
eval({dvd, A, B}) when eval(B) =/= 0 -> eval(A) / eval(B);
eval({neg, A})    -> -eval(A);
eval({abs, A})    -> abs(eval(A));
eval(N) when is_number(N) -> N.
```

### Exercise 3: Data Validator

```erlang
%% สร้าง validator.erl ที่ validate ข้อมูล user
%% ข้อมูล user: #{name, age, email, role}

-module(validator).
-export([validate_user/1]).

validate_user(#{name := Name, age := Age, email := Email, role := Role} = User)
    when is_binary(Name),
         byte_size(Name) > 0,
         is_integer(Age),
         Age >= 0, Age =< 150,
         is_binary(Email),
         is_atom(Role) ->
    case validate_email(Email) of
        true  -> {ok, User};
        false -> {error, invalid_email}
    end;
validate_user(User) ->
    {error, {invalid_user_data, User}}.

validate_email(Email) ->
    case binary:split(Email, <<"@">>) of
        [Local, Domain] when byte_size(Local) > 0,
                             byte_size(Domain) > 0 ->
            true;
        _ ->
            false
    end.

%% ทดสอบ
%% 1> validator:validate_user(#{name => <<"Alice">>, age => 30,
%%                               email => <<"alice@example.com">>,
%%                               role => user}).
%% {ok, #{...}}
%%
%% 2> validator:validate_user(#{name => <<>>, age => 30,
%%                               email => <<"bad_email">>,
%%                               role => user}).
%% {error, {invalid_user_data, ...}}
```

### Exercise 4: Binary Protocol

```erlang
%% สร้าง protocol.erl สำหรับ parse/encode binary protocol:
%% Message format:
%%   [Version:8] [Type:8] [Payload Length:16] [Payload:N bytes]
%% Types: 1=ping, 2=pong, 3=data

-module(protocol).
-export([encode/1, decode/1]).

encode({ping}) ->
    <<1:8, 1:8, 0:16>>;
encode({pong}) ->
    <<1:8, 2:8, 0:16>>;
encode({data, Payload}) when is_binary(Payload) ->
    Len = byte_size(Payload),
    <<1:8, 3:8, Len:16, Payload/binary>>.

decode(<<1:8, 1:8, 0:16>>) ->
    {ok, {ping}};
decode(<<1:8, 2:8, 0:16>>) ->
    {ok, {pong}};
decode(<<1:8, 3:8, Len:16, Payload:Len/binary>>) ->
    {ok, {data, Payload}};
decode(<<Version:8, _/binary>>) when Version =/= 1 ->
    {error, {unsupported_version, Version}};
decode(_) ->
    {error, invalid_message}.

%% ทดสอบ
%% 1> Msg = protocol:encode({data, <<"Hello!">>}).
%% <<1,3,0,6,72,101,108,108,111,33>>
%% 2> protocol:decode(Msg).
%% {ok, {data, <<"Hello!">>}}
```

---

## สรุป Part 03

ใน Part นี้คุณได้เรียนรู้:

✅ Pattern Matching คืออะไรและทำไมถึงสำคัญ  
✅ Match กับ literal values, variables, wildcards  
✅ Tuple pattern matching และ tagged tuples  
✅ List pattern matching และ Head/Tail  
✅ Binary pattern matching สำหรับ protocol parsing  
✅ Map pattern matching  
✅ Guards — เงื่อนไขเพิ่มเติมสำหรับ pattern  
✅ Multiple clause functions  
✅ Case expression  
✅ As-patterns และ nested matching  
✅ Common idioms: Result, Option, Accumulator patterns  

---

## ต่อไป: [Part 04 — Functions และ Modules](../part04/README.md)

ใน Part ถัดไปเราจะเรียนรู้:
- การประกาศ Module
- Function exports และ visibility
- Higher-order functions
- Fun/Lambda ขั้นสูง
- Tail recursion
- Module attributes

---

*Part 03/100 | [← ก่อนหน้า](../part02/README.md) | [ถัดไป →](../part04/README.md)*
