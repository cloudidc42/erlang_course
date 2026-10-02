# Part 27: Background Jobs และ Task Queues

> **"Process everything asynchronously — never block the user"**  
> ประมวลผลทุกอย่างแบบ asynchronous — อย่าทำให้ user รอ

---

## สารบัญ

1. [Background Jobs คืออะไร?](#1-background-jobs-คืออะไร)
2. [Simple Job Queue ด้วย GenServer](#2-simple-job-queue-ด้วย-genserver)
3. [Worker Pool Pattern](#3-worker-pool-pattern)
4. [Persistent Job Queue](#4-persistent-job-queue)
5. [Job Scheduling](#5-job-scheduling)
6. [Priority Queue](#6-priority-queue)
7. [Dead Letter Queue](#7-dead-letter-queue)
8. [Monitoring and Metrics](#8-monitoring-and-metrics)
9. [แบบฝึกหัด](#9-แบบฝึกหัด)

---

## 1. Background Jobs คืออะไร?

```
Background Jobs Pattern:

Web Request    → [API Handler] → put_job → [Queue]
                      ↓                       ↓
               return {ok, job_id}       [Workers]
                                              ↓
                                         process job
                                              ↓
                                         store result

ใช้เมื่อ:
- ส่ง email (ช้า)
- Resize images
- Generate reports
- Call external APIs
- Process large files
- Send notifications

ข้อดี:
- Response time เร็ว
- Retry failures ได้
- Scale workers ได้อิสระ
- Track job status ได้
```

---

## 2. Simple Job Queue ด้วย GenServer

```erlang
%% job_queue.erl — Simple in-memory job queue
-module(job_queue).
-behaviour(gen_server).

-export([start_link/0, enqueue/1, enqueue/2, dequeue/0,
         status/1, size/0]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

-record(job, {
    id       :: binary(),
    type     :: atom(),
    payload  :: term(),
    status   :: pending | running | done | failed,
    attempts :: non_neg_integer(),
    created  :: integer(),
    result   :: term()
}).

-record(state, {
    pending :: queue:queue(),
    jobs    :: #{binary() => #job{}}
}).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

enqueue(Job) -> enqueue(undefined, Job).
enqueue(Type, Payload) ->
    gen_server:call(?MODULE, {enqueue, Type, Payload}).

dequeue() ->
    gen_server:call(?MODULE, dequeue).

status(JobId) ->
    gen_server:call(?MODULE, {status, JobId}).

size() ->
    gen_server:call(?MODULE, size).

init([]) ->
    {ok, #state{pending=queue:new(), jobs=#{}}}.

handle_call({enqueue, Type, Payload}, _From, S) ->
    Id = make_id(),
    Job = #job{
        id=Id, type=Type, payload=Payload,
        status=pending, attempts=0,
        created=erlang:system_time(second),
        result=undefined
    },
    Queue2 = queue:in(Id, S#state.pending),
    Jobs2  = (S#state.jobs)#{Id => Job},
    {reply, {ok, Id}, S#state{pending=Queue2, jobs=Jobs2}};

handle_call(dequeue, _From, S) ->
    case queue:out(S#state.pending) of
        {empty, _} ->
            {reply, empty, S};
        {{value, Id}, Queue2} ->
            Job = maps:get(Id, S#state.jobs),
            Job2 = Job#job{status=running, attempts=Job#job.attempts+1},
            Jobs2 = (S#state.jobs)#{Id => Job2},
            {reply, {ok, Job2}, S#state{pending=Queue2, jobs=Jobs2}}
    end;

handle_call({status, Id}, _From, S) ->
    case maps:find(Id, S#state.jobs) of
        {ok, Job} -> {reply, {ok, Job}, S};
        error     -> {reply, {error, not_found}, S}
    end;

handle_call(size, _From, S) ->
    {reply, queue:len(S#state.pending), S};

handle_cast({complete, Id, Result}, S) ->
    Jobs2 = maps:update_with(Id,
        fun(J) -> J#job{status=done, result=Result} end,
        S#state.jobs),
    {noreply, S#state{jobs=Jobs2}};

handle_cast({fail, Id, Reason}, S) ->
    Jobs2 = maps:update_with(Id,
        fun(J) -> J#job{status=failed, result=Reason} end,
        S#state.jobs),
    {noreply, S#state{jobs=Jobs2}};

handle_info(_, S) -> {noreply, S}.

make_id() ->
    binary:encode_hex(crypto:strong_rand_bytes(8)).
```

---

## 3. Worker Pool Pattern

```erlang
%% job_worker.erl — Worker ที่ดึง job จาก queue
-module(job_worker).
-behaviour(gen_server).

-export([start_link/1]).
-export([init/1, handle_info/2, handle_cast/2, handle_call/3]).

start_link(Id) ->
    gen_server:start_link(?MODULE, [Id], []).

init([Id]) ->
    logger:info("Worker ~p started", [Id]),
    %% เริ่ม poll queue ทันที
    self() ! poll,
    {ok, #{id => Id}}.

handle_info(poll, State) ->
    case job_queue:dequeue() of
        empty ->
            %% ไม่มี job: รอ 500ms แล้ว poll ใหม่
            erlang:send_after(500, self(), poll),
            {noreply, State};
        {ok, Job} ->
            process_job(Job),
            %% poll ต่อทันที
            self() ! poll,
            {noreply, State}
    end;

handle_info(_, State) -> {noreply, State}.
handle_cast(_, State) -> {noreply, State}.
handle_call(_, _, State) -> {noreply, State}.

process_job(#job{id=Id, type=Type, payload=Payload}) ->
    logger:info("Processing job ~s type=~p", [Id, Type]),
    Result = try
        execute_job(Type, Payload)
    catch
        Class:Reason:Stack ->
            logger:error("Job ~s failed: ~p:~p~n~p",
                         [Id, Class, Reason, Stack]),
            gen_server:cast(job_queue, {fail, Id, {Class, Reason}}),
            error
    end,
    case Result of
        error -> ok;
        Value -> gen_server:cast(job_queue, {complete, Id, Value})
    end.

execute_job(send_email, #{to:=To, subject:=Subj, body:=Body}) ->
    mailer:send(To, Subj, Body);
execute_job(resize_image, #{path:=Path, width:=W, height:=H}) ->
    image_processor:resize(Path, W, H);
execute_job(Type, Payload) ->
    logger:warning("Unknown job type: ~p payload: ~p", [Type, Payload]),
    {error, unknown_type}.
```

---

## 4. Persistent Job Queue

```erlang
%% ใช้ ETS + Disk สำหรับ persistence
-module(persistent_job_queue).
-behaviour(gen_server).

-export([start_link/0, enqueue/2, ack/1, nack/2, pending_count/0]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

-record(job, {
    id        :: binary(),
    type      :: atom(),
    payload   :: term(),
    attempts  :: non_neg_integer(),
    max_attempts :: pos_integer(),
    scheduled :: integer(),   %% unix timestamp
    created   :: integer()
}).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

enqueue(Type, Payload) ->
    gen_server:call(?MODULE, {enqueue, Type, Payload}).

ack(JobId) ->
    gen_server:cast(?MODULE, {ack, JobId}).

nack(JobId, RetryAfter) ->
    gen_server:cast(?MODULE, {nack, JobId, RetryAfter}).

pending_count() ->
    gen_server:call(?MODULE, pending_count).

init([]) ->
    ets:new(jobs, [named_table, set, public, {keypos, #job.id}]),
    ets:new(scheduled, [named_table, ordered_set, public]),
    load_from_disk(),
    erlang:send_after(1000, self(), tick),
    {ok, #{}}.

handle_call({enqueue, Type, Payload}, _From, S) ->
    Now = erlang:system_time(second),
    Id = make_id(),
    Job = #job{
        id=Id, type=Type, payload=Payload,
        attempts=0, max_attempts=3,
        scheduled=Now, created=Now
    },
    ets:insert(jobs, Job),
    ets:insert(scheduled, {Now, Id}),
    persist(Job),
    {reply, {ok, Id}, S};

handle_call(pending_count, _From, S) ->
    Now = erlang:system_time(second),
    Count = ets:select_count(scheduled,
        [{{'$1','$2'}, [{'=<','$1',Now}], [true]}]),
    {reply, Count, S};

handle_cast({ack, Id}, S) ->
    ets:delete(jobs, Id),
    delete_persisted(Id),
    {noreply, S};

handle_cast({nack, Id, RetryAfter}, S) ->
    case ets:lookup(jobs, Id) of
        [Job] when Job#job.attempts < Job#job.max_attempts ->
            NewScheduled = erlang:system_time(second) + RetryAfter,
            ets:delete(scheduled, {Job#job.scheduled, Id}),
            Job2 = Job#job{attempts=Job#job.attempts+1, scheduled=NewScheduled},
            ets:insert(jobs, Job2),
            ets:insert(scheduled, {NewScheduled, Id}),
            persist(Job2);
        [Job] ->
            logger:error("Job ~s exceeded max retries", [Job#job.id]),
            ets:delete(jobs, Id),
            delete_persisted(Id)
    end,
    {noreply, S};

handle_info(tick, S) ->
    dispatch_ready_jobs(),
    erlang:send_after(1000, self(), tick),
    {noreply, S}.

dispatch_ready_jobs() ->
    Now = erlang:system_time(second),
    Ready = ets:select(scheduled,
        [{{'$1','$2'}, [{'=<','$1',Now}], ['$2']}],
        10),  %% batch 10 jobs
    case Ready of
        '$end_of_table' -> ok;
        {Ids, _} ->
            lists:foreach(fun dispatch_job/1, Ids)
    end.

dispatch_job(Id) ->
    case ets:lookup(jobs, Id) of
        [Job] ->
            job_dispatcher:dispatch(Job);
        [] ->
            ets:delete(scheduled, {0, Id})  %% stale
    end.

make_id() -> binary:encode_hex(crypto:strong_rand_bytes(8)).
persist(_Job) -> ok.       %% TODO: write to disk/db
load_from_disk() -> ok.    %% TODO: load on startup
delete_persisted(_Id) -> ok.
```

---

## 5. Job Scheduling

```erlang
%% cron-like job scheduler
-module(job_scheduler).
-behaviour(gen_server).

-export([start_link/0, add_job/3, remove_job/1, list_jobs/0]).
-export([init/1, handle_info/2, handle_cast/2, handle_call/3]).

-record(scheduled_job, {
    name    :: atom(),
    cron    :: binary(),     %% "*/5 * * * *"
    handler :: fun()
}).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

add_job(Name, Cron, Handler) ->
    gen_server:call(?MODULE, {add, Name, Cron, Handler}).

remove_job(Name) ->
    gen_server:cast(?MODULE, {remove, Name}).

list_jobs() ->
    gen_server:call(?MODULE, list).

init([]) ->
    erlang:send_after(60000, self(), tick),
    {ok, #{}}.

handle_info(tick, Jobs) ->
    Now = erlang:system_time(second),
    maps:foreach(fun(_Name, Job) ->
        case should_run(Job#scheduled_job.cron, Now) of
            true ->
                spawn(Job#scheduled_job.handler);
            false ->
                ok
        end
    end, Jobs),
    erlang:send_after(60000, self(), tick),
    {noreply, Jobs};

handle_call({add, Name, Cron, Handler}, _From, Jobs) ->
    Job = #scheduled_job{name=Name, cron=Cron, handler=Handler},
    {reply, ok, Jobs#{Name => Job}};

handle_call(list, _From, Jobs) ->
    {reply, maps:keys(Jobs), Jobs};

handle_cast({remove, Name}, Jobs) ->
    {noreply, maps:remove(Name, Jobs)}.

should_run(_Cron, _Now) ->
    %% TODO: implement cron expression matching
    false.

%% ตัวอย่างใช้งาน
setup_jobs() ->
    job_scheduler:add_job(cleanup_old_sessions,
        <<"0 * * * *">>,  %% every hour
        fun() -> session_store:cleanup() end),
    job_scheduler:add_job(send_daily_digest,
        <<"0 8 * * *">>,  %% 8am daily
        fun() -> email:send_digests() end).
```

---

## 6. Priority Queue

```erlang
%% priority_job_queue.erl
-module(priority_job_queue).
-behaviour(gen_server).

-export([start_link/0, enqueue/3, dequeue/0]).
-export([init/1, handle_call/3, handle_cast/2]).

%% Priority levels
-define(HIGH,   1).
-define(NORMAL, 5).
-define(LOW,    10).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

enqueue(Priority, Type, Payload) ->
    gen_server:call(?MODULE, {enqueue, Priority, Type, Payload}).

dequeue() ->
    gen_server:call(?MODULE, dequeue).

init([]) ->
    %% ใช้ ordered_set: key = {priority, timestamp, id}
    ets:new(pq, [named_table, ordered_set, public]),
    {ok, #{}}.

handle_call({enqueue, Priority, Type, Payload}, _From, S) ->
    Id  = make_id(),
    Now = erlang:monotonic_time(),
    Key = {Priority, Now, Id},
    ets:insert(pq, {Key, Type, Payload}),
    {reply, {ok, Id}, S};

handle_call(dequeue, _From, S) ->
    case ets:first(pq) of
        '$end_of_table' ->
            {reply, empty, S};
        Key ->
            [{Key, Type, Payload}] = ets:lookup(pq, Key),
            ets:delete(pq, Key),
            {_, _, Id} = Key,
            {reply, {ok, Id, Type, Payload}, S}
    end.

make_id() -> binary:encode_hex(crypto:strong_rand_bytes(4)).
```

---

## 7. Dead Letter Queue

```erlang
%% Jobs ที่ fail เกิน limit → ย้ายไป DLQ
-module(dead_letter_queue).
-behaviour(gen_server).

-export([start_link/0, move_to_dlq/2, list/0, retry/1, purge/0]).
-export([init/1, handle_call/3, handle_cast/2]).

-record(dead_job, {
    id         :: binary(),
    original   :: term(),
    failure    :: term(),
    failed_at  :: integer(),
    attempts   :: non_neg_integer()
}).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

move_to_dlq(Job, Reason) ->
    gen_server:cast(?MODULE, {add, Job, Reason}).

list() ->
    gen_server:call(?MODULE, list).

retry(JobId) ->
    gen_server:call(?MODULE, {retry, JobId}).

purge() ->
    gen_server:cast(?MODULE, purge).

init([]) ->
    {ok, #{}}.

handle_cast({add, Job, Reason}, DLQ) ->
    Dead = #dead_job{
        id       = maps:get(id, Job),
        original = Job,
        failure  = Reason,
        failed_at = erlang:system_time(second),
        attempts = maps:get(attempts, Job, 0)
    },
    {noreply, DLQ#{Dead#dead_job.id => Dead}};

handle_cast(purge, _DLQ) ->
    {noreply, #{}};

handle_call(list, _From, DLQ) ->
    {reply, maps:values(DLQ), DLQ};

handle_call({retry, Id}, _From, DLQ) ->
    case maps:find(Id, DLQ) of
        {ok, Dead} ->
            job_queue:enqueue(maps:get(type, Dead#dead_job.original),
                              maps:get(payload, Dead#dead_job.original)),
            {reply, ok, maps:remove(Id, DLQ)};
        error ->
            {reply, {error, not_found}, DLQ}
    end.
```

---

## 8. Monitoring and Metrics

```erlang
%% job_metrics.erl
-module(job_metrics).
-export([track_enqueue/1, track_start/1, track_complete/2, track_fail/2,
         stats/0]).

track_enqueue(Type) ->
    counter_inc({enqueued, Type}).

track_start(Type) ->
    counter_inc({started, Type}).

track_complete(Type, DurationMs) ->
    counter_inc({completed, Type}),
    histogram_record({duration_ms, Type}, DurationMs).

track_fail(Type, Reason) ->
    counter_inc({failed, Type}),
    counter_inc({failed_reason, Type, Reason}).

stats() ->
    #{
        enqueued  => get_counters(enqueued),
        started   => get_counters(started),
        completed => get_counters(completed),
        failed    => get_counters(failed),
        durations => get_histograms()
    }.

counter_inc(Key) ->
    ets:update_counter(job_metrics, Key, 1, {Key, 0}).

histogram_record(Key, Value) ->
    ets:insert(job_histograms, {erlang:unique_integer([monotonic]), Key, Value}).

get_counters(Prefix) ->
    Pattern = {{Prefix, '_'}, '$1'},
    ets:match(job_metrics, Pattern).

get_histograms() ->
    All = ets:tab2list(job_histograms),
    lists:foldl(fun({_, Key, Value}, Acc) ->
        maps:update_with(Key,
            fun(#{count:=C, sum:=S}) -> #{count=>C+1, sum=>S+Value} end,
            #{count=>1, sum=>Value},
            Acc)
    end, #{}, All).
```

---

## 9. แบบฝึกหัด

### Exercise: Email Job Queue

```erlang
%% สร้าง email job queue ที่:
%% 1. Enqueue email jobs
%% 2. Process 5 emails concurrently
%% 3. Retry 3 ครั้ง ถ้าล้มเหลว
%% 4. Move to DLQ หลัง 3 failures

%% email_job_server.erl
-module(email_job_server).
-behaviour(gen_server).

-export([start_link/0, send_email/3]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

-define(MAX_WORKERS, 5).
-define(MAX_RETRIES, 3).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

send_email(To, Subject, Body) ->
    JobId = job_queue:enqueue(send_email, #{
        to => To,
        subject => Subject,
        body => Body
    }),
    {ok, JobId}.

init([]) ->
    %% Start worker pool
    Workers = [spawn_link(fun worker_loop/0)
               || _ <- lists:seq(1, ?MAX_WORKERS)],
    {ok, #{workers => Workers}}.

worker_loop() ->
    case job_queue:dequeue() of
        empty ->
            timer:sleep(200),
            worker_loop();
        {ok, Job} ->
            Result = try
                #{to:=To, subject:=S, body:=B} = Job#job.payload,
                mailer:send(To, S, B)
            catch
                C:R -> {error, {C, R}}
            end,
            case Result of
                ok -> job_queue:ack(Job#job.id);
                {error, Reason} when Job#job.attempts < ?MAX_RETRIES ->
                    job_queue:nack(Job#job.id, 60);  %% retry in 60s
                {error, Reason} ->
                    dead_letter_queue:move_to_dlq(Job, Reason)
            end,
            worker_loop()
    end.
```

---

## สรุป Part 27

✅ Background jobs concept  
✅ Simple GenServer job queue  
✅ Worker pool pattern  
✅ Persistent job queue ด้วย ETS  
✅ Job scheduling (cron-like)  
✅ Priority queue  
✅ Dead letter queue  
✅ Monitoring and metrics

---

*Part 27/100 | [← ก่อนหน้า](../part26/README.md) | [ถัดไป →](../part28/README.md)*
