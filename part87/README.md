# Part 87: Message Queue Patterns

> **"A queue is a buffer between the pace of production and the pace of consumption"**  
> Queue คือตัวกั้นระหว่างความเร็วของการผลิตและความเร็วของการบริโภค

---

## สารบัญ

1. [Message Queue Fundamentals](#1-message-queue-fundamentals)
2. [Guaranteed Delivery with Outbox Pattern](#2-guaranteed-delivery-with-outbox-pattern)
3. [Ordered Processing](#3-ordered-processing)
4. [Deduplication](#4-deduplication)
5. [Dead Letter Queue](#5-dead-letter-queue)
6. [Priority Queue](#6-priority-queue)
7. [แบบฝึกหัด](#7-แบบฝึกหัด)

---

## 1. Message Queue Fundamentals

```
Message Queue Delivery Semantics
══════════════════════════════════════════════════

At-most-once (fire and forget)
  Producer sends → maybe consumed, maybe lost
  Use when: metrics, analytics, non-critical events
  Example: ets:insert(stats_buffer, {event, now()})

At-least-once (acknowledged delivery)
  Producer sends → consumer processes → acks
  May process same message TWICE if consumer crashes after processing but before ack
  Use when: emails, notifications, payment webhooks
  Requires: idempotent consumers

Exactly-once (transactional delivery)
  Atomically dequeue + process in same DB transaction
  Expensive but correct
  Use when: financial transactions, inventory changes

Pattern Selection:
  Async notification:       At-most-once
  Event sourcing:           At-least-once + idempotency
  Distributed transaction:  Exactly-once (or Saga)
```

---

## 2. Guaranteed Delivery with Outbox Pattern

```erlang
%% outbox.erl — transactional outbox for guaranteed delivery
-module(outbox).
-export([send/3, process_pending/0, start_publisher/0]).

%% Write message to outbox IN SAME TRANSACTION as business operation
%% This guarantees: if business op committed, message WILL be delivered eventually
send(AggregateId, EventType, Payload) ->
    db_pool:execute(
        "INSERT INTO outbox (id, aggregate_id, event_type, payload, created_at)
         VALUES (gen_random_uuid(), $1, $2, $3, NOW())",
        [AggregateId, atom_to_binary(EventType), json:encode(Payload)]).

%% Periodically scan outbox and publish unpublished messages
process_pending() ->
    case db_pool:query(
        "SELECT id, aggregate_id, event_type, payload
         FROM outbox
         WHERE published_at IS NULL
         ORDER BY created_at ASC
         LIMIT 100
         FOR UPDATE SKIP LOCKED", []) of  %% SKIP LOCKED prevents duplicate processing
        {ok, Rows} ->
            lists:foreach(fun({Id, AggId, EventType, Payload}) ->
                publish_and_mark(Id, AggId, EventType, Payload)
            end, Rows);
        {error, Reason} ->
            logger:error("Outbox scan failed: ~p", [Reason])
    end.

publish_and_mark(Id, AggId, EventType, Payload) ->
    Event = #{
        aggregate_id => AggId,
        event_type   => EventType,
        payload      => json:decode(Payload)
    },
    case kafka_producer:publish(EventType, Event) of
        ok ->
            db_pool:execute(
                "UPDATE outbox SET published_at = NOW() WHERE id = $1", [Id]);
        {error, Reason} ->
            logger:warning("Failed to publish outbox message ~p: ~p", [Id, Reason])
    end.

%% Background process that runs process_pending on a schedule
start_publisher() ->
    spawn_link(fun publisher_loop/0).

publisher_loop() ->
    process_pending(),
    timer:sleep(1000),
    publisher_loop().
```

---

## 3. Ordered Processing

```erlang
%% ordered_queue.erl — per-key ordering with parallel processing of different keys
-module(ordered_queue).
-behaviour(gen_server).

-export([start_link/0, enqueue/2, worker_done/2]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

%% Each OrderingKey gets its own sequential queue
%% Different keys process in parallel; same key is always sequential
-record(state, {
    queues = #{},          %% Key -> [Message]
    active = #{},          %% Key -> {WorkerPid, Message}
    worker_pool            %% Pool of worker processes
}).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

enqueue(OrderingKey, Message) ->
    gen_server:cast(?MODULE, {enqueue, OrderingKey, Message}).

worker_done(OrderingKey, Result) ->
    gen_server:cast(?MODULE, {done, OrderingKey, Result}).

init([]) ->
    {ok, #state{}}.

handle_cast({enqueue, Key, Msg}, #state{queues = Qs, active = Active} = State) ->
    Queue = maps:get(Key, Qs, []),
    NewQs = maps:put(Key, Queue ++ [Msg], Qs),
    NewState = State#state{queues = NewQs},
    case maps:is_key(Key, Active) of
        false -> {noreply, dispatch_next(Key, NewState)};
        true  -> {noreply, NewState}  %% already running for this key
    end;

handle_cast({done, Key, _Result}, #state{active = Active} = State) ->
    NewActive = maps:remove(Key, Active),
    NewState  = State#state{active = NewActive},
    %% Dispatch next message for this key if any
    {noreply, dispatch_next(Key, NewState)}.

dispatch_next(Key, #state{queues = Qs, active = Active} = State) ->
    case maps:get(Key, Qs, []) of
        [] ->
            State;
        [Msg | Rest] ->
            Worker = spawn_link(fun() ->
                process_message(Key, Msg)
            end),
            NewQs     = maps:put(Key, Rest, Qs),
            NewActive = maps:put(Key, {Worker, Msg}, Active),
            State#state{queues = NewQs, active = NewActive}
    end.

process_message(Key, Msg) ->
    %% Do actual work here
    Result = handle_message(Msg),
    ordered_queue:worker_done(Key, Result).

handle_message(_Msg) -> ok.  %% Override with actual handler
```

---

## 4. Deduplication

```erlang
%% dedup.erl — idempotent message processing with deduplication window
-module(dedup).
-behaviour(gen_server).

-export([start_link/0, process_once/3]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

-define(DEDUP_WINDOW_SECONDS, 3600).  %% 1 hour window
-define(CLEANUP_INTERVAL_MS, 60000).
-define(DEDUP_TABLE, dedup_seen).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

%% Process message exactly once within the dedup window
process_once(MessageId, Fun, _Opts) ->
    case try_mark_seen(MessageId) of
        new ->
            try
                Result = Fun(),
                {ok, {processed, Result}}
            catch
                E:R ->
                    unmark_seen(MessageId),  %% Allow retry on failure
                    {error, {E, R}}
            end;
        duplicate ->
            {ok, duplicate}
    end.

try_mark_seen(MessageId) ->
    Now = erlang:system_time(second),
    case ets:insert_new(?DEDUP_TABLE, {MessageId, Now}) of
        true  -> new;
        false ->
            %% Might be expired
            case ets:lookup(?DEDUP_TABLE, MessageId) of
                [{_, SeenAt}] when Now - SeenAt > ?DEDUP_WINDOW_SECONDS ->
                    ets:insert(?DEDUP_TABLE, {MessageId, Now}),
                    new;
                _ ->
                    duplicate
            end
    end.

unmark_seen(MessageId) ->
    ets:delete(?DEDUP_TABLE, MessageId).

init([]) ->
    ets:new(?DEDUP_TABLE, [named_table, public, {write_concurrency, true}]),
    schedule_cleanup(),
    {ok, #{}}.

handle_info(cleanup, State) ->
    Cutoff = erlang:system_time(second) - ?DEDUP_WINDOW_SECONDS,
    ets:select_delete(?DEDUP_TABLE, [{{'_', '$1'}, [{'<', '$1', Cutoff}], [true]}]),
    schedule_cleanup(),
    {noreply, State}.

schedule_cleanup() ->
    erlang:send_after(?CLEANUP_INTERVAL_MS, self(), cleanup).
```

---

## 5. Dead Letter Queue

```erlang
%% dlq.erl — dead letter queue for failed messages
-module(dlq).
-export([wrap_processor/3, send_to_dlq/3, reprocess/1, list_failed/0]).

-define(MAX_RETRIES, 3).
-define(RETRY_BACKOFF_MS, [1000, 5000, 30000]).

%% Wrap any message processor with retry + DLQ logic
wrap_processor(MessageId, Payload, ProcessFun) ->
    RetryCount = get_retry_count(MessageId),
    try
        ProcessFun(Payload),
        clear_retry_count(MessageId)
    catch
        E:R ->
            Attempt = RetryCount + 1,
            logger:warning("Message ~p failed (attempt ~p): ~p:~p",
                          [MessageId, Attempt, E, R]),
            case Attempt >= ?MAX_RETRIES of
                true ->
                    send_to_dlq(MessageId, Payload, {E, R});
                false ->
                    BackoffMs = lists:nth(min(Attempt, length(?RETRY_BACKOFF_MS)),
                                         ?RETRY_BACKOFF_MS),
                    set_retry_count(MessageId, Attempt),
                    schedule_retry(MessageId, Payload, ProcessFun, BackoffMs)
            end
    end.

send_to_dlq(MessageId, Payload, Error) ->
    ErrorStr = io_lib:format("~p", [Error]),
    db_pool:execute(
        "INSERT INTO dead_letter_queue
           (message_id, payload, error, failed_at, retry_count)
         VALUES ($1, $2, $3, NOW(), $4)
         ON CONFLICT (message_id)
         DO UPDATE SET error = $3, failed_at = NOW(), retry_count = $4",
        [MessageId, json:encode(Payload), list_to_binary(ErrorStr), ?MAX_RETRIES]).

reprocess(MessageId) ->
    case db_pool:query(
        "SELECT payload FROM dead_letter_queue WHERE message_id = $1",
        [MessageId]) of
        {ok, [{PayloadJson}]} ->
            Payload = json:decode(PayloadJson),
            db_pool:execute(
                "DELETE FROM dead_letter_queue WHERE message_id = $1",
                [MessageId]),
            {ok, Payload};  %% Caller should re-enqueue
        _ ->
            {error, not_found}
    end.

list_failed() ->
    db_pool:query(
        "SELECT message_id, error, failed_at, retry_count
         FROM dead_letter_queue
         ORDER BY failed_at DESC LIMIT 100", []).

schedule_retry(MessageId, Payload, ProcessFun, DelayMs) ->
    spawn(fun() ->
        timer:sleep(DelayMs),
        wrap_processor(MessageId, Payload, ProcessFun)
    end).

get_retry_count(_MessageId) -> 0.   %% simplified; use ETS in production
set_retry_count(_MessageId, _N) -> ok.
clear_retry_count(_MessageId) -> ok.
```

---

## 6. Priority Queue

```erlang
%% priority_queue.erl — multi-level priority queue for tasks
-module(priority_queue).
-behaviour(gen_server).

-export([start_link/0, enqueue/3, dequeue/0, size/0]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

%% Priority levels (lower number = higher priority)
-define(PRIORITIES, [critical, high, normal, low, background]).

-record(state, {
    queues :: #{atom() => queue:queue()}
}).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

enqueue(Priority, Message, _Opts) when is_atom(Priority) ->
    case lists:member(Priority, ?PRIORITIES) of
        true  -> gen_server:cast(?MODULE, {enqueue, Priority, Message});
        false -> {error, invalid_priority}
    end.

dequeue() ->
    gen_server:call(?MODULE, dequeue).

size() ->
    gen_server:call(?MODULE, size).

init([]) ->
    Queues = maps:from_list([{P, queue:new()} || P <- ?PRIORITIES]),
    {ok, #state{queues = Queues}}.

handle_cast({enqueue, Priority, Msg}, #state{queues = Qs} = State) ->
    Q     = maps:get(Priority, Qs),
    NewQ  = queue:in(Msg, Q),
    NewQs = maps:put(Priority, NewQ, Qs),
    {noreply, State#state{queues = NewQs}}.

handle_call(dequeue, _From, #state{queues = Qs} = State) ->
    {Reply, NewQs} = dequeue_by_priority(?PRIORITIES, Qs),
    {reply, Reply, State#state{queues = NewQs}};

handle_call(size, _From, #state{queues = Qs} = State) ->
    Total = maps:fold(fun(_, Q, Acc) -> Acc + queue:len(Q) end, 0, Qs),
    {reply, Total, State}.

dequeue_by_priority([], Qs) ->
    {empty, Qs};
dequeue_by_priority([Priority | Rest], Qs) ->
    Q = maps:get(Priority, Qs),
    case queue:out(Q) of
        {{value, Msg}, NewQ} ->
            {{ok, Priority, Msg}, maps:put(Priority, NewQ, Qs)};
        {empty, _} ->
            dequeue_by_priority(Rest, Qs)
    end.
```

---

## 7. แบบฝึกหัด

1. Implement exponential backoff with jitter สำหรับ DLQ retry
2. สร้าง queue monitor ที่ alert เมื่อ queue depth เกิน threshold
3. เพิ่ม message TTL — ทิ้งข้อความที่อยู่ใน queue นานเกินไป
4. Implement consumer group: หลาย consumers แบ่งงานกัน แต่ไม่ process ซ้ำกัน

---

## สรุป Part 87

✅ Delivery semantics: at-most-once, at-least-once, exactly-once  
✅ Outbox pattern: guarantee delivery via transactional write  
✅ Ordered processing: per-key serial, cross-key parallel  
✅ Deduplication: ETS-based seen window with expiry cleanup  
✅ Dead letter queue: retry with backoff, parking failed messages  
✅ Priority queue: multi-level with strict priority ordering  

---

*Part 87/100 | [← ก่อนหน้า](../part86/README.md) | [ถัดไป →](../part88/README.md)*
