# Part 08: String และ Binary

> **"In Erlang, strings are lists and binaries are bytes — know when to use each"**  
> ใน Erlang, strings คือ lists และ binaries คือ bytes — รู้ว่าเมื่อไหรควรใช้อะไร

---

## สารบัญ

1. [String ใน Erlang](#1-string-ใน-erlang)
2. [string Module](#2-string-module)
3. [Binary String Operations](#3-binary-string-operations)
4. [Regular Expressions](#4-regular-expressions)
5. [Unicode Handling](#5-unicode-handling)
6. [iolist](#6-iolist)
7. [Formatting กับ io:format](#7-formatting-กับ-ioformat)
8. [Binary Protocols](#8-binary-protocols)
9. [Performance Tips](#9-performance-tips)
10. [แบบฝึกหัด](#10-แบบฝึกหัด)

---

## 1. String ใน Erlang

```erlang
%% String ใน Erlang = list ของ Unicode code points (integers)
1> "hello".
"hello"
2> [104, 101, 108, 108, 111].
"hello"  %% Erlang แสดงเป็น string เพราะ printable ASCII

%% Unicode string
3> "สวัสดี".
[3626,3623,3633,3626,3604,3637]

%% Character literals
4> $h.    % 104
5> $A.    % 65
6> $\n.   % 10 (newline)
7> $\t.   % 9 (tab)
8> $\\.   % 92 (backslash)
9> $\".   % 34 (double quote)

%% String operations ใช้ lists functions
10> length("hello").
5
11> hd("hello").
104  % 'h'
12> tl("hello").
"ello"
13> "hello" ++ " world".
"hello world"
14> lists:reverse("hello").
"olleh"
```

### String เป็น List ของ Integers

```erlang
%% สิ่งที่ดูเหมือน string จริงๆ คือ list
[72, 101, 108, 108, 111] = "Hello"   % true!
[104, 101, 108, 108, 111] = "hello"  % true!

%% แปลง character เป็น code point
char_code(C) when is_integer(C) -> C.  % C คือ code point แล้ว!

%% แปลง code point เป็น string
code_to_string(Code) -> [Code].

%% ตรวจสอบว่า printable?
is_printable(Str) ->
    lists:all(fun(C) -> C >= 32 andalso C < 127 end, Str).
```

---

## 2. string Module

Erlang OTP มี `string` module ที่ทรงพลัง (รองรับ Unicode ด้วย)

### Basic Operations

```erlang
%% string:length/1 (Unicode aware)
1> string:length("hello").
5
2> string:length("สวัสดี").
6  % 6 grapheme clusters

%% string:substr/2,3 (deprecated, ใช้ slice แทน)
3> string:substr("hello world", 7).
"world"
4> string:substr("hello world", 1, 5).
"hello"

%% string:slice/3 (Unicode safe)
5> string:slice("hello world", 6).
"world"
6> string:slice("hello world", 0, 5).
"hello"

%% string:nth_lexeme/2
7> string:nth_lexeme("hello world foo", 2, " ").
"world"
```

### Case Conversion

```erlang
%% string:to_upper/1 กับ string:to_lower/1
1> string:to_upper("hello world").
"HELLO WORLD"
2> string:to_lower("HELLO WORLD").
"hello world"

%% Unicode case conversion
3> string:to_upper("café").
"CAFÉ"
4> string:to_lower("ΑΒΓΔ").
"αβγδ"

%% string:titlecase/1 (OTP 20+)
5> string:titlecase("hello world").
"Hello world"  % capitalize first char only!
```

### Trimming

```erlang
%% string:trim/1,2,3
1> string:trim("  hello  ").
"hello"
2> string:trim("  hello  ", leading).
"hello  "
3> string:trim("  hello  ", trailing).
"  hello"
4> string:trim("xxxhelloxxx", both, "x").
"hello"

%% string:strip/1,2,3 (deprecated)
5> string:strip("  hello  ").
"hello"
```

### Splitting

```erlang
%% string:tokens/2 (split กับ delimiter chars)
1> string:tokens("hello world foo", " ").
["hello","world","foo"]
2> string:tokens("a,b,,c", ",").
["a","b","c"]  % ไม่มี empty strings!

%% string:split/2,3
3> string:split("hello world foo", " ").
["hello","world foo"]  % split ที่ first occurrence
4> string:split("hello world foo", " ", all).
["hello","world","foo"]  % split ทั้งหมด
5> string:split("hello world foo", " ", leading).
["hello","world foo"]
6> string:split("hello world foo", " ", trailing).
["hello world","foo"]

%% Binary version
7> binary:split(<<"hello world">>, <<" ">>).
[<<"hello">>,<<"world">>]
8> binary:split(<<"a,b,c">>, <<",">>, [global]).
[<<"a">>,<<"b">>,<<"c">>]
```

### Finding

```erlang
%% string:find/2,3
1> string:find("hello world", "world").
"world"  % returns suffix from match
2> string:find("hello world", "xyz").
nomatch
3> string:find("hello world", "o", trailing).
"orld"  % search from right

%% string:str/2 (position, deprecated)
4> string:str("hello world", "world").
7  % 1-indexed position

%% Prefix/Suffix check
5> string:prefix("hello world", "hello").
" world"  % returns rest if match
6> string:prefix("hello world", "xyz").
nomatch
```

### Joining

```erlang
%% string:join/2
1> string:join(["hello", "world", "foo"], " ").
"hello world foo"
2> string:join(["a", "b", "c"], ",").
"a,b,c"
3> string:join([], ",").
[]
```

### Other String Operations

```erlang
%% string:equal/2,3
1> string:equal("Hello", "hello").
false
2> string:equal("Hello", "hello", true).  % case insensitive
true

%% string:to_integer/1
3> string:to_integer("42 abc").
{42, " abc"}
4> string:to_integer("abc").
{error, no_integer}

%% string:to_float/1
5> string:to_float("3.14 abc").
{3.14, " abc"}

%% string:copy/2
6> string:copies("ab", 3).
"ababab"

%% string:words/1,2
7> string:words("hello world foo").
3

%% string:reverse/1
8> string:reverse("hello").
"olleh"
```

---

## 3. Binary String Operations

Binary เหมาะสำหรับ high-performance string processing

### binary Module

```erlang
%% สร้าง binary
1> <<"hello">>.
<<"hello">>
2> <<"hello", " ", "world">>.
<<"hello world">>

%% binary:copy/1,2
3> binary:copy(<<"ab">>, 3).
<<"ababab">>

%% binary:part/2,3
4> binary:part(<<"hello world">>, 6, 5).
<<"world">>
5> binary:part(<<"hello world">>, {6, 5}).
<<"world">>

%% ขนาด
6> byte_size(<<"hello">>).
5
7> bit_size(<<"hello">>).
40
```

### binary:split/3

```erlang
%% แบ่ง binary
1> binary:split(<<"hello world">>, <<" ">>).
[<<"hello">>,<<"world">>]

2> binary:split(<<"a,b,c">>, <<",">>, [global]).
[<<"a">>,<<"b">>,<<"c">>]

3> binary:split(<<"a,,b">>, <<",">>, [global]).
[<<"a">>,<<>>,<<"b">>]  % รวม empty binaries!

%% กับ trim
4> binary:split(<<"a,,b">>, <<",">>, [global, trim]).
[<<"a">>,<<"b">>]

5> binary:split(<<"a,,b">>, <<",">>, [global, trim_all]).
[<<"a">>,<<"b">>]
```

### binary:replace/4

```erlang
%% แทนที่ใน binary
1> binary:replace(<<"hello world">>, <<"world">>, <<"erlang">>).
<<"hello erlang">>

2> binary:replace(<<"aababba">>, <<"ab">>, <<"X">>, [global]).
<<"aXXba">>

3> binary:replace(<<"hello">>, <<"ll">>, <<"LL">>).
<<"heLLo">>
```

### binary:match/3

```erlang
%% หา pattern ใน binary
1> binary:match(<<"hello world">>, <<"world">>).
{6, 5}  % {Position, Length}

2> binary:match(<<"hello world">>, <<"xyz">>).
nomatch

3> binary:matches(<<"hello world hello">>, <<"hello">>).
[{0,5},{12,5}]

%% กับ list ของ patterns
4> binary:match(<<"hello world">>, [<<"world">>, <<"hello">>]).
{0, 5}  % first match
```

### String ↔ Binary Conversion

```erlang
%% list <-> binary
1> list_to_binary("hello").
<<"hello">>
2> binary_to_list(<<"hello">>).
"hello"

%% integer <-> binary
3> integer_to_binary(42).
<<"42">>
4> binary_to_integer(<<"42">>).
42

%% float <-> binary
5> float_to_binary(3.14).
<<"3.14000000000000012434...e+0">>  % เยอะมาก!
6> float_to_binary(3.14, [{decimals, 2}]).
<<"3.14">>
7> binary_to_float(<<"3.14">>).
3.14

%% atom <-> binary
8> atom_to_binary(hello, utf8).
<<"hello">>
9> binary_to_atom(<<"hello">>, utf8).
hello
10> binary_to_existing_atom(<<"hello">>, utf8).  % safer!
hello
```

---

## 4. Regular Expressions

Erlang มี `re` module สำหรับ PCRE regular expressions

### Basic Usage

```erlang
%% re:run/2,3 - ค้นหา pattern
1> re:run("hello world", "world").
{match, [{6,5}]}

2> re:run("hello world", "xyz").
nomatch

%% ดึงค่าออกมา
3> re:run("hello world", "(\\w+)\\s+(\\w+)", [{capture, all, list}]).
{match,["hello world","hello","world"]}

4> re:run("2024-01-15", "(\\d{4})-(\\d{2})-(\\d{2})",
           [{capture, [1,2,3], list}]).
{match, ["2024","01","15"]}

%% Global match (ค้นหาทั้งหมด)
5> re:run("one two three", "\\w+", [global, {capture, all, list}]).
{match,[["one"],["two"],["three"]]}
```

### Named Captures

```erlang
%% Named capture groups
1> re:run("2024-01-15",
    "(?P<year>\\d{4})-(?P<month>\\d{2})-(?P<day>\\d{2})",
    [{capture, [year, month, day], list}]).
{match, ["2024", "01", "15"]}

%% Parse URL
parse_url(Url) ->
    Pattern = "(?P<scheme>\\w+)://(?P<host>[^/]+)(?P<path>/.*)?",
    case re:run(Url, Pattern, [{capture, [scheme,host,path], list}]) of
        {match, [Scheme, Host, Path]} ->
            #{scheme => Scheme, host => Host, path => Path};
        nomatch ->
            {error, invalid_url}
    end.

2> parse_url("https://example.com/path/to/page").
#{scheme => "https", host => "example.com", path => "/path/to/page"}
```

### re:replace/4

```erlang
%% แทนที่ด้วย regex
1> re:replace("hello world", "o", "0", [global, {return, list}]).
"hell0 w0rld"

2> re:replace("  hello  world  ", "\\s+", " ", [global, {return, binary}]).
<<" hello world ">>

%% Replace กับ references
3> re:replace("hello world", "(\\w+)", "[\\1]", [global, {return, list}]).
"[hello] [world]"
```

### re:split/3

```erlang
%% Split ด้วย regex
1> re:split("hello   world", "\\s+", [{return, list}]).
["hello","world"]

2> re:split("a1b2c3", "\\d", [{return, list}]).
["a","b","c",""]

3> re:split("a1b2c3", "\\d", [{return, list}, trim]).
["a","b","c"]
```

### Compile Pattern

```erlang
%% Compile pattern ก่อนใช้ซ้ำ (performance)
email_pattern() ->
    {ok, MP} = re:compile("^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$"),
    MP.

validate_email(Email) ->
    Pattern = email_pattern(),
    case re:run(Email, Pattern) of
        {match, _} -> valid;
        nomatch    -> invalid
    end.

%% ใช้ application:set_env หรือ module-level ตัวแปรเก็บ compiled pattern
-define(EMAIL_PATTERN, element(2, re:compile("^[^@]+@[^@]+\\.[^@]+$"))).
```

---

## 5. Unicode Handling

### Unicode Basics

```erlang
%% Erlang รองรับ Unicode เต็มรูปแบบ

%% Code points
1> $α.    % Greek alpha = 945
945
2> $ก.    % Thai ko kai = 3585
3585

%% String ที่มี Unicode
3> "สวัสดี".
[3626,3623,3633,3626,3604,3637]

%% แสดง Unicode string
4> io:format("~ts~n", ["สวัสดี"]).  % ใช้ ~ts ไม่ใช่ ~s!
สวัสดี
```

### unicode Module

```erlang
%% แปลง string เป็น UTF-8 binary
1> unicode:characters_to_binary("สวัสดี").
<<224,184,170,224,184,167,224,184,177,224,184,170,224,184,148,224,184,181>>

%% แปลง UTF-8 binary เป็น string
2> unicode:characters_to_list(<<224,184,170,224,184,167,...>>).
[3626,3623,3633,3626,3604,3637]

%% รองรับหลาย encoding
3> unicode:characters_to_binary("Hello", utf8).
<<"Hello">>

4> unicode:characters_to_binary([0x4e2d, 0x6587], utf8).  % Chinese
<<228,184,173,230,150,135>>

%% Character Categories
5> unicode:character_info($A).
{letter, {uppercase, 65}}
```

### UTF-8 Encoding/Decoding

```erlang
%% Encode string -> UTF-8 binary
encode_utf8(String) ->
    unicode:characters_to_binary(String, utf8).

%% Decode UTF-8 binary -> string
decode_utf8(Binary) ->
    case unicode:characters_to_list(Binary, utf8) of
        {error, Valid, _}   -> {error, {invalid_utf8, Valid}};
        {incomplete, _, _}  -> {error, incomplete_utf8};
        String              -> {ok, String}
    end.

%% Validate UTF-8
is_valid_utf8(Binary) ->
    case unicode:characters_to_list(Binary, utf8) of
        List when is_list(List) -> true;
        _ -> false
    end.
```

### String Normalization

```erlang
%% Unicode normalization (NFC, NFD, NFKC, NFKD)

%% "café" สามารถเป็น:
%% NFC: é เป็น 1 code point (U+00E9)
%% NFD: e + ́ เป็น 2 code points (U+0065, U+0301)

normalize(String, Form) ->
    unicode:characters_to_nfc_list(String).  % NFC only in OTP

%% หรือใช้ string:casefold สำหรับ comparison
string:casefold("CAFÉ") == string:casefold("café").
```

---

## 6. iolist

iolist คือ nested list/binary structure ที่ใช้สำหรับ efficient string building

```erlang
%% iolist = list | binary | char | nested iolist

%% ตัวอย่าง iolist
IO1 = "hello".                          % list
IO2 = <<"world">>.                      % binary
IO3 = $!.                               % char (integer)
IO4 = ["hello", " ", <<"world">>, $!].  % nested iolist

%% Convert iolist เป็น binary (efficient!)
iolist_to_binary(IO4).
%% <<"hello world!">>

%% iolist_size
iolist_size(IO4).
%% 12

%% ประโยชน์: สร้าง string โดยไม่ต้อง concatenate
build_html(Title, Body) ->
    ["<html><head><title>", Title, "</title></head>",
     "<body>", Body, "</body></html>"].

html = build_html("Test", "Hello World"),
iolist_to_binary(html).
%% <<"<html><head><title>Test</title></head><body>Hello World</body></html>">>
```

### iolist vs String Concatenation

```erlang
%% String concatenation ทำสำเนา!
build_slow(N) ->
    lists:foldl(fun(I, Acc) ->
        Acc ++ integer_to_list(I) ++ ","
    end, "", lists:seq(1, N)).
% O(n^2) เพราะ ++ ต้อง traverse ทั้ง string

%% iolist ไม่ทำสำเนาจนกว่าจำเป็น!
build_fast(N) ->
    Parts = [integer_to_list(I) ++ "," || I <- lists:seq(1, N)],
    iolist_to_binary(Parts).
% O(n) effectively

%% หรือ
build_fastest(N) ->
    iolist_to_binary(
        [[integer_to_binary(I), ","] || I <- lists:seq(1, N)]
    ).
```

### iolist Patterns

```erlang
%% HTML Builder
tag(Name, Content) ->
    ["<", Name, ">", Content, "</", Name, ">"].

%% JSON Builder (manual, ไม่ใช้ library)
json_string(S) when is_binary(S) ->
    ["\"", S, "\""];
json_string(S) when is_list(S) ->
    json_string(list_to_binary(S)).

json_object(Pairs) ->
    Parts = [["\"", K, "\":", json_value(V)] || {K, V} <- Pairs],
    ["{", join(Parts, ","), "}"].

json_value(V) when is_integer(V) -> integer_to_list(V);
json_value(V) when is_float(V)   -> float_to_list(V, [{decimals,6}]);
json_value(V) when is_binary(V)  -> json_string(V);
json_value(V) when is_list(V)    -> json_string(V);
json_value(true)  -> "true";
json_value(false) -> "false";
json_value(null)  -> "null".

join([], _) -> [];
join([H], _) -> [H];
join([H|T], Sep) -> [H, Sep | join(T, Sep)].

%% ตัวอย่าง
1> iolist_to_binary(json_object([
     {"name", <<"Alice">>},
     {"age", 30},
     {"active", true}
   ])).
<<"{\"name\":\"Alice\",\"age\":30,\"active\":true}">>
```

---

## 7. Formatting กับ io:format

### Format Strings

```erlang
%% Format specifiers:
~d   % integer (decimal)
~i   % integer (ignore)
~f   % float
~e   % float (scientific notation)
~g   % float (shorter of ~f/~e)
~s   % string (list of chars)
~ts  % Unicode string
~p   % any term (pretty print)
~w   % any term (write)
~q   % any term (with quotes)
~n   % newline
~N   % newline (platform-specific)
~r   % integer (base r)
~c   % character
~b   % integer (binary)
~o   % integer (octal)
~x   % integer (hex)
~X   % integer (HEX)
~+   % positive number
~-   % negative number
~t   % unicode character
~i   % ignore argument

%% ตัวอย่าง
1> io:format("~d~n", [42]).
42
2> io:format("~f~n", [3.14]).
3.140000
3> io:format("~.2f~n", [3.14159]).  % 2 decimal places
3.14
4> io:format("~e~n", [3.14159]).
3.14159e+0
5> io:format("~p~n", [{ok, [1,2,3]}]).
{ok,[1,2,3]}
6> io:format("~w~n", ["hello"]).
[104,101,108,108,111]  % แสดง as integers!
7> io:format("~s~n", ["hello"]).
hello
8> io:format("~ts~n", ["สวัสดี"]).  % unicode!
สวัสดี
```

### Width and Precision

```erlang
%% ~Nd - integer กว้าง N chars
1> io:format("|~10d|~n", [42]).
|        42|

%% ~-Nd - left-aligned
2> io:format("|~-10d|~n", [42]).
|42        |

%% ~N.Mf - float กว้าง N chars, M decimal places
3> io:format("|~10.2f|~n", [3.14159]).
|      3.14|

%% ~Ns - string กว้าง N chars
4> io:format("|~10s|~n", ["hello"]).
|     hello|

%% ~*d - width จาก argument
5> io:format("~*d~n", [10, 42]).
        42
```

### io_lib:format

```erlang
%% สร้าง string โดยไม่แสดง
1> io_lib:format("~p", [{ok, 42}]).
[[123,111,107,44,52,50,125]]  % iolist!

2> lists:flatten(io_lib:format("~p", [{ok, 42}])).
"{ok,42}"

3> iolist_to_binary(io_lib:format("~p", [{ok, 42}])).
<<"{ok,42}">>

%% Useful helper
format_to_binary(Format, Args) ->
    iolist_to_binary(io_lib:format(Format, Args)).

4> format_to_binary("Hello ~s, you are ~p years old", ["Alice", 30]).
<<"Hello Alice, you are 30 years old">>
```

---

## 8. Binary Protocols

### Custom Binary Protocol

```erlang
%% สร้าง custom binary protocol

%% Message Format:
%% [Version:8][Type:8][Length:32][Payload:N][Checksum:32]

-define(PROTOCOL_VERSION, 1).

-type msg_type() :: ping | pong | request | response | error.

%% Encode message
encode(Type, Payload) when is_binary(Payload) ->
    TypeByte = encode_type(Type),
    Length = byte_size(Payload),
    Checksum = checksum(Payload),
    <<?PROTOCOL_VERSION:8, TypeByte:8, Length:32, Payload/binary, Checksum:32>>.

%% Decode message
decode(<<Version:8, _/binary>>) when Version =/= ?PROTOCOL_VERSION ->
    {error, {unsupported_version, Version}};
decode(<<?PROTOCOL_VERSION:8, TypeByte:8, Length:32, 
         Payload:Length/binary, Checksum:32>>) ->
    case checksum(Payload) of
        Checksum ->
            Type = decode_type(TypeByte),
            {ok, {Type, Payload}};
        _ ->
            {error, checksum_mismatch}
    end;
decode(_) ->
    {error, invalid_message}.

encode_type(ping)     -> 1;
encode_type(pong)     -> 2;
encode_type(request)  -> 3;
encode_type(response) -> 4;
encode_type(error)    -> 5.

decode_type(1) -> ping;
decode_type(2) -> pong;
decode_type(3) -> request;
decode_type(4) -> response;
decode_type(5) -> error;
decode_type(N) -> {unknown, N}.

checksum(Bin) ->
    erlang:crc32(Bin).
```

### Frame-based Protocol

```erlang
%% Protocol ที่มี framing (สำหรับ stream protocols เช่น TCP)

%% Frame: [Length:32][Data:N]
frame(Data) when is_binary(Data) ->
    Len = byte_size(Data),
    <<Len:32, Data/binary>>.

%% Parse frames จาก stream
parse_frames(Buffer) ->
    parse_frames(Buffer, []).

parse_frames(<<Len:32, Data:Len/binary, Rest/binary>>, Frames) ->
    parse_frames(Rest, [Data|Frames]);
parse_frames(Incomplete, Frames) ->
    {lists:reverse(Frames), Incomplete}.

%% ทดสอบ
1> Buffer = <<
     5:32, "hello",
     5:32, "world",
     3:32, "foo",
     2:32, "in"  % incomplete!
   >>.
2> {Frames, Remaining} = parse_frames(Buffer).
3> Frames.
[<<"hello">>,<<"world">>,<<"foo">>]
4> Remaining.
<<0,0,0,2,105,110>>  % "in" ยังไม่ครบ
```

### Binary Parsing State Machine

```erlang
%% State machine สำหรับ parse binary data incrementally

-module(binary_parser).

init() -> {waiting_header, <<>>}.

feed({waiting_header, Buffer}, NewData) ->
    AllData = <<Buffer/binary, NewData/binary>>,
    case AllData of
        <<Len:32, Rest/binary>> ->
            feed({reading_payload, Len, <<>>}, Rest);
        _ ->
            {continue, {waiting_header, AllData}}
    end;

feed({reading_payload, Len, Buffer}, NewData) ->
    AllData = <<Buffer/binary, NewData/binary>>,
    case AllData of
        <<Payload:Len/binary, Rest/binary>> ->
            {message, Payload, feed(init(), Rest)};
        _ ->
            {continue, {reading_payload, Len, AllData}}
    end.
```

---

## 9. Performance Tips

### String Performance

```erlang
%% TIP 1: ใช้ Binary แทน String สำหรับ large data
%% String "hello" = 5 cons cells (~40 bytes)
%% Binary <<"hello">> = 5 bytes + overhead = ~20 bytes

%% TIP 2: ใช้ iolist แทน string concatenation
%% BAD:
build_bad(Parts) ->
    lists:foldl(fun(P, Acc) -> Acc ++ P end, "", Parts).

%% GOOD:
build_good(Parts) ->
    iolist_to_binary(Parts).

%% TIP 3: Compile regex pattern ก่อนใช้
%% BAD: compile ทุกครั้ง
validate_email_bad(Email) ->
    re:run(Email, "^[^@]+@[^@]+$") =/= nomatch.

%% GOOD: compile ครั้งเดียว, ใช้หลายครั้ง
-compile({parse_transform, my_app_transform}).
-define(EMAIL_RE, element(2, re:compile("^[^@]+@[^@]+$"))).

validate_email_good(Email) ->
    re:run(Email, ?EMAIL_RE) =/= nomatch.

%% TIP 4: ใช้ binary:split แทน re:split เมื่อทำได้
%% binary:split เร็วกว่า re:split มาก!

%% TIP 5: ใช้ binary pattern matching แทน regular functions
%% ตัวอย่าง: count occurrences
count_char(Binary, Char) when is_binary(Binary), is_integer(Char) ->
    count_char(Binary, Char, 0).

count_char(<<Char, Rest/binary>>, Char, Count) ->
    count_char(Rest, Char, Count + 1);
count_char(<<_, Rest/binary>>, Char, Count) ->
    count_char(Rest, Char, Count);
count_char(<<>>, _, Count) ->
    Count.
```

### Benchmark

```erlang
%% เปรียบเทียบ approaches
compare_approaches() ->
    Input = lists:duplicate(10000, "hello"),
    
    {T1, _} = timer:tc(fun() ->
        lists:foldl(fun(P, Acc) -> Acc ++ P end, "", Input)
    end),
    
    {T2, _} = timer:tc(fun() ->
        iolist_to_binary(Input)
    end),
    
    io:format("String concat: ~p us~n", [T1]),
    io:format("iolist:        ~p us~n", [T2]),
    io:format("Speedup: ~.1fx~n", [T1/T2]).
```

---

## 10. แบบฝึกหัด

### Exercise 1: String Utilities

```erlang
%% สร้าง str_utils.erl ที่มี:
%% - pad_left(Str, Width, Char)  : padding ทางซ้าย
%% - pad_right(Str, Width, Char) : padding ทางขวา
%% - center(Str, Width, Char)    : center alignment
%% - repeat(Str, N)              : repeat string N ครั้ง
%% - interpolate(Template, Vars) : "Hello #{name}!" -> "Hello Alice!"

-module(str_utils).
-export([pad_left/3, pad_right/3, center/3, repeat/2, interpolate/2]).

pad_left(Str, Width, Char) ->
    Len = string:length(Str),
    if Len >= Width -> Str;
       true ->
           Padding = lists:duplicate(Width - Len, Char),
           Padding ++ Str
    end.

pad_right(Str, Width, Char) ->
    Len = string:length(Str),
    if Len >= Width -> Str;
       true ->
           Padding = lists:duplicate(Width - Len, Char),
           Str ++ Padding
    end.

center(Str, Width, Char) ->
    Len = string:length(Str),
    if Len >= Width -> Str;
       true ->
           TotalPad = Width - Len,
           LeftPad = TotalPad div 2,
           RightPad = TotalPad - LeftPad,
           lists:duplicate(LeftPad, Char) ++ Str ++
           lists:duplicate(RightPad, Char)
    end.

repeat(Str, 0) -> "";
repeat(Str, N) when N > 0 ->
    string:copies(Str, N).

interpolate(Template, Vars) ->
    lists:foldl(fun({Key, Value}, Acc) ->
        Placeholder = "#{" ++ atom_to_list(Key) ++ "}",
        re:replace(Acc, Placeholder, Value, [{return, list}, global])
    end, Template, Vars).

%% ทดสอบ
%% 1> str_utils:pad_left("42", 5, $0).
%% "00042"
%% 2> str_utils:center("hi", 10, $-).
%% "----hi----"
%% 3> str_utils:interpolate("Hello #{name}! You are #{age}.", [{name,"Alice"},{age,"30"}]).
%% "Hello Alice! You are 30."
```

### Exercise 2: Binary Parser

```erlang
%% สร้าง bin_parser.erl สำหรับ parse binary data formats

-module(bin_parser).
-export([parse_tlv/1, encode_tlv/1,
         parse_varint/1, encode_varint/1]).

%% TLV (Type-Length-Value) Parser
parse_tlv(Binary) ->
    parse_tlv_items(Binary, []).

parse_tlv_items(<<>>, Items) ->
    {ok, lists:reverse(Items)};
parse_tlv_items(<<Type:8, Len:16, Value:Len/binary, Rest/binary>>, Items) ->
    parse_tlv_items(Rest, [{Type, Value}|Items]);
parse_tlv_items(_, _) ->
    {error, malformed_tlv}.

encode_tlv(Items) ->
    iolist_to_binary([
        <<Type:8, (byte_size(Value)):16, Value/binary>>
        || {Type, Value} <- Items
    ]).

%% Variable-length integer (protobuf style)
parse_varint(Bin) ->
    parse_varint(Bin, 0, 0).

parse_varint(<<1:1, Byte:7, Rest/binary>>, Shift, Acc) ->
    parse_varint(Rest, Shift + 7, Acc bor (Byte bsl Shift));
parse_varint(<<0:1, Byte:7, Rest/binary>>, Shift, Acc) ->
    {ok, Acc bor (Byte bsl Shift), Rest};
parse_varint(<<>>, _, _) ->
    {error, incomplete_varint}.

encode_varint(N) when N < 128 ->
    <<N:8>>;
encode_varint(N) ->
    Byte = N band 16#7F,
    Rest = N bsr 7,
    <<1:1, Byte:7, (encode_varint(Rest))/binary>>.

%% ทดสอบ
%% 1> Items = [{1, <<"name">>}, {2, <<"value">>}].
%% 2> Bin = bin_parser:encode_tlv(Items).
%% 3> bin_parser:parse_tlv(Bin).
%% {ok, [{1,<<"name">>},{2,<<"value">>}]}
```

### Exercise 3: Template Engine

```erlang
%% สร้าง template engine อย่างง่าย
%% รองรับ: {{variable}}, {{#if condition}}...{{/if}}
%%          {{#each list}}...{{/each}}

-module(template).
-export([render/2]).

render(Template, Vars) ->
    %% Simple variable substitution
    Bin = iolist_to_binary(Template),
    render_vars(Bin, Vars).

render_vars(Template, Vars) ->
    maps:fold(fun(Key, Value, Acc) ->
        Placeholder = <<"{{", (to_binary(Key))/binary, "}}">>,
        Replacement = to_binary(Value),
        binary:replace(Acc, Placeholder, Replacement, [global])
    end, Template, Vars).

to_binary(V) when is_binary(V)  -> V;
to_binary(V) when is_list(V)    -> list_to_binary(V);
to_binary(V) when is_integer(V) -> integer_to_binary(V);
to_binary(V) when is_float(V)   -> float_to_binary(V, [{decimals,2}]);
to_binary(V) when is_atom(V)    -> atom_to_binary(V, utf8).

%% ทดสอบ
%% 1> template:render(
%%      "Hello {{name}}! You have {{count}} messages.",
%%      #{name => <<"Alice">>, count => 5}
%%    ).
%% <<"Hello Alice! You have 5 messages.">>
```

---

## สรุป Part 08

ใน Part นี้คุณได้เรียนรู้:

✅ String ใน Erlang — list ของ integers  
✅ string Module — operations ทั้งหมด  
✅ Binary string operations  
✅ Regular expressions ด้วย `re` module  
✅ Unicode handling  
✅ iolist — efficient string building  
✅ io:format — format specifiers  
✅ Binary protocols — encoding/decoding  
✅ Performance tips  

---

## ต่อไป: [Part 09 — I/O และ File Operations](../part09/README.md)

ใน Part ถัดไปเราจะเรียนรู้:
- File reading และ writing
- Directory operations
- File permissions
- Streaming large files
- Standard I/O
- Port-based I/O

---

*Part 08/100 | [← ก่อนหน้า](../part07/README.md) | [ถัดไป →](../part09/README.md)*
