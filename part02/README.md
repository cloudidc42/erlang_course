# Part 02: ประเภทข้อมูลพื้นฐานและ Syntax

> **"In Erlang, everything is a term"**  
> ทุกอย่างใน Erlang คือ term — ข้อมูลที่ immutable และ type-safe

---

## สารบัญ

1. [ประเภทข้อมูลใน Erlang](#1-ประเภทข้อมูลใน-erlang)
2. [Integer](#2-integer)
3. [Float](#3-float)
4. [Atom](#4-atom)
5. [Boolean](#5-boolean)
6. [Bit String และ Binary](#6-bit-string-และ-binary)
7. [Reference](#7-reference)
8. [Fun (Function)](#8-fun-function)
9. [Port](#9-port)
10. [PID (Process Identifier)](#10-pid-process-identifier)
11. [Tuple](#11-tuple)
12. [List](#12-list)
13. [Map](#13-map)
14. [String](#14-string)
15. [Variables และ Binding](#15-variables-และ-binding)
16. [Operators](#16-operators)
17. [Comments](#17-comments)
18. [แบบฝึกหัด](#18-แบบฝึกหัด)

---

## 1. ประเภทข้อมูลใน Erlang

```
Erlang Data Types (Terms)
═════════════════════════

Primitive Types:
├── Integer       : 42, -7, 16#FF, 2#1010
├── Float         : 3.14, -0.5, 1.0e10  
├── Atom          : ok, error, hello, 'Hello World'
├── Boolean       : true, false (are atoms!)
├── Bit String    : <<0,1,2>>, <<"hello">>
├── Reference     : make_ref() 
├── Fun           : fun() -> ok end
├── Port          : port identifiers
└── PID           : <0.1.0>

Compound Types:
├── Tuple         : {1, 2, 3}, {ok, "value"}
├── List          : [1, 2, 3], [H|T]
└── Map           : #{key => value}

Special:
└── String        : "hello" (list of integers)
```

### ตรวจสอบ Type

```erlang
% ใช้ is_TYPE/1 functions
1> is_integer(42).
true
2> is_float(3.14).
true
3> is_atom(hello).
true
4> is_list([1,2,3]).
true
5> is_tuple({1,2}).
true
6> is_map(#{a => 1}).
true

% typecheck ด้วย typeof (custom)
7> typeof(X) ->
     if
       is_integer(X) -> integer;
       is_float(X)   -> float;
       is_atom(X)    -> atom;
       is_list(X)    -> list;
       is_tuple(X)   -> tuple;
       is_map(X)     -> map;
       true          -> unknown
     end.
```

---

## 2. Integer

### Integer ใน Erlang ไม่มีขอบเขต (Arbitrary Precision)

```erlang
% Integer ทั่วไป
1> 42.
42
2> -100.
-100
3> 0.
0

% Integer ขนาดใหญ่มาก (bignum)
4> 100 * 100 * 100 * 100 * 100 * 100 * 100 * 100 * 100 * 100.
100000000000000000000

% Factorial ของเลขใหญ่
5> F = fun(N) -> lists:foldl(fun(X, Acc) -> X * Acc end, 1, lists:seq(1, N)) end.
6> F(100).
93326215443944152681699238856266700490715968264381621468592963895217599993229915608941463976156518286253697920827223758251185210916864000000000000000000000000
```

### การเขียนตัวเลขในรูปแบบต่างๆ

```erlang
% Decimal (ฐาน 10)
1> 255.
255

% Hexadecimal (ฐาน 16)
2> 16#FF.
255
3> 16#ff.
255
4> 16#DEADBEEF.
3735928559

% Binary (ฐาน 2)
5> 2#11111111.
255
6> 2#10101010.
170

% Octal (ฐาน 8)
7> 8#377.
255

% Base N ทั่วไป
8> 36#ZZ.    % ฐาน 36
1295

% Character code
9> $A.       % ASCII code ของ 'A'
65
10> $a.
97
11> $0.
48
12> $\n.     % newline
10
13> $\t.     % tab
9
```

### Integer Operations

```erlang
% Arithmetic
1> 10 + 3.    % 13
2> 10 - 3.    % 7
3> 10 * 3.    % 30
4> 10 / 3.    % 3.3333... (float division!)
5> 10 div 3.  % 3 (integer division)
6> 10 rem 3.  % 1 (remainder)
7> -10 rem 3. % -1 (sign follows dividend)
8> abs(-5).   % 5

% Bitwise operations
9>  6 band 3.   % AND: 6=110, 3=011 -> 010 = 2
10> 6 bor 3.    % OR:  6=110, 3=011 -> 111 = 7  
11> 6 bxor 3.   % XOR: 6=110, 3=011 -> 101 = 5
12> bnot 6.     % NOT: depends on bit size
13> 6 bsl 2.    % Shift left: 6*4 = 24
14> 6 bsr 1.    % Shift right: 6/2 = 3

% Conversion
15> integer_to_list(42).   % "42"
16> list_to_integer("42"). % 42
17> integer_to_binary(42). % <<"42">>
```

---

## 3. Float

```erlang
% Float literals
1> 3.14.
3.14
2> -2.5.
-2.5
3> 1.0e10.
1.0e10
4> 1.5e-3.
1.5e-3

% Float ต้องมี digit ก่อนและหลัง decimal point
% ถูก:  3.14, 0.5, 1.0
% ผิด:  .5, 3.

% Float arithmetic
5> 3.14 + 2.0.
5.140000000000001  % floating point imprecision!
6> 0.1 + 0.2.
0.30000000000000004
```

### Float Functions

```erlang
% Math module
1> math:pi().
3.141592653589793
2> math:sqrt(16.0).
4.0
3> math:pow(2.0, 10.0).
1024.0
4> math:sin(math:pi() / 2).
1.0
5> math:cos(0.0).
1.0
6> math:log(math:exp(1.0)).
1.0
7> math:log2(8.0).
3.0
8> math:log10(1000.0).
3.0000000000000004

% Erlang built-in
9> trunc(3.9).    % truncate to integer: 3
10> round(3.5).   % round: 4
11> round(3.4).   % round: 3
12> floor(3.9).   % floor: 3
13> ceil(3.1).    % ceiling: 4
14> abs(-3.14).   % absolute: 3.14

% Conversion
15> float(42).     % integer to float: 42.0
16> float_to_list(3.14).   % "3.14000000000000012434..."
17> float_to_list(3.14, [{decimals, 2}]).  % "3.14"
18> list_to_float("3.14").  % 3.14
```

### Comparing Floats (ระวัง!)

```erlang
% อย่าเปรียบเทียบ float ตรงๆ
1> 0.1 + 0.2 == 0.3.
false  % !!!

% ใช้ epsilon comparison แทน
2> abs((0.1 + 0.2) - 0.3) < 1.0e-10.
true

% ฟังก์ชัน helper
float_equal(A, B) ->
    float_equal(A, B, 1.0e-10).

float_equal(A, B, Epsilon) ->
    abs(A - B) < Epsilon.
```

---

## 4. Atom

Atom เป็น constant ที่มีชื่อเป็น identifier — ใช้สำหรับแทนค่าที่ไม่เปลี่ยนแปลง

```erlang
% Atom ทั่วไป (ขึ้นต้นด้วยตัวพิมพ์เล็ก)
1> hello.
hello
2> world.
world
3> ok.
ok
4> error.
error
5> true.
true
6> false.
false

% Atom ที่มีตัวพิมพ์ใหญ่หรืออักขระพิเศษ (ใช้ single quotes)
7> 'Hello'.
'Hello'
8> 'Hello World'.
'Hello World'
9> 'hello-world'.
'hello-world'
10> 'it\'s ok'.
'it\'s ok'

% Atom ไม่จำเป็นต้องประกาศ — สร้างขึ้นตอน compile
11> atom_to_list(hello).
"hello"
12> list_to_atom("hello").
hello
13> atom_to_binary(hello).
<<"hello">>
14> binary_to_atom(<<"hello">>).
hello
```

### Atom Table

```erlang
% Atom จะถูกเก็บใน global atom table
% ขนาด default: 1,048,576 atoms
% ระวัง: ถ้า atom table เต็ย จะ crash!

% อย่าสร้าง atom แบบ dynamic จาก user input!
% BAD:
bad_example(UserInput) ->
    Atom = list_to_atom(UserInput),  % อันตราย!
    Atom.

% GOOD: ใช้ existing atom หรือ binary แทน
good_example(UserInput) ->
    Binary = list_to_binary(UserInput),  % safe
    Binary.

% ดู atom table size
15> erlang:system_info(atom_count).
12847
16> erlang:system_info(atom_limit).
1048576
```

### การใช้ Atom ในทางปฏิบัติ

```erlang
% Atom เป็น return values มาตรฐาน
process_result() ->
    case do_something() of
        {ok, Value}    -> Value;
        {error, Reason} -> handle_error(Reason)
    end.

% Status codes
handle_http_status(200) -> ok;
handle_http_status(404) -> not_found;
handle_http_status(500) -> server_error;
handle_http_status(_)   -> unknown.

% Module names เป็น atoms
lists:sort([3,1,2]).
io:format("Hello~n").
erlang:now().

% Function names เป็น atoms
Fun = fun lists:sort/1.
```

---

## 5. Boolean

Boolean ใน Erlang คือ Atom `true` และ `false`

```erlang
% Boolean values
1> true.
true
2> false.
false

% Boolean เป็น atoms!
3> is_atom(true).
true
4> is_boolean(true).
true

% Boolean operations
5> true and false.
false
6> true or false.
true
7> not true.
false
8> true xor false.
true

% Short-circuit operators
9> true andalso (1/0 > 0).   % ไม่ evaluate ส่วนขวาถ้าซ้ายเป็น false
false
10> false orelse (1/0 > 0).   % ไม่ evaluate ส่วนขวาถ้าซ้ายเป็น true
false

% Comparison (ได้ boolean)
11> 1 < 2.
true
12> 2 == 2.
true
13> 2 =/= 3.
true
14> "abc" == "abc".
true
```

### Difference: `and` vs `andalso`

```erlang
% 'and' evaluates both sides (strict)
true and false.   % ok
% false and (1/0 > 0).  % ERROR! evaluates 1/0

% 'andalso' is short-circuit
false andalso (1/0 > 0).  % ok! ไม่ evaluate ส่วนขวา

% ใช้ andalso/orelse ในทางปฏิบัติเสมอ
check_user(User) when is_record(User, user) andalso User#user.active ->
    process(User);
check_user(_) ->
    {error, invalid_user}.
```

---

## 6. Bit String และ Binary

Binary เป็นประเภทข้อมูลที่สำคัญมากใน Erlang สำหรับการทำงานกับ raw data

```erlang
% Binary literals
1> <<"hello">>.
<<"hello">>
2> <<1, 2, 3>>.
<<1,2,3>>
3> <<255, 0, 255>>.
<<255,0,255>>

% Binary string
4> <<"Hello, World!">>.
<<"Hello, World!">>

% Bit string (ไม่ต้องเป็น byte boundary)
5> <<1:1, 0:1, 1:1>>.   % 3 bits: 101
<<5:3>>

% Binary ใช้ memory น้อยกว่า List of chars
6> byte_size(<<"hello">>).
5
7> length("hello").
5
% แต่ binary ใช้ 5 bytes vs list ใช้ ~40 bytes

% Pattern matching กับ Binary
8> <<A, B, Rest/binary>> = <<"hello">>.
<<"hello">>
9> A.
104   % 'h'
10> B.
101   % 'e'
11> Rest.
<<"llo">>
```

### Binary Syntax

```erlang
% Syntax: <<Value:Size/Type-Specifiers>>

% Type specifiers:
% integer  - default
% float
% binary (or bytes)
% bitstring (or bits)
% utf8, utf16, utf32

% Size specifiers:
% N bits สำหรับ integer และ float
% N bytes สำหรับ binary

% Signed/Unsigned:
% signed (default สำหรับ integer)
% unsigned

% Endianness:
% big (default)
% little
% native

% ตัวอย่าง
1> <<16#DEADBEEF:32>>.
<<222,173,190,239>>

2> <<16#DEADBEEF:32/little>>.
<<239,190,173,222>>

3> <<3.14:64/float>>.
<<64,9,30,184,81,235,133,31>>

4> <<255:8/unsigned>>.
<<255>>

5> <<-1:8/signed>>.
<<255>>

% Building binary
6> Name = <<"Alice">>.
7> Age = 30.
8> <<Name/binary, " is ", (integer_to_binary(Age))/binary, " years old">>.
<<"Alice is 30 years old">>
```

### Binary Operations

```erlang
% ตรวจสอบ
1> is_binary(<<"hello">>).
true
2> byte_size(<<"hello">>).
5
3> bit_size(<<"hello">>).
40

% Concatenation
4> <<"hello">> ++ <<" world">>.  % ไม่ work! ++ สำหรับ list
5> <<<<"hello">>/binary, <<" world">>/binary>>.
<<"hello world">>
% หรือ
5> binary:list_to_bin([<<"hello">>, <<" ">>, <<"world">>]).
<<"hello world">>

% Splitting
6> binary:split(<<"hello world foo">>, <<" ">>).
[<<"hello">>,<<"world foo">>]
7> binary:split(<<"hello world foo">>, <<" ">>, [global]).
[<<"hello">>,<<"world">>,<<"foo">>]

% Finding
8> binary:match(<<"hello world">>, <<"world">>).
{6,5}
9> binary:matches(<<"hello world hello">>, <<"hello">>).
[{0,5},{12,5}]

% Replace
10> binary:replace(<<"hello world">>, <<"world">>, <<"erlang">>).
<<"hello erlang">>

% Conversion
11> binary_to_list(<<"hello">>).
"hello"
12> list_to_binary("hello").
<<"hello">>
13> binary_to_integer(<<"42">>).
42
14> integer_to_binary(42).
<<"42">>
```

---

## 7. Reference

Reference เป็นค่าที่ unique ทั่วทั้ง Erlang runtime

```erlang
% สร้าง Reference
1> Ref = make_ref().
#Ref<0.1234567890.1234567890.1234567890>

2> make_ref() == make_ref().
false  % ทุก ref ต่างกันเสมอ!

% ใช้งานจริง: correlation IDs
handle_request(Request) ->
    Ref = make_ref(),
    Pid ! {self(), Ref, process, Request},
    receive
        {Ref, Response} ->  % match กับ Ref เดิม
            {ok, Response}
    after 5000 ->
        {error, timeout}
    end.
```

---

## 8. Fun (Function)

Fun เป็น first-class function ใน Erlang (lambda/anonymous function)

```erlang
% Anonymous function (lambda)
1> Double = fun(X) -> X * 2 end.
#Fun<erl_eval.42.1...>

2> Double(5).
10

% Multi-clause fun
3> Classify = fun
     (X) when X > 0 -> positive;
     (X) when X < 0 -> negative;
     (0)            -> zero
   end.
4> Classify(5).
positive
5> Classify(-3).
negative
6> Classify(0).
zero

% Fun อ้างถึงจาก module
7> Sort = fun lists:sort/1.
8> Sort([3,1,2]).
[1,2,3]

% Fun ที่ capture ตัวแปรจาก outer scope (closure)
9> Multiplier = fun(Factor) ->
     fun(X) -> X * Factor end
   end.
10> Times3 = Multiplier(3).
11> Times3(7).
21

% Higher-order functions
12> lists:map(fun(X) -> X * X end, [1,2,3,4,5]).
[1,4,9,16,25]

13> lists:filter(fun(X) -> X > 3 end, [1,2,3,4,5]).
[4,5]

14> lists:foldl(fun(X, Acc) -> X + Acc end, 0, [1,2,3,4,5]).
15
```

---

## 9. Port

Port ใช้สำหรับสื่อสารกับ external programs

```erlang
% เปิด port ไปยัง external command
1> Port = open_port({spawn, "cat"}, [binary]).
#Port<0.1>

2> port_command(Port, <<"hello\n">>).
true

3> receive
     {Port, {data, Data}} -> Data
   end.
<<"hello\n">>
```

---

## 10. PID (Process Identifier)

PID คือ identifier ของ Erlang process

```erlang
% ดู PID ของ process ปัจจุบัน
1> self().
<0.85.0>

% Format ของ PID: <Node.Process.Serial>
% <0.85.0>  = Node 0, Process 85, Serial 0

% PID จาก node อื่น
% <2.100.0>  = Node 2, Process 100, Serial 0

% spawn process และได้ PID
2> Pid = spawn(fun() -> io:format("Hello from ~p~n", [self()]) end).
Hello from <0.88.0>
<0.88.0>

% ส่ง message ไปยัง PID
3> Pid ! hello.
hello

% ตรวจสอบ
4> is_pid(Pid).
true
5> is_process_alive(Pid).
false  % process จบแล้ว

6> pid_to_list(self()).
"<0.85.0>"
```

---

## 11. Tuple

Tuple คือ container ขนาดคงที่ที่มีหลาย element

```erlang
% สร้าง Tuple
1> {1, 2, 3}.
{1,2,3}

2> {ok, "success"}.
{ok,"success"}

3> {error, not_found, 404}.
{error,not_found,404}

% Tuple ขนาดต่างๆ
4> {}.           % empty tuple
{}
5> {single}.     % 1 element
{single}
6> {a, b, c, d, e}.  % 5 elements

% Mixed types
7> {42, 3.14, hello, "world", [1,2,3]}.
{42,3.14,hello,"world",[1,2,3]}

% Nested tuples
8> {{1, 2}, {3, 4}}.
{{1,2},{3,4}}
```

### Tuple Operations

```erlang
% ขนาด tuple
1> tuple_size({1, 2, 3}).
3

% เข้าถึง element (index เริ่มจาก 1!)
2> element(1, {a, b, c}).
a
3> element(2, {a, b, c}).
b
4> element(3, {a, b, c}).
c

% แก้ไข element (ได้ tuple ใหม่)
5> setelement(2, {a, b, c}, x).
{a,x,c}

% แปลง
6> tuple_to_list({1, 2, 3}).
[1,2,3]
7> list_to_tuple([1, 2, 3]).
{1,2,3}

% Pattern matching (วิธีหลักในการใช้ tuple)
8> {X, Y} = {10, 20}.
{10,20}
9> X.
10
10> Y.
20

% Tagged tuples (idiom มาตรฐาน)
11> {ok, Value} = {ok, 42}.
12> Value.
42

13> {error, Reason} = {error, not_found}.
14> Reason.
not_found
```

### Tuple Patterns ที่นิยมใช้

```erlang
% Result tuple pattern
process(Input) ->
    case validate(Input) of
        {ok, Valid}     -> {ok, transform(Valid)};
        {error, Reason} -> {error, Reason}
    end.

% Key-value pairs
Point = {x, 10}.
Color = {rgb, 255, 0, 0}.
Date  = {date, 2024, 1, 15}.

% 2-tuple (pair)
Pair = {key, value}.

% 3-tuple (triple)
Triple = {name, age, city}.
```

---

## 12. List

List เป็นประเภทข้อมูลหลักใน Erlang

```erlang
% สร้าง List
1> [1, 2, 3].
[1,2,3]

2> [a, b, c].
[a,b,c]

3> [].          % empty list
[]

% Mixed types
4> [1, hello, 3.14, "world"].
[1,hello,3.14,"world"]

% Nested lists
5> [[1,2], [3,4], [5,6]].
[[1,2],[3,4],[5,6]]

% Head | Tail syntax
6> [H|T] = [1, 2, 3, 4, 5].
7> H.
1
8> T.
[2,3,4,5]

% สร้าง list ด้วย |
9> [0 | [1, 2, 3]].
[0,1,2,3]

10> [a | [b | [c | []]]].
[a,b,c]
```

### List Operations

```erlang
% ความยาว
1> length([1, 2, 3]).
3

% รวม list
2> [1, 2] ++ [3, 4].
[1,2,3,4]

% ลบ elements
3> [1, 2, 3, 2, 1] -- [2, 1].
[3,2,1]  % ลบครั้งละ 1

% Head และ Tail
4> hd([1, 2, 3]).
1
5> tl([1, 2, 3]).
[2,3]

% Reverse
6> lists:reverse([1, 2, 3]).
[3,2,1]

% Sort
7> lists:sort([3, 1, 4, 1, 5, 9]).
[1,1,3,4,5,9]

% Map
8> lists:map(fun(X) -> X * 2 end, [1,2,3]).
[2,4,6]

% Filter
9> lists:filter(fun(X) -> X > 2 end, [1,2,3,4]).
[3,4]

% Fold
10> lists:foldl(fun(X, Acc) -> X + Acc end, 0, [1,2,3,4,5]).
15

% Flatten
11> lists:flatten([[1,2],[3,[4,5]]]).
[1,2,3,4,5]

% Member
12> lists:member(3, [1,2,3,4]).
true

% nth element
13> lists:nth(2, [a,b,c,d]).
b

% Last
14> lists:last([1,2,3]).
3
```

### List Comprehension

```erlang
% [Expression || Pattern <- List, Guard]

% ตัวอย่างพื้นฐาน
1> [X * 2 || X <- [1, 2, 3, 4, 5]].
[2,4,6,8,10]

% พร้อม filter
2> [X * 2 || X <- [1, 2, 3, 4, 5], X > 2].
[6,8,10]

% หลาย generators
3> [{X, Y} || X <- [1,2,3], Y <- [a,b]].
[{1,a},{1,b},{2,a},{2,b},{3,a},{3,b}]

% Pythagorean triples
4> [{A,B,C} || 
     C <- lists:seq(1, 20),
     B <- lists:seq(1, C),
     A <- lists:seq(1, B),
     A*A + B*B =:= C*C].
[{3,4,5},{6,8,10},{5,12,13},{8,15,17},{9,12,15}]

% Flatten nested lists
5> Nested = [[1,2,3],[4,5,6],[7,8,9]].
6> [X || Inner <- Nested, X <- Inner].
[1,2,3,4,5,6,7,8,9]
```

---

## 13. Map

Map เป็น key-value store ที่เพิ่มเข้ามาใน Erlang 17

```erlang
% สร้าง Map
1> #{}.                        % empty map
#{}
2> #{name => "Alice"}.
#{name => "Alice"}
3> #{name => "Alice", age => 30}.
#{age => 30, name => "Alice"}  % key เรียงตาม order

% Key ชนิดใดก็ได้
4> #{1 => one, 2 => two}.
#{1 => one, 2 => two}

5> #{<<"key">> => <<"value">>}.
#{<<"key">> => <<"value">>}

% Nested maps
6> #{user => #{name => "Bob", age => 25}}.
#{user => #{age => 25, name => "Bob"}}
```

### Map Operations

```erlang
% เข้าถึงค่า
1> M = #{name => "Alice", age => 30}.
2> maps:get(name, M).
"Alice"
3> maps:get(age, M).
30
4> maps:get(missing, M, "default").  % กับ default value
"default"

% Map:Key syntax (pattern matching)
5> #{name := Name} = M.
6> Name.
"Alice"

% Update map (ได้ map ใหม่)
7> M#{age => 31}.
#{age => 31, name => "Alice"}

% เพิ่ม key ใหม่
8> M#{email => "alice@example.com"}.
#{age => 30, email => "alice@example.com", name => "Alice"}

% ตรวจสอบ key
9> maps:is_key(name, M).
true
10> maps:is_key(email, M).
false

% ลบ key
11> maps:remove(age, M).
#{name => "Alice"}

% ขนาด
12> maps:size(M).
2

% Keys และ Values
13> maps:keys(M).
[age, name]  % sorted
14> maps:values(M).
[30, "Alice"]

% Convert
15> maps:to_list(M).
[{age,30},{name,"Alice"}]
16> maps:from_list([{a, 1}, {b, 2}]).
#{a => 1, b => 2}

% Map operations
17> maps:merge(#{a => 1}, #{b => 2}).
#{a => 1, b => 2}

18> maps:map(fun(_, V) -> V * 2 end, #{a => 1, b => 2}).
#{a => 2, b => 4}

19> maps:filter(fun(_, V) -> V > 1 end, #{a => 1, b => 2, c => 3}).
#{b => 2, c => 3}

20> maps:fold(fun(_, V, Acc) -> Acc + V end, 0, #{a => 1, b => 2, c => 3}).
6
```

---

## 14. String

String ใน Erlang คือ list of integers (Unicode code points)

```erlang
% String เป็น list!
1> "hello".
"hello"
2> [104, 101, 108, 108, 111].
"hello"  % Erlang แสดงเป็น string เพราะ printable chars

3> is_list("hello").
true
4> length("hello").
5

% เข้าถึง character
5> hd("hello").
104   % 'h' = 104

% String operations ใช้ lists module!
6> lists:reverse("hello").
"olleh"

% String concatenation
7> "hello" ++ " " ++ "world".
"hello world"

% String comparison
8> "abc" == "abc".
true
9> "abc" < "abd".
true  % lexicographic comparison

% String functions
10> string:to_upper("hello").
"HELLO"
11> string:to_lower("HELLO").
"hello"
12> string:len("hello").
5
13> string:substr("hello world", 7).
"world"
14> string:tokens("a,b,c", ",").
["a","b","c"]
15> string:join(["a","b","c"], ",").
"a,b,c"
```

### String vs Binary

```erlang
% String (list) ใช้ memory มาก
% "hello" = [104, 101, 108, 108, 111]
% แต่ละ character ใช้ ~8 bytes (on 64-bit) = 40 bytes!

% Binary ใช้ memory น้อยกว่า
% <<"hello">> = 5 bytes!

% ใน production ใช้ binary สำหรับ string manipulation

% แปลงระหว่างกัน
1> list_to_binary("hello").
<<"hello">>
2> binary_to_list(<<"hello">>).
"hello"

% atom_to_list/list_to_atom
3> atom_to_list(hello).
"hello"
4> list_to_atom("hello").
hello
```

### Unicode

```erlang
% Unicode support
1> "สวัสดี".
[3626,3623,3633,3626,3604,3637]  % Unicode code points

2> unicode:characters_to_binary("สวัสดี").
<<224,184,170,224,184,167,224,184,177,224,184,170,224,184,148,224,184,181>>

3> unicode:characters_to_list(<<"hello">>).
"hello"

% io:format กับ Unicode
4> io:format("~ts~n", ["สวัสดี"]).
สวัสดี
ok

% String module รองรับ Unicode
5> string:length("สวัสดี").
6  % 6 grapheme clusters
6> string:to_upper("café").
"CAFÉ"
```

---

## 15. Variables และ Binding

### Single Assignment (Immutability)

```erlang
% ตัวแปรใน Erlang ขึ้นต้นด้วยตัวพิมพ์ใหญ่
1> X = 42.
42

% ไม่สามารถ reassign ได้!
2> X = 100.
** exception error: no match of right hand side value 100

% X ถูก "bind" ค่าแล้ว ไม่สามารถเปลี่ยนได้
% นี่คือ immutability ของ Erlang

% ใน function ใหม่หรือ scope ใหม่ ตัวแปรชื่อเดิมได้
my_function() ->
    X = 1,
    Y = 2,
    Z = X + Y,
    Z.

% ตัวแปร _ (underscore) = ไม่ใช้ค่า
ignore_result() ->
    _ = some_function(),
    ok.

% _Prefix = แสดงว่าอาจไม่ใช้ แต่ไม่ warning
maybe_use(X, _Unused) ->
    X.
```

### Pattern Matching ในการ Assign

```erlang
% Pattern matching คือหัวใจของ Erlang
1> {A, B} = {1, 2}.
{1,2}
2> A.
1
3> B.
2

% Match กับ List
4> [H|T] = [1,2,3,4,5].
5> H.
1
6> T.
[2,3,4,5]

% Nested pattern
7> {X, [Y|_]} = {hello, [1,2,3]}.
8> X.
hello
9> Y.
1

% Match กับ specific values
10> {ok, Value} = {ok, 42}.
11> Value.
42

% ถ้า pattern ไม่ match จะ error!
12> {ok, V} = {error, something}.
** exception error: no match of right hand side value {error,something}
```

---

## 16. Operators

### Arithmetic Operators

```erlang
% Integer arithmetic
+    % addition
-    % subtraction
*    % multiplication
div  % integer division
rem  % remainder

% Float arithmetic
/    % division (เสมอ float)

% Examples
1> 10 + 3.     % 13
2> 10 - 3.     % 7
3> 10 * 3.     % 30
4> 10 / 3.     % 3.3333...
5> 10 div 3.   % 3
6> 10 rem 3.   % 1
7> -(-5).      % 5 (unary minus)
```

### Comparison Operators

```erlang
==    % equal (เปรียบค่า: 1 == 1.0 -> true)
/=    % not equal
=:=   % exactly equal (เปรียบ type+value: 1 =:= 1.0 -> false)
=/=   % not exactly equal
<     % less than
>     % greater than
<=    % less than or equal
>=    % greater than or equal

% Examples
1> 1 == 1.0.    % true (ค่าเท่ากัน แม้ต่าง type)
2> 1 =:= 1.0.  % false (type ต่างกัน)
3> 1 /= 2.     % true
4> 1 =/= 1.0.  % true

% Erlang term ordering (สำคัญ!)
% number < atom < reference < fun < port < pid < tuple < map < list < bitstring
5> 1 < a.      % true (number < atom)
6> a < [].     % true (atom < list)
7> {} < [].    % true (tuple < list)
```

### Boolean Operators

```erlang
not    % logical NOT
and    % logical AND (strict)
or     % logical OR (strict)
xor    % logical XOR
andalso  % short-circuit AND
orelse   % short-circuit OR

% Examples
1> not true.         % false
2> true and false.   % false
3> true or false.    % true
4> true xor true.    % false
5> true andalso false.  % false (short-circuit)
6> false orelse true.   % true (short-circuit)
```

### Bitwise Operators

```erlang
band   % bitwise AND
bor    % bitwise OR
bxor   % bitwise XOR
bnot   % bitwise NOT
bsl    % bit shift left
bsr    % bit shift right

% Examples
1> 6 band 3.    % 2  (110 AND 011 = 010)
2> 6 bor 3.     % 7  (110 OR  011 = 111)
3> 6 bxor 3.    % 5  (110 XOR 011 = 101)
4> 6 bsl 1.     % 12 (1100)
5> 6 bsr 1.     % 3  (011)
```

### List Operators

```erlang
++   % concatenation
--   % subtraction

1> [1,2] ++ [3,4].     % [1,2,3,4]
2> [1,2,3] -- [2].     % [1,3]
3> [1,2,2,3] -- [2,3]. % [1,2]  (ลบครั้งละ 1 ตัว)
```

### Match Operator

```erlang
=    % pattern match / bind

1> X = 5.    % bind X to 5
2> 5 = 5.    % match succeeds
3> 5 = 6.    % ** exception error: no match
```

---

## 17. Comments

```erlang
% Comment เริ่มด้วย % และไปถึงสิ้นบรรทัด

%% สองเปอร์เซ็นต์นิยมใช้สำหรับ module-level comments
%% และ section comments

%%% สามเปอร์เซ็นต์สำหรับ documentation

% ไม่มี multi-line comments ใน Erlang!
% ถ้าต้องการ multi-line ต้องใส่ % ทุกบรรทัด

%% ------------------------------------------------------------------
%% Function: process/1
%% Purpose : Process the input and return result
%% Args    : Input - the data to process
%% Returns : {ok, Result} | {error, Reason}
%% ------------------------------------------------------------------
process(Input) ->
    % TODO: implement this
    {ok, Input}.
```

---

## 18. แบบฝึกหัด

### Exercise 1: Type Exploration

```erlang
%% สร้างไฟล์ type_explorer.erl
%% ฟังก์ชัน type_of/1 ที่คืน atom บอกประเภทของ input

type_of(X) when is_integer(X)  -> integer;
type_of(X) when is_float(X)    -> float;
type_of(X) when is_atom(X)     -> atom;
type_of(X) when is_binary(X)   -> binary;
type_of(X) when is_list(X)     -> list;
type_of(X) when is_tuple(X)    -> tuple;
type_of(X) when is_map(X)      -> map;
type_of(X) when is_pid(X)      -> pid;
type_of(X) when is_function(X) -> function;
type_of(_)                     -> unknown.
```

### Exercise 2: Data Conversion

```erlang
%% สร้างไฟล์ converter.erl
%% ฟังก์ชัน:
%% - to_string/1 : แปลงทุก type เป็น string
%% - to_binary/1 : แปลงทุก type เป็น binary
%% - parse_int/1 : parse string/binary เป็น integer safely

to_string(X) when is_integer(X) -> integer_to_list(X);
to_string(X) when is_float(X)   -> float_to_list(X, [{decimals, 6}]);
to_string(X) when is_atom(X)    -> atom_to_list(X);
to_string(X) when is_binary(X)  -> binary_to_list(X);
to_string(X) when is_list(X)    -> X.  % assume it's already a string

to_binary(X) when is_integer(X) -> integer_to_binary(X);
to_binary(X) when is_float(X)   -> float_to_binary(X, [{decimals, 6}]);
to_binary(X) when is_atom(X)    -> atom_to_binary(X, utf8);
to_binary(X) when is_list(X)    -> list_to_binary(X);
to_binary(X) when is_binary(X)  -> X.

parse_int(X) when is_binary(X) ->
    parse_int(binary_to_list(X));
parse_int(X) when is_list(X) ->
    try
        {ok, list_to_integer(X)}
    catch
        _:_ -> {error, invalid_integer}
    end;
parse_int(X) when is_integer(X) ->
    {ok, X}.
```

### Exercise 3: Calculator ขั้นสูง

```erlang
%% สร้าง calculator.erl ที่รองรับ:
%% calc("2 + 3")   -> 5
%% calc("10 / 2")  -> 5.0
%% calc("2 ^ 10")  -> 1024
%% calc("10 mod 3") -> 1

-module(calculator).
-export([calc/1]).

calc(Expr) when is_list(Expr) ->
    Tokens = string:tokens(Expr, " "),
    case Tokens of
        [A, "+", B]   -> to_num(A) + to_num(B);
        [A, "-", B]   -> to_num(A) - to_num(B);
        [A, "*", B]   -> to_num(A) * to_num(B);
        [A, "/", B]   -> to_num(A) / to_num(B);
        [A, "^", B]   -> math:pow(to_num(A), to_num(B));
        [A, "mod", B] -> to_num(A) rem trunc(to_num(B));
        _             -> {error, invalid_expression}
    end.

to_num(S) ->
    try list_to_integer(S)
    catch _:_ ->
        list_to_float(S)
    end.
```

---

## สรุป Part 02

ใน Part นี้คุณได้เรียนรู้:

✅ ประเภทข้อมูลทั้งหมดใน Erlang  
✅ Integer — arbitrary precision, หลายฐาน  
✅ Float — IEEE 754, math module  
✅ Atom — immutable constants  
✅ Boolean — เป็น atoms  
✅ Bit String และ Binary — efficient data  
✅ Reference — unique identifiers  
✅ Fun — first-class functions  
✅ Tuple — fixed-size containers  
✅ List — variable-size sequences  
✅ Map — key-value stores  
✅ String — list of integers  
✅ Variables — single assignment  
✅ Operators — arithmetic, comparison, boolean, bitwise  

---

## ต่อไป: [Part 03 — Pattern Matching](../part03/README.md)

ใน Part ถัดไปเราจะเรียนรู้ Pattern Matching อย่างลึกซึ้ง:
- Pattern matching พื้นฐานถึงขั้นสูง
- Guards
- Multiple clause functions
- Destructuring
- การใช้ pattern matching ใน lists, tuples, maps, binaries

---

*Part 02/100 | [← ก่อนหน้า](../part01/README.md) | [ถัดไป →](../part03/README.md)*
