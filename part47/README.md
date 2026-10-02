# Part 47: IoT Platform with Erlang

> **"Millions of devices, one reliable platform — Erlang was born for this"**  
> อุปกรณ์นับล้าน, platform เดียวที่เชื่อถือได้ — Erlang ถูกสร้างมาสำหรับงานนี้

---

## สารบัญ

1. [IoT Architecture](#1-iot-architecture)
2. [MQTT Server](#2-mqtt-server)
3. [Device Registry](#3-device-registry)
4. [Telemetry Ingestion](#4-telemetry-ingestion)
5. [Rule Engine](#5-rule-engine)
6. [Device Commands](#6-device-commands)
7. [Time-Series Storage](#7-time-series-storage)
8. [แบบฝึกหัด](#8-แบบฝึกหัด)

---

## 1. IoT Architecture

```
IoT Platform Architecture:

  [IoT Devices]
      | MQTT / CoAP / HTTP
      ↓
  [Protocol Gateway]
      | normalized
      ↓
  [Device Registry] → [Auth Service]
      |
  [Telemetry Ingester] → [Time-Series DB]
      |
  [Rule Engine] → [Alert Service] → [Notifications]
      |
  [Command Dispatcher] → [Devices]

Scale: 1M+ concurrent device connections on Erlang nodes
```

---

## 2. MQTT Server

```erlang
%% mqtt_handler.erl — MQTT over TCP using ranch
-module(mqtt_handler).
-behaviour(gen_server).
-export([start_link/4]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

-define(CONNECT,    1).
-define(CONNACK,    2).
-define(PUBLISH,    3).
-define(SUBSCRIBE,  8).
-define(SUBACK,     9).
-define(PINGREQ,   12).
-define(PINGRESP,  13).
-define(DISCONNECT,14).

-record(state, {
    socket,
    transport,
    client_id,
    username,
    buffer = <<>>,
    keepalive = 60
}).

start_link(Ref, Socket, Transport, Opts) ->
    {ok, proc_lib:spawn_link(?MODULE, init,
                              [{Ref, Socket, Transport, Opts}])}.

init({Ref, Socket, Transport, _Opts}) ->
    ok = ranch:accept_ack(Ref),
    Transport:setopts(Socket, [{active, once}, binary]),
    gen_server:enter_loop(?MODULE, [], #state{socket=Socket, transport=Transport}).

handle_info({tcp, Sock, Data}, #state{buffer=Buf} = S) ->
    S#state.transport:setopts(Sock, [{active, once}]),
    Buffer2 = <<Buf/binary, Data/binary>>,
    {Packets, Rest} = decode_mqtt_packets(Buffer2),
    NewState = lists:foldl(fun handle_packet/2, S#state{buffer=Rest}, Packets),
    {noreply, NewState};

handle_info({tcp_closed, _}, S) -> {stop, normal, S};
handle_info({tcp_error, _, _}, S) -> {stop, normal, S}.

handle_packet({connect, #{client_id:=CId, username:=U, password:=P}}, S) ->
    case device_auth:authenticate(U, P) of
        {ok, DeviceId} ->
            device_registry:register(CId, self()),
            send_connack(S#state.socket, S#state.transport, 0),
            S#state{client_id=CId, username=U};
        {error, _} ->
            send_connack(S#state.socket, S#state.transport, 5),
            S
    end;

handle_packet({publish, #{topic:=T, payload:=Payload, qos:=QoS}},
              #state{client_id=CId} = S) ->
    telemetry_ingester:ingest(CId, T, Payload),
    rule_engine:process(CId, T, Payload),
    S;

handle_packet({subscribe, #{topics:=Topics, packet_id:=PktId}}, S) ->
    [subscription_manager:subscribe(S#state.client_id, Topic) || {Topic, _QoS} <- Topics],
    send_suback(S#state.socket, S#state.transport, PktId, Topics),
    S;

handle_packet(pingreq, S) ->
    send_pingresp(S#state.socket, S#state.transport),
    S;

handle_packet(disconnect, S) ->
    device_registry:unregister(S#state.client_id),
    {stop, normal, S};

handle_packet(_, S) -> S.

send_connack(Sock, Transport, ReturnCode) ->
    Transport:send(Sock, <<16#20:8, 2:8, 0:8, ReturnCode:8>>).

send_pingresp(Sock, Transport) ->
    Transport:send(Sock, <<16#D0:8, 0:8>>).

send_suback(Sock, Transport, PacketId, Topics) ->
    GrantedQoS = [0 || _ <- Topics],
    Payload    = list_to_binary(GrantedQoS),
    Transport:send(Sock, <<16#90:8, (2 + length(Topics)):8,
                            PacketId:16/big, Payload/binary>>).

decode_mqtt_packets(Buffer) ->
    {[], Buffer}.   %% simplified; real impl would parse properly

handle_call(_, _, S) -> {reply, ok, S}.
handle_cast(_, S)    -> {noreply, S}.
```

---

## 3. Device Registry

```erlang
%% device_registry.erl — manage connected device processes
-module(device_registry).
-behaviour(gen_server).
-export([start_link/0, register/2, unregister/1,
         get_pid/1, list/0, count/0]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

-define(TABLE, device_registry_table).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

register(ClientId, Pid) ->
    gen_server:call(?MODULE, {register, ClientId, Pid}).

unregister(ClientId) ->
    gen_server:cast(?MODULE, {unregister, ClientId}).

get_pid(ClientId) ->
    case ets:lookup(?TABLE, ClientId) of
        [{ClientId, Pid, _}] when is_process_alive(Pid) -> {ok, Pid};
        _ -> {error, not_connected}
    end.

list() ->
    [{CId, Pid} || {CId, Pid, _} <- ets:tab2list(?TABLE),
                   is_process_alive(Pid)].

count() -> length(list()).

init([]) ->
    ets:new(?TABLE, [named_table, set, protected]),
    {ok, #{}}.

handle_call({register, CId, Pid}, _From, State) ->
    %% Monitor the device process
    Ref = erlang:monitor(process, Pid),
    ets:insert(?TABLE, {CId, Pid, Ref}),
    metrics:gauge(connected_devices, count(), #{}),
    {reply, ok, maps:put(Ref, CId, State)}.

handle_info({'DOWN', Ref, process, _, _}, State) ->
    case maps:find(Ref, State) of
        {ok, CId} ->
            ets:delete(?TABLE, CId),
            metrics:gauge(connected_devices, count(), #{});
        error -> ok
    end,
    {noreply, maps:remove(Ref, State)}.

handle_cast({unregister, CId}, State) ->
    ets:delete(?TABLE, CId),
    {noreply, State}.
```

---

## 4. Telemetry Ingestion

```erlang
%% telemetry_ingester.erl — high-throughput telemetry ingestion
-module(telemetry_ingester).
-behaviour(gen_server).
-export([start_link/0, ingest/3]).
-export([init/1, handle_cast/2, handle_info/2, handle_call/3]).

-define(BATCH_SIZE, 500).
-define(FLUSH_INTERVAL, 1000).

-record(state, {
    buffer = [],
    count  = 0
}).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

ingest(DeviceId, Topic, Payload) ->
    gen_server:cast(?MODULE, {ingest, DeviceId, Topic, Payload}).

init([]) ->
    erlang:send_after(?FLUSH_INTERVAL, self(), flush),
    {ok, #state{}}.

handle_cast({ingest, DeviceId, Topic, Payload},
            #state{buffer=Buf, count=N} = S) ->
    Point = #{
        device_id  => DeviceId,
        topic      => Topic,
        payload    => Payload,
        ingested_at => os:system_time(millisecond)
    },
    NewState = S#state{buffer=[Point | Buf], count=N+1},
    case NewState#state.count >= ?BATCH_SIZE of
        true  -> flush_buffer(NewState);
        false -> {noreply, NewState}
    end.

handle_info(flush, S) ->
    erlang:send_after(?FLUSH_INTERVAL, self(), flush),
    flush_buffer(S).

flush_buffer(#state{buffer=[]} = S) ->
    {noreply, S};
flush_buffer(#state{buffer=Buf} = S) ->
    spawn(fun() -> write_batch(lists:reverse(Buf)) end),
    {noreply, S#state{buffer=[], count=0}}.

write_batch(Points) ->
    %% Write to time-series DB (TimescaleDB or custom ETS)
    timeseries:bulk_insert(Points).

handle_call(_, _, S) -> {reply, ok, S}.
```

---

## 5. Rule Engine

```erlang
%% rule_engine.erl — process telemetry against rules
-module(rule_engine).
-export([process/3, add_rule/1, remove_rule/1, list_rules/0]).

-define(RULES_TABLE, iot_rules).

%% Rule: #{id, device_id, topic, condition, action}
%% Condition: fun(Payload) -> bool()
%% Action: {alert, Msg} | {command, DeviceId, Cmd} | {webhook, Url}

add_rule(Rule = #{id := Id}) ->
    ets:insert(?RULES_TABLE, {Id, Rule}).

remove_rule(Id) ->
    ets:delete(?RULES_TABLE, Id).

list_rules() ->
    [Rule || {_, Rule} <- ets:tab2list(?RULES_TABLE)].

process(DeviceId, Topic, Payload) ->
    Data = parse_payload(Payload),
    Rules = get_matching_rules(DeviceId, Topic),
    lists:foreach(fun(Rule) ->
        case evaluate_condition(maps:get(condition, Rule), Data) of
            true  -> execute_action(maps:get(action, Rule), DeviceId, Data);
            false -> ok
        end
    end, Rules).

get_matching_rules(DeviceId, Topic) ->
    All = [R || {_, R} <- ets:tab2list(?RULES_TABLE)],
    [R || R <- All,
          matches(maps:get(device_id, R, <<"*">>), DeviceId),
          matches_topic(maps:get(topic, R, <<"#">>), Topic)].

matches(<<"*">>, _)    -> true;
matches(P, S) -> P =:= S.

matches_topic(<<"#">>, _)    -> true;
matches_topic(Rule, Topic) ->
    RuleParts  = binary:split(Rule, <<"/">>, [global]),
    TopicParts = binary:split(Topic, <<"/">>, [global]),
    match_parts(RuleParts, TopicParts).

match_parts([], [])              -> true;
match_parts([<<"#">>|_], _)      -> true;
match_parts([<<"+">>|R], [_|T])  -> match_parts(R, T);
match_parts([P|R], [P|T])        -> match_parts(R, T);
match_parts(_, _)                -> false.

evaluate_condition(Cond, Data) when is_function(Cond, 1) ->
    try Cond(Data) catch _:_ -> false end;
evaluate_condition({gt, Field, Threshold}, Data) ->
    maps:get(Field, Data, 0) > Threshold;
evaluate_condition({lt, Field, Threshold}, Data) ->
    maps:get(Field, Data, 0) < Threshold;
evaluate_condition({eq, Field, Value}, Data) ->
    maps:get(Field, Data) =:= Value.

execute_action({alert, Msg}, DeviceId, _Data) ->
    alert_service:send(#{device => DeviceId, message => Msg});
execute_action({command, TargetDevice, Cmd}, _DeviceId, _Data) ->
    device_commander:send(TargetDevice, Cmd);
execute_action({webhook, Url}, DeviceId, Data) ->
    Body = jsx:encode(#{device => DeviceId, data => Data}),
    hackney:request(post, Url, [{<<"content-type">>, <<"application/json">>}], Body, []).

parse_payload(Payload) when is_binary(Payload) ->
    try jsx:decode(Payload, [return_maps])
    catch _:_ -> #{raw => Payload}
    end.
```

---

## 6. Device Commands

```erlang
%% device_commander.erl — send commands to connected devices
-module(device_commander).
-export([send/2, send_and_wait/3]).

send(DeviceId, Command) ->
    case device_registry:get_pid(DeviceId) of
        {ok, Pid} ->
            Payload = jsx:encode(Command),
            Topic = <<"cmd/", DeviceId/binary>>,
            Pid ! {publish, Topic, Payload},
            ok;
        {error, not_connected} ->
            %% Queue for when device reconnects
            queued_command_store:enqueue(DeviceId, Command)
    end.

send_and_wait(DeviceId, Command, TimeoutMs) ->
    Ref = make_ref(),
    CmdWithCorr = Command#{correlation_id => term_to_binary(Ref)},
    send(DeviceId, CmdWithCorr),
    receive
        {command_response, Ref, Response} -> {ok, Response}
    after TimeoutMs ->
        {error, timeout}
    end.
```

---

## 7. Time-Series Storage

```erlang
%% timeseries.erl — ETS-based time-series for IoT data
-module(timeseries).
-export([start/0, bulk_insert/1, query_range/3, aggregate/4]).

-define(TS_TABLE, iot_timeseries).
-define(MAX_AGE, 3600 * 24 * 7).  %% keep 7 days

start() ->
    ets:new(?TS_TABLE, [named_table, ordered_set, public,
                        {write_concurrency, true}]),
    erlang:send_after(3600_000, self(), cleanup).

bulk_insert(Points) ->
    [ets:insert(?TS_TABLE,
                {{maps:get(device_id, P),
                  maps:get(ingested_at, P)},
                 maps:get(payload, P)})
     || P <- Points].

query_range(DeviceId, FromMs, ToMs) ->
    ets:select(?TS_TABLE, [
        {{{DeviceId, '$1'}, '$2'},
         [{'>=', '$1', FromMs}, {'=<', '$1', ToMs}],
         [{{DeviceId, '$1', '$2'}}]}
    ]).

aggregate(DeviceId, FromMs, ToMs, {field, Field}) ->
    Points = query_range(DeviceId, FromMs, ToMs),
    Values = [maps:get(Field, jsx:decode(P, [return_maps]), 0)
              || {_, _, P} <- Points,
                 is_map(catch jsx:decode(P, [return_maps]))],
    case Values of
        [] -> #{count => 0};
        _  -> #{
            count => length(Values),
            sum   => lists:sum(Values),
            min   => lists:min(Values),
            max   => lists:max(Values),
            mean  => lists:sum(Values) div length(Values)
        }
    end.
```

---

## 8. แบบฝึกหัด

1. เพิ่ม over-the-air firmware update: stream binary ไปยัง device ผ่าน MQTT
2. สร้าง geo-fencing rule: alert เมื่อ GPS coordinates ออกนอก boundary
3. Implement device shadow: เก็บ last-known state สำหรับ offline devices
4. เพิ่ม fleet management: bulk command to all devices matching a tag

---

## สรุป Part 47

✅ MQTT server ด้วย ranch  
✅ Device registry ด้วย process monitoring  
✅ High-throughput telemetry ingestion  
✅ Rule engine ด้วย topic matching  
✅ Device command dispatch  
✅ Time-series storage  

---

*Part 47/100 | [← ก่อนหน้า](../part46/README.md) | [ถัดไป →](../part48/README.md)*
