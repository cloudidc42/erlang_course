# Part 05: Lists และ Recursion

> **"Lists are the fundamental data structure in functional programming"**  
> List คือโครงสร้างข้อมูลพื้นฐานในการเขียนโปรแกรมเชิงฟังก์ชัน

---

## สารบัญ

1. [List Internals](#1-list-internals)
2. [List Construction Patterns](#2-list-construction-patterns)
3. [Recursion Patterns](#3-recursion-patterns)
4. [Tail Recursion Patterns ขั้นสูง](#4-tail-recursion-patterns-ขั้นสูง)
5. [List Comprehensions ขั้นสูง](#5-list-comprehensions-ขั้นสูง)
6. [lists Module ทั้งหมด](#6-lists-module-ทั้งหมด)
7. [Sorting และ Searching](#7-sorting-และ-searching)
8. [List Algorithms](#8-list-algorithms)
9. [Performance Considerations](#9-performance-considerations)
10. [แบบฝึกหัด](#10-แบบฝึกหัด)

---

## 1. List Internals

### Structure ของ List

```
List ใน Erlang คือ Linked List ของ Cons Cells

[1, 2, 3] แสดงเป็น:
┌───┬───┐   ┌───┬───┐   ┌───┬───┐
│ 1 │ ●─┼──>│ 2 │ ●─┼──>│ 3 │ []│
└───┴───┘   └───┴───┘   └───┴───┘
  car  cdr   car  cdr   car  cdr

หรือเขียนแบบ Erlang:
[1 | [2 | [3 | []]]]
```

### Memory Layout

```erlang
%% แต่ละ cons cell ใช้ 2 words (16 bytes บน 64-bit)
%% List [1,2,3] = 3 cons cells = 48 bytes
%% + value ของแต่ละ element

%% เทียบกับ Tuple {1,2,3} = 4 words = 32 bytes (compact)
%% ดังนั้น Tuple ประหยัดกว่า List สำหรับ fixed-size data
```

### Head/Tail Operations

```erlang
%% hd/1 และ tl/1 เป็น O(1)
1> hd([1,2,3]).
1
2> tl([1,2,3]).
[2,3]

%% length/1 เป็น O(n)!
3> length([1,2,3,4,5]).
5

%% [H|T] destructuring เป็น O(1)
[H|T] = [1,2,3,4,5].
% H = 1, T = [2,3,4,5]

%% การ prepend เป็น O(1)
4> [0 | [1,2,3]].
[0,1,2,3]

%% การ append เป็น O(n) เพราะต้อง traverse ทั้ง list แรก
5> [1,2,3] ++ [4,5,6].
[1,2,3,4,5,6]  % expensive!
```

---

## 2. List Construction Patterns

### Building Lists Efficiently

```erlang
%% Pattern 1: Prepend แล้ว reverse (efficient)
build_list_efficient(N) ->
    build_list_efficient(N, []).

build_list_efficient(0, Acc) ->
    lists:reverse(Acc);
build_list_efficient(N, Acc) ->
    build_list_efficient(N-1, [N|Acc]).

1> build_list_efficient(5).
[1,2,3,4,5]

%% Pattern 2: Append (inefficient สำหรับ large lists)
build_list_slow(0) -> [];
build_list_slow(N) -> build_list_slow(N-1) ++ [N].

%% Pattern 3: List Comprehension
2> [X || X <- lists:seq(1, 5)].
[1,2,3,4,5]

%% Pattern 4: lists:seq/2
3> lists:seq(1, 10).
[1,2,3,4,5,6,7,8,9,10]
4> lists:seq(1, 10, 2).   % step 2
[1,3,5,7,9]
5> lists:seq(10, 1, -1).  % countdown
[10,9,8,7,6,5,4,3,2,1]
```

### Proper vs Improper Lists

```erlang
%% Proper list: ลงท้ายด้วย []
[1, 2, 3]           = [1|[2|[3|[]]]]  % proper
[]                  % proper (empty)

%% Improper list: ลงท้ายด้วยอย่างอื่น
[1 | 2]             % improper
[1, 2 | 3]          % improper

%% ตรวจสอบ
6> is_list([1,2,3]).
true
7> is_list([1|2]).
true  % is_list/1 ไม่ตรวจ proper/improper!

%% ตรวจ proper list
is_proper_list([]) -> true;
is_proper_list([_|T]) -> is_proper_list(T);
is_proper_list(_) -> false.

%% Improper list ใช้ใน special cases เช่น iolist
iolist_example() ->
    %% iolist: list, binary, หรือ nested iolist
    ["hello", <<" ">>, ["world", $!]].
    %% io:format, file:write รองรับ iolist โดยตรง
```

---

## 3. Recursion Patterns

### Basic Recursion

```erlang
%% Template สำหรับ list recursion
process_list([]) ->
    base_case_result;
process_list([H|T]) ->
    Result = process_element(H),
    Rest = process_list(T),
    combine(Result, Rest).

%% Examples
sum([]) -> 0;
sum([H|T]) -> H + sum(T).

product([]) -> 1;
product([H|T]) -> H * product(T).

max_of([X]) -> X;
max_of([H|T]) ->
    MaxRest = max_of(T),
    if H > MaxRest -> H; true -> MaxRest end.

%% หรือใช้ erlang:max/2
max_of2([X]) -> X;
max_of2([H|T]) -> erlang:max(H, max_of2(T)).
```

### Accumulator Pattern (Tail Recursive)

```erlang
%% Template
process_with_acc(List) ->
    process_with_acc(List, InitialAcc).

process_with_acc([], Acc) ->
    finalize(Acc);
process_with_acc([H|T], Acc) ->
    NewAcc = update_acc(Acc, H),
    process_with_acc(T, NewAcc).

%% Concrete examples
sum_tr(List) -> sum_tr(List, 0).
sum_tr([], Acc) -> Acc;
sum_tr([H|T], Acc) -> sum_tr(T, H + Acc).

product_tr(List) -> product_tr(List, 1).
product_tr([], Acc) -> Acc;
product_tr([H|T], Acc) -> product_tr(T, H * Acc).

reverse(List) -> reverse(List, []).
reverse([], Acc) -> Acc;
reverse([H|T], Acc) -> reverse(T, [H|Acc]).

%% Count elements matching predicate
count_if(Pred, List) -> count_if(Pred, List, 0).
count_if(_, [], Acc) -> Acc;
count_if(Pred, [H|T], Acc) ->
    case Pred(H) of
        true  -> count_if(Pred, T, Acc + 1);
        false -> count_if(Pred, T, Acc)
    end.
```

### Mutual Recursion

```erlang
%% สองฟังก์ชัน recursive ที่เรียกกันเอง

%% ตัวอย่าง: even/odd
is_even(0) -> true;
is_even(N) when N > 0 -> is_odd(N - 1).

is_odd(0) -> false;
is_odd(N) when N > 0 -> is_even(N - 1).

1> is_even(4).
true
2> is_odd(7).
true

%% ตัวอย่าง: flatten nested list
flatten([]) -> [];
flatten([H|T]) when is_list(H) ->
    flatten(H) ++ flatten(T);
flatten([H|T]) ->
    [H | flatten(T)].

3> flatten([[1,2],[3,[4,5]],6]).
[1,2,3,4,5,6]
```

### Tree Recursion

```erlang
%% Traverse tree structure
-type tree(T) :: {node, T, tree(T), tree(T)} | nil.

%% Count nodes
count_nodes(nil) -> 0;
count_nodes({node, _, Left, Right}) ->
    1 + count_nodes(Left) + count_nodes(Right).

%% Sum all values
sum_tree(nil) -> 0;
sum_tree({node, Val, Left, Right}) ->
    Val + sum_tree(Left) + sum_tree(Right).

%% Find depth
depth(nil) -> 0;
depth({node, _, Left, Right}) ->
    1 + erlang:max(depth(Left), depth(Right)).

%% In-order traversal
inorder(nil) -> [];
inorder({node, Val, Left, Right}) ->
    inorder(Left) ++ [Val] ++ inorder(Right).

%% Build balanced BST from sorted list
build_bst([]) -> nil;
build_bst(List) ->
    Mid = length(List) div 2,
    {Left, [Value|Right]} = lists:split(Mid, List),
    {node, Value, build_bst(Left), build_bst(Right)}.

%% ทดสอบ
1> T = build_bst([1,2,3,4,5,6,7]).
{node,4,{node,2,{node,1,nil,nil},{node,3,nil,nil}},
      {node,6,{node,5,nil,nil},{node,7,nil,nil}}}

2> inorder(T).
[1,2,3,4,5,6,7]

3> depth(T).
3
```

---

## 4. Tail Recursion Patterns ขั้นสูง

### CPS (Continuation-Passing Style)

```erlang
%% CPS ทำให้ทุก function เป็น tail call

%% Normal fibonacci
fib(0) -> 0;
fib(1) -> 1;
fib(N) -> fib(N-1) + fib(N-2).

%% CPS fibonacci (tail recursive!)
fib_cps(N) -> fib_cps(N, fun(X) -> X end).

fib_cps(0, K) -> K(0);
fib_cps(1, K) -> K(1);
fib_cps(N, K) ->
    fib_cps(N-1, fun(V1) ->
        fib_cps(N-2, fun(V2) ->
            K(V1 + V2)
        end)
    end).

%% แต่ CPS fibonacci ไม่ได้ช่วยเรื่อง stack (ยังใช้ heap)
%% สำหรับ fibonacci ดีที่สุดคือ iterative approach
fib_iter(N) -> fib_iter(N, 0, 1).
fib_iter(0, A, _) -> A;
fib_iter(N, A, B) -> fib_iter(N-1, B, A+B).

1> fib_iter(30).
832040
```

### Work Queue Pattern

```erlang
%% Process items ด้วย work queue (BFS)
bfs_traverse(Graph, StartNode) ->
    bfs_traverse(Graph, [StartNode], [], []).

bfs_traverse(_, [], _, Visited) ->
    lists:reverse(Visited);
bfs_traverse(Graph, [Node|Queue], Seen, Visited) ->
    case lists:member(Node, Seen) of
        true ->
            bfs_traverse(Graph, Queue, Seen, Visited);
        false ->
            Neighbors = get_neighbors(Graph, Node),
            NewQueue = Queue ++ Neighbors,
            bfs_traverse(Graph, NewQueue, [Node|Seen], [Node|Visited])
    end.
```

### Chunk Processing

```erlang
%% Process list เป็น chunks
process_in_chunks(List, ChunkSize) ->
    process_in_chunks(List, ChunkSize, []).

process_in_chunks([], _, Results) ->
    lists:reverse(Results);
process_in_chunks(List, ChunkSize, Results) ->
    {Chunk, Rest} = safe_split(List, ChunkSize),
    Result = process_chunk(Chunk),
    process_in_chunks(Rest, ChunkSize, [Result|Results]).

safe_split(List, N) when length(List) >= N ->
    lists:split(N, List);
safe_split(List, _) ->
    {List, []}.

%% ทดสอบ
process_chunk(Chunk) ->
    lists:sum(Chunk).

1> process_in_chunks(lists:seq(1, 10), 3).
[6, 15, 24, 10]
% [1+2+3, 4+5+6, 7+8+9, 10]
```

---

## 5. List Comprehensions ขั้นสูง

### Multiple Generators

```erlang
%% Cartesian product
1> [{X, Y} || X <- [1,2,3], Y <- [a,b]].
[{1,a},{1,b},{2,a},{2,b},{3,a},{3,b}]

%% Nested comprehension
2> [[X*Y || Y <- [1,2,3]] || X <- [1,2,3]].
[[1,2,3],[2,4,6],[3,6,9]]

%% Matrix multiplication
matrix_mult(A, B) ->
    BT = transpose(B),
    [[dot_product(Row, Col) || Col <- BT] || Row <- A].

transpose([[]|_]) -> [];
transpose(Matrix) ->
    [lists:map(fun hd/1, Matrix) |
     transpose(lists:map(fun tl/1, Matrix))].

dot_product(V1, V2) ->
    lists:sum([X*Y || {X,Y} <- lists:zip(V1, V2)]).

3> A = [[1,2],[3,4]].
4> B = [[5,6],[7,8]].
5> matrix_mult(A, B).
[[19,22],[43,50]]
```

### Guards ใน Comprehensions

```erlang
%% Filter ใน comprehension
1> [X || X <- lists:seq(1, 20), X rem 3 =:= 0].
[3,6,9,12,15,18]

%% หลาย conditions
2> [X || X <- lists:seq(1, 100),
         X rem 3 =:= 0,
         X rem 5 =:= 0].
[15,30,45,60,75,90]

%% Pattern matching ใน generator
3> Numbers = [{1, odd}, {2, even}, {3, odd}, {4, even}].
4> [N || {N, odd} <- Numbers].
[1,3]

%% Flatmap (map + flatten)
5> [Y || X <- [[1,2],[3,4],[5,6]], Y <- X].
[1,2,3,4,5,6]
```

### Binary Comprehensions

```erlang
%% Binary comprehension: << expr || bin-gen, filter >>

%% Convert list to binary
1> << <<X>> || X <- [72, 101, 108, 108, 111] >>.
<<"Hello">>

%% Filter binary content
2> << X || <<X>> <= <<"Hello World">>, X =/= $  >>.
<<"HelloWorld">>

%% Uppercase (ASCII only)
to_upper(Bin) ->
    << (if X >= $a, X =< $z -> X - 32; true -> X end)
       || <<X>> <= Bin >>.

3> to_upper(<<"hello world">>).
<<"HELLO WORLD">>

%% Parse binary data
parse_bytes(Bin) ->
    [X || <<X>> <= Bin].

4> parse_bytes(<<1,2,3,4,5>>).
[1,2,3,4,5]
```

---

## 6. lists Module ทั้งหมด

### Basic Operations

```erlang
%% lists:append/2, /1
1> lists:append([1,2], [3,4]).
[1,2,3,4]
2> lists:append([[1,2], [3,4], [5,6]]).
[1,2,3,4,5,6]

%% lists:flatten/1, /2
3> lists:flatten([[1,[2,3]],[4,[5,[6]]]]).
[1,2,3,4,5,6]
4> lists:flatten([[1,[2,3]],[4]], [5,6]).  % กับ tail
[1,2,3,4,5,6]

%% lists:reverse/1, /2
5> lists:reverse([1,2,3]).
[3,2,1]
6> lists:reverse([1,2,3], [4,5]).  % reverse + prepend tail
[3,2,1,4,5]

%% lists:nth/2
7> lists:nth(3, [a,b,c,d,e]).
c

%% lists:last/1
8> lists:last([1,2,3,4,5]).
5

%% lists:droplast/1
9> lists:droplast([1,2,3,4,5]).
[1,2,3,4]

%% lists:sublist/2, /3
10> lists:sublist([a,b,c,d,e], 3).
[a,b,c]
11> lists:sublist([a,b,c,d,e], 2, 3).
[b,c,d]
```

### Search Operations

```erlang
%% lists:member/2
1> lists:member(3, [1,2,3,4,5]).
true
2> lists:member(6, [1,2,3,4,5]).
false

%% lists:search/2 (ค้นหาแบบ complex)
3> lists:search(fun(X) -> X > 3 end, [1,2,3,4,5]).
{value, 4}
4> lists:search(fun(X) -> X > 10 end, [1,2,3,4,5]).
false

%% lists:keyfind/3 (สำหรับ list of tuples)
5> lists:keyfind(b, 1, [{a,1},{b,2},{c,3}]).
{b,2}
6> lists:keyfind(99, 2, [{a,1},{b,2},{c,3}]).
false

%% lists:keysearch/3
7> lists:keysearch(b, 1, [{a,1},{b,2},{c,3}]).
{value, {b,2}}

%% lists:keyember/3
8> lists:keymember(b, 1, [{a,1},{b,2},{c,3}]).
true
```

### Transformation Operations

```erlang
%% lists:map/2
1> lists:map(fun(X) -> X * 2 end, [1,2,3]).
[2,4,6]

%% lists:flatmap/2 (map + flatten)
2> lists:flatmap(fun(X) -> [X, X*2] end, [1,2,3]).
[1,2,2,4,3,6]

%% lists:filtermap/2
3> lists:filtermap(fun(X) ->
     if X rem 2 =:= 0 -> {true, X div 2};
        true -> false
     end
   end, [1,2,3,4,5,6]).
[1,2,3]

%% lists:foldl/3, foldr/3
4> lists:foldl(fun(X, Acc) -> X + Acc end, 0, [1,2,3,4,5]).
15
5> lists:foldr(fun(X, Acc) -> [X*2|Acc] end, [], [1,2,3]).
[2,4,6]

%% lists:mapfoldl/3
6> lists:mapfoldl(fun(X, Acc) -> {X*2, Acc+X} end, 0, [1,2,3,4,5]).
{[2,4,6,8,10], 15}

%% lists:partition/2
7> lists:partition(fun(X) -> X rem 2 =:= 0 end, [1,2,3,4,5,6]).
{[2,4,6],[1,3,5]}

%% lists:unzip/1
8> lists:unzip([{1,a},{2,b},{3,c}]).
{[1,2,3],[a,b,c]}

%% lists:zip/2, zip3/3
9> lists:zip([1,2,3],[a,b,c]).
[{1,a},{2,b},{3,c}]

10> lists:zip3([1,2,3],[a,b,c],[x,y,z]).
[{1,a,x},{2,b,y},{3,c,z}]

%% lists:zipwith/3
11> lists:zipwith(fun(X,Y) -> X+Y end, [1,2,3],[10,20,30]).
[11,22,33]
```

### Sort Operations

```erlang
%% lists:sort/1, /2
1> lists:sort([3,1,4,1,5,9,2,6]).
[1,1,2,3,4,5,6,9]

2> lists:sort(fun(A,B) -> A > B end, [3,1,4,1,5]).
[9,5,4,3,1,1]   %% sort ใน documentation บอกลำดับ descending

%% lists:usort/1, /2 (unique sort)
3> lists:usort([3,1,4,1,5,9,2,6]).
[1,2,3,4,5,6,9]  %% ลบ duplicate!

%% lists:keysort/2
4> lists:keysort(2, [{a,3},{b,1},{c,2}]).
[{b,1},{c,2},{a,3}]

%% lists:ukeysort/2
5> lists:ukeysort(1, [{a,3},{a,1},{b,2}]).
[{a,3},{b,2}]  %% เก็บ first occurrence

%% lists:merge/2
6> lists:merge([1,3,5],[2,4,6]).
[1,2,3,4,5,6]
```

### Set Operations

```erlang
%% lists:subtract/2
1> lists:subtract([1,2,3,4,5], [2,4]).
[1,3,5]

%% lists:usort สำหรับ unique elements
unique(List) -> lists:usort(List).

%% Manual set operations
union(A, B) -> lists:usort(A ++ B).
intersection(A, B) -> [X || X <- A, lists:member(X, B)].
difference(A, B) -> [X || X <- A, not lists:member(X, B)].

2> union([1,2,3],[2,3,4]).
[1,2,3,4]
3> intersection([1,2,3],[2,3,4]).
[2,3]
4> difference([1,2,3],[2,3,4]).
[1]
```

---

## 7. Sorting และ Searching

### Sorting Algorithms

```erlang
%% Merge Sort (functional style)
merge_sort([]) -> [];
merge_sort([X]) -> [X];
merge_sort(List) ->
    {Left, Right} = lists:split(length(List) div 2, List),
    merge(merge_sort(Left), merge_sort(Right)).

merge([], Right) -> Right;
merge(Left, []) -> Left;
merge([H1|T1] = Left, [H2|T2] = Right) ->
    if H1 =< H2 -> [H1 | merge(T1, Right)];
       true      -> [H2 | merge(Left, T2)]
    end.

1> merge_sort([5,2,8,1,9,3,7,4,6]).
[1,2,3,4,5,6,7,8,9]

%% Quick Sort
quicksort([]) -> [];
quicksort([Pivot|Rest]) ->
    Smaller = [X || X <- Rest, X < Pivot],
    Larger  = [X || X <- Rest, X >= Pivot],
    quicksort(Smaller) ++ [Pivot] ++ quicksort(Larger).

2> quicksort([5,2,8,1,9,3,7,4,6]).
[1,2,3,4,5,6,7,8,9]
```

### Binary Search

```erlang
%% Binary search ใน sorted list
binary_search(List, Target) ->
    binary_search(list_to_tuple(List), Target, 1, length(List)).

binary_search(_, _, Low, High) when Low > High ->
    not_found;
binary_search(Arr, Target, Low, High) ->
    Mid = (Low + High) div 2,
    MidVal = element(Mid, Arr),
    if MidVal =:= Target -> {found, Mid};
       MidVal < Target   -> binary_search(Arr, Target, Mid+1, High);
       true              -> binary_search(Arr, Target, Low, Mid-1)
    end.

1> binary_search([1,3,5,7,9,11,13,15], 7).
{found, 4}
2> binary_search([1,3,5,7,9,11,13,15], 6).
not_found
```

---

## 8. List Algorithms

### Common Algorithms

```erlang
%% ============================================================
%% Group By
%% ============================================================
group_by(Fun, List) ->
    lists:foldl(
        fun(Item, Acc) ->
            Key = Fun(Item),
            maps:update_with(Key, fun(V) -> [Item|V] end, [Item], Acc)
        end,
        #{},
        List
    ).

1> group_by(fun(X) -> X rem 3 end, [1,2,3,4,5,6,7,8,9]).
#{0 => [9,6,3], 1 => [7,4,1], 2 => [8,5,2]}

%% ============================================================
%% Sliding Window
%% ============================================================
sliding_window(List, Size) ->
    sliding_window(List, Size, []).

sliding_window(List, Size, Acc) when length(List) < Size ->
    lists:reverse(Acc);
sliding_window(List, Size, Acc) ->
    Window = lists:sublist(List, Size),
    sliding_window(tl(List), Size, [Window|Acc]).

2> sliding_window([1,2,3,4,5,6], 3).
[[1,2,3],[2,3,4],[3,4,5],[4,5,6]]

%% ============================================================
%% Running Sum
%% ============================================================
running_sum(List) ->
    element(1, lists:mapfoldl(
        fun(X, Acc) -> {X + Acc, X + Acc} end,
        0, List
    )).

3> running_sum([1,2,3,4,5]).
[1,3,6,10,15]

%% ============================================================
%% Find Duplicates
%% ============================================================
find_duplicates(List) ->
    find_duplicates(lists:sort(List), []).

find_duplicates([], Dups) -> lists:usort(Dups);
find_duplicates([_], Dups) -> lists:usort(Dups);
find_duplicates([X,X|T], Dups) ->
    find_duplicates([X|T], [X|Dups]);
find_duplicates([_|T], Dups) ->
    find_duplicates(T, Dups).

4> find_duplicates([1,2,3,2,4,3,5,1]).
[1,2,3]

%% ============================================================
%% Rotate List
%% ============================================================
rotate_left([], _) -> [];
rotate_left(List, N) ->
    Len = length(List),
    Shift = N rem Len,
    {Front, Back} = lists:split(Shift, List),
    Back ++ Front.

rotate_right(List, N) ->
    rotate_left(List, length(List) - (N rem length(List))).

5> rotate_left([1,2,3,4,5], 2).
[3,4,5,1,2]
6> rotate_right([1,2,3,4,5], 2).
[4,5,1,2,3]

%% ============================================================
%% Transpose (matrix)
%% ============================================================
transpose([]) -> [];
transpose([[]|_]) -> [];
transpose(Matrix) ->
    [lists:map(fun hd/1, Matrix) | 
     transpose(lists:map(fun tl/1, Matrix))].

7> transpose([[1,2,3],[4,5,6],[7,8,9]]).
[[1,4,7],[2,5,8],[3,6,9]]

%% ============================================================
%% Chunk
%% ============================================================
chunk([], _) -> [];
chunk(List, N) when length(List) =< N -> [List];
chunk(List, N) ->
    {Chunk, Rest} = lists:split(N, List),
    [Chunk | chunk(Rest, N)].

8> chunk([1,2,3,4,5,6,7], 3).
[[1,2,3],[4,5,6],[7]]

%% ============================================================
%% Interleave
%% ============================================================
interleave([], _) -> [];
interleave(_, []) -> [];
interleave([H1|T1], [H2|T2]) ->
    [H1, H2 | interleave(T1, T2)].

9> interleave([1,3,5], [2,4,6]).
[1,2,3,4,5,6]
```

---

## 9. Performance Considerations

### List Operations Complexity

```
Operation             Complexity  Notes
────────────────────────────────────────────────────
length(List)          O(n)        traverse ทั้ง list
[H|T]                 O(1)        pattern match
hd(List)              O(1)        
tl(List)              O(1)        
[X|List] (prepend)    O(1)        efficient!
List ++ [X] (append)  O(n)        traverse List
lists:reverse/1       O(n)        
lists:nth/2           O(n)        ไม่ efficient
lists:sort/1          O(n log n)  
lists:member/2        O(n)        
lists:map/2           O(n)        
lists:filter/2        O(n)        
lists:foldl/3         O(n)        
```

### Tips สำหรับ Performance

```erlang
%% TIP 1: Build list ด้วย prepend แล้ว reverse
%% WRONG: ต่อท้าย O(n) ต่อครั้ง
build_wrong(N) ->
    build_wrong(N, []).
build_wrong(0, Acc) -> Acc;
build_wrong(N, Acc) ->
    build_wrong(N-1, Acc ++ [N]).  %% O(n^2) total!

%% CORRECT: prepend แล้ว reverse ตอนสุดท้าย
build_correct(N) ->
    build_correct(N, []).
build_correct(0, Acc) -> lists:reverse(Acc);  %% reverse ครั้งเดียว
build_correct(N, Acc) ->
    build_correct(N-1, [N|Acc]).  %% O(n) total

%% TIP 2: ใช้ tuple แทน list สำหรับ fixed-size data
%% List [1,2,3]  = 3 cons cells + 3 values (overhead มาก)
%% Tuple {1,2,3} = 1 allocation (compact มาก)

%% TIP 3: ใช้ binary แทน string
%% "hello" = [h,e,l,l,o] = 5 cons cells = ~40 bytes
%% <<"hello">> = 5 bytes

%% TIP 4: Avoid repeated length/1 calls
bad_example(List) ->
    N = length(List),
    if N > 100 ->
        process_large(List, N);  %% N = length ครั้งแรก
    true ->
        process_small(List, N)
    end.

good_example(List) ->
    case length(List) of  %% เรียก length ครั้งเดียว
        N when N > 100 -> process_large(List, N);
        N -> process_small(List, N)
    end.

%% TIP 5: ใช้ lists:foldl สำหรับ accumulation
%% แทน recursive function ที่เขียนเอง (likely มี bug)
sum_good(List) ->
    lists:foldl(fun(X, Acc) -> X + Acc end, 0, List).
```

### Benchmarking

```erlang
%% Benchmark ด้วย timer:tc
benchmark(Fun, Args) ->
    {Time, Result} = timer:tc(erlang, apply, [Fun, Args]),
    io:format("Time: ~p microseconds~n", [Time]),
    Result.

%% ทดสอบ
1> benchmark(fun lists:sort/1, [lists:seq(1000, 1, -1)]).
Time: 234 microseconds
[1,2,...,1000]

%% Benchmark แบบ repeat
benchmark_n(Fun, Args, N) ->
    Times = [element(1, timer:tc(erlang, apply, [Fun, Args])) 
             || _ <- lists:seq(1, N)],
    Sum = lists:sum(Times),
    #{
        total_us    => Sum,
        avg_us      => Sum div N,
        min_us      => lists:min(Times),
        max_us      => lists:max(Times),
        n           => N
    }.

2> benchmark_n(fun lists:sort/1, [lists:seq(1000, 1, -1)], 100).
#{total_us => 23400, avg_us => 234, min_us => 180, max_us => 520, n => 100}
```

---

## 10. แบบฝึกหัด

### Exercise 1: List Operations Library

```erlang
%% สร้าง list_ops.erl ที่ implement:
%% - take/2, drop/2       : เอา/ทิ้ง N elements แรก
%% - take_while/2         : เอา elements จนกว่า predicate ผิด
%% - drop_while/2         : ทิ้ง elements จนกว่า predicate ผิด
%% - span/2               : แบ่งเป็น 2 list ด้วย predicate
%% - group_consecutive/1  : group elements ที่ติดกัน

-module(list_ops).
-export([take/2, drop/2, take_while/2, drop_while/2, 
         span/2, group_consecutive/1]).

take(List, N) -> take(List, N, []).
take([], _, Acc) -> lists:reverse(Acc);
take(_, 0, Acc) -> lists:reverse(Acc);
take([H|T], N, Acc) -> take(T, N-1, [H|Acc]).

drop(List, 0) -> List;
drop([], _) -> [];
drop([_|T], N) -> drop(T, N-1).

take_while([], _) -> [];
take_while([H|T], Pred) ->
    case Pred(H) of
        true  -> [H | take_while(T, Pred)];
        false -> []
    end.

drop_while([], _) -> [];
drop_while([H|T] = List, Pred) ->
    case Pred(H) of
        true  -> drop_while(T, Pred);
        false -> List
    end.

span(List, Pred) ->
    {take_while(List, Pred), drop_while(List, Pred)}.

group_consecutive([]) -> [];
group_consecutive([H|T]) -> 
    group_consecutive(T, H, [H], []).

group_consecutive([], _, Group, Groups) ->
    lists:reverse([lists:reverse(Group)|Groups]);
group_consecutive([H|T], Prev, Group, Groups) when H =:= Prev + 1 ->
    group_consecutive(T, H, [H|Group], Groups);
group_consecutive([H|T], _, Group, Groups) ->
    group_consecutive(T, H, [H], [lists:reverse(Group)|Groups]).

%% ทดสอบ
%% 1> list_ops:take([1,2,3,4,5], 3).
%% [1,2,3]
%% 2> list_ops:group_consecutive([1,2,3,5,6,8,9,10]).
%% [[1,2,3],[5,6],[8,9,10]]
```

### Exercise 2: Statistics Module

```erlang
%% สร้าง stats.erl ที่คำนวณ:
%% - mean/1    : ค่าเฉลี่ย
%% - median/1  : ค่ามัธยฐาน
%% - mode/1    : ค่าที่พบบ่อยที่สุด
%% - std_dev/1 : Standard deviation
%% - variance/1: Variance

-module(stats).
-export([mean/1, median/1, mode/1, variance/1, std_dev/1]).

mean([]) -> undefined;
mean(List) ->
    lists:sum(List) / length(List).

median([]) -> undefined;
median(List) ->
    Sorted = lists:sort(List),
    N = length(Sorted),
    case N rem 2 of
        1 -> lists:nth((N + 1) div 2, Sorted);
        0 ->
            M1 = lists:nth(N div 2, Sorted),
            M2 = lists:nth(N div 2 + 1, Sorted),
            (M1 + M2) / 2
    end.

mode([]) -> undefined;
mode(List) ->
    Counts = lists:foldl(
        fun(X, Acc) ->
            maps:update_with(X, fun(V) -> V+1 end, 1, Acc)
        end,
        #{}, List
    ),
    {Mode, _} = maps:fold(
        fun(K, V, {BestK, BestV}) ->
            if V > BestV -> {K, V}; true -> {BestK, BestV} end
        end,
        {undefined, 0},
        Counts
    ),
    Mode.

variance([]) -> undefined;
variance(List) ->
    Avg = mean(List),
    SumSquaredDiffs = lists:sum(
        [(X - Avg) * (X - Avg) || X <- List]
    ),
    SumSquaredDiffs / length(List).

std_dev(List) ->
    math:sqrt(variance(List)).

%% ทดสอบ
%% 1> stats:mean([1,2,3,4,5]).
%% 3.0
%% 2> stats:median([1,2,3,4,5]).
%% 3
%% 3> stats:mode([1,2,2,3,3,3,4]).
%% 3
%% 4> stats:std_dev([2,4,4,4,5,5,7,9]).
%% 2.0
```

### Exercise 3: Graph Algorithms

```erlang
%% สร้าง graph.erl ที่ implement:
%% - bfs/2  : Breadth-First Search
%% - dfs/2  : Depth-First Search
%% - shortest_path/3 : หา shortest path (BFS)
%% - has_cycle/1 : ตรวจสอบว่า graph มี cycle

%% Graph represented as #{Node => [Neighbors]}

-module(graph).
-export([bfs/2, dfs/2, shortest_path/3]).

bfs(Graph, Start) ->
    bfs(Graph, [Start], sets:new(), []).

bfs(_, [], _, Visited) ->
    lists:reverse(Visited);
bfs(Graph, [Node|Queue], Seen, Visited) ->
    case sets:is_element(Node, Seen) of
        true ->
            bfs(Graph, Queue, Seen, Visited);
        false ->
            Neighbors = maps:get(Node, Graph, []),
            NewQueue = Queue ++ Neighbors,
            NewSeen = sets:add_element(Node, Seen),
            bfs(Graph, NewQueue, NewSeen, [Node|Visited])
    end.

dfs(Graph, Start) ->
    dfs(Graph, [Start], sets:new(), []).

dfs(_, [], _, Visited) ->
    lists:reverse(Visited);
dfs(Graph, [Node|Stack], Seen, Visited) ->
    case sets:is_element(Node, Seen) of
        true ->
            dfs(Graph, Stack, Seen, Visited);
        false ->
            Neighbors = maps:get(Node, Graph, []),
            NewStack = Neighbors ++ Stack,  %% DFS: prepend
            NewSeen = sets:add_element(Node, Seen),
            dfs(Graph, NewStack, NewSeen, [Node|Visited])
    end.

shortest_path(Graph, Start, End) ->
    bfs_path(Graph, [{Start, [Start]}], sets:new(), End).

bfs_path(_, [], _, _) ->
    not_found;
bfs_path(Graph, [{Node, Path}|Queue], Seen, End) ->
    case Node =:= End of
        true -> lists:reverse(Path);
        false ->
            case sets:is_element(Node, Seen) of
                true -> bfs_path(Graph, Queue, Seen, End);
                false ->
                    Neighbors = maps:get(Node, Graph, []),
                    NewPaths = [{N, [N|Path]} || N <- Neighbors,
                                                 not sets:is_element(N, Seen)],
                    NewSeen = sets:add_element(Node, Seen),
                    bfs_path(Graph, Queue ++ NewPaths, NewSeen, End)
            end
    end.

%% ทดสอบ
%% G = #{a => [b,c], b => [d], c => [d,e], d => [f], e => [f], f => []}.
%% 1> graph:bfs(G, a).
%% [a,b,c,d,e,f]
%% 2> graph:shortest_path(G, a, f).
%% [a,b,d,f] หรือ [a,c,d,f]
```

---

## สรุป Part 05

ใน Part นี้คุณได้เรียนรู้:

✅ List internals — linked list, cons cells  
✅ List construction patterns — prepend + reverse  
✅ Proper vs improper lists  
✅ Recursion patterns — basic, accumulator, mutual, tree  
✅ Tail recursion ขั้นสูง — CPS, work queue, chunking  
✅ List comprehensions — generators, guards, binary  
✅ lists module ทั้งหมด — basic, search, transform, sort  
✅ Sorting algorithms — merge sort, quick sort, binary search  
✅ List algorithms — group by, sliding window, chunk, interleave  
✅ Performance considerations  

---

## ต่อไป: [Part 06 — Tuples, Maps และ Records](../part06/README.md)

ใน Part ถัดไปเราจะเรียนรู้:
- Tuple operations ขั้นสูง
- Map operations ขั้นสูง
- Records — structured data
- proplist
- Comparison และ เลือกใช้ structure ที่เหมาะสม

---

*Part 05/100 | [← ก่อนหน้า](../part04/README.md) | [ถัดไป →](../part06/README.md)*
