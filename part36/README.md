# Part 36: Real-World Project — Job Queue System

> **"Decouple producers from consumers — the core of scalable systems"**  
> แยก producers จาก consumers — หัวใจของ scalable systems

---

## สารบัญ

1. [System Overview](#1-system-overview)
2. [Job Definition](#2-job-definition)
3. [Queue Server](#3-queue-server)
4. [Worker Supervisor](#4-worker-supervisor)
5. [Job Handlers](#5-job-handlers)
6. [Scheduler](#6-scheduler)
7. [Dashboard API](#7-dashboard-api)
8. [Monitoring](#8-monitoring)
9. [Testing](#9-testing)

---

## 1. System Overview

```
Job Queue System:

[Producers]           [Queue]          [Workers]
  Web API    ──put──► job_queue ─get─► worker_1
  Scheduler  ──put──► job_queue ─get─► worker_2
  Cron jobs  ──put──► job_queue ─get─► worker_3
                          │
                     [Job Store]
                          │
                     [Dead Letter Queue]

Job lifecycle:
pending → running → done
                 → failed (retry 1)
                 → failed (retry 2)
                 → failed (retry 3) → dead_letter
```

---

## 2. Job Definition

```erlang
%% job.erl — Job data structure
-module(job).
-export([new/2, new/3]).

-record(job, {
    id          :: binary(),
    type        :: atom(),
    payload     :: map(),
    status      :: pending | running | done | failed | dead,
    priority    :: 1..10,
    attempts    :: non_neg_integer(),
    max_attempts :: pos_integer(),
    queue       :: atom(),           %% default | email | urgent
    scheduled_at :: integer(),       %% unix timestamp
    started_at   :: integer() | undefined,
    done_at      :: integer() | undefined,
    result       :: term(),
    error        :: term(),
    created_at   :: integer()
}).

-type job() :: #job{}.
-export_type([job/0]).

new(Type, Payload) ->
    new(Type, Payload, []).

new(Type, Payload, Opts) ->
    #job{
        id          = make_id(),
        type        = Type,
        payload     = Payload,
        status      = pending,
        priority    = proplists:get_value(priority, Opts, 5),
        attempts    = 0,
        max_attempts = proplists:get_value(max_attempts, Opts, 3),
        queue       = proplists:get_value(queue, Opts, default),
        scheduled_at = proplists:get_value(scheduled_at, Opts,
                           erlang:system_time(second)),
        started_at  = undefined,
        done_at     = undefined,
        result      = undefined,
        error       = undefined,
        created_at  = erlang:system_time(second)
    }.

make_id() ->
    binary:encode_hex(crypto:strong_rand_bytes(8)).
```

---

## 3. Queue Server

```erlang
%% job_queue_server.erl
-module(job_queue_server).
-behaviour(gen_server).

-export([start_link/0, enqueue/1, dequeue/1, ack/2, nack/2,
         stats/0, list_jobs/1]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

enqueue(Job) ->
    gen_server:call(?MODULE, {enqueue, Job}).

dequeue(Queue) ->
    gen_server:call(?MODULE, {dequeue, Queue}).

ack(JobId, Result) ->
    gen_server:cast(?MODULE, {ack, JobId, Result}).

nack(JobId, Error) ->
    gen_server:cast(?MODULE, {nack, JobId, Error}).

stats() ->
    gen_server:call(?MODULE, stats).

list_jobs(Status) ->
    gen_server:call(?MODULE, {list, Status}).

init([]) ->
    %% Priority queue: ordered_set keyed by {priority, scheduled_at, id}
    ets:new(job_queue, [named_table, ordered_set, protected,
                        {keypos, 1}]),
    ets:new(job_store, [named_table, set, protected,
                        {keypos, #job.id}]),
    ets:new(job_counters, [named_table, set, public]),
    erlang:send_after(5000, self(), dispatch_scheduled),
    {ok, #{}}.

handle_call({enqueue, Job}, _From, S) ->
    Key = {Job#job.priority, Job#job.scheduled_at, Job#job.id},
    ets:insert(job_queue, {Key, Job#job.queue}),
    ets:insert(job_store, Job),
    counter_inc(Job#job.queue, pending),
    {reply, {ok, Job#job.id}, S};

handle_call({dequeue, Queue}, _From, S) ->
    Now = erlang:system_time(second),
    %% Find first job in the given queue that's ready to run
    Result = find_ready_job(Queue, Now),
    case Result of
        {ok, Key, Job} ->
            ets:delete(job_queue, Key),
            Job2 = Job#job{status=running,
                           started_at=Now,
                           attempts=Job#job.attempts+1},
            ets:insert(job_store, Job2),
            counter_dec(Queue, pending),
            counter_inc(Queue, running),
            {reply, {ok, Job2}, S};
        empty ->
            {reply, empty, S}
    end;

handle_call(stats, _From, S) ->
    Stats = #{
        pending => get_counter(default, pending),
        running => get_counter(default, running),
        done    => get_counter(default, done),
        failed  => get_counter(default, failed),
        queue_size => ets:info(job_queue, size)
    },
    {reply, Stats, S};

handle_call({list, Status}, _From, S) ->
    Jobs = ets:select(job_store,
        [{#job{status='$1', _='_'}, [{'==','$1',Status}], ['$_']}]),
    {reply, Jobs, S};

handle_cast({ack, JobId, Result}, S) ->
    case ets:lookup(job_store, JobId) of
        [Job] ->
            Job2 = Job#job{status=done, result=Result,
                           done_at=erlang:system_time(second)},
            ets:insert(job_store, Job2),
            counter_dec(Job#job.queue, running),
            counter_inc(Job#job.queue, done);
        [] -> ok
    end,
    {noreply, S};

handle_cast({nack, JobId, Error}, S) ->
    case ets:lookup(job_store, JobId) of
        [Job] ->
            if Job#job.attempts < Job#job.max_attempts ->
                %% Retry with exponential backoff
                Delay = retry_delay(Job#job.attempts),
                ScheduledAt = erlang:system_time(second) + Delay,
                Job2 = Job#job{status=pending, error=Error,
                               scheduled_at=ScheduledAt},
                ets:insert(job_store, Job2),
                Key = {Job2#job.priority, ScheduledAt, JobId},
                ets:insert(job_queue, {Key, Job2#job.queue}),
                counter_dec(Job#job.queue, running),
                counter_inc(Job#job.queue, pending);
            true ->
                %% Max retries: move to dead letter
                Job2 = Job#job{status=dead, error=Error,
                               done_at=erlang:system_time(second)},
                ets:insert(job_store, Job2),
                counter_dec(Job#job.queue, running),
                counter_inc(Job#job.queue, dead),
                logger:error("Job ~s dead after ~p attempts: ~p",
                             [JobId, Job#job.attempts, Error])
            end;
        [] -> ok
    end,
    {noreply, S};

handle_info(dispatch_scheduled, S) ->
    erlang:send_after(5000, self(), dispatch_scheduled),
    {noreply, S};
handle_info(_, S) -> {noreply, S}.

find_ready_job(Queue, Now) ->
    %% Use ets:select for efficient lookup
    Pattern = {{'$1', '$2', '$3'}, Queue},
    case ets:match(job_queue, Pattern, 1) of
        {[[P, ScheduledAt, Id]], _} when ScheduledAt =< Now ->
            case ets:lookup(job_store, Id) of
                [Job] -> {ok, {P, ScheduledAt, Id}, Job};
                []    -> empty
            end;
        _ -> empty
    end.

retry_delay(1) -> 30;
retry_delay(2) -> 300;
retry_delay(_) -> 3600.

counter_inc(Queue, Status) ->
    ets:update_counter(job_counters, {Queue, Status}, 1, {{Queue, Status}, 0}).

counter_dec(Queue, Status) ->
    ets:update_counter(job_counters, {Queue, Status}, -1, {{Queue, Status}, 0}).

get_counter(Queue, Status) ->
    case ets:lookup(job_counters, {Queue, Status}) of
        [{{_,_}, N}] -> N;
        []           -> 0
    end.
```

---

## 4. Worker Supervisor

```erlang
%% job_worker_sup.erl
-module(job_worker_sup).
-behaviour(supervisor).

-export([start_link/0, init/1]).

start_link() ->
    supervisor:start_link({local, ?MODULE}, ?MODULE, []).

init([]) ->
    %% Start N workers per queue
    Workers = lists:flatmap(fun({Queue, Count}) ->
        [#{id      => {worker, Queue, I},
           start   => {job_worker, start_link, [Queue]},
           restart => permanent,
           type    => worker}
         || I <- lists:seq(1, Count)]
    end, queue_config()),
    {ok, {{one_for_one, 10, 60}, Workers}}.

queue_config() ->
    [
        {default, 5},   %% 5 workers for default queue
        {email,   2},   %% 2 workers for email queue
        {urgent,  10}   %% 10 workers for urgent queue
    ].
```

---

## 5. Job Handlers

```erlang
%% job_worker.erl
-module(job_worker).
-behaviour(gen_server).

-export([start_link/1]).
-export([init/1, handle_info/2, handle_call/3]).

start_link(Queue) ->
    gen_server:start_link(?MODULE, [Queue], []).

init([Queue]) ->
    self() ! poll,
    {ok, #{queue => Queue}}.

handle_info(poll, #{queue := Queue} = S) ->
    case job_queue_server:dequeue(Queue) of
        empty ->
            erlang:send_after(500, self(), poll);
        {ok, Job} ->
            execute(Job),
            self() ! poll
    end,
    {noreply, S};
handle_info(_, S) -> {noreply, S}.

handle_call(_, _, S) -> {reply, ok, S}.

execute(Job) ->
    T1 = erlang:monotonic_time(millisecond),
    try
        Result = dispatch(Job),
        T2 = erlang:monotonic_time(millisecond),
        logger:info("Job ~s done in ~pms", [Job#job.id, T2-T1]),
        job_queue_server:ack(Job#job.id, Result)
    catch
        Class:Reason:Stack ->
            logger:error("Job ~s failed: ~p:~p~n~p",
                         [Job#job.id, Class, Reason, Stack]),
            job_queue_server:nack(Job#job.id, {Class, Reason})
    end.

%% Dispatch to handler modules
dispatch(#job{type=send_email, payload=P}) ->
    email_handler:handle(P);
dispatch(#job{type=resize_image, payload=P}) ->
    image_handler:handle(P);
dispatch(#job{type=generate_report, payload=P}) ->
    report_handler:handle(P);
dispatch(#job{type=Type, payload=P}) ->
    logger:warning("Unknown job type: ~p", [Type]),
    {error, unknown_type}.

%% email_handler.erl
-module(email_handler).
-export([handle/1]).

handle(#{to:=To, subject:=Subject, body:=Body}) ->
    %% Send via SMTP
    smtplib:send(#{
        from    => <<"noreply@myapp.com">>,
        to      => To,
        subject => Subject,
        body    => Body
    }).
```

---

## 6. Scheduler

```erlang
%% job_scheduler.erl — Cron-like job scheduler
-module(job_scheduler).
-behaviour(gen_server).

-export([start_link/0, add/3, remove/1, list/0]).
-export([init/1, handle_info/2, handle_call/3]).

-record(schedule, {
    id      :: binary(),
    name    :: binary(),
    cron    :: binary(),   %% "*/5 * * * *"
    type    :: atom(),
    payload :: map(),
    last_run :: integer() | undefined
}).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

add(Name, Cron, {Type, Payload}) ->
    gen_server:call(?MODULE, {add, Name, Cron, Type, Payload}).

remove(Id) ->
    gen_server:cast(?MODULE, {remove, Id}).

list() ->
    gen_server:call(?MODULE, list).

init([]) ->
    erlang:send_after(60000, self(), tick),
    {ok, #{}}.

handle_info(tick, Schedules) ->
    Now = erlang:system_time(second),
    maps:foreach(fun(Id, S) ->
        case should_run(S#schedule.cron, Now) of
            true ->
                Job = job:new(S#schedule.type, S#schedule.payload),
                job_queue_server:enqueue(Job),
                logger:info("Scheduled job ~s triggered", [S#schedule.name]);
            false ->
                ok
        end
    end, Schedules),
    erlang:send_after(60000, self(), tick),
    {noreply, Schedules};

handle_call({add, Name, Cron, Type, Payload}, _From, Schedules) ->
    Id = job:make_id(),
    S  = #schedule{id=Id, name=Name, cron=Cron,
                   type=Type, payload=Payload, last_run=undefined},
    {reply, {ok, Id}, Schedules#{Id => S}};

handle_call(list, _From, Schedules) ->
    {reply, maps:values(Schedules), Schedules};

handle_cast({remove, Id}, Schedules) ->
    {noreply, maps:remove(Id, Schedules)}.

should_run(_Cron, _Now) ->
    %% Simple: always run (TODO: implement cron parser)
    true.
```

---

## 7. Dashboard API

```erlang
%% job_dashboard_handler.erl
-module(job_dashboard_handler).
-export([init/2]).

init(Req, State) ->
    Path = cowboy_req:path(Req),
    Method = cowboy_req:method(Req),
    handle(Method, Path, Req, State).

handle(<<"GET">>, <<"/api/jobs/stats">>, Req, State) ->
    Stats = job_queue_server:stats(),
    reply_json(200, Stats, Req, State);

handle(<<"GET">>, <<"/api/jobs">>, Req, State) ->
    Qs = cowboy_req:parse_qs(Req),
    Status = case lists:keyfind(<<"status">>, 1, Qs) of
        {_, S} -> binary_to_atom(S, utf8);
        false  -> pending
    end,
    Jobs = job_queue_server:list_jobs(Status),
    Data = [job_to_map(J) || J <- Jobs],
    reply_json(200, Data, Req, State);

handle(<<"POST">>, <<"/api/jobs/retry">>, Req, State) ->
    {ok, Body, Req2} = cowboy_req:read_body(Req),
    #{<<"job_id">> := JobId} = jsx:decode(Body, [return_maps]),
    %% Re-enqueue dead job
    case job_queue_server:list_jobs(dead) of
        Jobs ->
            case lists:keyfind(JobId, #job.id, Jobs) of
                Job when is_record(Job, job) ->
                    Job2 = Job#job{status=pending, attempts=0, error=undefined},
                    job_queue_server:enqueue(Job2),
                    reply_json(200, #{status => <<"requeued">>}, Req2, State);
                false ->
                    reply_json(404, #{error => <<"not found">>}, Req2, State)
            end
    end.

job_to_map(J) ->
    #{
        id          => J#job.id,
        type        => J#job.type,
        status      => J#job.status,
        attempts    => J#job.attempts,
        created_at  => J#job.created_at,
        error       => J#job.error
    }.

reply_json(Status, Data, Req, State) ->
    Req2 = cowboy_req:reply(Status,
        #{<<"content-type">> => <<"application/json">>},
        jsx:encode(Data), Req),
    {ok, Req2, State}.
```

---

## 8. Monitoring

```erlang
%% job_monitor.erl — Alerting for job queue health
-module(job_monitor).
-behaviour(gen_server).

-export([start_link/0]).
-export([init/1, handle_info/2]).

-define(CHECK_INTERVAL, 30000).
-define(MAX_PENDING,    1000).
-define(MAX_DEAD,       100).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

init([]) ->
    erlang:send_after(?CHECK_INTERVAL, self(), check),
    {ok, #{}}.

handle_info(check, S) ->
    Stats = job_queue_server:stats(),
    Pending = maps:get(pending, Stats, 0),
    Dead    = maps:get(dead,    Stats, 0),

    if
        Pending > ?MAX_PENDING ->
            alert(queue_overload, #{pending => Pending});
        true -> ok
    end,

    if
        Dead > ?MAX_DEAD ->
            alert(dead_jobs_accumulating, #{dead => Dead});
        true -> ok
    end,

    %% Report metrics
    telemetry:execute([job_queue, stats], Stats, #{}),

    erlang:send_after(?CHECK_INTERVAL, self(), check),
    {noreply, S};
handle_info(_, S) -> {noreply, S}.

alert(Type, Data) ->
    logger:warning("Job queue alert: ~p ~p", [Type, Data]),
    %% Send to Slack/PagerDuty/etc
    notify:send(Type, Data).
```

---

## 9. Testing

```erlang
%% job_system_tests.erl
-module(job_system_tests).
-include_lib("eunit/include/eunit.hrl").

enqueue_dequeue_test() ->
    job_queue_server:start_link(),
    Job = job:new(test_job, #{data => <<"test">>}),
    {ok, _Id} = job_queue_server:enqueue(Job),
    {ok, Got} = job_queue_server:dequeue(default),
    ?assertEqual(running, Got#job.status),
    ?assertEqual(test_job, Got#job.type).

retry_on_failure_test() ->
    Job = job:new(will_fail, #{}, [{max_attempts, 3}]),
    {ok, Id} = job_queue_server:enqueue(Job),
    {ok, J1} = job_queue_server:dequeue(default),
    job_queue_server:nack(Id, <<"first failure">>),

    %% After nack, job should be re-queued (pending)
    timer:sleep(100),
    {ok, J2} = job_queue_server:dequeue(default),
    ?assertEqual(2, J2#job.attempts).

dead_after_max_retries_test() ->
    Job = job:new(will_die, #{}, [{max_attempts, 1}]),
    {ok, Id} = job_queue_server:enqueue(Job),
    {ok, _}  = job_queue_server:dequeue(default),
    job_queue_server:nack(Id, <<"fatal error">>),
    timer:sleep(100),
    [DeadJob] = job_queue_server:list_jobs(dead),
    ?assertEqual(dead, DeadJob#job.status).
```

---

## สรุป Part 36

✅ Job queue architecture  
✅ Job record definition  
✅ Priority queue server ด้วย ETS  
✅ Worker supervisor (multiple queues)  
✅ Job handlers (email, image, report)  
✅ Cron-like scheduler  
✅ Dashboard API  
✅ Monitoring and alerting  
✅ Unit tests

---

*Part 36/100 | [← ก่อนหน้า](../part35/README.md) | [ถัดไป →](../part37/README.md)*
