# Part 13: Supervisor — Fault Tolerant Systems

> **"Supervisors are the backbone of Erlang's fault tolerance"**  
> Supervisor คือกระดูกสันหลังของ fault tolerance ใน Erlang

---

## สารบัญ

1. [Supervisor คืออะไร?](#1-supervisor-คืออะไร)
2. [Child Specification](#2-child-specification)
3. [Restart Strategies](#3-restart-strategies)
4. [Intensity และ Period](#4-intensity-และ-period)
5. [Static Children](#5-static-children)
6. [Dynamic Children](#6-dynamic-children)
7. [Supervisor Tree](#7-supervisor-tree)
8. [simple_one_for_one (deprecated → simple_one_for_one)](#8-simple_one_for_one)
9. [Supervisor ใน Production](#9-supervisor-ใน-production)
10. [แบบฝึกหัด](#10-แบบฝึกหัด)

---

## 1. Supervisor คืออะไร?

```
Supervisor คือ process ที่:
- ดูแล child processes
- Restart ถ้า child crash
- รู้ว่าจะ restart อย่างไร (strategy)
- มี threshold: restart ถี่เกินไป → ตัวเองก็ crash (bubble up)

Supervision Tree:
    Application
        │
    Supervisor (root)
       ├── Worker1
       ├── Worker2
       └── Supervisor2
              ├── Worker3
              └── Worker4
```

---

## 2. Child Specification

```erlang
%% OTP 18+ format (map)
ChildSpec = #{
    id       => child_id,          %% unique identifier
    start    => {Module, Fun, Args}, %% {M, F, A}
    restart  => permanent,         %% permanent | transient | temporary
    shutdown => 5000,              %% ms | brutal_kill | infinity
    type     => worker,            %% worker | supervisor
    modules  => [Module]           %% for hot code upgrade
}.

%% ตัวอย่างจริง
worker_spec(Id, Module, Args) ->
    #{
        id       => Id,
        start    => {Module, start_link, Args},
        restart  => permanent,
        shutdown => 5000,
        type     => worker,
        modules  => [Module]
    }.

supervisor_spec(Id, Module, Args) ->
    #{
        id       => Id,
        start    => {Module, start_link, Args},
        restart  => permanent,
        shutdown => infinity,
        type     => supervisor,
        modules  => [Module]
    }.
```

### Restart Values

```erlang
%% permanent — restart เสมอ (default)
%%   ใช้กับ: critical workers, supervisors

%% transient — restart เฉพาะตอน abnormal exit
%%   ใช้กับ: tasks ที่ปกติจบเอง แต่ถ้า crash ต้อง restart

%% temporary — ไม่ restart เลย
%%   ใช้กับ: one-shot tasks, short-lived workers
```

### Shutdown Values

```erlang
%% 5000 (ms) — ส่ง shutdown signal, รอ N ms แล้ว kill
%%   default สำหรับ worker

%% infinity — รอจน child terminate เอง
%%   ใช้กับ supervisor children

%% brutal_kill — kill ทันที
%%   ใช้กับ: stateless workers, quick shutdown
```

---

## 3. Restart Strategies

```erlang
%% one_for_one — restart เฉพาะ child ที่ crash
%%   children เป็น independent จากกัน

%% one_for_all — ถ้า child หนึ่ง crash → restart ทั้งหมด
%%   children มี dependency กัน

%% rest_for_one — crash child + children ที่ start ทีหลัง restart
%%   children มี ordered dependency

%% simple_one_for_one — dynamic children ที่มี spec เดียวกัน
%%   ใช้สำหรับ worker pools
```

### one_for_one

```
Before:  W1  W2  W3
Crash W2: W1  💥  W3
After:   W1  W2' W3    (W2 restarted, W1, W3 ไม่กระทบ)
```

### one_for_all

```
Before:  W1  W2  W3
Crash W2: W1  💥  W3
After:   W1' W2' W3'   (ทั้งหมด restart)
```

### rest_for_one

```
Before:  W1  W2  W3  W4
Crash W2: W1  💥  W3  W4
After:   W1  W2' W3' W4'  (W2, W3, W4 restart, W1 ไม่กระทบ)
```

---

## 4. Intensity และ Period

```erlang
%% {MaxRestarts, Period}
%% ถ้า restart > MaxRestarts ครั้งใน Period วินาที → supervisor crash

{ok, {#{strategy => one_for_one,
        intensity => 3,    %% max 3 restarts
        period => 5},      %% within 5 seconds
      Children}}.

%% ค่า default: intensity=1, period=5
%% → restart ได้แค่ 1 ครั้งใน 5 วินาที

%% production: intensity=10, period=60
%% → restart ได้ 10 ครั้งใน 60 วินาที ก่อน crash
```

---

## 5. Static Children

```erlang
-module(my_sup).
-behaviour(supervisor).
-export([start_link/0, init/1]).

start_link() ->
    supervisor:start_link({local, ?MODULE}, ?MODULE, []).

init([]) ->
    SupFlags = #{
        strategy  => one_for_one,
        intensity => 5,
        period    => 30
    },
    Children = [
        #{id       => cache_server,
          start    => {cache_server, start_link, []},
          restart  => permanent,
          shutdown => 5000,
          type     => worker,
          modules  => [cache_server]},

        #{id       => job_queue,
          start    => {job_queue, start_link, []},
          restart  => permanent,
          shutdown => 10000,
          type     => worker,
          modules  => [job_queue]},

        #{id       => worker_sup,
          start    => {worker_sup, start_link, []},
          restart  => permanent,
          shutdown => infinity,
          type     => supervisor,
          modules  => [worker_sup]}
    ],
    {ok, {SupFlags, Children}}.
```

---

## 6. Dynamic Children

```erlang
%% เพิ่ม child หลัง supervisor เริ่มแล้ว
supervisor:start_child(SupRef, ChildSpec).

%% ลบ child
supervisor:terminate_child(SupRef, ChildId).
supervisor:delete_child(SupRef, ChildId).

%% Restart child
supervisor:restart_child(SupRef, ChildId).

%% ดู children
supervisor:which_children(SupRef).
%% [{Id, Pid, Type, Modules}, ...]

%% Count
supervisor:count_children(SupRef).
%% #{specs => N, active => N, supervisors => N, workers => N}

%% ตัวอย่าง: Dynamic worker manager
-module(dynamic_sup).
-behaviour(supervisor).
-export([start_link/0, start_worker/1, stop_worker/1, init/1]).

start_link() ->
    supervisor:start_link({local, ?MODULE}, ?MODULE, []).

start_worker(WorkerId) ->
    ChildSpec = #{
        id       => WorkerId,
        start    => {my_worker, start_link, [WorkerId]},
        restart  => temporary,
        shutdown => 5000,
        type     => worker,
        modules  => [my_worker]
    },
    supervisor:start_child(?MODULE, ChildSpec).

stop_worker(WorkerId) ->
    supervisor:terminate_child(?MODULE, WorkerId),
    supervisor:delete_child(?MODULE, WorkerId).

init([]) ->
    {ok, {#{strategy => one_for_one, intensity => 3, period => 5}, []}}.
```

---

## 7. Supervisor Tree

```erlang
%% Application Supervisor Tree ตัวอย่าง

%% myapp_sup (root, one_for_one)
%%   ├── myapp_config_server (GenServer, permanent)
%%   ├── myapp_db_sup (Supervisor, permanent)
%%   │     ├── myapp_db_pool (one_for_all)
%%   │     │     ├── db_conn_1 (worker)
%%   │     │     ├── db_conn_2 (worker)
%%   │     │     └── db_conn_3 (worker)
%%   │     └── myapp_db_monitor (worker)
%%   └── myapp_web_sup (Supervisor, permanent)
%%         ├── myapp_cowboy_listener (worker)
%%         └── myapp_request_sup (simple_one_for_one)
%%               ├── request_handler_1 (temporary)
%%               └── request_handler_2 (temporary)

%% Root Supervisor
-module(myapp_sup).
-behaviour(supervisor).
-export([start_link/0, init/1]).

start_link() ->
    supervisor:start_link({local, ?MODULE}, ?MODULE, []).

init([]) ->
    Children = [
        child(myapp_config_server, worker),
        child(myapp_db_sup, supervisor),
        child(myapp_web_sup, supervisor)
    ],
    {ok, {#{strategy => one_for_one, intensity => 3, period => 30},
          Children}}.

child(Module, Type) ->
    Shutdown = case Type of
        supervisor -> infinity;
        worker     -> 5000
    end,
    #{id => Module, start => {Module, start_link, []},
      restart => permanent, shutdown => Shutdown,
      type => Type, modules => [Module]}.
```

---

## 8. simple_one_for_one

```erlang
%% simple_one_for_one: ทุก children ใช้ spec เดียวกัน
%% start children แบบ dynamic ด้วย supervisor:start_child/2

-module(worker_pool_sup).
-behaviour(supervisor).
-export([start_link/0, start_worker/1, init/1]).

start_link() ->
    supervisor:start_link({local, ?MODULE}, ?MODULE, []).

start_worker(Args) ->
    supervisor:start_child(?MODULE, Args).

init([]) ->
    ChildSpec = #{
        id       => worker,
        start    => {pool_worker, start_link, []},
        restart  => temporary,
        shutdown => 5000,
        type     => worker,
        modules  => [pool_worker]
    },
    {ok, {#{strategy => simple_one_for_one,
            intensity => 0,
            period => 1},
          [ChildSpec]}}.

%% เรียกใช้
worker_pool_sup:start_link(),
{ok, W1} = worker_pool_sup:start_worker([job1]),
{ok, W2} = worker_pool_sup:start_worker([job2]),

%% stop
supervisor:terminate_child(worker_pool_sup, W1).
```

---

## 9. Supervisor ใน Production

```erlang
%% Observer — GUI สำหรับดู supervision tree
observer:start().

%% Programmatic inspection
Children = supervisor:which_children(my_sup),
lists:foreach(fun({Id, Pid, Type, _}) ->
    case Pid of
        undefined -> io:format("~p: not running~n", [Id]);
        _ when Type =:= worker ->
            {memory, Mem} = process_info(Pid, memory),
            io:format("~p: ~p (mem: ~pKB)~n", [Id, Pid, Mem div 1024]);
        _ ->
            io:format("~p: supervisor ~p~n", [Id, Pid])
    end
end, Children).

%% Graceful shutdown
init:stop().          %% normal application stop
application:stop(App). %% stop specific app

%% timeout ใน terminate
terminate(Reason, State) ->
    %% ทำ cleanup ที่จำเป็น
    %% แต่อย่าใช้เวลานาน (ต้องจบก่อน shutdown timeout)
    flush_buffers(State),
    ok.
```

### Best Practices

```erlang
%% 1. Tree structure: supervisor ซ้อน supervisor
%% 2. ใช้ permanent สำหรับ critical services
%% 3. ใช้ transient สำหรับ tasks ที่ปกติจบเอง
%% 4. ใช้ temporary สำหรับ one-shot jobs
%% 5. Intensity/Period ควรสัมพันธ์กับ recovery time
%% 6. Shutdown: worker=5000, supervisor=infinity
%% 7. อย่า start supervisor ที่ intensity=0 (จะไม่ restart เลย)
```

---

## 10. แบบฝึกหัด

### Exercise: Application Supervisor Tree

```erlang
%% สร้าง supervision tree สำหรับ mini web app
%% - Root supervisor (one_for_one)
%% - Config server (GenServer, permanent)
%% - Database supervisor (one_for_all)
%%   - DB connection pool (3 workers)
%% - HTTP supervisor (one_for_one)
%%   - Listener process

-module(miniapp_sup).
-behaviour(supervisor).
-export([start_link/0, init/1]).

start_link() ->
    supervisor:start_link({local, ?MODULE}, ?MODULE, []).

init([]) ->
    {ok, {
        #{strategy => one_for_one, intensity => 5, period => 30},
        [
            #{id => config, start => {config_server, start_link, []},
              restart => permanent, shutdown => 5000,
              type => worker, modules => [config_server]},

            #{id => db_sup, start => {db_sup, start_link, []},
              restart => permanent, shutdown => infinity,
              type => supervisor, modules => [db_sup]},

            #{id => http_sup, start => {http_sup, start_link, []},
              restart => permanent, shutdown => infinity,
              type => supervisor, modules => [http_sup]}
        ]
    }}.

%% DB Supervisor
-module(db_sup).
-behaviour(supervisor).
-export([start_link/0, init/1]).

start_link() ->
    supervisor:start_link({local, ?MODULE}, ?MODULE, []).

init([]) ->
    Workers = [#{id => {db_conn, N},
                 start => {db_conn, start_link, [N]},
                 restart => permanent,
                 shutdown => 5000,
                 type => worker,
                 modules => [db_conn]}
               || N <- lists:seq(1, 3)],
    {ok, {#{strategy => one_for_all, intensity => 3, period => 60}, Workers}}.
```

---

## สรุป Part 13

✅ Supervisor behaviour  
✅ Child specification: id, start, restart, shutdown, type  
✅ Restart types: permanent, transient, temporary  
✅ Strategies: one_for_one, one_for_all, rest_for_one, simple_one_for_one  
✅ Intensity และ Period — restart threshold  
✅ Static vs Dynamic children  
✅ Supervision tree design  
✅ Production considerations

---

*Part 13/100 | [← ก่อนหน้า](../part12/README.md) | [ถัดไป →](../part14/README.md)*
