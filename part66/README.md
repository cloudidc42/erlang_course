# Part 66: Performance Optimization

> **"Measure first, optimize second — intuition lies"**  
> วัดก่อน optimize ทีหลัง — สัญชาตญาณมักโกหก

---

## สารบัญ

1. [Profiling Tools](#1-profiling-tools)
2. [Memory Optimization](#2-memory-optimization)
3. [CPU Optimization](#3-cpu-optimization)
4. [Process Optimization](#4-process-optimization)
5. [Binary Performance](#5-binary-performance)
6. [ETS Performance](#6-ets-performance)
7. [Benchmarking](#7-benchmarking)
8. [แบบฝึกหัด](#8-แบบฝึกหัด)

---

## 1. Profiling Tools

```erlang
%% profiling.erl — Erlang profiling tools
-module(profiling).
-export([profile_function/2, profile_module/2, check_hotspots/0]).

%% fprof: detailed per-function profiling
profile_function(Fun, Args) ->
    fprof:apply(Fun, Args),
    fprof:profile(),
    fprof:analyse([{dest, "/tmp/fprof_result.txt"},
                   {details, true},
                   {totals, true}]),
    file:read_file("/tmp/fprof_result.txt").

%% cprof: lightweight call count profiling
profile_module(Module, Workload) ->
    cprof:start(),
    Workload(),
    cprof:pause(),
    Analysis = cprof:analyse(Module),
    cprof:stop(),
    Analysis.

%% eprof: wall-clock profiling for concurrency
profile_concurrent(Pids, Workload) ->
    eprof:start_profiling(Pids),
    Workload(),
    eprof:stop_profiling(),
    eprof:analyse(procs),
    eprof:log("eprof_result.txt").

%% Check which processes are consuming most CPU
check_hotspots() ->
    %% Enable scheduler wall time
    erlang:system_flag(scheduler_wall_time, true),
    Before = erlang:statistics(scheduler_wall_time),
    timer:sleep(1000),
    After = erlang:statistics(scheduler_wall_time),

    %% Calculate utilization per scheduler
    Utilization = [{Id,
                    (AT - AB) / (TT - TB) * 100}
                   || {{Id, AB, TB}, {Id, AT, TT}} <- lists:zip(Before, After)],

    %% Find processes with long message queues (potential bottlenecks)
    HeavyMsgQ = [begin
                     {_, Q} = erlang:process_info(P, message_queue_len),
                     {P, Q}
                 end
                 || P <- processes(),
                    erlang:process_info(P) =/= undefined,
                    begin
                        {_, Q} = erlang:process_info(P, message_queue_len),
                        Q > 1000
                    end],

    #{scheduler_utilization => Utilization, heavy_msg_queues => HeavyMsgQ}.

%% Reductions per process (CPU proxy)
top_processes_by_reductions(N) ->
    All = [{P, erlang:process_info(P, [reductions, message_queue_len, current_function])}
           || P <- processes(),
              erlang:process_info(P) =/= undefined],
    Sorted = lists:reverse(lists:sort(
        [{R, P, Info} || {P, [{reductions, R}|Info] = _} <- All,
                          is_integer(R)]
    )),
    lists:sublist(Sorted, N).
```

---

## 2. Memory Optimization

```erlang
%% memory_opt.erl — reduce memory usage
-module(memory_opt).
-export([shrink_process/1, efficient_accumulation/1]).

%% Process heap shrinking: force GC after large operations
shrink_process(Pid) ->
    erlang:garbage_collect(Pid, [{async, make_ref()}]).

%% Binary references: avoid holding large binaries unintentionally
%% Bad: long-lived process keeps reference to large binary
bad_binary_leak() ->
    LargeBin = binary:copy(<<"x">>, 1024 * 1024),
    %% If we keep only a sub-binary, the ENTIRE original is kept in memory!
    _Sub = binary:part(LargeBin, 0, 10),
    %% LargeBin's ref-count = 2 (LargeBin + _Sub point to same data)
    %% Solution: copy the sub-binary to release the original
    _Safe = binary:copy(binary:part(LargeBin, 0, 10)).

%% Atom table: atoms never garbage collected!
bad_dynamic_atoms() ->
    %% This LEAKS memory — atom table grows forever
    %% binary_to_atom(UserInput, utf8).

    %% Safe: use existing atoms only
    %% binary_to_existing_atom(UserInput, utf8).

    %% Or keep as binary
    ok.

%% Efficient list accumulation
efficient_accumulation(List) ->
    %% Bad: O(n²) — appending to end
    _Bad = lists:foldl(fun(X, Acc) -> Acc ++ [X * 2] end, [], List),

    %% Good: prepend and reverse — O(n)
    Reversed = lists:foldl(fun(X, Acc) -> [X * 2 | Acc] end, [], List),
    lists:reverse(Reversed).

%% Map vs Record memory comparison
map_vs_record() ->
    %% Record: fixed structure, no key overhead
    %% -record(point, {x, y, z}).
    %% Point = #point{x=1, y=2, z=3}
    %% Memory: 4 words (tag + 3 fields)

    %% Map: flexible, higher overhead
    %% Point = #{x => 1, y => 2, z => 3}
    %% Memory: ~10 words (header, hash array, etc.)

    %% Use records when structure is known at compile time
    %% Use maps for dynamic structures or when fields vary
    ok.

%% ETS memory: calculate bytes per entry
ets_memory_per_entry(Table) ->
    case ets:info(Table, size) of
        0 -> 0;
        N ->
            Words = ets:info(Table, memory),
            Bytes = Words * erlang:system_info(wordsize),
            Bytes div N  %% bytes per entry
    end.
```

---

## 3. CPU Optimization

```erlang
%% cpu_opt.erl — reduce CPU usage
-module(cpu_opt).
-export([tail_recursive/1, pattern_matching_order/1]).

%% Tail recursion: stack stays flat
%% BAD: body recursion — stack grows
sum_bad([])     -> 0;
sum_bad([H|T])  -> H + sum_bad(T).  %% NOT tail recursive

%% GOOD: tail recursive with accumulator
sum_good(List) -> sum_good(List, 0).
sum_good([], Acc)    -> Acc;
sum_good([H|T], Acc) -> sum_good(T, H + Acc).  %% tail call!

%% Pattern matching: put most common cases first
%% Bad order: rare cases before common ones
parse_bad(<<0:8, _/binary>>) -> type_zero;      %% rare
parse_bad(<<1:8, _/binary>>) -> type_one;       %% rare
parse_bad(<<Data/binary>>)   -> {unknown, Data}; %% common

%% Good order: common cases first  
parse_good(<<Data/binary>>) when byte_size(Data) > 100 ->
    {large, Data};  %% most common
parse_good(<<0:8, _/binary>>) -> type_zero;
parse_good(<<1:8, _/binary>>) -> type_one;
parse_good(Data) -> {unknown, Data}.

%% Avoid repeated computation
inefficient(List) ->
    [X || X <- List, length(List) > 10].  %% length called for EACH element!

efficient(List) ->
    Len = length(List),
    case Len > 10 of
        true  -> List;
        false -> []
    end.

%% Guard optimization: fastest guards first
%% Guards short-circuit: if first guard fails, rest not evaluated
fast_guard(X) when is_integer(X), X > 0, X < 1000000 -> positive_small;
fast_guard(X) when is_integer(X) -> integer;
fast_guard(X) when is_binary(X)  -> binary;
fast_guard(_) -> other.

%% String/binary operations: use binary:match over re for simple patterns
fast_contains(Haystack, Needle) ->
    case binary:match(Haystack, Needle) of
        {_, _}  -> true;
        nomatch -> false
    end.
    %% binary:match ~10x faster than re:run for literals

%% Avoid ets:foldl for large queries: use ets:select with match spec
%% Bad: scan all, filter in Erlang
bad_query(Table, MinAge) ->
    ets:foldl(fun({_Id, Age, _}, Acc) when Age >= MinAge -> Acc + 1;
                 (_, Acc) -> Acc
              end, 0, Table).

%% Good: use match spec — filtering done in C
good_query(Table, MinAge) ->
    ets:select_count(Table, [{{'$1', '$2', '$3'},
                              [{'>=', '$2', MinAge}],
                              [true]}]).
```

---

## 4. Process Optimization

```erlang
%% process_opt.erl — optimize process usage
-module(process_opt).

%% Process spawn cost: ~microsecond, but accumulates
%% Don't spawn a process per HTTP request for simple operations

%% Use a pool of persistent workers instead
-module(request_handler_pool).
-behaviour(gen_server).

%% Bad: spawn per request
handle_request_bad(Request) ->
    spawn_link(fun() ->
        process_request(Request),
        reply(Request, ok)
    end).

%% Good: use pooled worker
handle_request_good(Request) ->
    worker_pool:run(request_pool, fun(Worker) ->
        gen_server:call(Worker, {process, Request})
    end).

%% Avoid large message copies: use references to shared data
%% Bad: copy large binary in every message
send_large_bad(Workers, LargeBin) ->
    [W ! {data, LargeBin} || W <- Workers].
    %% LargeBin (>64 bytes refc binary) shares off-heap memory — actually OK!
    %% But for truly large shared data, use ETS

%% Good: for truly large shared state, use ETS
send_large_good(Workers, LargeData) ->
    Key = make_ref(),
    ets:insert(shared_data, {Key, LargeData}),
    [W ! {data_key, Key} || W <- Workers].
    %% Workers fetch from ETS: one copy, multiple readers

%% Selective receive: pattern matching in mailbox
%% Bad: scan entire mailbox for specific messages
receive_slow(Ref) ->
    receive
        {reply, Ref, Value} -> Value  %% may scan many unrelated messages
    after 5000 -> timeout
    end.

%% Good: use {monitor_ref, value} pattern — unique reference filters quickly
receive_fast() ->
    Ref = make_ref(),
    spawn(fun() -> self() ! {Ref, result} end),
    receive
        {Ref, Value} -> Value  %% unique ref = fast mailbox scan
    after 5000 -> timeout
    end.
```

---

## 5. Binary Performance

```erlang
%% binary_perf.erl — binary operation optimization
-module(binary_perf).
-export([build_fast/1, parse_fast/1, match_fast/2]).

%% Build binaries efficiently using iolist
build_fast(Parts) ->
    %% Bad: repeated concatenation O(n²)
    % lists:foldl(fun(P, Acc) -> <<Acc/binary, P/binary>> end, <<>>, Parts)

    %% Good: iolist → single allocation
    iolist_to_binary(Parts).

%% Parse binary with pattern matching (zero-copy where possible)
parse_fast(Bin) ->
    <<Magic:4/binary,
      Version:8,
      Length:32/big,
      Payload:Length/binary,
      _Rest/binary>> = Bin,
    #{magic => Magic, version => Version, payload => Payload}.

%% Match multiple patterns efficiently
match_fast(Haystack, Needles) ->
    %% binary:matches returns all occurrences efficiently
    binary:matches(Haystack, Needles).

%% Binary comprehension (OTP 27+)
binary_comprehension_example() ->
    Bytes = <<1, 2, 3, 4, 5>>,
    %% Double each byte
    Doubled = << <<(B * 2)>> || <<B>> <= Bytes >>,
    Doubled.

%% Splitting binaries: split is faster than pattern match for variable lengths
split_csv_line(Line) ->
    binary:split(Line, <<",">>, [global]).

%% Avoid binary_to_list / list_to_binary when not needed
validate_utf8(Bin) ->
    %% Use unicode:characters_to_binary — returns {error, ...} on invalid UTF-8
    case unicode:characters_to_binary(Bin, utf8) of
        B when is_binary(B) -> {ok, B};
        {error, _, _}       -> {error, invalid_utf8};
        {incomplete, _, _}  -> {error, incomplete_utf8}
    end.
```

---

## 6. ETS Performance

```erlang
%% ets_perf.erl — ETS optimization techniques
-module(ets_perf).

%% Table type selection:
%% set: O(1) lookup, no duplicates — default, best for key-value
%% ordered_set: O(log n) lookup, sorted — for range queries
%% bag: O(1), allows duplicates same key — for multi-value
%% duplicate_bag: O(1), allows identical records — rarely used

%% Concurrency flags matter for performance under load
create_optimal_tables() ->
    %% Read-heavy cache: maximize read throughput
    ets:new(cache, [named_table, set, public,
                    {read_concurrency, true}]),

    %% Write-heavy counters: minimize write contention
    ets:new(counters, [named_table, set, public,
                       {write_concurrency, true}]),

    %% Mixed: both flags together
    ets:new(hot_table, [named_table, set, public,
                        {read_concurrency, true},
                        {write_concurrency, true},
                        {decentralized_counters, true}]).

%% Bulk operations: insert_list is faster than repeated insert
bulk_insert(Table, Records) ->
    ets:insert(Table, Records).  %% Pass a list: atomic + faster than N inserts

%% Select with compiled match spec is faster than repeated lookups
compiled_select_example(Table, MinScore) ->
    %% Compile once, use many times
    MS = ets:fun2ms(fun({Id, Score, _}) when Score > MinScore -> Id end),
    ets:select(Table, MS).

%% Tab2list is slow for large tables: use first/next for streaming
stream_table(Table) ->
    stream_table(Table, ets:first(Table), []).

stream_table(_Table, '$end_of_table', Acc) ->
    lists:reverse(Acc);
stream_table(Table, Key, Acc) ->
    case ets:lookup(Table, Key) of
        [Record] -> stream_table(Table, ets:next(Table, Key), [Record | Acc]);
        []       -> stream_table(Table, ets:next(Table, Key), Acc)
    end.

%% Atomic counter without locks (faster than gen_server counter)
-define(COUNTER_TABLE, perf_counters).

atomic_increment(Key) ->
    ets:update_counter(?COUNTER_TABLE, Key, {2, 1}, {Key, 0}).

atomic_get(Key) ->
    case ets:lookup(?COUNTER_TABLE, Key) of
        [{Key, N}] -> N;
        []         -> 0
    end.
```

---

## 7. Benchmarking

```erlang
%% benchmark.erl — micro-benchmarking utilities
-module(benchmark).
-export([time/2, compare/2, warm_up/2, bench/3]).

%% Simple timing
time(Label, Fun) ->
    T1 = erlang:monotonic_time(microsecond),
    Result = Fun(),
    T2 = erlang:monotonic_time(microsecond),
    io:format("~s: ~p µs~n", [Label, T2 - T1]),
    Result.

%% Compare two implementations
compare(FunA, FunB) ->
    N = 10000,
    %% Warm up JIT/caches
    [FunA() || _ <- lists:seq(1, 100)],
    [FunB() || _ <- lists:seq(1, 100)],
    %% Measure
    T1 = erlang:monotonic_time(microsecond),
    [FunA() || _ <- lists:seq(1, N)],
    T2 = erlang:monotonic_time(microsecond),
    [FunB() || _ <- lists:seq(1, N)],
    T3 = erlang:monotonic_time(microsecond),
    TimeA = (T2 - T1) / N,
    TimeB = (T3 - T2) / N,
    io:format("A: ~.2f µs/op, B: ~.2f µs/op, ratio: ~.2fx~n",
              [TimeA, TimeB, TimeA/TimeB]).

%% Warm up (important for JIT)
warm_up(Fun, N) ->
    [Fun() || _ <- lists:seq(1, N)],
    ok.

%% Statistically meaningful benchmark
bench(Label, Fun, Opts) ->
    Warmup     = maps:get(warmup, Opts, 1000),
    Iterations = maps:get(iterations, Opts, 10000),
    warm_up(Fun, Warmup),
    Times = [begin
                 T1 = erlang:monotonic_time(nanosecond),
                 Fun(),
                 erlang:monotonic_time(nanosecond) - T1
             end || _ <- lists:seq(1, Iterations)],
    Sorted = lists:sort(Times),
    Total  = lists:sum(Times),
    Mean   = Total div Iterations,
    Median = lists:nth(Iterations div 2, Sorted),
    P95    = lists:nth(round(Iterations * 0.95), Sorted),
    P99    = lists:nth(round(Iterations * 0.99), Sorted),
    io:format("~s: mean=~pns median=~pns p95=~pns p99=~pns~n",
              [Label, Mean, Median, P95, P99]).
```

---

## 8. แบบฝึกหัด

1. Profile ระบบที่คุณสร้างใน Part 46 (e-commerce): ค้นหาและแก้ bottleneck
2. Benchmark: `lists:map` vs list comprehension vs `lists:foldl` สำหรับ transformation
3. เขียน benchmark เพื่อหา ETS read concurrency threshold
4. Optimize function ที่ใช้ `string:split` → `binary:split` แล้ววัดความต่าง

---

## สรุป Part 66

✅ Profiling tools: fprof, cprof, eprof  
✅ Memory: avoid binary leaks, atom table, efficient accumulation  
✅ CPU: tail recursion, pattern matching order, match spec  
✅ Process: pooling, reference sharing, selective receive  
✅ Binary: iolist building, zero-copy parsing, comprehensions  
✅ ETS: concurrency flags, bulk ops, compiled match spec  
✅ Benchmarking: warm-up, statistical measurements  

---

*Part 66/100 | [← ก่อนหน้า](../part65/README.md) | [ถัดไป →](../part67/README.md)*
