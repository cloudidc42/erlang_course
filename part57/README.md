# Part 57: Functional Programming Patterns

> **"Functions that transform, not mutate — the Erlang way of thinking"**  
> Functions ที่แปลง ไม่ใช่แก้ไข — วิธีคิดแบบ Erlang

---

## สารบัญ

1. [Higher-Order Functions](#1-higher-order-functions)
2. [Monadic Patterns](#2-monadic-patterns)
3. [Currying and Partial Application](#3-currying-and-partial-application)
4. [Transducers](#4-transducers)
5. [Recursive Patterns](#5-recursive-patterns)
6. [Point-Free Style](#6-point-free-style)
7. [แบบฝึกหัด](#7-แบบฝึกหัด)

---

## 1. Higher-Order Functions

```erlang
%% fn.erl — higher order function utilities
-module(fn).
-export([compose/1, pipe/2, identity/1, const/1,
         map/2, filter/2, reduce/3, flat_map/2,
         memoize/1, once/1, throttle/2]).

%% Function composition: compose([f, g, h]) = f(g(h(x)))
compose([])    -> fun identity/1;
compose([F])   -> F;
compose([F|Fs]) ->
    G = compose(Fs),
    fun(X) -> F(G(X)) end.

%% Pipe: apply list of functions left-to-right
pipe(Value, Funs) ->
    lists:foldl(fun(F, V) -> F(V) end, Value, Funs).

identity(X) -> X.
const(X) -> fun(_) -> X end.

%% Map/filter/reduce on lists
map(List, F)       -> lists:map(F, List).
filter(List, Pred) -> lists:filter(Pred, List).
reduce(List, Acc, F) -> lists:foldl(F, Acc, List).

flat_map(List, F) ->
    lists:flatten([F(X) || X <- List]).

%% Memoize: cache function results
memoize(F) ->
    Cache = ets:new(memo_cache, [set]),
    fun(X) ->
        case ets:lookup(Cache, X) of
            [{X, V}] -> V;
            [] ->
                V = F(X),
                ets:insert(Cache, {X, V}),
                V
        end
    end.

%% Once: function that can only be called once
once(F) ->
    Ref = make_ref(),
    fun(X) ->
        case get(Ref) of
            called -> {error, already_called};
            undefined ->
                put(Ref, called),
                F(X)
        end
    end.

%% Throttle: limit to N calls per millisecond window
throttle(F, WindowMs) ->
    Table = ets:new(throttle, [set]),
    ets:insert(Table, {count, 0, os:system_time(millisecond)}),
    fun(X) ->
        Now  = os:system_time(millisecond),
        [{count, N, WinStart}] = ets:lookup(Table, count),
        case Now - WinStart >= WindowMs of
            true  ->
                ets:insert(Table, {count, 1, Now}),
                F(X);
            false when N < 10 ->
                ets:update_element(Table, count, {2, N+1}),
                F(X);
            false ->
                {throttled, try_again_after, WinStart + WindowMs - Now}
        end
    end.
```

---

## 2. Monadic Patterns

```erlang
%% result.erl — Result monad for error handling
-module(result).
-export([ok/1, err/1, bind/2, map/2, map_err/2, unwrap/1,
         sequence/1, traverse/2]).

ok(Value) -> {ok, Value}.
err(Reason) -> {error, Reason}.

%% bind/2: chain operations that might fail
bind({ok, V}, F) ->
    try F(V)
    catch C:E -> {error, {C, E}}
    end;
bind({error, _} = Err, _F) ->
    Err.

%% map/2: transform the success value
map({ok, V}, F)       -> {ok, F(V)};
map({error, _} = E, _) -> E.

%% map_err/2: transform the error value
map_err({ok, _} = V,   _) -> V;
map_err({error, E},    F)  -> {error, F(E)}.

unwrap({ok, V})    -> V;
unwrap({error, E}) -> error(E).

%% sequence: [result()] → result([])
sequence(Results) ->
    lists:foldl(fun
        ({ok, V}, {ok, Acc}) -> {ok, [V | Acc]};
        ({error, _} = E, _)  -> E;
        (_, {error, _} = E)  -> E
    end, {ok, []}, Results).

%% traverse: apply F to each element, fail on first error
traverse(List, F) ->
    sequence([F(X) || X <- List]).

%% Usage example:
example() ->
    result:bind(
        result:ok(5),
        fun(N) ->
            result:bind(
                result:ok(N * 2),
                fun(M) ->
                    result:ok(M + 1)
                end)
        end).  %% => {ok, 11}
```

---

## 3. Currying and Partial Application

```erlang
%% curry.erl — currying and partial application
-module(curry).
-export([curry/1, partial/2, flip/1]).

%% Partial application: fix first N arguments
partial(F, Args) when is_function(F), is_list(Args) ->
    fun(MoreArgs) when is_list(MoreArgs) ->
        apply(F, Args ++ MoreArgs);
       (Arg) ->
        apply(F, Args ++ [Arg])
    end.

%% Flip: swap first two arguments
flip(F) ->
    fun(A, B) -> F(B, A) end.

%% Usage
add(A, B) -> A + B.

examples() ->
    Add5 = partial(fun add/2, [5]),
    Add5(3),   %% => 8
    Add5(10),  %% => 15

    %% Flip division
    Div = fun(A, B) -> A / B end,
    DivBy2 = partial(flip(Div), [2]),
    DivBy2(10).  %% => 5.0

%% Auto-curry: wrap function to accept args one at a time
curry(F) when is_function(F) ->
    {arity, N} = erlang:fun_info(F, arity),
    curry(F, N, []).

curry(F, 0, Args) ->
    apply(F, lists:reverse(Args));
curry(F, N, Args) ->
    fun(Arg) -> curry(F, N-1, [Arg | Args]) end.

%% Usage
curried_add() ->
    CAdd = curry(fun(A, B) -> A + B end),
    Add5 = CAdd(5),
    Add5(3).    %% => 8
```

---

## 4. Transducers

```erlang
%% transducer.erl — composable transformations
-module(transducer).
-export([mapping/1, filtering/1, taking/1, transduce/3]).

%% Transducer: (reducer → reducer) — transformation of reducing function

mapping(F) ->
    fun(Reducer) ->
        fun(Acc, Item) -> Reducer(Acc, F(Item)) end
    end.

filtering(Pred) ->
    fun(Reducer) ->
        fun(Acc, Item) ->
            case Pred(Item) of
                true  -> Reducer(Acc, Item);
                false -> Acc
            end
        end
    end.

taking(N) ->
    Count = counters:new(1, []),
    fun(Reducer) ->
        fun(Acc, Item) ->
            counters:add(Count, 1, 1),
            case counters:get(Count, 1) =< N of
                true  -> Reducer(Acc, Item);
                false -> throw({halt, Acc})
            end
        end
    end.

%% Compose transducers (right-to-left application)
compose(Xforms) ->
    fun(Reducer) ->
        lists:foldl(fun(Xf, R) -> Xf(R) end, Reducer, lists:reverse(Xforms))
    end.

%% Run transducer over collection
transduce(Collection, Xform, Init) ->
    Reducer = fun(Acc, Item) -> [Item | Acc] end,
    TransducedReducer = Xform(Reducer),
    Result = try
        lists:foldl(fun(Item, Acc) ->
            TransducedReducer(Acc, Item)
        end, Init, Collection)
    catch
        throw:{halt, FinalAcc} -> FinalAcc
    end,
    lists:reverse(Result).

%% Usage: map + filter + take in one pass
example() ->
    Xform = compose([
        mapping(fun(X) -> X * 2 end),
        filtering(fun(X) -> X > 4 end),
        taking(3)
    ]),
    transduce([1, 2, 3, 4, 5, 6, 7, 8], Xform, []).
    %% => [6, 8, 10]
```

---

## 5. Recursive Patterns

```erlang
%% recursive.erl — advanced recursion patterns
-module(recursive).
-export([unfold/3, scan/3, zip_with/3, group_by/2,
         tree_map/2, tree_fold/3]).

%% Unfold: generate list from seed
unfold(Seed, Fun, Acc) ->
    case Fun(Seed) of
        done             -> lists:reverse(Acc);
        {Value, NewSeed} -> unfold(NewSeed, Fun, [Value | Acc])
    end.

%% Scan: like fold but keep all intermediate values
scan([], _, Acc) -> lists:reverse(Acc);
scan([H|T], F, [Prev|_] = Acc) ->
    scan(T, F, [F(Prev, H) | Acc]).

scan(List, Init, F) ->
    scan(List, F, [Init]).

%% ZipWith: combine two lists with a function
zip_with([], _, _) -> [];
zip_with(_, [], _) -> [];
zip_with([H1|T1], [H2|T2], F) ->
    [F(H1, H2) | zip_with(T1, T2, F)].

%% GroupBy: group list elements by key
group_by(List, KeyFun) ->
    lists:foldl(fun(Item, Acc) ->
        Key = KeyFun(Item),
        maps:update_with(Key, fun(Existing) -> [Item | Existing] end,
                         [Item], Acc)
    end, #{}, List).

%% Tree traversal (tree = {Value, [Children]})
tree_map({Value, Children}, F) ->
    {F(Value), [tree_map(C, F) || C <- Children]}.

tree_fold({Value, Children}, Acc, F) ->
    ChildAcc = lists:foldl(fun(C, A) -> tree_fold(C, A, F) end, Acc, Children),
    F(Value, ChildAcc).
```

---

## 6. Point-Free Style

```erlang
%% Using compose and partial for point-free programming
-module(pointfree).
-export([examples/0]).

examples() ->
    %% Traditional style
    Doubles = [X * 2 || X <- lists:seq(1, 5)],

    %% Point-free style
    Double  = fun(X) -> X * 2 end,
    IsEven  = fun(X) -> X rem 2 =:= 0 end,
    Process = fn:compose([
        fun(List) -> lists:filter(IsEven, List) end,
        fun(List) -> lists:map(Double, List) end
    ]),
    Result = Process(lists:seq(1, 10)),

    %% Pipe style (left-to-right)
    Result2 = fn:pipe(lists:seq(1, 10), [
        fun(List) -> lists:map(Double, List) end,
        fun(List) -> lists:filter(IsEven, List) end,
        fun(List) -> lists:sum(List) end
    ]),

    #{traditional => Doubles, composed => Result, piped => Result2}.
```

---

## 7. แบบฝึกหัด

1. Implement `result:do/2` macro-like function using parse_transform
2. สร้าง lazy version ของ `fn:map` ที่ไม่ evaluate จนกว่าจะถาม
3. เขียน `maybe` monad: `{just, V}` หรือ `nothing`
4. ใช้ transducers ประมวลผล CSV file ขนาด 1GB แบบ streaming

---

## สรุป Part 57

✅ Higher-order functions (compose, pipe, memoize, once)  
✅ Result monad สำหรับ error handling  
✅ Currying และ partial application  
✅ Transducers สำหรับ composable transformation  
✅ Advanced recursion (unfold, scan, group_by, tree)  
✅ Point-free style ด้วย compose/pipe  

---

*Part 57/100 | [← ก่อนหน้า](../part56/README.md) | [ถัดไป →](../part58/README.md)*
