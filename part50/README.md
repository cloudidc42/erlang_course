# Part 50: World-Class Erlang Patterns

> **"Erlang's secret: make the common case fast, and the failure case handled"**  
> ความลับของ Erlang: ทำให้กรณีปกติเร็ว และจัดการกรณีที่ล้มเหลวได้เสมอ

---

## สารบัญ

1. [Erlang Design Principles](#1-erlang-design-principles)
2. [Let It Crash Philosophy](#2-let-it-crash-philosophy)
3. [Process Design Patterns](#3-process-design-patterns)
4. [Supervisor Trees สำหรับ Production](#4-supervisor-trees-สำหรับ-production)
5. [Massive Concurrency Patterns](#5-massive-concurrency-patterns)
6. [Zero-Downtime Architecture](#6-zero-downtime-architecture)
7. [Production Checklist](#7-production-checklist)
8. [World-Class Case Studies](#8-world-class-case-studies)

---

## 1. Erlang Design Principles

```
Erlang Design Principles (from Joe Armstrong):

1. Everything is a process
2. Processes are strongly isolated
3. Process creation and destruction is lightweight
4. Message passing is the ONLY way to communicate
5. Error handling is non-local
6. Processes have unique names
7. Processes should not share names
8. If you can do it in parallel, do it
9. A failed process should be logged with a reason
10. Do not communicate by sharing memory; share memory by communicating

The "Erlang Style":
  - Small processes, each doing ONE thing
  - Fail fast, restart cleanly
  - No defensive programming — trust your data
  - Supervision is infrastructure, not business logic
  - State in processes, not global variables
```

---

## 2. Let It Crash Philosophy

```erlang
%% BAD: Defensive programming — Erlang style violation
safe_divide(A, B) ->
    if B =:= 0 ->
        {error, division_by_zero};
    true ->
        {ok, A div B}
    end.

%% GOOD: Let it crash, supervise it
divide(A, B) -> A div B.  %% Will crash on B=0 — that's correct!

%% The SUPERVISOR handles failures, not the worker
%% If divide/2 crashes with {badarith,...}, the supervisor:
%%   1. Logs the crash
%%   2. Restarts the process
%%   3. Continues serving other requests

%% BAD: Catching everything
process_request(Req) ->
    try
        do_work(Req)
    catch
        _:_ -> {error, something_went_wrong}   %% hiding real bugs!
    end.

%% GOOD: Catch only what you know how to handle
process_request(Req) ->
    try do_work(Req)
    catch
        error:{db_error, Reason} ->
            logger:error("DB error: ~p", [Reason]),
            {error, service_unavailable};
        error:{validation, Field, Msg} ->
            {error, #{field => Field, message => Msg}}
        %% Let everything else crash and be restarted
    end.
```

---

## 3. Process Design Patterns

```erlang
%% Pattern 1: Server Process (stateful, long-lived)
-module(counter).
-behaviour(gen_server).
-export([start_link/0, increment/0, get/0]).
-export([init/1, handle_call/3, handle_cast/2]).

start_link() -> gen_server:start_link({local, ?MODULE}, ?MODULE, 0, []).
increment()  -> gen_server:cast(?MODULE, increment).
get()        -> gen_server:call(?MODULE, get).

init(N) -> {ok, N}.
handle_cast(increment, N) -> {noreply, N+1}.
handle_call(get, _, N) -> {reply, N, N}.

%% Pattern 2: Worker Process (one task, short-lived)
process_job(Job) ->
    spawn(fun() ->
        Result = do_heavy_work(Job),
        notify_completion(Job, Result)
    end).

%% Pattern 3: Pool of identical workers
-module(worker_pool).
start_pool(Size) ->
    [spawn_link(fun worker_loop/0) || _ <- lists:seq(1, Size)].

worker_loop() ->
    receive
        {work, Task, ReplyTo} ->
            Result = execute(Task),
            ReplyTo ! {result, Result},
            worker_loop();
        stop -> ok
    end.

%% Pattern 4: Pipeline (chain of transformations)
pipeline(Input, Steps) ->
    lists:foldl(fun(Step, Acc) ->
        receive_and_forward(Step, Acc)
    end, Input, Steps).

%% Pattern 5: Event Hub (fan-out to multiple subscribers)
-module(event_hub).
-behaviour(gen_server).

publish(Event) ->
    gen_server:cast(?MODULE, {publish, Event}).

subscribe(Pid, EventType) ->
    gen_server:call(?MODULE, {subscribe, Pid, EventType}).
```

---

## 4. Supervisor Trees สำหรับ Production

```erlang
%% app_sup.erl — production supervisor tree
-module(app_sup).
-behaviour(supervisor).
-export([start_link/0, init/1]).

start_link() ->
    supervisor:start_link({local, ?MODULE}, ?MODULE, []).

init([]) ->
    SupFlags = #{
        strategy  => one_for_one,
        intensity => 10,   %% max 10 restarts
        period    => 60    %% within 60 seconds
    },
    Children = [
        %% Infrastructure layer
        child(db_sup,      supervisor, permanent),
        child(cache_sup,   supervisor, permanent),
        child(metrics,     worker,     permanent),

        %% Business logic layer
        child(auth_server, worker, permanent),
        child(job_sup,     supervisor, permanent),

        %% HTTP layer (last, depends on above)
        child(http_sup, supervisor, permanent)
    ],
    {ok, {SupFlags, Children}}.

child(Name, Type, Restart) ->
    #{id => Name, start => {Name, start_link, []},
      restart => Restart, type => Type,
      shutdown => case Type of supervisor -> infinity; _ -> 5000 end}.

%% http_sup.erl — separate HTTP supervisor
-module(http_sup).
-behaviour(supervisor).

init([]) ->
    {ok, {#{strategy => one_for_all}, [
        %% one_for_all: if HTTP listener dies, restart all HTTP workers
        child(cowboy_listener, worker, permanent),
        child(session_manager, worker, permanent)
    ]}}.
```

---

## 5. Massive Concurrency Patterns

```erlang
%% Handling millions of concurrent connections

%% Pattern: Per-connection lightweight process
%% Each TCP connection = 1 Erlang process (~2KB initial heap)

%% 1 million connections:
%%   Memory: 1,000,000 * 2KB = 2GB
%%   With pools: much less

%% Pattern: Partition by key to avoid bottlenecks
-module(partitioned_store).
-define(NUM_PARTITIONS, 32).

get_partition(Key) ->
    erlang:phash2(Key, ?NUM_PARTITIONS).

get_worker(Key) ->
    PartitionId = get_partition(Key),
    list_to_atom("store_worker_" ++ integer_to_list(PartitionId)).

get(Key) ->
    gen_server:call(get_worker(Key), {get, Key}).

put(Key, Value) ->
    gen_server:call(get_worker(Key), {put, Key, Value}).

%% Pattern: Avoid gen_server bottleneck for reads
%% Use ETS directly for reads, gen_server only for writes
-module(fast_store).
-define(TABLE, fast_store_table).

read(Key) ->
    %% No gen_server call — direct ETS read
    case ets:lookup(?TABLE, Key) of
        [{Key, Value}] -> {ok, Value};
        [] -> {error, not_found}
    end.

write(Key, Value) ->
    %% Serialize writes through gen_server
    gen_server:call(fast_store_server, {write, Key, Value}).
```

---

## 6. Zero-Downtime Architecture

```erlang
%% hot_upgrade.erl — performing hot upgrade safely
-module(hot_upgrade).
-export([prepare/1, execute/1, verify/1, rollback/1]).

prepare(Release) ->
    logger:info("Preparing upgrade to ~s", [Release]),
    %% 1. Unpack release
    ok = release_handler:unpack_release(Release),
    %% 2. Check for conflicts
    Conflicts = check_conflicts(Release),
    case Conflicts of
        [] -> {ok, ready};
        _  -> {error, {conflicts, Conflicts}}
    end.

execute(Release) ->
    logger:info("Executing upgrade to ~s", [Release]),
    %% 1. Install new code
    case release_handler:install_release(Release) of
        {ok, _, _} ->
            logger:info("Release ~s installed", [Release]),
            {ok, installed};
        {error, Reason} ->
            {error, Reason}
    end.

verify(Release) ->
    %% Check all key services are healthy after upgrade
    Checks = [
        fun() -> http_health_check("http://localhost:8080/health") end,
        fun() -> db:query("SELECT 1", []) end,
        fun() -> gen_server:call(myapp_server, ping) end
    ],
    Results = [C() || C <- Checks],
    case lists:all(fun(ok) -> true; ({ok, _}) -> true; (_) -> false end, Results) of
        true  ->
            release_handler:make_permanent(Release),
            {ok, verified};
        false ->
            {error, health_check_failed}
    end.

rollback(PreviousRelease) ->
    logger:warning("Rolling back to ~s", [PreviousRelease]),
    case release_handler:install_release(PreviousRelease) of
        {ok, _, _} ->
            release_handler:make_permanent(PreviousRelease),
            {ok, rolled_back};
        {error, Reason} ->
            {error, Reason}
    end.

check_conflicts(_Release) -> [].   %% check appup files etc
http_health_check(_Url) -> ok.     %% HTTP GET and check 200
```

---

## 7. Production Checklist

```
Pre-Production Checklist:

Infrastructure:
  ✓ Erlang OTP 26+ with SMP and async threads
  ✓ ulimit -n 65536 (file descriptors)
  ✓ vm.args: +P 1000000 (max processes)
  ✓ vm.args: +Q 65536 (port limit)
  ✓ Heartbeat enabled: -heart
  ✓ Crash dump location: ERL_CRASH_DUMP

Code:
  ✓ No global state (no put/get except debug)
  ✓ All gen_server calls have timeouts
  ✓ All external calls wrapped with try/catch or monitors
  ✓ DB queries use parameterized statements only
  ✓ Secrets from environment variables only
  ✓ log level configurable at runtime

Testing:
  ✓ Unit test coverage > 80%
  ✓ Integration tests passing
  ✓ Load test: 2x expected peak
  ✓ Chaos test: node failure, DB failure, network partition

Monitoring:
  ✓ /health endpoint returning 200
  ✓ Metrics endpoint at /metrics
  ✓ Alerting on error rate, latency, memory
  ✓ Log aggregation configured
  ✓ Distributed tracing enabled

Operations:
  ✓ Deployment script with rollback
  ✓ Runbook documented
  ✓ On-call rotation configured
  ✓ Backup and restore tested
```

---

## 8. World-Class Case Studies

```
Erlang / BEAM in Production:

1. WhatsApp (Facebook/Meta)
   - 2 million connections per server
   - 900M+ users served by small engineering team
   - Key: Erlang's lightweight processes + message passing

2. Discord
   - Elixir (Erlang VM) for presence system
   - 5 million concurrent users in a single guild
   - Key: GenServer + ETS for O(1) lookups

3. Ericsson Telecommunications
   - AXD301 switch: 99.9999999% uptime (9 nines)
   - Hot code upgrades while serving traffic
   - Key: OTP supervision trees + hot upgrades

4. RabbitMQ
   - Message broker written in Erlang
   - Used by millions of applications
   - Key: Reliable message delivery + clustering

5. CouchDB
   - Distributed document database
   - "Crash only" design
   - Key: Erlang's fault tolerance + Mnesia patterns

Lessons from production systems:
  - Process-per-connection scales to millions
  - Let it crash > defensive programming
  - Supervision trees make failures recoverable
  - ETS for read-heavy, gen_server for write serialization
  - Hot code upgrades require careful state migration
  - Distribution is hard — test network partitions
```

---

## สรุป Part 50: หลักเหตุใจ Erlang ระดับโลก

✅ Erlang design principles  
✅ Let it crash philosophy  
✅ Process design patterns  
✅ Production supervisor trees  
✅ Massive concurrency (millions of connections)  
✅ Zero-downtime hot upgrades  
✅ Production checklist  
✅ Real-world case studies  

---

**ยินดีด้วย! คุณเรียน Erlang ครบ 50 Parts แล้ว!**  
ตั้งแต่ Hello World จนถึงระบบ distributed ระดับ WhatsApp

Parts 51-100 จะครอบคลุม:
- Advanced distributed patterns
- Real-world projects (fintech, gaming, telecom)
- Performance tuning ขั้นสูง  
- Contributing to open source Erlang

---

*Part 50/100 | [← ก่อนหน้า](../part49/README.md) | [ถัดไป →](../part51/README.md)*
