# Part 75: Real-Time Event Streaming

> **"Data in motion is more valuable than data at rest"**  
> ข้อมูลที่เคลื่อนไหวมีคุณค่ามากกว่าข้อมูลที่หยุดนิ่ง

---

## สารบัญ

1. [Event Streaming Architecture](#1-event-streaming-architecture)
2. [Kafka Producer](#2-kafka-producer)
3. [Kafka Consumer with Backpressure](#3-kafka-consumer-with-backpressure)
4. [Stream Processing Pipeline](#4-stream-processing-pipeline)
5. [In-Memory Event Bus](#5-in-memory-event-bus)
6. [Time-Series Aggregation](#6-time-series-aggregation)
7. [แบบฝึกหัด](#7-แบบฝึกหัด)

---

## 1. Event Streaming Architecture

```
Erlang Event Streaming Stack
════════════════════════════════════════════════════════

PRODUCERS               BROKERS           CONSUMERS
┌──────────┐            ┌──────────┐      ┌──────────────┐
│ API Svcs │──events──▶│ pg/Kafka │─────▶│ Projections  │
│ Domain   │            │          │      │ Analytics    │
│ Services │            │ Topics:  │      │ Notifications│
│ Sensors  │            │ orders   │      │ ML Pipeline  │
└──────────┘            │ payments │      └──────────────┘
                        │ metrics  │
                        └──────────┘

Patterns:
  • Fan-out: one event → many consumers (pub/sub)
  • Work queue: one event → one consumer (load balanced)
  • Event sourcing replay: replay entire topic
  • Stream join: correlate events from different topics
```

---

## 2. Kafka Producer

```erlang
%% kafka_producer.erl — reliable Kafka producer with local buffer
-module(kafka_producer).
-behaviour(gen_server).

-export([start_link/0, produce/3, produce_batch/2]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

-define(FLUSH_INTERVAL, 100).   % ms
-define(BATCH_SIZE, 500).
-define(LINGER_MS, 10).         % wait for more messages

-record(state, {
    client      :: pid() | undefined,
    buffer      = #{} :: #{Topic :: binary() => list()},
    flush_timer :: reference()
}).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

produce(Topic, Key, Value) ->
    gen_server:cast(?MODULE, {produce, Topic, Key, Value}).

produce_batch(Topic, Messages) ->
    gen_server:cast(?MODULE, {produce_batch, Topic, Messages}).

init([]) ->
    {ok, Client} = brod:start_client(kafka_brokers(), client1, []),
    TRef = schedule_flush(),
    {ok, #state{client = Client, flush_timer = TRef}}.

handle_cast({produce, Topic, Key, Value}, State) ->
    TopicBuffer = maps:get(Topic, State#state.buffer, []),
    Msg = #{key => Key, value => Value, ts => timestamp()},
    NewBuffer = maps:put(Topic, [Msg | TopicBuffer], State#state.buffer),
    NewState = State#state{buffer = NewBuffer},
    case total_buffered(NewBuffer) >= ?BATCH_SIZE of
        true  -> do_flush(NewState);
        false -> {noreply, NewState}
    end;

handle_cast({produce_batch, Topic, Messages}, State) ->
    TopicBuffer = maps:get(Topic, State#state.buffer, []),
    NewBuffer = maps:put(Topic, Messages ++ TopicBuffer, State#state.buffer),
    {noreply, State#state{buffer = NewBuffer}}.

handle_info(flush, State) ->
    {noreply, NewState} = do_flush(State),
    TRef = schedule_flush(),
    {noreply, NewState#state{flush_timer = TRef}}.

handle_call(_Req, _From, State) -> {reply, ok, State}.

do_flush(#state{buffer = Buffer} = State) when map_size(Buffer) =:= 0 ->
    {noreply, State};
do_flush(State) ->
    maps:foreach(fun(Topic, Messages) ->
        Msgs = [{maps:get(key, M), maps:get(value, M)}
                || M <- lists:reverse(Messages)],
        case brod:produce_sync(State#state.client, Topic, 0, <<>>, Msgs) of
            ok -> ok;
            {error, Reason} ->
                logger:error("Kafka produce failed",
                             #{topic => Topic, reason => Reason})
        end
    end, State#state.buffer),
    {noreply, State#state{buffer = #{}}}.

total_buffered(Buffer) ->
    lists:sum([length(Msgs) || Msgs <- maps:values(Buffer)]).

schedule_flush() ->
    erlang:send_after(?FLUSH_INTERVAL, self(), flush).

kafka_brokers() ->
    [{"kafka1.internal", 9092}, {"kafka2.internal", 9092}].

timestamp() -> erlang:system_time(millisecond).
```

---

## 3. Kafka Consumer with Backpressure

```erlang
%% kafka_consumer.erl — consumer group with backpressure
-module(kafka_consumer).
-behaviour(gen_server).
-behaviour(brod_group_subscriber).

-export([start_link/2]).
-export([init/2, handle_message/4]).

-record(state, {
    topic     :: binary(),
    handler   :: module(),
    in_flight = 0   :: integer(),
    max_in_flight   :: integer()
}).

-define(MAX_IN_FLIGHT, 100).

start_link(Topic, Handler) ->
    ConsumerConfig = [
        {begin_offset, latest},
        {max_bytes, 1048576},      % 1MB per fetch
        {max_wait_time, 500}
    ],
    GroupConfig = [
        {offset_commit_policy, commit_to_kafka_v2},
        {offset_commit_interval_seconds, 5}
    ],
    brod:start_link_group_subscriber(
        client1,
        <<"myapp_consumer_group">>,
        [Topic],
        GroupConfig,
        ConsumerConfig,
        ?MODULE,
        [Topic, Handler]
    ).

init(_GroupId, [Topic, Handler]) ->
    {ok, #state{topic = Topic, handler = Handler,
                max_in_flight = ?MAX_IN_FLIGHT}}.

handle_message(Topic, Partition, Message, State) ->
    #kafka_message{offset = Offset, key = Key, value = Value} = Message,

    %% Backpressure: wait if too many messages in flight
    NewState = wait_for_capacity(State),

    %% Process asynchronously
    Self = self(),
    spawn_link(fun() ->
        Result = (NewState#state.handler):process(Topic, Key, Value),
        gen_server:cast(Self, {done, Offset, Partition, Result})
    end),

    InFlight = NewState#state.in_flight + 1,
    {ok, ack, NewState#state{in_flight = InFlight}}.

wait_for_capacity(#state{in_flight = N, max_in_flight = Max} = State)
  when N >= Max ->
    receive
        {done, _, _, _} -> State#state{in_flight = N - 1}
    after 5000 ->
        logger:warning("Consumer backpressure: ~p in-flight", [N]),
        State
    end;
wait_for_capacity(State) -> State.
```

---

## 4. Stream Processing Pipeline

```erlang
%% stream_pipeline.erl — composable stream processing
-module(stream_pipeline).
-export([new/0, map/2, filter/2, flat_map/2, batch/2,
         window/2, sink/2, run/2]).

%% Build a pipeline as a list of stage functions
new() -> [].

map(Pipeline, Fun) ->
    [fun(Events) -> lists:map(Fun, Events) end | Pipeline].

filter(Pipeline, Pred) ->
    [fun(Events) -> lists:filter(Pred, Events) end | Pipeline].

flat_map(Pipeline, Fun) ->
    [fun(Events) -> lists:flatmap(Fun, Events) end | Pipeline].

batch(Pipeline, Size) ->
    [fun(Events) -> batch_events(Events, Size, []) end | Pipeline].

window(Pipeline, WindowMs) ->
    [fun(Events) -> window_events(Events, WindowMs) end | Pipeline].

sink(Pipeline, SinkFun) ->
    [fun(Events) -> SinkFun(Events), Events end | Pipeline].

run(Pipeline, Events) ->
    Stages = lists:reverse(Pipeline),
    lists:foldl(fun(Stage, Evs) -> Stage(Evs) end, Events, Stages).

batch_events([], _Size, Acc) ->
    [lists:reverse(Acc)];
batch_events(Events, Size, _Acc) when length(Events) =< Size ->
    [Events];
batch_events(Events, Size, Acc) ->
    {Batch, Rest} = lists:split(Size, Events),
    [Batch | batch_events(Rest, Size, Acc)].

window_events(Events, WindowMs) ->
    Now = erlang:system_time(millisecond),
    Cutoff = Now - WindowMs,
    lists:filter(fun(E) -> maps:get(timestamp, E, Now) >= Cutoff end, Events).
```

```erlang
%% Example: Order processing pipeline
order_pipeline() ->
    stream_pipeline:new()
    |> stream_pipeline:filter(fun(E) -> maps:get(type, E) =:= order_placed end)
    |> stream_pipeline:map(fun enrich_with_customer/1)
    |> stream_pipeline:filter(fun(E) -> maps:get(amount, E) > 0 end)
    |> stream_pipeline:batch(50)
    |> stream_pipeline:sink(fun(Batch) ->
        analytics_service:record_orders(Batch)
       end).

%% Run the pipeline
process_events(Events) ->
    Pipeline = order_pipeline(),
    stream_pipeline:run(Pipeline, Events).

enrich_with_customer(Event) ->
    UserId = maps:get(user_id, Event),
    case customer_cache:get(UserId) of
        {ok, Customer} -> maps:merge(Event, #{customer => Customer});
        _ -> Event
    end.
```

---

## 5. In-Memory Event Bus

```erlang
%% event_bus.erl — fast in-process pub/sub using pg
-module(event_bus).
-export([publish/2, subscribe/2, unsubscribe/2]).

-define(GROUP_PREFIX, <<"event_bus.">>).

publish(Topic, Event) ->
    Group = topic_group(Topic),
    Members = pg:get_members(Group),
    lists:foreach(fun(Pid) ->
        Pid ! {event, Topic, Event}
    end, Members),
    ok.

subscribe(Topic, Pid) ->
    Group = topic_group(Topic),
    pg:join(Group, Pid).

unsubscribe(Topic, Pid) ->
    Group = topic_group(Topic),
    pg:leave(Group, Pid).

topic_group(Topic) when is_atom(Topic) ->
    topic_group(atom_to_binary(Topic));
topic_group(Topic) when is_binary(Topic) ->
    binary_to_atom(<<?GROUP_PREFIX/binary, Topic/binary>>).
```

```erlang
%% event_bus_subscriber.erl — helper behaviour for subscribers
-module(event_bus_subscriber).
-export([start_link/2]).

-callback handle_event(Topic :: atom(), Event :: map()) -> ok.

start_link(Module, Topics) ->
    Pid = spawn_link(fun() -> init_subscriber(Module, Topics) end),
    {ok, Pid}.

init_subscriber(Module, Topics) ->
    Pid = self(),
    lists:foreach(fun(T) -> event_bus:subscribe(T, Pid) end, Topics),
    subscriber_loop(Module).

subscriber_loop(Module) ->
    receive
        {event, Topic, Event} ->
            try Module:handle_event(Topic, Event)
            catch E:R -> logger:error("Handler error", #{module => Module,
                                                         error => E, reason => R})
            end,
            subscriber_loop(Module);
        stop -> ok
    end.
```

---

## 6. Time-Series Aggregation

```erlang
%% time_series.erl — sliding window aggregations
-module(time_series).
-behaviour(gen_server).

-export([start_link/0, record/3, query/3]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

-define(RESOLUTION_SECONDS, 60).    % 1-minute buckets
-define(RETENTION_HOURS, 24).

-record(state, {
    series :: ets:tid()
}).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

record(Metric, Value, Tags) ->
    gen_server:cast(?MODULE, {record, Metric, Value, Tags}).

%% Query: aggregate between From and To (unix timestamps)
query(Metric, From, To) ->
    gen_server:call(?MODULE, {query, Metric, From, To}).

init([]) ->
    Tid = ets:new(time_series, [named_table, public,
                                {write_concurrency, true}]),
    erlang:send_after(300000, self(), cleanup),
    {ok, #state{series = Tid}}.

handle_cast({record, Metric, Value, Tags}, State) ->
    Bucket = current_bucket(),
    Key    = {Metric, Bucket, Tags},
    ets:update_counter(State#state.series, Key, [{2, 1}, {3, Value}],
                       {Key, 0, 0, Value, Value}),
    %% Also update min/max
    update_min_max(State#state.series, Key, Value),
    {noreply, State}.

handle_call({query, Metric, From, To}, _From, State) ->
    FromBucket = From div ?RESOLUTION_SECONDS,
    ToBucket   = To   div ?RESOLUTION_SECONDS,
    Pattern    = [{ {{Metric, '$1', '_'}, '$2', '$3', '_', '_'},
                    [{'>=', '$1', FromBucket}, {'=<', '$1', ToBucket}],
                    [{{bucket, '$1', count, '$2', sum, '$3'}}] }],
    Results = ets:select(State#state.series, Pattern),
    Aggregated = aggregate_buckets(Results),
    {reply, Aggregated, State}.

handle_info(cleanup, State) ->
    Cutoff = current_bucket() - (?RETENTION_HOURS * 3600 div ?RESOLUTION_SECONDS),
    ets:select_delete(State#state.series,
        [{ {{'_', '$1', '_'}, '_', '_', '_', '_'},
           [{'<', '$1', Cutoff}], [true] }]),
    erlang:send_after(300000, self(), cleanup),
    {noreply, State}.

current_bucket() ->
    erlang:system_time(second) div ?RESOLUTION_SECONDS.

update_min_max(Tid, Key, Value) ->
    case ets:lookup(Tid, Key) of
        [] -> ok;
        [{Key, Count, Sum, Min, Max}] ->
            ets:insert(Tid, {Key, Count, Sum, min(Min, Value), max(Max, Value)})
    end.

aggregate_buckets(Rows) ->
    TotalCount = lists:sum([C || {bucket, _, count, C, sum, _} <- Rows]),
    TotalSum   = lists:sum([S || {bucket, _, count, _, sum, S} <- Rows]),
    Avg = case TotalCount of
        0 -> 0;
        N -> TotalSum / N
    end,
    #{count => TotalCount, sum => TotalSum, avg => Avg}.
```

---

## 7. แบบฝึกหัด

1. เพิ่ม dead letter queue สำหรับ Kafka messages ที่ process ไม่ผ่าน
2. Implement event replay: load events จาก beginning ของ topic
3. สร้าง stream join: correlate events จาก 2 topics ภายใน window
4. เพิ่ม compression ใน `kafka_producer.erl` (snappy/gzip)

---

## สรุป Part 75

✅ Event streaming architecture: producers → brokers → consumers  
✅ Kafka producer: batched, buffered, with flush interval  
✅ Kafka consumer: backpressure via in-flight counter  
✅ Stream pipeline: composable map/filter/batch/window/sink  
✅ In-memory event bus: pg-based pub/sub for local processes  
✅ Time-series aggregation: minute buckets with sliding window queries  

---

*Part 75/100 | [← ก่อนหน้า](../part74/README.md) | [ถัดไป →](../part76/README.md)*
