# Part 60: BEAM VM Internals

> **"Understanding the machine makes you master of it"**  
> เข้าใจเครื่องยนต์ทำให้คุณเป็นนายของมัน

---

## สารบัญ

1. [BEAM Architecture](#1-beam-architecture)
2. [Schedulers and Reduction Count](#2-schedulers-and-reduction-count)
3. [Process Heap and GC](#3-process-heap-and-gc)
4. [Binary Handling](#4-binary-handling)
5. [ETS Internals](#5-ets-internals)
6. [VM Tuning](#6-vm-tuning)
7. [แบบฝึกหัด](#7-แบบฝึกหัด)

---

## 1. BEAM Architecture

```
BEAM Virtual Machine Architecture
══════════════════════════════════════════════════════════
  ┌─────────────────────────────────────────────────────┐
  │                   Erlang Code                        │
  │  module.beam  →  bytecode  →  loaded into VM         │
  └───────────────────────┬─────────────────────────────┘
                          │
  ┌───────────────────────▼─────────────────────────────┐
  │               Scheduler (1 per CPU core)             │
  │  run queue → pick process → execute reductions       │
  │  preempt after 2000 reductions                       │
  └───────────────────────┬─────────────────────────────┘
                          │
  ┌───────────────────────▼─────────────────────────────┐
  │              Process (Erlang "green thread")         │
  │  heap, stack, mailbox, PCB (Process Control Block)   │
  │  default heap: 233 words (expandable)                │
  └─────────────────────────────────────────────────────┘

Key VM data structures:
  - PCB: process descriptor (priority, state, registers, ...)
  - Heap: private to each process — no sharing, no locks needed
  - Message queue (mailbox): linked list of message terms
  - Call stack: stored on process heap
  - ETS: shared off-heap storage (lock per table)
```

```erlang
%% Inspect VM internals at runtime
vm_stats() ->
    #{
        %% Schedulers
        schedulers        => erlang:system_info(schedulers),
        schedulers_online => erlang:system_info(schedulers_online),

        %% Processes
        process_count   => erlang:system_info(process_count),
        process_limit   => erlang:system_info(process_limit),

        %% Memory
        total_memory    => erlang:memory(total),
        process_memory  => erlang:memory(processes),
        system_memory   => erlang:memory(system),
        ets_memory      => erlang:memory(ets),
        binary_memory   => erlang:memory(binary),
        atom_memory     => erlang:memory(atom),

        %% Run queues
        run_queue       => erlang:statistics(run_queue),
        run_queue_lengths => erlang:statistics(run_queue_lengths)
    }.
```

---

## 2. Schedulers and Reduction Count

```erlang
%% reductions.erl — understanding scheduler preemption
-module(reductions).
-export([count_reductions/1, yield_example/0, scheduler_info/0]).

%% Each function call consumes ~1 reduction
%% Preemption happens every 2000 reductions
%% This ensures fair CPU sharing across all processes

count_reductions(F) ->
    {Reductions1, _} = erlang:process_info(self(), reductions),
    Result = F(),
    {Reductions2, _} = erlang:process_info(self(), reductions),
    {Result, Reductions2 - Reductions1}.

%% Voluntarily yield the scheduler
yield_example() ->
    spawn(fun() ->
        loop(0)
    end).

loop(N) when N < 1_000_000 ->
    %% erlang:yield() gives up the scheduler voluntarily
    %% Good for CPU-intensive tasks to avoid hogging
    case N rem 1000 of
        0 -> erlang:yield();
        _ -> ok
    end,
    loop(N + 1);
loop(_) -> done.

scheduler_info() ->
    [{scheduler_id, erlang:system_info(scheduler_id)} |
     [{Id, erlang:statistics({scheduler_wall_time, Id})}
      || Id <- lists:seq(1, erlang:system_info(schedulers_online))]].

%% Dirty schedulers: for NIF/BIF that block
%% +SDcpu N  → N dirty CPU schedulers
%% +SDio N   → N dirty I/O schedulers
dirty_nif_info() ->
    #{
        dirty_cpu_schedulers    => erlang:system_info(dirty_cpu_schedulers),
        dirty_cpu_schedulers_online => erlang:system_info(dirty_cpu_schedulers_online),
        dirty_io_schedulers     => erlang:system_info(dirty_io_schedulers)
    }.
```

---

## 3. Process Heap and GC

```erlang
%% heap_gc.erl — understanding per-process GC
-module(heap_gc).
-export([heap_info/1, force_gc/1, set_heap_size/2]).

%% Each Erlang process has its OWN heap — no stop-the-world GC
%% GC is per-process: only the GC'd process stops
%% Minor GC: collect young generation (fast)
%% Major GC: collect entire heap (slower, rarer)

heap_info(Pid) ->
    {_, HeapSize}  = erlang:process_info(Pid, heap_size),
    {_, StackSize} = erlang:process_info(Pid, stack_size),
    {_, TotalHeap} = erlang:process_info(Pid, total_heap_size),
    {_, Reds}      = erlang:process_info(Pid, reductions),
    {_, MsgQLen}   = erlang:process_info(Pid, message_queue_len),
    #{
        heap_size       => HeapSize,    %% current heap words
        stack_size      => StackSize,   %% stack words
        total_heap_size => TotalHeap,   %% heap + stack + old heap
        reductions      => Reds,
        message_queue_len => MsgQLen
    }.

%% Force GC on a specific process (e.g., after freeing large data)
force_gc(Pid) ->
    erlang:garbage_collect(Pid).

%% Tune heap size for a process that you know will be large
set_heap_size(Pid, MinHeapSize) ->
    erlang:process_flag(Pid, min_heap_size, MinHeapSize).

%% GC statistics
gc_stats() ->
    {GCs, WordsReclaimed, 0} = erlang:statistics(garbage_collection),
    #{
        number_of_gcs      => GCs,
        words_reclaimed    => WordsReclaimed
    }.

%% Spawn with large heap to avoid early GC
spawn_large_process(Fun) ->
    spawn_opt(Fun, [
        {min_heap_size, 65536},   %% 64K words initial heap
        {fullsweep_after, 0}      %% full GC every minor GC (for short-lived)
    ]).

%% For long-lived processes with predictable memory
spawn_long_lived(Fun) ->
    spawn_opt(Fun, [
        {min_heap_size, 1024},
        {fullsweep_after, 100}    %% full sweep after 100 minor GCs
    ]).
```

---

## 4. Binary Handling

```erlang
%% binary_internals.erl — understanding binary memory model
-module(binary_internals).
-export([binary_types/0, refc_binary_info/0, binary_optimization/0]).

%% Two types of binaries in BEAM:
%% 1. Heap binary: <= 64 bytes, stored on process heap (copied on send)
%% 2. Refc binary: > 64 bytes, stored off-heap in binary heap
%%    All processes share the same physical memory (reference counted)
%%    Only a "ProcBin" pointer is copied when sending between processes

binary_types() ->
    Small = <<"hello">>,          %% heap binary (5 bytes, on heap)
    Large = binary:copy(<<"x">>, 100),  %% refc binary (100 bytes, off-heap)

    %% Check where a binary lives
    SmallInfo = erts_debug:size(Small),   %% includes in process heap
    LargeInfo = erts_debug:size(Large),   %% only counts ProcBin pointer

    #{small_size => SmallInfo, large_size => LargeInfo,
      small => Small, large => Large}.

%% Monitor refc binary count
refc_binary_info() ->
    erlang:memory(binary).  %% total bytes in binary heap

%% Optimization: avoid copying large binaries
binary_optimization() ->
    %% BAD: creates copies of sub-binaries
    _Bad = fun(Bin) ->
        Part1 = binary:part(Bin, 0, 10),
        Part2 = binary:part(Bin, 10, 10),
        {Part1, Part2}   %% Part1 and Part2 may reference original
    end,

    %% GOOD: use binary pattern matching (zero-copy sub-binary)
    _Good = fun(Bin) ->
        <<Part1:10/binary, Part2:10/binary, _/binary>> = Bin,
        {Part1, Part2}   %% guaranteed sub-binary references, no copy
    end,

    %% Force copy when you want to release original large binary
    release_original = fun(SubBin) ->
        binary:copy(SubBin)  %% creates an independent heap binary copy
    end,

    ok.

%% Binary builder: use iolist instead of concatenation
build_http_response(StatusCode, Headers, Body) ->
    %% BAD: O(n²) — each ++ creates a new binary
    % <<"HTTP/1.1 ">> ++ integer_to_binary(StatusCode) ++ ...

    %% GOOD: build as iolist, convert once at the end
    StatusLine = [<<"HTTP/1.1 ">>, integer_to_binary(StatusCode), <<"\r\n">>],
    HeaderLines = [[K, <<": ">>, V, <<"\r\n">>] || {K, V} <- Headers],
    Response = [StatusLine, HeaderLines, <<"\r\n">>, Body],
    iolist_to_binary(Response).
```

---

## 5. ETS Internals

```erlang
%% ets_internals.erl — understanding ETS implementation
-module(ets_internals).
-export([table_info/1, memory_analysis/0, concurrency_model/0]).

table_info(Table) ->
    #{
        type      => ets:info(Table, type),
        keypos    => ets:info(Table, keypos),
        size      => ets:info(Table, size),        %% number of objects
        memory    => ets:info(Table, memory),      %% words used
        owner     => ets:info(Table, owner),       %% controlling process
        protection => ets:info(Table, protection), %% public/protected/private
        read_concurrency  => ets:info(Table, read_concurrency),
        write_concurrency => ets:info(Table, write_concurrency)
    }.

%% ETS memory analysis across all tables
memory_analysis() ->
    AllTables = ets:all(),
    Info = [{ets:info(T, name),
             ets:info(T, size),
             ets:info(T, memory) * erlang:system_info(wordsize)}
            || T <- AllTables, is_atom(ets:info(T, name))],
    Total = lists:sum([Mem || {_, _, Mem} <- Info]),
    #{tables => Info, total_bytes => Total}.

%% ETS concurrency: fine-grained locking
concurrency_model() ->
    %% {read_concurrency, true}: optimize for concurrent reads
    %%   → uses reader-writer locks (multiple readers OR one writer)
    ReadHeavy = ets:new(read_heavy, [
        public, set,
        {read_concurrency, true}
    ]),

    %% {write_concurrency, true}: optimize for concurrent writes
    %%   → splits table into multiple segments with separate locks
    WriteHeavy = ets:new(write_heavy, [
        public, set,
        {write_concurrency, true}
    ]),

    %% Both: best for read-write mix
    Mixed = ets:new(mixed, [
        public, set,
        {read_concurrency, true},
        {write_concurrency, true}
    ]),

    {ReadHeavy, WriteHeavy, Mixed}.
```

---

## 6. VM Tuning

```erlang
%% vm_tuning.erl — runtime VM configuration
-module(vm_tuning).

%% vm.args — startup configuration (in release)
%%
%% ## Schedulers
%% +S 8:8          → 8 schedulers, 8 online (match CPU count)
%% +SDcpu 4        → 4 dirty CPU schedulers
%% +SDio 10        → 10 dirty I/O schedulers
%%
%% ## Memory
%% +MBas aobf      → allocator: address-order best-fit (reduces fragmentation)
%% +MBlmbcs 512    → max block in mbcs: 512 KB (reduce fragmentation)
%%
%% ## Process limits
%% +P 1000000      → max 1M processes
%% +Q 65535        → max 65535 ports
%%
%% ## Distribution
%% -kernel net_ticktime 60   → heartbeat every 60s (not 15)
%% -kernel dist_auto_connect never  → manual node connections
%%
%% ## GC
%% +e 1024         → expand heap factor (higher = less GC, more memory)

%% Runtime tuning
tune_at_runtime() ->
    %% Adjust schedulers online
    erlang:system_flag(schedulers_online, erlang:system_info(schedulers)),

    %% Adjust process limit (only up to compile-time max)
    %% erlang:system_flag(process_count, 2000000),

    %% Busy wait (spin instead of sleep when no work — latency vs CPU)
    erlang:system_flag(scheduler_wall_time, true),

    %% Enable scheduler stats for monitoring
    erlang:system_flag(scheduler_wall_time, true).

profiling_commands() ->
    %% eprof: wall-clock profiling
    %% eprof:start_profiling([Pid]).
    %% eprof:stop_profiling().
    %% eprof:analyze().

    %% fprof: detailed call graph profiling
    %% fprof:apply(Fun, Args).
    %% fprof:profile().
    %% fprof:analyse([{dest, "fprof.analysis"}]).

    %% cprof: lightweight call count profiling
    %% cprof:start().
    %% ... run workload ...
    %% cprof:pause().
    %% cprof:analyse(MyModule).
    ok.
```

---

## 7. แบบฝึกหัด

1. เขียน GenServer ที่ log heap size ของตัวเองทุก 1 วินาที
2. ทดสอบ: spawn 100,000 processes แล้ว measure รวม memory ที่ใช้
3. สร้าง benchmark เปรียบเทียบ ETS read_concurrency=true vs false
4. Profile function ด้วย fprof แล้ว identify hotspot

---

## สรุป Part 60

✅ BEAM architecture: schedulers, processes, heap  
✅ Reduction count และ scheduler preemption  
✅ Per-process GC: minor/major, heap size tuning  
✅ Binary types: heap binary vs refc binary  
✅ ETS locking models: read/write concurrency  
✅ VM startup args และ runtime tuning  

---

*Part 60/100 | [← ก่อนหน้า](../part59/README.md) | [ถัดไป →](../part61/README.md)*
