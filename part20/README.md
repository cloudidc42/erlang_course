# Part 20: Debugging และ Profiling

> **"Profile before optimizing — measure, don't guess"**  
> Profile ก่อน optimize — วัด อย่าเดา

---

## สารบัญ

1. [Debugging Tools](#1-debugging-tools)
2. [erlang Debugger](#2-erlang-debugger)
3. [dbg — Tracing](#3-dbg--tracing)
4. [recon Library](#4-recon-library)
5. [Observer](#5-observer)
6. [Profiling ด้วย fprof](#6-profiling-ด้วย-fprof)
7. [eprof — Time Profiler](#7-eprof--time-profiler)
8. [cprof — Call Profiler](#8-cprof--call-profiler)
9. [Memory Analysis](#9-memory-analysis)
10. [แบบฝึกหัด](#10-แบบฝึกหัด)

---

## 1. Debugging Tools

### เครื่องมือที่มีใน OTP

```erlang
%% 1. Debugger (GUI) — breakpoints, step-through
debugger:start().

%% 2. dbg — function/message tracing
dbg:tracer().
dbg:p(all, c).
dbg:tpl(Module, Function, '_', []).

%% 3. sys module — inspect GenServer state
sys:get_state(ServerRef).
sys:get_status(ServerRef).
sys:trace(ServerRef, true).

%% 4. process_info
process_info(Pid, [status, memory, message_queue_len, current_function]).

%% 5. erlang:trace
erlang:trace(Pid, true, [call, {tracer, self()}]).

%% 6. Observer — GUI สำหรับ system monitoring
observer:start().

%% 7. crash dumps
%% เมื่อ VM crash: erl_crash.dump
crashdump_viewer:start().  %% GUI viewer
```

---

## 2. erlang Debugger

```erlang
%% เปิด Debugger GUI
debugger:start().

%% ต้อง compile ด้วย debug_info
erlc +debug_info my_module.erl
%% หรือใน rebar.config: {erl_opts, [debug_info]}

%% Attach debugger ไปที่ module
int:i(my_module).    %% interpret (enable debugging)
int:ni(my_module).   %% ถอด interpretation

%% Programmatic breakpoint
int:break(my_module, 25).      %% breakpoint ที่ line 25
int:delete_break(my_module, 25).
int:break_in(my_module, my_fun, 1). %% breakpoint เมื่อเข้า my_fun/1

%% ดู breakpoints
int:all_breaks().

%% ใช้ GUI: Module → View → Attach สำหรับ step-through
%% n = next, s = step, c = continue, f = finish function
```

---

## 3. dbg — Tracing

```erlang
%% dbg: powerful tracing สำหรับ production debugging

%% เริ่ม tracer (print to console)
dbg:tracer().

%% เริ่ม tracer ไปยัง file
dbg:tracer(file, "/tmp/trace.log").

%% เลือก processes
dbg:p(all, c).      %% ทุก processes, trace calls
dbg:p(Pid, c).      %% specific process
dbg:p(new, c).      %% processes ที่สร้างใหม่เท่านั้น

%% Trace flags:
%% c = calls
%% r = return values
%% m = messages sent/received
%% s = sent messages
%% 'receive' = received messages
%% p = process events

%% Trace function calls
dbg:tpl(mymod, myfun, 1, []).  %% trace mymod:myfun/1
dbg:tpl(mymod, '_', '_', []). %% trace ทุก functions ใน mymod
dbg:tpl(mymod, myfun, '_', [{'_', [], [{return_trace}]}]).  %% + return value

%% Trace with match spec
MS = dbg:fun2ms(fun([X]) when X > 100 -> return_trace() end).
dbg:tpl(mymod, myfun, MS).

%% หยุด tracing
dbg:ctpl().   %% clear all trace patterns
dbg:stop().   %% stop tracer

%% ตัวอย่าง: trace specific call path
trace_request(RequestId) ->
    dbg:tracer(),
    dbg:p(all, [c, r]),
    dbg:tpl(request_handler, '_', '_',
        dbg:fun2ms(fun([ReqId|_]) when ReqId =:= RequestId -> return_trace() end)),
    ok.
```

---

## 4. recon Library

```erlang
%% recon: production debugging library
%% rebar.config: {deps, [{recon, "2.5.4"}]}

%% Process info
recon:info(Pid).
recon:info(Pid, [memory, message_queue_len, status]).

%% Top N processes by memory/reductions/etc
recon:proc_count(memory, 10).
recon:proc_count(message_queue_len, 5).
recon:proc_count(reductions, 10).

%% Window (compare over time)
recon:proc_window(memory, 5, 1000).    %% top 5, sample 1s apart

%% Port info
recon:port_info(Port).
recon:port_count(memory, 10).

%% Tracing (safer than raw dbg for production)
recon_trace:calls({Module, Function, Arity}, N).
%% N = max number of calls to trace

recon_trace:calls({mymod, myfun, 1}, 100,
    [{pid, all}, {scope, global}]).

%% ดู function calls ที่กำลัง execute
recon:scheduler_usage(1000).   %% scheduler utilization

%% Memory analysis
recon_alloc:memory(usage).
recon_alloc:memory(allocated).
recon_alloc:sbcs_to_mbcs(1000).  %% fragmentation check
```

---

## 5. Observer

```erlang
%% Observer: GUI tool สำหรับ system monitoring

observer:start().

%% Tabs:
%% System     — node info, memory, scheduler
%% Load Charts — CPU, memory graphs
%% Memory     — allocator details
%% Applications — supervision tree visualization
%% Processes  — process list with sorting
%% Ports      — network/file ports
%% ETS        — ETS table browser
%% Mnesia     — Mnesia table browser
%% Trace Overview — tracing configuration

%% Remote observer
%% connect ไปยัง remote node แล้ว
observer:start().
%% ใน Nodes menu เลือก node ที่ต้องการ observe

%% Programmatic node info (ไม่ต้องใช้ GUI)
erlang:memory().
%% [{total, N}, {processes, N}, {system, N}, {atom, N}, ...]

erlang:system_info(process_count).
erlang:system_info(scheduler_id).
erlang:statistics(scheduler_wall_time).
erlang:statistics(run_queue).
erlang:statistics(reductions).
```

---

## 6. Profiling ด้วย fprof

```erlang
%% fprof: function profiling (overhead ~10x slower)

%% Profile ด้วย fun
fprof:apply(fun() -> my_function(args) end).
fprof:profile().
fprof:analyse().

%% หรือ step by step
fprof:trace([start]).
my_function(args).
fprof:trace([stop]).
fprof:profile().
fprof:analyse([totals, details]).

%% Output to file
fprof:analyse([{dest, "/tmp/fprof.txt"}]).

%% ตัวอย่าง output:
%% {analysis_opts, ...}
%% {totals, ...}
%% [ {Caller, {Own, Acc, Calls}, [{Called, Calls}]}, ...]
%%
%% Own = time in function itself (excluding callees)
%% Acc = accumulated time (including callees)

%% Profile specific process
fprof:trace([start, {procs, Pid}]).
...
fprof:trace([stop]).
fprof:profile().
fprof:analyse().
```

---

## 7. eprof — Time Profiler

```erlang
%% eprof: per-process time profiling (lower overhead than fprof)

%% เริ่ม profiling
eprof:start_profiling([self()]).

%% รัน code
do_work().

%% หยุดและ analyze
eprof:stop_profiling(),
eprof:analyze(total).
%% หรือ
eprof:log("/tmp/eprof.log").
eprof:analyze(procs).

%% Profile all processes
eprof:profile(fun() ->
    lists:sort([rand:uniform(100) || _ <- lists:seq(1, 10000)])
end).
eprof:analyze(total).

%% Output ตัวอย่าง:
%% FUNCTION                 CALLS    %     TIME  [uS / CALLS]
%% ---------                -----  ---     ----  [----------]
%% lists:sort/1              1000  35.2     3521  [      3.52]
%% rand:uniform/1           10000  52.1     5210  [      0.52]
```

---

## 8. cprof — Call Profiler

```erlang
%% cprof: นับจำนวน function calls (minimal overhead)

cprof:start().
%% รัน code
lists:sort([rand:uniform(100) || _ <- lists:seq(1, 10000)]).
cprof:pause().

%% Analyze
cprof:analyse().
%% หรือ module เฉพาะ
cprof:analyse(lists).

%% Output:
%% {lists, 29001,
%%  [{lists:sort/1, 1},
%%   {lists:sort/2, 9999},
%%   {lists:merge/2, 9001},
%%   ...]}

cprof:restart().  %% reset counters
cprof:stop().
```

---

## 9. Memory Analysis

```erlang
%% Memory usage breakdown
erlang:memory().
%% [{total,N}, {processes,N}, {processes_used,N},
%%  {system,N}, {atom,N}, {atom_used,N},
%%  {binary,N}, {code,N}, {ets,N}]

%% Garbage collection stats
erlang:statistics(garbage_collection).
%% {NumberGCs, WordsReclaimed, 0}

%% Process memory
process_info(Pid, memory).
process_info(Pid, garbage_collection).
process_info(Pid, heap_size).  %% words

%% Binary memory
process_info(Pid, binary).
%% [{Ref, Size, Count}] รายการ binaries ที่ process อ้างถึง

%% Trigger GC
erlang:garbage_collect().
erlang:garbage_collect(Pid).

%% Find memory leaks — processes ที่ memory โต
find_memory_leaks() ->
    Procs = [{process_info(P, memory), P} || P <- processes()],
    Sorted = lists:sort(fun({A,_}, {B,_}) -> A > B end, Procs),
    lists:sublist(Sorted, 10).

%% Binary leak — binaries ที่ไม่ถูก GC
check_binary_memory() ->
    M = erlang:memory(binary),
    io:format("Binary memory: ~p MB~n", [M div 1048576]).

%% หลังจาก GC ทุก process
[erlang:garbage_collect(P) || P <- processes()].
```

---

## 10. แบบฝึกหัด

### Exercise: Profile Sorting

```erlang
%% Profile sorting algorithms ต่างๆ แล้วเปรียบเทียบ

-module(sort_bench).
-export([run/0]).

run() ->
    Data = [rand:uniform(1000) || _ <- lists:seq(1, 10000)],
    
    %% Benchmark แต่ละ sort
    Results = [
        {erlang_sort,    bench(fun() -> lists:sort(Data) end, 100)},
        {merge_sort,     bench(fun() -> my_merge_sort(Data) end, 100)},
        {quick_sort,     bench(fun() -> my_quick_sort(Data) end, 100)}
    ],
    
    lists:foreach(fun({Name, Time}) ->
        io:format("~p: ~p ms~n", [Name, Time])
    end, lists:sort(fun({_,A}, {_,B}) -> A =< B end, Results)).

bench(Fun, N) ->
    T1 = erlang:monotonic_time(millisecond),
    lists:foreach(fun(_) -> Fun() end, lists:seq(1, N)),
    T2 = erlang:monotonic_time(millisecond),
    T2 - T1.

my_merge_sort([]) -> [];
my_merge_sort([X]) -> [X];
my_merge_sort(List) ->
    {Left, Right} = split(List),
    merge(my_merge_sort(Left), my_merge_sort(Right)).

split(List) ->
    N = length(List) div 2,
    lists:split(N, List).

merge([], Right) -> Right;
merge(Left, []) -> Left;
merge([H1|T1], [H2|T2]) when H1 =< H2 ->
    [H1 | merge(T1, [H2|T2])];
merge(Left, [H2|T2]) ->
    [H2 | merge(Left, T2)].

my_quick_sort([]) -> [];
my_quick_sort([Pivot|Rest]) ->
    {Less, Greater} = lists:partition(fun(X) -> X < Pivot end, Rest),
    my_quick_sort(Less) ++ [Pivot] ++ my_quick_sort(Greater).
```

---

## สรุป Part 20

✅ Debugging tools overview  
✅ erlang Debugger GUI  
✅ dbg: function tracing  
✅ recon: production-safe debugging  
✅ Observer: system monitoring GUI  
✅ fprof: detailed function profiling  
✅ eprof: time profiling  
✅ cprof: call counting  
✅ Memory analysis tools

---

*Part 20/100 | [← ก่อนหน้า](../part19/README.md) | [ถัดไป →](../part21/README.md)*

---

## ระดับ 2 เสร็จแล้ว! ต่อไป: ระดับ Web Development

Part 21-30 จะครอบคลุม Cowboy HTTP Server, RESTful APIs, WebSockets, และ Web Application Development
