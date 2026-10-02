# Part 43: Stream Processing

> **"Process data as it flows, not after it arrives"**  
> ประมวลผลข้อมูลขณะที่ไหลผ่าน ไม่ใช่หลังจากมาถึง

---

## สารบัญ

1. [Stream Concepts ใน Erlang](#1-stream-concepts-ใน-erlang)
2. [Producer-Consumer Pipeline](#2-producer-consumer-pipeline)
3. [Windowing](#3-windowing)
4. [Stream Aggregation](#4-stream-aggregation)
5. [Backpressure in Streams](#5-backpressure-in-streams)
6. [Kafka Integration](#6-kafka-integration)
7. [Real-Time Dashboard](#7-real-time-dashboard)
8. [แบบฝึกหัด](#8-แบบฝึกหัด)

---

## 1. Stream Concepts ใน Erlang

```erlang
%% Erlang lists เป็น lazy streams ด้วย list comprehension
%% แต่สำหรับ infinite / real streams ใช้ process pipeline

%% Lazy stream ด้วย fun
-module(stream).
-export([from_list/1, map/2, filter/2, take/2, to_list/1,
         range/2, unfold/2]).

%% Stream type: fun() -> {Value, NextStream} | done
from_list([])      -> fun() -> done end;
from_list([H | T]) -> fun() -> {H, from_list(T)} end.

range(From, To) when From > To -> fun() -> done end;
range(From, To) ->
    fun() -> {From, range(From+1, To)} end.

unfold(Seed, Fun) ->
    fun() ->
        case Fun(Seed) of
            {Value, NewSeed} -> {Value, unfold(NewSeed, Fun)};
            done             -> done
        end
    end.

map(Stream, F) ->
    fun() ->
        case Stream() of
            done       -> done;
            {V, Next}  -> {F(V), map(Next, F)}
        end
    end.

filter(Stream, Pred) ->
    fun() ->
        case Stream() of
            done -> done;
            {V, Next} ->
                case Pred(V) of
                    true  -> {V, filter(Next, Pred)};
                    false -> (filter(Next, Pred))()
                end
        end
    end.

take(_, 0) -> fun() -> done end;
take(Stream, N) ->
    fun() ->
        case Stream() of
            done      -> done;
            {V, Next} -> {V, take(Next, N-1)}
        end
    end.

to_list(Stream) ->
    case Stream() of
        done      -> [];
        {V, Next} -> [V | to_list(Next)]
    end.

%% Example: first 10 even squares from 1..∞
example() ->
    S1 = range(1, 1000000),
    S2 = filter(S1, fun(X) -> X rem 2 =:= 0 end),
    S3 = map(S2, fun(X) -> X * X end),
    S4 = take(S3, 10),
    to_list(S4).
```

---

## 2. Producer-Consumer Pipeline

```erlang
%% stream_pipeline.erl — process-based stream pipeline
-module(stream_pipeline).
-export([build/1, run/2]).

%% Pipeline = list of stage specs
%% Stage = {transform, Fun} | {filter, Pred} | {sink, Fun}
%%       | {batch, N}        | {parallel, N, Fun}

build(Stages) ->
    %% Build pipeline as chain of processes
    Stages.

run(Source, Stages) ->
    Self    = self(),
    Stages2 = Stages ++ [{sink, fun(X) -> Self ! {result, X} end}],
    Chain   = build_chain(Stages2),
    %% Feed source into first stage
    lists:foreach(fun(Item) ->
        hd(Chain) ! {item, Item}
    end, Source),
    hd(Chain) ! eos,
    collect_results().

build_chain([Stage]) ->
    [spawn_stage(Stage, terminal)];
build_chain([Stage | Rest]) ->
    DownChain = build_chain(Rest),
    [spawn_stage(Stage, hd(DownChain)) | DownChain].

spawn_stage({transform, Fun}, Next) ->
    spawn(fun() -> transform_loop(Fun, Next) end);
spawn_stage({filter, Pred}, Next) ->
    spawn(fun() -> filter_loop(Pred, Next) end);
spawn_stage({batch, N}, Next) ->
    spawn(fun() -> batch_loop(N, [], Next) end);
spawn_stage({sink, Fun}, _) ->
    spawn(fun() -> sink_loop(Fun) end).

transform_loop(Fun, Next) ->
    receive
        {item, X} -> Next ! {item, Fun(X)}, transform_loop(Fun, Next);
        eos       -> Next ! eos
    end.

filter_loop(Pred, Next) ->
    receive
        {item, X} ->
            case Pred(X) of
                true  -> Next ! {item, X};
                false -> ok
            end,
            filter_loop(Pred, Next);
        eos -> Next ! eos
    end.

batch_loop(N, Acc, Next) ->
    receive
        {item, X} ->
            NewAcc = [X | Acc],
            case length(NewAcc) >= N of
                true  ->
                    Next ! {item, lists:reverse(NewAcc)},
                    batch_loop(N, [], Next);
                false ->
                    batch_loop(N, NewAcc, Next)
            end;
        eos ->
            case Acc of
                [] -> ok;
                _  -> Next ! {item, lists:reverse(Acc)}
            end,
            Next ! eos
    end.

sink_loop(Fun) ->
    receive
        {item, X} -> Fun(X), sink_loop(Fun);
        eos       -> ok
    end.

collect_results() ->
    receive
        {result, X} -> [X | collect_results()]
    after 100 -> []
    end.
```

---

## 3. Windowing

```erlang
%% windowing.erl — tumbling and sliding windows
-module(windowing).
-behaviour(gen_server).
-export([start_link/2, push/2, get_window/0]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

-record(state, {
    type,        %% tumbling | sliding
    size_ms,
    events = [],
    window_start
}).

start_link(Type, SizeMs) ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, {Type, SizeMs}, []).

push(Timestamp, Value) ->
    gen_server:cast(?MODULE, {push, Timestamp, Value}).

get_window() ->
    gen_server:call(?MODULE, get_window).

init({Type, SizeMs}) ->
    erlang:send_after(SizeMs, self(), tick),
    {ok, #state{type=Type, size_ms=SizeMs,
                window_start=os:system_time(millisecond)}}.

handle_cast({push, Ts, Val}, #state{events=E} = S) ->
    {noreply, S#state{events=[{Ts, Val} | E]}}.

handle_info(tick, #state{type=tumbling, size_ms=SMs,
                          events=Events, window_start=WS} = S) ->
    Now = os:system_time(millisecond),
    emit_window(tumbling, Events, WS, Now),
    erlang:send_after(SMs, self(), tick),
    {noreply, S#state{events=[], window_start=Now}};

handle_info(tick, #state{type=sliding, size_ms=SMs, events=Events} = S) ->
    Now      = os:system_time(millisecond),
    Cutoff   = Now - SMs,
    Current  = [{Ts, V} || {Ts, V} <- Events, Ts >= Cutoff],
    emit_window(sliding, Current, Now - SMs, Now),
    erlang:send_after(SMs div 10, self(), tick),  %% slide every 10%
    {noreply, S#state{events=Current}};

handle_info(_, S) -> {noreply, S}.
handle_call(get_window, _, #state{events=E} = S) -> {reply, E, S}.

emit_window(Type, Events, From, To) ->
    Vals = [V || {_, V} <- Events],
    case Vals of
        [] -> ok;
        _ ->
            Stats = #{
                type  => Type,
                from  => From,
                to    => To,
                count => length(Vals),
                sum   => lists:sum(Vals),
                min   => lists:min(Vals),
                max   => lists:max(Vals),
                mean  => lists:sum(Vals) div length(Vals)
            },
            logger:info("Window: ~p", [Stats])
    end.
```

---

## 4. Stream Aggregation

```erlang
%% stream_agg.erl — real-time aggregation over event streams
-module(stream_agg).
-export([start/0, push_event/2, get_stats/1]).

-define(TABLE, stream_agg_table).

start() ->
    ets:new(?TABLE, [named_table, set, public,
                     {write_concurrency, true}]).

push_event(Key, Value) when is_number(Value) ->
    case ets:lookup(?TABLE, Key) of
        [] ->
            ets:insert(?TABLE, {Key, #{
                count => 1, sum => Value,
                min => Value, max => Value,
                last => Value
            }});
        [{Key, Stats}] ->
            ets:insert(?TABLE, {Key, Stats#{
                count => maps:get(count, Stats) + 1,
                sum   => maps:get(sum, Stats) + Value,
                min   => min(maps:get(min, Stats), Value),
                max   => max(maps:get(max, Stats), Value),
                last  => Value
            }})
    end.

get_stats(Key) ->
    case ets:lookup(?TABLE, Key) of
        [{Key, Stats}] ->
            Count = maps:get(count, Stats),
            Stats#{mean => maps:get(sum, Stats) div Count};
        [] ->
            not_found
    end.
```

---

## 5. Backpressure in Streams

```erlang
%% stream_bp.erl — backpressure with demand-driven pull
-module(stream_bp).
-behaviour(gen_server).
-export([start_producer/1, start_consumer/2, request/2]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

%% Producer: emits items on demand
start_producer(Source) ->
    gen_server:start({local, producer}, ?MODULE,
                     {producer, Source}, []).

%% Consumer: processes items, requests more when ready
start_consumer(Name, ProducerPid) ->
    gen_server:start({local, Name}, ?MODULE,
                     {consumer, ProducerPid}, []).

request(ProducerPid, N) ->
    gen_server:cast(ProducerPid, {demand, self(), N}).

init({producer, Source}) ->
    {ok, {producer, Source}};
init({consumer, Producer}) ->
    request(Producer, 10),  %% Initial demand
    {ok, {consumer, Producer, []}}.

handle_cast({demand, Consumer, N}, {producer, Source}) ->
    Items = take_items(Source, N),
    Consumer ! {items, Items},
    {noreply, {producer, Source}};

handle_cast(_, S) -> {noreply, S}.

handle_info({items, Items}, {consumer, Producer, Buffer}) ->
    %% Process items
    lists:foreach(fun process_item/1, Items),
    %% Request more only when buffer is low
    case length(Buffer) < 5 of
        true  -> request(Producer, 10);
        false -> ok
    end,
    {noreply, {consumer, Producer, Buffer}};

handle_info(_, S) -> {noreply, S}.
handle_call(_, _, S) -> {reply, ok, S}.

take_items(Source, N) -> lists:sublist(Source, N).
process_item(Item) -> logger:info("Processing: ~p", [Item]).
```

---

## 6. Kafka Integration

```erlang
%% kafka_consumer.erl — brod Kafka consumer
%% deps: {brod, "3.16.8"}
-module(kafka_consumer).
-behaviour(brod_group_subscriber).
-export([start_link/0, init/2, handle_message/4]).

start_link() ->
    KafkaHosts = [{<<"kafka">>, 9092}],
    ok = brod:start_client(KafkaHosts, kafka_client, []),
    GroupConfig = [
        {offset_commit_policy, commit_to_kafka_v2},
        {offset_commit_interval_seconds, 5}
    ],
    brod:start_link_group_subscriber(
        kafka_client,
        <<"my-consumer-group">>,
        [<<"events">>],
        GroupConfig,
        _ConsumerConfig = [],
        ?MODULE,
        _InitArgs = []
    ).

init(_GroupId, _State) ->
    {ok, #{}}.

handle_message(_Topic, Partition, Message, State) ->
    #kafka_message{
        offset = Offset,
        key    = Key,
        value  = Value
    } = Message,
    logger:info("Partition ~p offset ~p key=~p",
                [Partition, Offset, Key]),
    %% Process event
    Event = jsx:decode(Value, [return_maps]),
    handle_event(Event),
    {ok, ack, State}.

handle_event(#{<<"type">> := Type} = Event) ->
    logger:info("Event type=~s data=~p", [Type, Event]).
```

---

## 7. Real-Time Dashboard

```erlang
%% dashboard_ws_handler.erl — push live stats via WebSocket
-module(dashboard_ws_handler).
-behaviour(cowboy_websocket).
-export([init/2, websocket_init/1, websocket_handle/2,
         websocket_info/2, terminate/3]).

init(Req, State) ->
    {cowboy_websocket, Req, State}.

websocket_init(State) ->
    %% Subscribe to telemetry events
    telemetry:attach(dashboard_handler,
                     [myapp, metrics, snapshot],
                     fun(_, M, _, _) ->
                         self() ! {metrics, M}
                     end, #{}),
    %% Send periodic snapshots
    erlang:send_after(1000, self(), send_snapshot),
    {ok, State}.

websocket_handle(ping, State) ->
    {reply, pong, State};
websocket_handle({text, <<"subscribe:", Topic/binary>>}, State) ->
    {ok, maps:put(topics, [Topic | maps:get(topics, State, [])], State)};
websocket_handle(_, State) ->
    {ok, State}.

websocket_info(send_snapshot, State) ->
    erlang:send_after(1000, self(), send_snapshot),
    Snapshot = get_current_metrics(),
    Json = jsx:encode(#{type => <<"snapshot">>, data => Snapshot}),
    {reply, {text, Json}, State};

websocket_info({metrics, Meta}, State) ->
    Json = jsx:encode(#{type => <<"metric">>, data => Meta}),
    {reply, {text, Json}, State};

websocket_info(_, State) ->
    {ok, State}.

terminate(_Reason, _Req, _State) ->
    telemetry:detach(dashboard_handler),
    ok.

get_current_metrics() ->
    #{
        timestamp  => os:system_time(second),
        memory_mb  => erlang:memory(total) div (1024*1024),
        processes  => erlang:system_info(process_count),
        queue_depth => 0   %% job_queue:depth()
    }.
```

---

## 8. แบบฝึกหัด

1. สร้าง pipeline: read from file → parse CSV → filter rows → aggregate by key → write summary
2. Implement `flat_map` สำหรับ lazy stream: แต่ละ element ผลิต 0-N elements
3. ทดสอบ tumbling window ขนาด 5 วินาที ด้วยข้อมูล sensor readings
4. เขียน Kafka producer ที่ส่ง events พร้อม headers

---

## สรุป Part 43

✅ Lazy streams ด้วย fun  
✅ Process-based stream pipeline  
✅ Tumbling/sliding windows  
✅ Real-time aggregation  
✅ Backpressure ด้วย demand-driven pull  
✅ Kafka integration ด้วย brod  
✅ WebSocket live dashboard  

---

*Part 43/100 | [← ก่อนหน้า](../part42/README.md) | [ถัดไป →](../part44/README.md)*
