# Part 11: Processes และ Concurrency พื้นฐาน

> **"In Erlang, everything is a process. Processes are cheap. Create them freely."**  
> ใน Erlang ทุกอย่างคือ process — processes ราคาถูก สร้างได้อิสระ

---

## สารบัญ

1. [BEAM Process Model](#1-beam-process-model)
2. [สร้าง Process](#2-สร้าง-process)
3. [Process Identity](#3-process-identity)
4. [Message Passing](#4-message-passing)
5. [Receive Expression](#5-receive-expression)
6. [Process Lifecycle](#6-process-lifecycle)
7. [Linking Processes](#7-linking-processes)
8. [Monitoring Processes](#8-monitoring-processes)
9. [Process Dictionary](#9-process-dictionary)
10. [Registered Processes](#10-registered-processes)
11. [Process Patterns](#11-process-patterns)
12. [แบบฝึกหัด](#12-แบบฝึกหัด)

---

## 1. BEAM Process Model

```
BEAM Process:
├── ขนาดเริ่มต้น: ~2KB heap (เพิ่มขึ้นได้)
├── Lightweight: ไม่ใช่ OS thread
├── Isolated: ไม่แชร์ memory กัน
├── Preemptive: BEAM scheduler ดูแล
├── Max: 134 ล้าน processes (default ~1 ล้าน)
└── Communication: message passing เท่านั้น

OS Thread: ~8MB stack, expensive context switch
BEAM Process: ~2KB heap, cheap context switch (microseconds)
```

### เปรียบเทียบกับ Thread

```erlang
%% Thread-based (Java/C++):
%% - แชร์ memory → race conditions
%% - locks, mutexes → deadlocks
%% - crash thread หนึ่ง → อาจทำลายทั้งหมด

%% Process-based (Erlang):
%% - ไม่แชร์ memory → no race conditions
%% - ไม่มี locks → no deadlocks
%% - crash process หนึ่ง → ไม่กระทบอื่น
```

---

## 2. สร้าง Process

### spawn/1,3

```erlang
%% spawn/1 — ส่ง fun
Pid = spawn(fun() ->
    io:format("Hello from ~p~n", [self()])
end).

%% spawn/3 — ส่ง Module:Function, Args
Pid = spawn(mymod, myfun, [Arg1, Arg2]).

%% spawn_link/1,3 — spawn และ link ไปด้วย
Pid = spawn_link(fun() -> do_work() end).

%% spawn_monitor/1,3 — spawn และ monitor
{Pid, Ref} = spawn_monitor(fun() -> do_work() end).

%% spawn_opt — ตั้งค่า process
Pid = spawn_opt(fun() -> do_work() end, [
    {min_heap_size, 1024},     %% เริ่มต้นด้วย heap ขนาดใหญ่
    {priority, high},          %% priority: low, normal, high, max
    {fullsweep_after, 1000},   %% GC tuning
    link,                      %% เหมือน spawn_link
    monitor                    %% เหมือน spawn_monitor
]).
```

### Process Arguments

```erlang
%% ส่ง initial state ผ่าน closure
start_counter(Initial) ->
    spawn(fun() -> counter_loop(Initial) end).

counter_loop(N) ->
    receive
        {increment, From} ->
            From ! {count, N + 1},
            counter_loop(N + 1);
        {get, From} ->
            From ! {count, N},
            counter_loop(N);
        stop ->
            ok
    end.

%% ทดสอบ
C = start_counter(0),
C ! {increment, self()},
receive {count, N} -> io:format("Count: ~p~n", [N]) end.
```

---

## 3. Process Identity

```erlang
%% self() — PID ของ process ปัจจุบัน
Pid = self().  %% <0.100.0>

%% PID format: <Node.Creation.Serial>
%% Local:   <0.100.0>
%% Remote:  <mynode@host.100.0>

%% is_process_alive/1
is_process_alive(Pid).  %% true | false

%% node/1 — node ของ process
node(Pid).  %% node@host

%% process_info/1,2
process_info(Pid).
%% [{registered_name, []}, {current_function, ...},
%%  {initial_call, ...}, {status, waiting},
%%  {message_queue_len, 0}, {heap_size, ...}, ...]

process_info(Pid, message_queue_len).
%% {message_queue_len, 0}

process_info(Pid, [status, memory, message_queue_len]).
%% [{status, waiting}, {memory, 2000}, {message_queue_len, 0}]

%% processes() — list ทุก process
length(processes()).  %% จำนวน processes ทั้งหมด
```

---

## 4. Message Passing

```erlang
%% ! operator — ส่ง message
Pid ! Message.

%% ส่งไป named process
registered_name ! Message.

%% ส่งไป remote process
{registered_name, 'node@host'} ! Message.

%% Message คือ Erlang term ใดก็ได้
Pid ! hello.
Pid ! {ok, 42}.
Pid ! [1, 2, 3].
Pid ! #{key => value}.

%% ส่ง message แล้วรับ reply (request-response)
request(Pid, Request) ->
    Ref = make_ref(),  %% unique reference
    Pid ! {self(), Ref, Request},
    receive
        {Ref, Response} -> Response
    after 5000 ->
        {error, timeout}
    end.

%% ฝั่ง server ตอบ
handle(Sender, Ref, Request) ->
    Response = process_request(Request),
    Sender ! {Ref, Response}.
```

---

## 5. Receive Expression

```erlang
%% Receive — รอ message ที่ match
receive
    Pattern1 -> Body1;
    Pattern2 -> Body2;
    Pattern3 when Guard -> Body3
end.

%% Receive with timeout
receive
    Message -> handle(Message)
after
    Timeout ->   %% milliseconds, 0 = non-blocking, infinity = ไม่หมดเวลา
        handle_timeout()
end.

%% Flush mailbox
flush_mailbox() ->
    receive
        _ -> flush_mailbox()
    after 0 ->
        ok
    end.

%% Selective receive — match เฉพาะ message ที่ต้องการ
wait_for_ref(Ref) ->
    receive
        {Ref, Result} -> Result
        %% message อื่นๆ ยังอยู่ใน mailbox!
    after 5000 ->
        {error, timeout}
    end.
```

### Mailbox Model

```erlang
%% BEAM mailbox:
%% - FIFO queue สำหรับแต่ละ process
%% - Selective receive: ค้นหา pattern ใน mailbox
%% - Unmatched messages อยู่ใน mailbox ต่อไป
%% - mailbox ใหญ่ = slow selective receive

%% ตรวจสอบ mailbox size
{message_queue_len, N} = process_info(self(), message_queue_len).
```

---

## 6. Process Lifecycle

```erlang
%% Start → Running → Waiting/Suspended → Dead

%% Normal exit
exit(normal).    %% ออกปกติ
%%   หรือ function ทำงานจนจบ

%% Abnormal exit
exit(reason).    %% ออกด้วย reason
exit({error, something}).

%% Kill (ไม่สามารถ catch ได้)
exit(Pid, kill).

%% Process status
process_info(Pid, status).
%% running, runnable, waiting, suspended, garbage_collecting, exiting

%% Trapping exits — รับ exit signals เป็น messages
process_flag(trap_exit, true).
%% ตอนนี้ exit signals มาเป็น:
%% {'EXIT', Pid, Reason}
```

---

## 7. Linking Processes

```erlang
%% Link: ถ้า process หนึ่ง crash, อีก process ก็ตายด้วย

%% Link ขณะ spawn
Pid = spawn_link(fun() -> ... end).

%% Link ทีหลัง
link(Pid).

%% Unlink
unlink(Pid).

%% Exit propagation:
%% A linked B → A crash → B receives exit signal → B crash
%% (เว้นแต่ B trap_exit = true)

%% trap_exit: รับ exit signal เป็น message
process_flag(trap_exit, true),
Pid = spawn_link(fun() -> exit(crashed) end),
receive
    {'EXIT', Pid, crashed} ->
        io:format("Process ~p crashed~n", [Pid])
after 1000 ->
    timeout
end.

%% Process Group: link หลายๆ process เข้าด้วยกัน
%% ทุก process ใน group ตายพร้อมกันถ้าหนึ่งตาย
```

### Link Diagram

```
Process A ----link---- Process B
    |
  crash
    |
  exit signal ──→ Process B (crash ถ้าไม่ trap)
```

---

## 8. Monitoring Processes

```erlang
%% Monitor: observe โดยไม่ถูกกระทบ (one-directional)

%% สร้าง monitor
Ref = monitor(process, Pid).

%% ยกเลิก monitor
demonitor(Ref).
demonitor(Ref, [flush]).  %% flush pending messages ด้วย

%% รับ DOWN message เมื่อ process ตาย
receive
    {'DOWN', Ref, process, Pid, Reason} ->
        io:format("Process ~p died: ~p~n", [Pid, Reason])
end.

%% spawn_monitor: spawn และ monitor ในครั้งเดียว
{Pid, Ref} = spawn_monitor(fun() -> do_work() end).
receive
    {'DOWN', Ref, process, Pid, normal} ->
        io:format("Worker completed~n");
    {'DOWN', Ref, process, Pid, Reason} ->
        io:format("Worker failed: ~p~n", [Reason])
end.
```

### Link vs Monitor

```
Feature         Link        Monitor
Direction       Bidirectional   Unidirectional
Effect          Both die    Only monitor gets notified
Signal          {'EXIT',...} {'DOWN',...}
Use case        Tightly coupled  Loosely coupled
```

---

## 9. Process Dictionary

```erlang
%% Process dictionary: key-value store ของแต่ละ process
%% ใช้น้อยที่สุด — ทำให้โค้ดยากอ่าน

put(key, value).
get(key).           %% undefined ถ้าไม่มี
erase(key).
get().              %% ดึง dict ทั้งหมด: [{key, value}, ...]

%% ตัวอย่างการใช้
put(request_id, make_ref()),
put(user_id, 12345),

%% log ด้วย context จาก process dict
log_with_context(Level, Msg) ->
    ReqId = get(request_id),
    UserId = get(user_id),
    logger:log(Level, "~s [req=~p, user=~p]", [Msg, ReqId, UserId]).
```

---

## 10. Registered Processes

```erlang
%% ตั้งชื่อ process เพื่อ find ได้โดยไม่ต้องรู้ PID

%% Register
register(my_server, Pid).

%% Unregister
unregister(my_server).

%% ส่ง message
my_server ! {request, Data}.

%% ค้นหา PID จากชื่อ
Pid = whereis(my_server).  %% undefined ถ้าไม่มี

%% List ชื่อทั้งหมด
registered().

%% Safe send
safe_send(Name, Msg) ->
    case whereis(Name) of
        undefined -> {error, not_registered};
        Pid       -> Pid ! Msg, ok
    end.
```

---

## 11. Process Patterns

### Worker Pool

```erlang
%% Simple worker pool
-module(worker_pool).
-export([start/2, submit/2]).

start(PoolName, NumWorkers) ->
    Workers = [spawn_link(fun() -> worker_loop() end)
               || _ <- lists:seq(1, NumWorkers)],
    register(PoolName, spawn_link(fun() ->
        pool_loop(Workers, Workers)
    end)).

pool_loop([], AllWorkers) ->
    receive
        {done, Worker} ->
            pool_loop([Worker], AllWorkers)
    end;
pool_loop([Worker|Rest], AllWorkers) ->
    receive
        {submit, Task} ->
            Worker ! {task, Task},
            pool_loop(Rest, AllWorkers);
        {done, FinishedWorker} ->
            pool_loop([FinishedWorker|[Worker|Rest]], AllWorkers)
    end.

worker_loop() ->
    receive
        {task, Fun} ->
            try Fun()
            catch _:_ -> ok
            end,
            %% notify pool
            worker_loop()
    end.

submit(Pool, Task) ->
    Pool ! {submit, Task}.
```

### Server Loop Pattern

```erlang
-module(simple_server).
-export([start/0, call/2, cast/2]).

start() ->
    register(?MODULE, spawn_link(fun() -> loop(#{}) end)).

loop(State) ->
    receive
        {call, From, Ref, Request} ->
            {Reply, NewState} = handle_call(Request, State),
            From ! {Ref, Reply},
            loop(NewState);
        {cast, Message} ->
            NewState = handle_cast(Message, State),
            loop(NewState)
    end.

call(Server, Request) ->
    Ref = make_ref(),
    Server ! {call, self(), Ref, Request},
    receive
        {Ref, Reply} -> Reply
    after 5000 ->
        {error, timeout}
    end.

cast(Server, Message) ->
    Server ! {cast, Message}.

handle_call({get, Key}, State) ->
    {maps:get(Key, State, undefined), State};
handle_call({put, Key, Val}, State) ->
    {ok, State#{Key => Val}}.

handle_cast({delete, Key}, State) ->
    maps:remove(Key, State).
```

---

## 12. แบบฝึกหัด

### Exercise 1: Process Counter

```erlang
%% สร้าง counter process ที่:
%% - รับ increment, decrement, reset, get
%% - ส่ง reply กลับ
%% - มี limit (ไม่ลบต่ำกว่า 0, ไม่เกิน Max)

-module(bounded_counter).
-export([start/2, increment/1, decrement/1, get/1, reset/1]).

start(Initial, Max) ->
    spawn(fun() -> loop(Initial, Max) end).

loop(N, Max) ->
    receive
        {increment, From} when N < Max ->
            From ! {ok, N + 1},
            loop(N + 1, Max);
        {increment, From} ->
            From ! {error, at_maximum},
            loop(N, Max);
        {decrement, From} when N > 0 ->
            From ! {ok, N - 1},
            loop(N - 1, Max);
        {decrement, From} ->
            From ! {error, at_minimum},
            loop(N, Max);
        {get, From} ->
            From ! {ok, N},
            loop(N, Max);
        {reset, From} ->
            From ! {ok, 0},
            loop(0, Max)
    end.

increment(Pid) -> rpc(Pid, increment).
decrement(Pid) -> rpc(Pid, decrement).
get(Pid) -> rpc(Pid, get).
reset(Pid) -> rpc(Pid, reset).

rpc(Pid, Msg) ->
    Pid ! {Msg, self()},
    receive Reply -> Reply
    after 1000 -> {error, timeout}
    end.
```

### Exercise 2: Process Supervisor (ง่าย)

```erlang
%% ง่ายๆ: restart worker ถ้า crash

-module(simple_supervisor).
-export([start/1]).

start(WorkerFun) ->
    spawn(fun() -> supervise(WorkerFun) end).

supervise(WorkerFun) ->
    process_flag(trap_exit, true),
    Worker = spawn_link(WorkerFun),
    io:format("Started worker ~p~n", [Worker]),
    wait_for_crash(WorkerFun, Worker).

wait_for_crash(WorkerFun, Worker) ->
    receive
        {'EXIT', Worker, normal} ->
            io:format("Worker finished normally~n");
        {'EXIT', Worker, Reason} ->
            io:format("Worker crashed: ~p, restarting...~n", [Reason]),
            timer:sleep(1000),
            supervise(WorkerFun)
    end.
```

---

## สรุป Part 11

✅ BEAM process model — lightweight, isolated  
✅ spawn/1,3, spawn_link, spawn_monitor  
✅ Message passing ด้วย ! operator  
✅ Receive expression ด้วย pattern matching  
✅ Process lifecycle  
✅ Link — bidirectional, both die  
✅ Monitor — unidirectional, only notified  
✅ Process dictionary  
✅ Registered processes  
✅ Worker pool และ server loop patterns

---

*Part 11/100 | [← ก่อนหน้า](../part10/README.md) | [ถัดไป →](../part12/README.md)*
