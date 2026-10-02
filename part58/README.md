# Part 58: Erlang Type System and Dialyzer

> **"Types catch bugs before runtime — Dialyzer catches them before you ship"**  
> Types ตรวจจับ bugs ก่อน runtime — Dialyzer ตรวจจับก่อนที่คุณจะ ship

---

## สารบัญ

1. [Type Specifications](#1-type-specifications)
2. [Custom Types](#2-custom-types)
3. [Opaque Types](#3-opaque-types)
4. [Dialyzer Analysis](#4-dialyzer-analysis)
5. [Success Typings](#5-success-typings)
6. [Type Guards](#6-type-guards)
7. [แบบฝึกหัด](#7-แบบฝึกหัด)

---

## 1. Type Specifications

```erlang
%% typed_module.erl — complete type specifications
-module(typed_module).

%% Basic types: integer, float, atom, binary, pid, reference, fun, port
%% Compound: {T1, T2}, [T], #{K => V}, T | T2 (union)

%% Function specs
-spec add(integer(), integer()) -> integer().
add(A, B) -> A + B.

-spec divide(number(), number()) -> {ok, float()} | {error, division_by_zero}.
divide(_, 0) -> {error, division_by_zero};
divide(A, B) -> {ok, A / B}.

-spec process_list([integer()]) -> {sum, integer(), count, non_neg_integer()}.
process_list(List) ->
    {sum, lists:sum(List), count, length(List)}.

%% Multiple clauses with different return types
-spec format_value(atom() | integer() | binary()) -> binary().
format_value(A) when is_atom(A)    -> atom_to_binary(A, utf8);
format_value(N) when is_integer(N) -> integer_to_binary(N);
format_value(B) when is_binary(B)  -> B.

%% Higher-order function specs
-spec apply_to_list([A], fun((A) -> B)) -> [B].
apply_to_list(List, Fun) -> lists:map(Fun, List).

%% Callback type
-type handler() :: fun((term()) -> {ok, term()} | {error, term()}).
-spec register_handler(atom(), handler()) -> ok.
register_handler(Name, Handler) ->
    ets:insert(handlers, {Name, Handler}),
    ok.
```

---

## 2. Custom Types

```erlang
%% types.erl — defining and using custom types
-module(types).

%% Simple type alias
-type user_id()   :: pos_integer().
-type session_id() :: binary().
-type timestamp() :: non_neg_integer().  %% unix seconds

%% Record-based type
-record(user, {
    id         :: user_id(),
    name       :: binary(),
    email      :: binary(),
    created_at :: timestamp(),
    role       :: admin | user | moderator
}).
-type user() :: #user{}.

%% Union types
-type json_value() ::
    null |
    boolean() |
    number() |
    binary() |
    [json_value()] |
    #{binary() => json_value()}.

%% Result type
-type result(T) :: {ok, T} | {error, error_reason()}.
-type error_reason() ::
    not_found |
    unauthorized |
    {invalid_field, atom()} |
    {db_error, term()}.

%% Function specs using custom types
-spec find_user(user_id()) -> result(user()).
find_user(UserId) ->
    case db:query("SELECT * FROM users WHERE id = $1", [UserId]) of
        {ok, [Row | _]} -> {ok, row_to_user(Row)};
        {ok, []}        -> {error, not_found};
        {error, Reason} -> {error, {db_error, Reason}}
    end.

-spec create_user(binary(), binary()) -> result(user()).
create_user(Name, Email) ->
    case validate_email(Email) of
        ok ->
            {ok, #{<<"id">> := Id}} = db:execute(
                "INSERT INTO users (name, email) VALUES ($1, $2) RETURNING id",
                [Name, Email]),
            find_user(Id);
        {error, invalid_email} ->
            {error, {invalid_field, email}}
    end.

row_to_user(Row) ->
    #user{
        id         = maps:get(<<"id">>, Row),
        name       = maps:get(<<"name">>, Row),
        email      = maps:get(<<"email">>, Row),
        created_at = maps:get(<<"created_at">>, Row),
        role       = binary_to_atom(maps:get(<<"role">>, Row), utf8)
    }.

validate_email(Email) when is_binary(Email) ->
    case re:run(Email, <<"^[^@]+@[^@]+\\.[^@]+$">>) of
        {match, _} -> ok;
        nomatch    -> {error, invalid_email}
    end.
```

---

## 3. Opaque Types

```erlang
%% queue_typed.erl — opaque type encapsulation
-module(queue_typed).
-export([new/0, push/2, pop/1, size/1, to_list/1]).

%% Opaque type: internals hidden from other modules
-opaque queue(T) :: {non_neg_integer(), [T], [T]}.

-spec new() -> queue(_).
new() -> {0, [], []}.

-spec push(queue(T), T) -> queue(T).
push({N, In, Out}, Item) ->
    {N+1, [Item | In], Out}.

-spec pop(queue(T)) -> {T, queue(T)} | empty.
pop({0, [], []}) -> empty;
pop({N, In, [H | T]}) ->
    {{value, H}, {N-1, In, T}};
pop({N, In, []}) ->
    [H | T] = lists:reverse(In),
    {{value, H}, {N-1, [], T}}.

-spec size(queue(_)) -> non_neg_integer().
size({N, _, _}) -> N.

-spec to_list(queue(T)) -> [T].
to_list({_, In, Out}) ->
    Out ++ lists:reverse(In).
```

---

## 4. Dialyzer Analysis

```bash
# Run Dialyzer on your project
rebar3 dialyzer

# Build PLT manually (first time)
dialyzer --build_plt --apps erts kernel stdlib

# Check specific files
dialyzer --src src/mymodule.erl

# Check with extra warnings
dialyzer --Wunmatched_returns --Wunderspecs --Wno_return src/

# Output formats
dialyzer src/ --format short    # brief
dialyzer src/ --format long     # detailed
dialyzer src/ --format raw      # for tooling

# Common Dialyzer warnings:
# - The call will never return since it differs in argument/return
# - Function will never be called
# - The variable can never match
# - The call might throw an exception
```

```erlang
%% dialyzer_examples.erl — understanding Dialyzer warnings

%% Warning: "The call will never return"
bad_always_error(X) ->
    case X of
        1 -> good;
        2 -> good
        %% Dialyzer warns: no catch-all, but function signature says it returns atom()
    end.

%% Warning: unreachable code
dead_code(X) ->
    case X > 0 of
        true  -> positive;
        false -> negative;
        other -> impossible  %% Dialyzer: this clause can never match
    end.

%% Type annotation that Dialyzer uses
-spec safe_divide(number(), number()) -> number().
safe_divide(A, B) ->
    A / B.  %% Dialyzer knows this can crash with badarith — add error handling
```

---

## 5. Success Typings

```erlang
%% Dialyzer uses "success typings": infer what CAN succeed, not what MUST succeed
%% This means Dialyzer only warns when something CANNOT work, never when it MIGHT work

%% Example: Dialyzer is conservative — won't warn unless certain
possible_crash(X) ->
    X + 1.  %% Dialyzer infers X must be number() — warns if you pass non-number

%% Dialyzer vs type checkers:
%% Type checkers (TypeScript, Haskell): must prove type IS correct
%% Dialyzer: must disprove — warns only when type CANNOT be correct

%% This means Dialyzer has no false positives but may have false negatives
%% A function Dialyzer doesn't warn about might still crash at runtime
```

---

## 6. Type Guards

```erlang
%% type_guards.erl — using guards for runtime type checking
-module(type_guards).
-export([assert_string/1, is_user/1, validate_config/1]).

%% Type assertion helpers
assert_string(X) when is_binary(X)   -> X;
assert_string(X) when is_list(X)     -> list_to_binary(X);
assert_string(X) when is_atom(X)     -> atom_to_binary(X, utf8);
assert_string(X) -> error({expected_string, got, X}).

%% Runtime record validation
is_user(#{id := Id, name := Name, email := Email})
        when is_integer(Id), is_binary(Name), is_binary(Email) ->
    true;
is_user(_) -> false.

%% Config validation with type info
validate_config(Config) when is_map(Config) ->
    Required = [
        {port,    fun is_integer/1},
        {host,    fun is_binary/1},
        {workers, fun(N) -> is_integer(N) andalso N > 0 end}
    ],
    Errors = [
        {Field, missing_or_invalid}
        || {Field, Validator} <- Required,
           not (case maps:find(Field, Config) of
                    {ok, V} -> Validator(V);
                    error   -> false
                end)
    ],
    case Errors of
        [] -> {ok, Config};
        _  -> {error, {invalid_config, Errors}}
    end.
```

---

## 7. แบบฝึกหัด

1. Add `-spec` to all exported functions in a module and run Dialyzer
2. สร้าง opaque type สำหรับ `encrypted_value()` ที่ซ่อน implementation
3. Fix 5 Dialyzer warnings จาก project จริงๆ
4. เขียน `validate_map/2` ที่ใช้ type spec เพื่อ validate map structure

---

## สรุป Part 58

✅ Function type specifications (-spec)  
✅ Custom types และ opaque types  
✅ Dialyzer static analysis  
✅ Understanding success typings  
✅ Runtime type guards  

---

*Part 58/100 | [← ก่อนหน้า](../part57/README.md) | [ถัดไป →](../part59/README.md)*
