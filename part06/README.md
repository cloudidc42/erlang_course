# Part 06: Tuples, Maps และ Records

> **"Choose your data structure wisely — it shapes your code"**  
> การเลือก data structure ที่เหมาะสม ส่งผลต่อ code ทั้งหมด

---

## สารบัญ

1. [Tuples ขั้นสูง](#1-tuples-ขั้นสูง)
2. [Maps ขั้นสูง](#2-maps-ขั้นสูง)
3. [Records](#3-records)
4. [Proplists](#4-proplists)
5. [เปรียบเทียบและเลือกใช้ Data Structures](#5-เปรียบเทียบและเลือกใช้-data-structures)
6. [Nested Data Structures](#6-nested-data-structures)
7. [Data Transformation Patterns](#7-data-transformation-patterns)
8. [แบบฝึกหัด](#8-แบบฝึกหัด)

---

## 1. Tuples ขั้นสูง

### Tuple Operations

```erlang
%% สร้างและ access
1> T = {alice, 30, "alice@example.com"}.
{alice, 30, "alice@example.com"}

2> element(1, T).
alice
3> element(2, T).
30

%% Update (ได้ tuple ใหม่)
4> setelement(3, T, "new@email.com").
{alice, 30, "new@email.com"}

%% Tuple info
5> tuple_size(T).
3
6> is_tuple(T).
true

%% Convert
7> tuple_to_list(T).
[alice, 30, "alice@example.com"]
8> list_to_tuple([1, 2, 3]).
{1, 2, 3}

%% Append to tuple (ไม่มี built-in, ทำผ่าน list)
append_to_tuple(Tuple, Element) ->
    list_to_tuple(tuple_to_list(Tuple) ++ [Element]).

9> append_to_tuple({1,2,3}, 4).
{1,2,3,4}
```

### Tuple Patterns ที่นิยม

```erlang
%% Tagged tuple เป็น idiom หลักของ Erlang
%% {Tag, Data...}

%% Result types
{ok, Value}
{error, Reason}
{error, Code, Message}

%% Status tuples
{status, active}
{status, inactive}
{name, "Alice"}
{age, 30}

%% Records แบบ old-school (ก่อนจะมี records)
%% {user, Name, Age, Email}
alice() -> {user, "Alice", 30, "alice@example.com"}.
name({user, Name, _, _}) -> Name.
age({user, _, Age, _}) -> Age.

%% Coordinate pairs
{x, 10}
{y, 20}
{point, {10, 20}}
{rect, {0,0}, {100, 100}}

%% Date/Time
{date, 2024, 1, 15}
{time, 12, 30, 0}
{datetime, {2024,1,15}, {12,30,0}}
```

### Tuple Key-Value Store

```erlang
%% เมื่อ keys รู้ล่วงหน้าและคงที่ ใช้ tuple ได้
%% แต่ถ้า keys dynamic ใช้ map ดีกว่า

%% Example: configuration tuple
-define(DEFAULT_CONFIG, {config, 
    8080,    %% port
    "localhost",  %% host
    100,     %% max_connections
    5000     %% timeout
}).

get_port({config, Port, _, _, _}) -> Port.
get_host({config, _, Host, _, _}) -> Host.
set_port({config, _, Host, MC, T}, Port) ->
    {config, Port, Host, MC, T}.
```

---

## 2. Maps ขั้นสูง

### Map Creation Patterns

```erlang
%% Literal
1> M = #{name => "Alice", age => 30}.

%% From list
2> maps:from_list([{a, 1}, {b, 2}, {c, 3}]).
#{a => 1, b => 2, c => 3}

%% Combine multiple maps
3> maps:merge(#{a => 1}, #{b => 2}).
#{a => 1, b => 2}

%% Merge กับ conflict resolution
merge_with(F, Map1, Map2) ->
    maps:fold(fun(Key, Val2, Acc) ->
        case maps:find(Key, Acc) of
            {ok, Val1} -> Acc#{Key => F(Key, Val1, Val2)};
            error      -> Acc#{Key => Val2}
        end
    end, Map1, Map2).

%% Keep larger value on conflict
4> merge_with(fun(_, V1, V2) -> max(V1, V2) end,
              #{a => 5, b => 3},
              #{a => 2, c => 10}).
#{a => 5, b => 3, c => 10}
```

### Map Access Patterns

```erlang
%% Pattern matching (crash ถ้าไม่มี key)
#{name := Name} = #{name => "Alice", age => 30}.

%% maps:get/2 (crash ถ้าไม่มี key)
Name = maps:get(name, #{name => "Alice"}).

%% maps:get/3 กับ default (safe)
Name = maps:get(name, Map, undefined).

%% maps:find/2 (returns {ok, Value} | error)
case maps:find(name, Map) of
    {ok, Name} -> process(Name);
    error      -> handle_missing()
end.

%% ตรวจสอบ key
maps:is_key(name, Map).

%% Get or error
get_required(Key, Map) ->
    case maps:find(Key, Map) of
        {ok, Value} -> {ok, Value};
        error       -> {error, {missing_key, Key}}
    end.
```

### Map Transformations

```erlang
%% maps:map/2
1> maps:map(fun(_, V) -> V * 2 end, #{a => 1, b => 2, c => 3}).
#{a => 2, b => 4, c => 6}

%% maps:filter/2
2> maps:filter(fun(_, V) -> V > 1 end, #{a => 1, b => 2, c => 3}).
#{b => 2, c => 3}

%% maps:fold/3
3> maps:fold(fun(K, V, Acc) -> Acc + V end, 0, #{a => 1, b => 2, c => 3}).
6

%% maps:filtermap/2 (filter + map ในครั้งเดียว)
4> maps:filtermap(
     fun(_, V) when V > 1 -> {true, V * 10};
        (_, _) -> false
     end,
     #{a => 1, b => 2, c => 3}).
#{b => 20, c => 30}

%% select/invert keys
select_keys(Map, Keys) ->
    maps:filter(fun(K, _) -> lists:member(K, Keys) end, Map).

invert_map(Map) ->
    maps:fold(fun(K, V, Acc) -> Acc#{V => K} end, #{}, Map).

5> select_keys(#{a=>1,b=>2,c=>3,d=>4}, [a,c]).
#{a => 1, c => 3}

6> invert_map(#{a=>1,b=>2,c=>3}).
#{1=>a, 2=>b, 3=>c}
```

### Update Patterns

```erlang
%% Simple update (fails ถ้าไม่มี key)
Map#{age := 31}.

%% Add new key (ใช้ => แทน :=)
Map#{email => "alice@example.com"}.

%% Conditional update
update_if_exists(Map, Key, Fun) ->
    case maps:find(Key, Map) of
        {ok, Value} -> Map#{Key => Fun(Value)};
        error       -> Map
    end.

%% Increment counter
increment(Map, Key) ->
    maps:update_with(Key, fun(V) -> V + 1 end, 1, Map).

1> M0 = #{}.
2> M1 = increment(M0, a).
#{a => 1}
3> M2 = increment(M1, a).
#{a => 2}
4> M3 = increment(M2, b).
#{a => 2, b => 1}

%% Deep update
deep_update(Map, [Key], Fun) ->
    maps:update_with(Key, Fun, Map);
deep_update(Map, [Key|Rest], Fun) ->
    SubMap = maps:get(Key, Map, #{}),
    Map#{Key => deep_update(SubMap, Rest, Fun)}.

5> M = #{user => #{profile => #{age => 30}}}.
6> deep_update(M, [user, profile, age], fun(A) -> A + 1 end).
#{user => #{profile => #{age => 31}}}
```

### Map Performance

```erlang
%% Maps ใน Erlang implement เป็น HAMT (Hash Array Mapped Trie)
%% การ access/update: O(log n)
%% การ create: O(n)

%% สำหรับ small maps (<32 entries), Erlang ใช้ flat representation
%% ซึ่ง faster กว่า HAMT

%% Benchmark
benchmark_map_access(N) ->
    Map = maps:from_list([{I, I} || I <- lists:seq(1, N)]),
    {Time, _} = timer:tc(fun() ->
        lists:foreach(fun(I) -> maps:get(I, Map) end,
                      lists:seq(1, N))
    end),
    Time / N.  % microseconds per access
```

---

## 3. Records

Records คือ named tuple ที่มี named fields — ทำให้ code อ่านง่ายขึ้น

### Declaration

```erlang
%% Records ประกาศใน .hrl file หรือ .erl file
-record(user, {
    id,
    name,
    age,
    email,
    active = true,
    created_at
}).

%% Records กับ type specs
-record(point, {
    x :: number(),
    y :: number(),
    z = 0 :: number()
}).

-record(config, {
    host     = "localhost" :: string(),
    port     = 8080        :: pos_integer(),
    timeout  = 5000        :: pos_integer(),
    debug    = false       :: boolean()
}).
```

### สร้างและใช้ Records

```erlang
%% สร้าง record
Alice = #user{
    id        = 1,
    name      = "Alice",
    age       = 30,
    email     = "alice@example.com",
    created_at = erlang:timestamp()
}.

%% Access fields
Alice#user.name.        % "Alice"
Alice#user.age.         % 30
Alice#user.active.      % true (default)

%% Update record (ได้ record ใหม่!)
Alice2 = Alice#user{age = 31}.
Alice3 = Alice#user{age = 31, email = "new@email.com"}.

%% Pattern matching
case User of
    #user{active = true, name = Name} ->
        io:format("Active user: ~s~n", [Name]);
    #user{active = false} ->
        io:format("Inactive user~n")
end.

%% ใน function clause
process_user(#user{id = Id, name = Name, active = true}) ->
    {ok, {Id, Name}};
process_user(#user{active = false}) ->
    {error, inactive}.
```

### Record Macros

```erlang
%% เข้าถึง record ด้วย macro
-record(user, {name, age}).

%% ?record_info(fields, RecordName) -> [FieldNames]
%% ใช้ใน module ที่ define record

fields() -> record_info(fields, user).
% [name, age]

index(Field) -> 
    list_to_tuple([0|record_info(fields, user)]),
    % ...หรือ
    Record = #user{},
    Size = tuple_size(Record),
    find_index(Field, record_info(fields, user), 2).

%% Useful macro: IS_RECORD
-define(IS_USER(X), is_record(X, user)).

%% ใน guard
process(User) when ?IS_USER(User) ->
    %% safe to access User#user.name
    User#user.name.
```

### Record vs Map Comparison

```erlang
%% RECORD
-record(person, {name, age, email}).

create_record(Name, Age, Email) ->
    #person{name=Name, age=Age, email=Email}.

get_name_record(P) -> P#person.name.

update_age_record(P, Age) -> P#person{age=Age}.

%% MAP
create_map(Name, Age, Email) ->
    #{name => Name, age => Age, email => Email}.

get_name_map(M) -> maps:get(name, M).

update_age_map(M, Age) -> M#{age => Age}.

%% ใช้ Record เมื่อ:
%% 1. fields ที่รู้ล่วงหน้าและไม่เปลี่ยน
%% 2. ต้องการ compile-time checks
%% 3. Legacy code
%% 4. Records ใน ETS/Mnesia

%% ใช้ Map เมื่อ:
%% 1. Dynamic keys
%% 2. จะ serialize เป็น JSON
%% 3. ผ่าน network boundaries
%% 4. Keys ที่ optional มากๆ
%% 5. Modern Erlang code (recommended)
```

### Records ใน ETS (ตัวอย่าง)

```erlang
%% Records เข้ากันได้ดีกับ ETS
-record(session, {
    id      :: binary(),
    user_id :: pos_integer(),
    token   :: binary(),
    expires :: integer()
}).

create_session_table() ->
    ets:new(sessions, [
        set,
        named_table,
        public,
        {keypos, #session.id}  %% ใช้ field เป็น key!
    ]).

store_session(Session = #session{}) ->
    ets:insert(sessions, Session).

find_session(Id) ->
    case ets:lookup(sessions, Id) of
        [Session] -> {ok, Session};
        []        -> not_found
    end.
```

---

## 4. Proplists

Proplist คือ list ของ `{key, value}` หรือ atoms

```erlang
%% Proplist = [{key, value}] หรือ [key] (atom shorthand)

%% สร้าง proplist
Props = [
    {host, "localhost"},
    {port, 8080},
    {debug, true},
    verbose    %% shorthand สำหรับ {verbose, true}
].

%% Access
proplists:get_value(host, Props).          % "localhost"
proplists:get_value(port, Props).          % 8080
proplists:get_value(missing, Props).       % undefined
proplists:get_value(missing, Props, "def").% "def"

proplists:is_defined(debug, Props).        % true
proplists:is_defined(absent, Props).       % false

proplists:get_all_values(host, Props).     % ["localhost"]

%% ลบ key
proplists:delete(port, Props).

%% Normalize (convert atoms to {atom, true})
proplists:normalize(Props, []).

%% To/from Map
maps:from_list(Props).
maps:to_list(SomeMap).
```

### Proplist Use Cases

```erlang
%% นิยมใช้สำหรับ configuration options
connect(Host, Port, Opts) ->
    Timeout = proplists:get_value(timeout, Opts, 5000),
    SSL     = proplists:get_value(ssl, Opts, false),
    Debug   = proplists:get_value(debug, Opts, false),
    
    if SSL -> ssl_connect(Host, Port, Timeout, Debug);
       true -> tcp_connect(Host, Port, Timeout, Debug)
    end.

%% ใช้งาน
connect("localhost", 8080, [
    {timeout, 10000},
    ssl,           %% shorthand สำหรับ {ssl, true}
    {debug, true}
]).

%% รับ opts กับ default values
parse_opts(Opts) ->
    #{
        timeout   => proplists:get_value(timeout, Opts, 5000),
        retries   => proplists:get_value(retries, Opts, 3),
        verbose   => proplists:get_bool(verbose, Opts),
        pool_size => proplists:get_value(pool_size, Opts, 10)
    }.
```

---

## 5. เปรียบเทียบและเลือกใช้ Data Structures

### เปรียบเทียบ

```
                Tuple   Map     Record   List    Proplist
────────────────────────────────────────────────────────
Access          O(1)   O(log n)  O(1)   O(n)     O(n)
Update          O(n)   O(log n)  O(n)   O(n)     O(n)
Pattern Match   ★★★★★  ★★★      ★★★★★  ★★★★    ★★
Type Safety     Runtime Dialyzer Compile Runtime  Runtime
Dynamic Keys    ✗       ✓        ✗      ✓         ✓
Memory          Small   Medium   Small  Large     Large
JSON-friendly   ✗       ✓        ✗      ✓         ✗
Named Fields    ✗       ✓        ✓      ✗         ✓
OTP Compatible  ✓       ✓        ✓      ✓         ✓
```

### เมื่อไหรควรใช้อะไร

```erlang
%% ======================================================
%% Tuple: ใช้เมื่อ...
%% ======================================================
%% 1. Fixed structure, ขนาดเล็ก
%% 2. Tagged results: {ok, V}, {error, R}
%% 3. Coordinates: {X, Y}, {X, Y, Z}
%% 4. Key-value pair: {key, value}

%% GOOD
result(ok, Value) -> {ok, Value}.
point(X, Y) -> {X, Y}.
status(Code, Msg) -> {Code, Msg}.

%% ======================================================
%% Map: ใช้เมื่อ...
%% ======================================================
%% 1. Dynamic keys
%% 2. JSON data
%% 3. Configuration
%% 4. Key-value store
%% 5. Optional fields

%% GOOD
user(Name, Age) ->
    #{name => Name, age => Age}.

config(Opts) ->
    maps:merge(default_config(), maps:from_list(Opts)).

%% ======================================================
%% Record: ใช้เมื่อ...
%% ======================================================
%% 1. Well-known, stable structure
%% 2. ETS/Mnesia tables
%% 3. Performance critical (ขนาดน้อยกว่า map)
%% 4. Legacy code ที่ต้องการ backward compat

%% GOOD
-record(request, {method, path, headers, body}).
-record(state, {socket, buffer, timeout}).

%% ======================================================
%% List: ใช้เมื่อ...
%% ======================================================
%% 1. Sequential data
%% 2. Variable length
%% 3. Processing element by element
%% 4. Stack operations

%% ======================================================
%% Proplist: ใช้เมื่อ...
%% ======================================================
%% 1. Configuration options สำหรับ functions
%% 2. Backward-compatible APIs
%% 3. Optional arguments
```

---

## 6. Nested Data Structures

### Nested Maps

```erlang
%% สร้าง nested structure
Company = #{
    name => "Acme Corp",
    address => #{
        street => "123 Main St",
        city => "Springfield",
        country => "US"
    },
    employees => [
        #{name => "Alice", role => cto},
        #{name => "Bob", role => engineer}
    ]
}.

%% Access nested
City = maps:get(city, maps:get(address, Company)).
%% หรือใช้ pattern matching
#{address := #{city := City}} = Company.

%% Deep access helper
get_in(Map, []) -> Map;
get_in(Map, [Key|Rest]) ->
    get_in(maps:get(Key, Map), Rest).

1> get_in(Company, [address, city]).
"Springfield"
```

### Lens Pattern (Functional)

```erlang
%% Lens ช่วย get/update nested structures อย่าง elegant

%% Simple lens
make_lens(Get, Set) ->
    {Get, Set}.

lens_get({Get, _}, Structure) ->
    Get(Structure).

lens_set({_, Set}, Structure, Value) ->
    Set(Structure, Value).

lens_update(Lens, Structure, Fun) ->
    Value = lens_get(Lens, Structure),
    lens_set(Lens, Structure, Fun(Value)).

%% Map lens
map_lens(Key) ->
    make_lens(
        fun(Map) -> maps:get(Key, Map) end,
        fun(Map, Value) -> Map#{Key => Value} end
    ).

%% Compose lenses
lens_compose(Inner, Outer) ->
    make_lens(
        fun(S) -> lens_get(Inner, lens_get(Outer, S)) end,
        fun(S, V) ->
            OuterVal = lens_get(Outer, S),
            NewOuterVal = lens_set(Inner, OuterVal, V),
            lens_set(Outer, S, NewOuterVal)
        end
    ).

%% ใช้งาน
AddressLens = map_lens(address).
CityLens = map_lens(city).
AddressCityLens = lens_compose(CityLens, AddressLens).

Company = #{address => #{city => "Old City"}}.
lens_update(AddressCityLens, Company, fun(_) -> "New City" end).
% #{address => #{city => "New City"}}
```

---

## 7. Data Transformation Patterns

### JSON-like Transformation

```erlang
%% Erlang Map <-> JSON-like transformation

%% สมมุติว่าได้ JSON parsed เป็น map
ParsedJson = #{
    <<"name">> => <<"Alice">>,
    <<"age">> => 30,
    <<"address">> => #{
        <<"city">> => <<"Bangkok">>
    }
}.

%% Convert binary keys to atoms (careful! atom table limit)
binary_keys_to_atoms(Map) when is_map(Map) ->
    maps:fold(fun(K, V, Acc) ->
        NewK = binary_to_existing_atom(K, utf8),
        NewV = if is_map(V) -> binary_keys_to_atoms(V);
                  true -> V
               end,
        Acc#{NewK => NewV}
    end, #{}, Map).

%% Convert atom keys to binary
atom_keys_to_binary(Map) when is_map(Map) ->
    maps:fold(fun(K, V, Acc) ->
        NewK = atom_to_binary(K, utf8),
        NewV = if is_map(V) -> atom_keys_to_binary(V);
                  true -> V
               end,
        Acc#{NewK => NewV}
    end, #{}, Map).
```

### Record to Map Conversion

```erlang
%% Macro-based conversion
-record(user, {id, name, age, email}).

record_to_map(Record = #user{}) ->
    Fields = record_info(fields, user),
    Values = tl(tuple_to_list(Record)),
    maps:from_list(lists:zip(Fields, Values)).

%% หรือ explicit
user_to_map(#user{id=I, name=N, age=A, email=E}) ->
    #{id => I, name => N, age => A, email => E}.

map_to_user(#{id := I, name := N, age := A, email := E}) ->
    #user{id=I, name=N, age=A, email=E}.
```

### Data Validation Pipeline

```erlang
%% Pipeline ของ transformations และ validations

-module(user_pipeline).

process_user_input(Input) ->
    Steps = [
        fun parse_input/1,
        fun validate_required_fields/1,
        fun validate_email/1,
        fun validate_age/1,
        fun normalize_fields/1,
        fun enrich_with_defaults/1
    ],
    
    lists:foldl(fun
        (Step, {ok, Data}) ->
            Step(Data);
        (_, {error, _} = Error) ->
            Error
    end, {ok, Input}, Steps).

parse_input(Input) when is_map(Input) ->
    {ok, Input};
parse_input(_) ->
    {error, invalid_input}.

validate_required_fields(Data) ->
    Required = [name, email],
    Missing = [F || F <- Required, not maps:is_key(F, Data)],
    case Missing of
        []   -> {ok, Data};
        Keys -> {error, {missing_fields, Keys}}
    end.

validate_email(Data = #{email := Email}) ->
    case re:run(Email, "^[^@]+@[^@]+\\.[^@]+$") of
        {match, _} -> {ok, Data};
        nomatch    -> {error, invalid_email}
    end.

validate_age(Data = #{age := Age}) when is_integer(Age), Age >= 0, Age =< 150 ->
    {ok, Data};
validate_age(Data) when not is_map_key(age, Data) ->
    {ok, Data};
validate_age(_) ->
    {error, invalid_age}.

normalize_fields(Data = #{name := Name}) when is_list(Name) ->
    {ok, Data#{name => list_to_binary(Name)}};
normalize_fields(Data) ->
    {ok, Data}.

enrich_with_defaults(Data) ->
    Defaults = #{
        role => user,
        active => true,
        created_at => erlang:system_time(second)
    },
    {ok, maps:merge(Defaults, Data)}.

%% ทดสอบ
test() ->
    Input = #{name => "Alice", email => "alice@example.com", age => 30},
    process_user_input(Input).
```

---

## 8. แบบฝึกหัด

### Exercise 1: User Management System

```erlang
%% สร้าง user_mgmt.erl ที่ใช้ Records และ Maps ร่วมกัน

-module(user_mgmt).
-record(user, {
    id        :: pos_integer(),
    username  :: binary(),
    email     :: binary(),
    role      = user :: admin | user | moderator,
    active    = true :: boolean(),
    created_at        :: integer()
}).

-export([create/2, get/2, update/3, delete/2, list_active/1]).

create(DB, #{username := UN, email := E} = Input) ->
    Id = generate_id(),
    User = #user{
        id         = Id,
        username   = UN,
        email      = E,
        role       = maps:get(role, Input, user),
        created_at = erlang:system_time(second)
    },
    {ok, DB#{Id => User}, User}.

get(DB, Id) ->
    case maps:find(Id, DB) of
        {ok, User} -> {ok, User};
        error      -> {error, not_found}
    end.

update(DB, Id, Changes) ->
    case maps:find(Id, DB) of
        {ok, User} ->
            Updated = apply_changes(User, Changes),
            {ok, DB#{Id => Updated}, Updated};
        error ->
            {error, not_found}
    end.

delete(DB, Id) ->
    case maps:is_key(Id, DB) of
        true  -> {ok, maps:remove(Id, DB)};
        false -> {error, not_found}
    end.

list_active(DB) ->
    [User || {_, User} <- maps:to_list(DB),
             User#user.active =:= true].

%% Private
generate_id() -> erlang:unique_integer([positive]).

apply_changes(User, Changes) ->
    maps:fold(fun
        (email, V, U) when is_binary(V) -> U#user{email = V};
        (role, V, U) when V =:= admin; V =:= user; V =:= moderator ->
            U#user{role = V};
        (active, V, U) when is_boolean(V) -> U#user{active = V};
        (_, _, U) -> U  % ignore unknown fields
    end, User, Changes).
```

### Exercise 2: Configuration Manager

```erlang
%% สร้าง config_mgr.erl
%% ที่จัดการ configuration ด้วย Map และ support:
%% - Load จาก proplist
%% - Deep merge
%% - Path-based access (a.b.c)
%% - Type validation
%% - Default values

-module(config_mgr).
-export([new/0, load/2, get/2, get/3, set/3, merge/2]).

new() -> #{}.

load(Config, Proplist) ->
    merge(Config, proplist_to_map(Proplist)).

get(Config, Path) ->
    get(Config, Path, undefined).

get(Config, Path, Default) when is_atom(Path) ->
    maps:get(Path, Config, Default);
get(Config, Path, Default) when is_list(Path) ->
    get_path(Config, Path, Default).

get_path(Config, [], _Default) -> Config;
get_path(Config, [Key|Rest], Default) ->
    case maps:find(Key, Config) of
        {ok, Sub} -> get_path(Sub, Rest, Default);
        error     -> Default
    end.

set(Config, Path, Value) when is_atom(Path) ->
    Config#{Path => Value};
set(Config, [Key], Value) ->
    Config#{Key => Value};
set(Config, [Key|Rest], Value) ->
    Sub = maps:get(Key, Config, #{}),
    Config#{Key => set(Sub, Rest, Value)}.

merge(Base, Override) ->
    maps:fold(fun(K, V, Acc) ->
        case {maps:find(K, Acc), is_map(V)} of
            {{ok, OldV}, true} when is_map(OldV) ->
                Acc#{K => merge(OldV, V)};
            _ ->
                Acc#{K => V}
        end
    end, Base, Override).

proplist_to_map(List) ->
    lists:foldl(fun
        ({K, V}, Acc) -> Acc#{K => V};
        (K, Acc) when is_atom(K) -> Acc#{K => true}
    end, #{}, List).

%% ทดสอบ
test() ->
    Config = new(),
    Config2 = load(Config, [
        {database, #{host => "localhost", port => 5432}},
        {app, #{port => 8080}},
        debug
    ]),
    get(Config2, [database, host]),     % "localhost"
    get(Config2, [app, port]),          % 8080
    get(Config2, debug),                % true
    get(Config2, missing, "default").   % "default"
```

### Exercise 3: Inventory System

```erlang
%% สร้าง inventory.erl ที่จัดการ inventory ด้วย nested maps
%% รองรับ: add_item, remove_item, update_quantity
%%          find_by_category, get_low_stock, total_value

-module(inventory).
-export([new/0, add_item/2, remove_item/2,
         update_quantity/3, find_by_category/2,
         get_low_stock/2, total_value/1]).

-type item() :: #{
    id       := binary(),
    name     := binary(),
    category := atom(),
    quantity := non_neg_integer(),
    price    := float()
}.

new() -> #{}.

add_item(Inventory, Item = #{id := Id}) ->
    case maps:is_key(Id, Inventory) of
        true  -> {error, item_exists};
        false -> {ok, Inventory#{Id => Item}}
    end.

remove_item(Inventory, Id) ->
    case maps:is_key(Id, Inventory) of
        true  -> {ok, maps:remove(Id, Inventory)};
        false -> {error, not_found}
    end.

update_quantity(Inventory, Id, Delta) ->
    case maps:find(Id, Inventory) of
        {ok, Item = #{quantity := Q}} ->
            NewQ = Q + Delta,
            if NewQ < 0 -> {error, insufficient_stock};
               true ->
                   {ok, Inventory#{Id => Item#{quantity => NewQ}}}
            end;
        error -> {error, not_found}
    end.

find_by_category(Inventory, Category) ->
    maps:filter(fun(_, #{category := C}) -> C =:= Category end,
                Inventory).

get_low_stock(Inventory, Threshold) ->
    maps:filter(fun(_, #{quantity := Q}) -> Q =< Threshold end,
                Inventory).

total_value(Inventory) ->
    maps:fold(fun(_, #{quantity := Q, price := P}, Acc) ->
        Acc + Q * P
    end, 0.0, Inventory).
```

---

## สรุป Part 06

ใน Part นี้คุณได้เรียนรู้:

✅ Tuple operations ขั้นสูงและ patterns  
✅ Map creation, access, update, transformation  
✅ Records — declaration, creation, access, update  
✅ Records ใน ETS  
✅ Proplists — use cases และ API  
✅ เปรียบเทียบ data structures  
✅ Nested data structures  
✅ Lens pattern  
✅ Data transformation pipelines  

---

## ต่อไป: [Part 07 — Control Flow](../part07/README.md)

ใน Part ถัดไปเราจะเรียนรู้:
- if expression
- case expression ขั้นสูง
- receive (สำหรับ process messages)
- try/catch/after
- throw, error, exit
- Guards ขั้นสูง

---

*Part 06/100 | [← ก่อนหน้า](../part05/README.md) | [ถัดไป →](../part07/README.md)*
