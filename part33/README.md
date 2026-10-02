# Part 33: Performance Optimization

> **"Measure first, optimize second — never guess"**  
> วัดก่อน optimize ทีหลัง — อย่าเดา

---

## สารบัญ

1. [Performance Profiling Workflow](#1-performance-profiling-workflow)
2. [Benchmarking ด้วย timer:tc](#2-benchmarking-ด้วย-timertc)
3. [Memory Optimization](#3-memory-optimization)
4. [Binary Optimization](#4-binary-optimization)
5. [ETS vs Process State](#5-ets-vs-process-state)
6. [Tail Recursion Patterns](#6-tail-recursion-patterns)
7. [Compiler Optimizations](#7-compiler-optimizations)
8. [Process Optimization](#8-process-optimization)
9. [แบบฝึกหัด](#9-แบบฝึกหัด)

---

## 1. Performance Profiling Workflow

```
Workflow:
1. Measure: เก็บ baseline metrics
2. Profile: หา hotspot ด้วย fprof/eprof
3. Analyze: เข้าใจว่าอะไรช้า
4. Optimize: แก้ hotspot
5. Measure again: ยืนยัน improvement
6. Repeat ถ้าจำเป็น

Tools:
- timer:tc          — micro-benchmark
- fprof             — call graph profiler
- eprof             — per-function time profiler
- cprof             — call count profiler (ต่ำสุด overhead)
- recon_alloc       — memory allocator analysis
- erlang:statistics — VM statistics
- observer          — GUI overview
```

---

## 2. Benchmarking ด้วย timer:tc

```erlang
%% timer:tc/1,2,3 — วัดเวลา
bench(Fun) ->
    {Micros, Result} = timer:tc(Fun),
    io:format("Time: ~p µs (~p ms)~n",
              [Micros, Micros div 1000]),
    Result.

bench(Fun, N) ->
    Times = [begin {T, _} = timer:tc(Fun), T end
             || _ <- lists:seq(1, N)],
    Sorted = lists:sort(Times),
    Min  = hd(Sorted),
    Max  = lists:last(Sorted),
    Mean = lists:sum(Times) div N,
    P50  = lists:nth(N div 2, Sorted),
    P95  = lists:nth(round(N * 0.95), Sorted),
    P99  = lists:nth(round(N * 0.99), Sorted),
    io:format("N=~p Min=~pµs Max=~pµs Mean=~pµs P50=~pµs P95=~pµs P99=~pµs~n",
              [N, Min, Max, Mean, P50, P95, P99]).

%% เปรียบเทียบ 2 implementations
compare(Fun1, Fun2, N) ->
    T1 = bench_avg(Fun1, N),
    T2 = bench_avg(Fun2, N),
    Ratio = T1 / max(T2, 1),
    io:format("Fun1: ~pµs | Fun2: ~pµs | Ratio: ~.2f~n",
              [T1, T2, Ratio]).

bench_avg(Fun, N) ->
    Times = [begin {T, _} = timer:tc(Fun), T end
             || _ <- lists:seq(1, N)],
    lists:sum(Times) div N.
```

---

## 3. Memory Optimization

```erlang
%% ตรวจสอบ memory usage
erlang:memory().
%% [{total, 50MB}, {processes, 30MB}, {ets, 5MB}, {binary, 10MB}, ...]

erlang:memory(processes).
erlang:memory(binary).

%% Process memory
process_info(self(), memory).        %% bytes
process_info(self(), heap_size).     %% words
process_info(self(), stack_size).    %% words
process_info(self(), binary).        %% [{BinId, Size, Count}]

%% หา processes ที่กิน memory มากที่สุด
top_memory_processes(N) ->
    All = processes(),
    WithMem = [{process_info(P, memory), P} || P <- All,
               process_info(P, memory) =/= undefined],
    Sorted = lists:sort(fun({M1,_},{M2,_}) -> M1 > M2 end, WithMem),
    lists:sublist(Sorted, N).

%% Binary memory หลุดรั่ว
%% Large binaries (>64 bytes) จะไม่ GC ทันที
%% ต้องรอ major GC

%% Force GC
erlang:garbage_collect(Pid).

%% ป้องกัน binary leak ใน long-running processes
defragment_binary(Pid) ->
    erlang:garbage_collect(Pid).

%% ใช้ binary:copy เมื่อ sub-binary จาก binary ใหญ่
good_practice() ->
    LargeBin = read_file(),
    SmallBin = binary:part(LargeBin, 0, 100),
    %% LargeBin ยังไม่ GC เพราะ SmallBin reference อยู่!

    %% Fix: copy ออกมา
    SmallBinCopy = binary:copy(SmallBin),
    %% ตอนนี้ LargeBin GC ได้
    {SmallBinCopy}.

read_file() -> <<>>.  %% placeholder
```

---

## 4. Binary Optimization

```erlang
%% Binary Performance Tips:

%% 1. ใช้ binary matching แทน list operations
%% BAD: convert to list
parse_slow(Bin) ->
    List = binary_to_list(Bin),
    string:split(List, ",").

%% GOOD: binary matching
parse_fast(Bin) ->
    binary:split(Bin, <<",">>, [global]).

%% 2. Build binaries ด้วย iolist แทน <<>>
%% BAD: O(n²) concatenation
build_slow(Parts) ->
    lists:foldl(fun(P, Acc) -> <<Acc/binary, P/binary>> end, <<>>, Parts).

%% GOOD: iolist_to_binary
build_fast(Parts) ->
    iolist_to_binary(Parts).

%% 3. Use binary comprehension
bytes_to_hex(Bin) ->
    <<begin
        H = N bsr 4,
        L = N band 15,
        HChar = if H < 10 -> H + $0; true -> H - 10 + $a end,
        LChar = if L < 10 -> L + $0; true -> L - 10 + $a end,
        <<HChar, LChar>>
      end || <<N>> <= Bin>>.

%% 4. Pattern match on binaries (very fast)
is_http_get(<<$G, $E, $T, $\s, _/binary>>) -> true;
is_http_get(_) -> false.

%% 5. ใช้ binary:match และ binary:split
count_lines(Bin) ->
    length(binary:split(Bin, <<"\n">>, [global])).

find_header(Bin, Header) ->
    Pattern = <<Header/binary, ": ">>,
    case binary:match(Bin, Pattern) of
        {Start, Len} ->
            Rest = binary:part(Bin, Start + Len, byte_size(Bin) - Start - Len),
            {ok, binary:part(Rest, 0, find_newline(Rest))};
        nomatch ->
            not_found
    end.

find_newline(Bin) ->
    case binary:match(Bin, <<"\r\n">>) of
        {Pos, _} -> Pos;
        nomatch   -> byte_size(Bin)
    end.
```

---

## 5. ETS vs Process State

```erlang
%% เมื่อไหรควรใช้ ETS แทน Process State

%% Process State (GenServer):
%% + Simple, safe, OTP-idiomatic
%% + State isolation
%% - Single-process bottleneck
%% - ต้อง message pass สำหรับทุก access

%% ETS:
%% + Concurrent reads (read_concurrency)
%% + Concurrent writes (write_concurrency)
%% + ไม่ต้องผ่าน process
%% - No transactions
%% - Global shared state

%% Use ETS when:
%% - High-read, low-write (caches)
%% - Many processes need same data
%% - Speed > safety

%% Benchmark ตัวอย่าง
benchmark_comparison() ->
    %% ETS read
    ets:new(bench, [named_table, set, {read_concurrency, true}]),
    ets:insert(bench, {key, value}),
    {T1, _} = timer:tc(fun() ->
        [ets:lookup(bench, key) || _ <- lists:seq(1, 100000)]
    end),

    %% GenServer read
    bench_server:start_link(),
    bench_server:put(key, value),
    {T2, _} = timer:tc(fun() ->
        [bench_server:get(key) || _ <- lists:seq(1, 100000)]
    end),

    io:format("ETS: ~pµs | GenServer: ~pµs | Speedup: ~.1fx~n",
              [T1, T2, T2/T1]).
```

---

## 6. Tail Recursion Patterns

```erlang
%% Tail recursion — ไม่สะสม stack frames

%% BAD: non-tail recursive (O(n) stack)
sum_bad([]) -> 0;
sum_bad([H|T]) -> H + sum_bad(T).  %% NOT tail call!

%% GOOD: tail recursive accumulator
sum_good(List) -> sum_acc(List, 0).
sum_acc([], Acc) -> Acc;
sum_acc([H|T], Acc) -> sum_acc(T, Acc + H).

%% Map with tail recursion
map_tr(Fun, List) -> map_tr(Fun, List, []).
map_tr(_, [], Acc) -> lists:reverse(Acc);
map_tr(Fun, [H|T], Acc) -> map_tr(Fun, T, [Fun(H)|Acc]).

%% Filter with tail recursion
filter_tr(Pred, List) -> filter_tr(Pred, List, []).
filter_tr(_, [], Acc) -> lists:reverse(Acc);
filter_tr(Pred, [H|T], Acc) ->
    case Pred(H) of
        true  -> filter_tr(Pred, T, [H|Acc]);
        false -> filter_tr(Pred, T, Acc)
    end.

%% Body-recursive is OK for small lists — lists:map uses it
%% Only worry for large lists or deep recursion (> 10000 elements)

%% Erlang compiler optimizes:
%% lists:map, lists:foldl are optimized
%% Use built-in list functions when possible
```

---

## 7. Compiler Optimizations

```erlang
%% 1. Compile with optimizations
%% erlc +native mymod.erl          — HiPE native code (deprecated)
%% erlc +{optimize, 3} mymod.erl  — max optimization

%% 2. Avoid dynamic dispatch in hot paths
%% BAD: dynamic call
call_dynamic(Mod, Fun, Args) ->
    apply(Mod, Fun, Args).

%% GOOD: static call (compiler can optimize)
call_known_module(Args) ->
    my_module:known_function(Args).

%% 3. Use pattern matching early
%% BAD:
handle(Msg) ->
    Type = maps:get(type, Msg),
    if Type =:= foo -> handle_foo(Msg);
       Type =:= bar -> handle_bar(Msg)
    end.

%% GOOD: match in function head
handle(#{type := foo} = Msg) -> handle_foo(Msg);
handle(#{type := bar} = Msg) -> handle_bar(Msg).

%% 4. Avoid creating garbage in hot paths
%% BAD: creates atom on every call
status_string(ok)    -> atom_to_list(ok);
status_string(error) -> atom_to_list(error).

%% GOOD: compile-time constant
status_string(ok)    -> "ok";
status_string(error) -> "error".

%% 5. Integer operations > float operations
round_to_int(F) -> round(F).  %% avoid float when possible
```

---

## 8. Process Optimization

```erlang
%% Process mailbox tips

%% 1. Process messages quickly — ป้องกัน mailbox bloat
%% ตรวจสอบ message queue length
process_info(Pid, message_queue_len).

%% Monitor large mailboxes
check_mailbox_sizes() ->
    [io:format("~p: ~p msgs~n", [P, N])
     || P <- processes(),
        {message_queue_len, N} <- [process_info(P, message_queue_len)],
        N > 1000].

%% 2. ใช้ selective receive อย่างระวัง
%% Selective receive scans mailbox → O(n) สำหรับทุก receive

%% BAD: selective receive ใน loop
server_loop() ->
    receive
        {specific_msg, Data} -> handle(Data)
    end,
    server_loop().

%% GOOD: accept all messages, dispatch in code
server_loop(State) ->
    receive
        Msg -> handle_msg(Msg, State)
    end.

handle_msg({specific_msg, Data}, State) -> ...;
handle_msg({other_msg, Data},    State) -> ...;
handle_msg(unknown_msg,          State) ->
    logger:warning("Unknown: ~p", [unknown_msg]),
    State.

%% 3. Process flags สำหรับ performance
process_flag(priority, high).      %% ให้ priority สูงกว่า
process_flag(max_heap_size, #{    %% จำกัด heap growth
    size => 1024*1024,
    kill => false,
    error_logger => true
}).
```

---

## 9. แบบฝึกหัด

### Exercise: JSON Parsing Benchmark

```erlang
%% เปรียบเทียบ JSON libraries
-module(json_bench).
-export([run/0]).

run() ->
    Data = generate_json(1000),
    N = 1000,

    %% jsx
    bench:bench(fun() -> jsx:decode(Data) end, N),

    %% jiffy (NIF-based, faster)
    bench:bench(fun() -> jiffy:decode(Data) end, N),

    %% Analyze: memory allocation during parsing
    {memory, Before} = process_info(self(), memory),
    jsx:decode(Data),
    {memory, After} = process_info(self(), memory),
    io:format("Memory delta: ~p bytes~n", [After - Before]).

generate_json(N) ->
    Items = [#{<<"id">> => I, <<"name">> => <<"item">>,
               <<"value">> => I * 1.5} || I <- lists:seq(1, N)],
    jsx:encode(Items).
```

---

## สรุป Part 33

✅ Profiling workflow  
✅ Benchmarking ด้วย timer:tc  
✅ Memory optimization (binary leak, GC)  
✅ Binary optimization (iolist, matching)  
✅ ETS vs Process State trade-offs  
✅ Tail recursion patterns  
✅ Compiler optimizations  
✅ Process mailbox optimization

---

*Part 33/100 | [← ก่อนหน้า](../part32/README.md) | [ถัดไป →](../part34/README.md)*
