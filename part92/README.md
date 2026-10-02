# Part 92: IoT Platform — Device Management at Scale

> **"A billion devices, each with a heartbeat — your job is to hear them all"**  
> อุปกรณ์หนึ่งพันล้านชิ้น แต่ละชิ้นมีชีพจร — งานของคุณคือได้ยินทุกชิ้น

---

## สารบัญ

1. [IoT Platform Architecture](#1-iot-platform-architecture)
2. [Device Registry](#2-device-registry)
3. [MQTT Protocol Handler](#3-mqtt-protocol-handler)
4. [Device Shadow (Digital Twin)](#4-device-shadow-digital-twin)
5. [Telemetry Ingestion Pipeline](#5-telemetry-ingestion-pipeline)
6. [OTA Firmware Updates](#6-ota-firmware-updates)
7. [แบบฝึกหัด](#7-แบบฝึกหัด)

---

## 1. IoT Platform Architecture

```
IoT Platform at Scale
═══════════════════════════════════════════════════════════

           Devices (millions)
               │ MQTT / MQTTS
               ▼
       ┌───────────────┐
       │  MQTT Broker  │  emqx / vernemq (cluster)
       │  (Erlang!)    │  
       └───────┬───────┘
               │ Erlang message passing
               ▼
   ┌─────────────────────────────┐
   │    IoT Core (this system)   │
   │                             │
   │  device_registry (ETS)      │  which devices exist
   │  device_shadow (per device) │  desired vs reported state
   │  command_dispatcher         │  send commands to devices
   │  telemetry_ingestor         │  receive sensor data
   │  ota_manager                │  firmware rollout
   └─────────────────────────────┘
               │
       ┌───────┴───────┐
       │               │
  PostgreSQL       Kafka → Analytics

Scale targets:
  1M concurrent connections per MQTT broker node
  10M messages per second ingested
  100k OTA updates in parallel
  99.99% uptime (4.4 minutes downtime per year)
```

---

## 2. Device Registry

```erlang
%% device_registry.erl — fast device lookup and management
-module(device_registry).
-behaviour(gen_server).

-export([start_link/0, register/2, deregister/1,
         get_device/1, list_by_group/1, update_status/2]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

-define(TABLE, device_registry).

-record(device, {
    id,
    type,          %% sensor | actuator | gateway
    group_id,
    firmware_version,
    status = offline :: online | offline,
    last_seen :: integer() | undefined,
    shadow_pid :: pid() | undefined,
    metadata = #{}
}).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

register(DeviceId, Config) ->
    gen_server:call(?MODULE, {register, DeviceId, Config}).

deregister(DeviceId) ->
    gen_server:cast(?MODULE, {deregister, DeviceId}).

get_device(DeviceId) ->
    case ets:lookup(?TABLE, DeviceId) of
        [Device] -> {ok, Device};
        []       -> {error, not_found}
    end.

list_by_group(GroupId) ->
    ets:select(?TABLE, [
        {#device{group_id = GroupId, _ = '_'}, [], ['$_']}
    ]).

update_status(DeviceId, Status) ->
    gen_server:cast(?MODULE, {status, DeviceId, Status}).

init([]) ->
    ets:new(?TABLE, [named_table, public,
                     {keypos, #device.id},
                     {read_concurrency, true}]),
    {ok, #{}}.

handle_call({register, DeviceId, Config}, _From, State) ->
    Device = #device{
        id               = DeviceId,
        type             = maps:get(type, Config, sensor),
        group_id         = maps:get(group_id, Config),
        firmware_version = maps:get(firmware_version, Config, <<"unknown">>),
        metadata         = maps:get(metadata, Config, #{})
    },
    ets:insert(?TABLE, Device),
    %% Start shadow process for this device
    {ok, ShadowPid} = device_shadow_sup:start_shadow(DeviceId),
    ets:update_element(?TABLE, DeviceId, {#device.shadow_pid, ShadowPid}),
    {reply, ok, State}.

handle_cast({deregister, DeviceId}, State) ->
    case ets:lookup(?TABLE, DeviceId) of
        [#device{shadow_pid = SPid}] when is_pid(SPid) ->
            device_shadow:stop(SPid);
        _ -> ok
    end,
    ets:delete(?TABLE, DeviceId),
    {noreply, State};

handle_cast({status, DeviceId, Status}, State) ->
    Now = erlang:system_time(second),
    ets:update_element(?TABLE, DeviceId, [
        {#device.status, Status},
        {#device.last_seen, Now}
    ]),
    {noreply, State}.

handle_info(_Info, State) -> {noreply, State}.
```

---

## 3. MQTT Protocol Handler

```erlang
%% mqtt_handler.erl — process MQTT messages from/to devices
-module(mqtt_handler).
-export([on_message_publish/2, on_client_connected/2,
         on_client_disconnected/3]).

%% Called by MQTT broker hook when device publishes a message
on_message_publish(Message, _Env) ->
    Topic   = maps:get(topic, Message),
    Payload = maps:get(payload, Message),
    case parse_topic(Topic) of
        {telemetry, DeviceId} ->
            handle_telemetry(DeviceId, Payload);
        {shadow_update, DeviceId} ->
            handle_shadow_update(DeviceId, Payload);
        {command_response, DeviceId, CommandId} ->
            handle_command_response(DeviceId, CommandId, Payload);
        unknown ->
            ok
    end,
    {ok, Message}.

on_client_connected(#{clientid := DeviceId}, _ConnInfo) ->
    logger:info("Device ~s connected", [DeviceId]),
    device_registry:update_status(DeviceId, online),
    %% Send pending commands
    command_queue:flush_pending(DeviceId).

on_client_disconnected(#{clientid := DeviceId}, Reason, _ConnInfo) ->
    logger:info("Device ~s disconnected: ~p", [DeviceId, Reason]),
    device_registry:update_status(DeviceId, offline).

handle_telemetry(DeviceId, Payload) ->
    case json:decode(Payload) of
        {ok, Data} ->
            telemetry_ingestor:ingest(DeviceId, Data);
        _ ->
            logger:warning("Invalid telemetry from ~s", [DeviceId])
    end.

handle_shadow_update(DeviceId, Payload) ->
    case json:decode(Payload) of
        {ok, #{<<"state">> := #{<<"reported">> := Reported}}} ->
            device_shadow:update_reported(DeviceId, Reported);
        _ ->
            ok
    end.

handle_command_response(DeviceId, CommandId, Payload) ->
    case json:decode(Payload) of
        {ok, #{<<"status">> := Status}} ->
            command_dispatcher:ack(DeviceId, CommandId, Status);
        _ ->
            ok
    end.

parse_topic(<<"devices/", Rest/binary>>) ->
    case binary:split(Rest, <<"/">>) of
        [DeviceId, <<"telemetry">>] -> {telemetry, DeviceId};
        [DeviceId, <<"shadow/update">>] -> {shadow_update, DeviceId};
        [DeviceId, <<"commands/", CommandId/binary>>] ->
            {command_response, DeviceId, CommandId};
        _ -> unknown
    end;
parse_topic(_) -> unknown.
```

---

## 4. Device Shadow (Digital Twin)

```erlang
%% device_shadow.erl — maintains desired vs reported state
-module(device_shadow).
-behaviour(gen_server).

-export([start_link/1, get/1, update_desired/2, update_reported/2,
         get_delta/1, stop/1]).
-export([init/1, handle_call/3, handle_cast/2]).

%% Shadow: the "digital twin" pattern
%% - desired: what we want the device to do
%% - reported: what device says it is doing
%% - delta: what's different (send to device to reconcile)

-record(shadow, {
    device_id,
    desired  = #{} :: map(),
    reported = #{} :: map(),
    version  = 0   :: integer()
}).

start_link(DeviceId) ->
    gen_server:start_link(?MODULE, DeviceId, []).

get(DeviceId) ->
    case device_registry:get_device(DeviceId) of
        {ok, #device{shadow_pid = Pid}} -> gen_server:call(Pid, get);
        Err -> Err
    end.

update_desired(DeviceId, NewDesired) ->
    with_shadow(DeviceId, fun(Pid) ->
        gen_server:call(Pid, {update_desired, NewDesired})
    end).

update_reported(DeviceId, NewReported) ->
    with_shadow(DeviceId, fun(Pid) ->
        gen_server:cast(Pid, {update_reported, NewReported})
    end).

get_delta(DeviceId) ->
    with_shadow(DeviceId, fun(Pid) ->
        gen_server:call(Pid, get_delta)
    end).

init(DeviceId) ->
    {ok, #shadow{device_id = DeviceId}}.

handle_call(get, _From, Shadow) ->
    {reply, {ok, shadow_to_map(Shadow)}, Shadow};

handle_call({update_desired, New}, _From,
            #shadow{desired = Old, version = V, device_id = DeviceId} = Shadow) ->
    Desired   = maps:merge(Old, New),
    NewShadow = Shadow#shadow{desired = Desired, version = V + 1},
    publish_delta_to_device(DeviceId, compute_delta(NewShadow)),
    {reply, {ok, V + 1}, NewShadow};

handle_call(get_delta, _From, Shadow) ->
    Delta = compute_delta(Shadow),
    {reply, {ok, Delta}, Shadow}.

handle_cast({update_reported, New}, #shadow{reported = Old, version = V} = Shadow) ->
    Reported  = maps:merge(Old, New),
    NewShadow = Shadow#shadow{reported = Reported, version = V + 1},
    {noreply, NewShadow}.

compute_delta(#shadow{desired = Desired, reported = Reported}) ->
    maps:fold(fun(Key, DesiredVal, Acc) ->
        case maps:get(Key, Reported, undefined) of
            DesiredVal -> Acc;                    %% in sync
            _          -> maps:put(Key, DesiredVal, Acc)
        end
    end, #{}, Desired).

publish_delta_to_device(DeviceId, Delta) when map_size(Delta) > 0 ->
    Topic   = <<"devices/", DeviceId/binary, "/shadow/delta">>,
    Payload = json:encode(#{state => #{desired => Delta}}),
    mqtt_broker:publish(Topic, Payload);
publish_delta_to_device(_, _) -> ok.

shadow_to_map(#shadow{desired = D, reported = R, version = V}) ->
    #{desired => D, reported => R, version => V, delta => compute_delta(#shadow{desired=D, reported=R})}.

with_shadow(DeviceId, Fun) ->
    case device_registry:get_device(DeviceId) of
        {ok, #device{shadow_pid = Pid}} when is_pid(Pid) -> Fun(Pid);
        {ok, _} -> {error, no_shadow};
        Err -> Err
    end.

-record(device, {id, type, group_id, firmware_version,
                 status, last_seen, shadow_pid, metadata}).
stop(Pid) -> gen_server:stop(Pid).
```

---

## 5. Telemetry Ingestion Pipeline

```erlang
%% telemetry_ingestor.erl — high-throughput sensor data ingestion
-module(telemetry_ingestor).
-behaviour(gen_server).

-export([start_link/0, ingest/2]).
-export([init/1, handle_cast/2, handle_info/2]).

-define(BATCH_SIZE, 1000).
-define(FLUSH_INTERVAL_MS, 500).

-record(state, {
    buffer = [] :: list(),
    count  = 0  :: integer()
}).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

ingest(DeviceId, Data) ->
    gen_server:cast(?MODULE, {ingest, DeviceId, Data}).

init([]) ->
    schedule_flush(),
    {ok, #state{}}.

handle_cast({ingest, DeviceId, Data},
            #state{buffer = Buf, count = N} = State) ->
    Record = #{
        device_id  => DeviceId,
        data       => Data,
        ingested_at => erlang:system_time(millisecond)
    },
    NewBuf   = [Record | Buf],
    NewCount = N + 1,
    case NewCount >= ?BATCH_SIZE of
        true  ->
            flush_batch(NewBuf),
            {noreply, #state{}};
        false ->
            {noreply, State#state{buffer = NewBuf, count = NewCount}}
    end.

handle_info(flush, #state{buffer = []} = State) ->
    schedule_flush(),
    {noreply, State};
handle_info(flush, #state{buffer = Buf} = State) ->
    flush_batch(Buf),
    schedule_flush(),
    {noreply, #state{}}.

flush_batch(Records) ->
    %% Write to time-series database or Kafka
    spawn(fun() ->
        case kafka_producer:publish_batch(<<"iot.telemetry">>, Records) of
            ok    -> ok;
            Error -> logger:error("Telemetry flush failed: ~p", [Error])
        end
    end).

schedule_flush() ->
    erlang:send_after(?FLUSH_INTERVAL_MS, self(), flush).
```

---

## 6. OTA Firmware Updates

```erlang
%% ota_manager.erl — over-the-air firmware rollout
-module(ota_manager).
-behaviour(gen_server).

-export([start_link/0, create_rollout/3, cancel_rollout/1, get_status/1]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

-define(BATCH_SIZE, 100).        %% devices per batch
-define(BATCH_INTERVAL_MS, 5000)  .%% delay between batches

-record(rollout, {
    id,
    firmware_url,
    target_group,
    progress = 0,
    total,
    status = pending :: pending | running | completed | cancelled,
    failures = []
}).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

create_rollout(FirmwareUrl, GroupId, Opts) ->
    gen_server:call(?MODULE, {create, FirmwareUrl, GroupId, Opts}).

cancel_rollout(RolloutId) ->
    gen_server:cast(?MODULE, {cancel, RolloutId}).

get_status(RolloutId) ->
    gen_server:call(?MODULE, {status, RolloutId}).

init([]) ->
    {ok, #{rollouts => #{}}}.

handle_call({create, Url, GroupId, _Opts}, _From, #{rollouts := Rs} = State) ->
    Devices = device_registry:list_by_group(GroupId),
    Id = crypto:strong_rand_bytes(8),
    Rollout = #rollout{
        id           = Id,
        firmware_url = Url,
        target_group = GroupId,
        total        = length(Devices),
        status       = running
    },
    schedule_next_batch(Id, Devices),
    {reply, {ok, Id}, State#{rollouts => maps:put(Id, Rollout, Rs)}};

handle_call({status, Id}, _From, #{rollouts := Rs} = State) ->
    case maps:get(Id, Rs, undefined) of
        undefined -> {reply, {error, not_found}, State};
        Rollout   -> {reply, {ok, rollout_to_map(Rollout)}, State}
    end.

handle_cast({cancel, Id}, #{rollouts := Rs} = State) ->
    case maps:get(Id, Rs, undefined) of
        undefined -> {noreply, State};
        Rollout ->
            NewRollout = Rollout#rollout{status = cancelled},
            {noreply, State#{rollouts => maps:put(Id, NewRollout, Rs)}}
    end;

handle_cast({device_ack, RolloutId, DeviceId, ok}, #{rollouts := Rs} = State) ->
    Rollout = maps:get(RolloutId, Rs),
    NewRollout = Rollout#rollout{progress = Rollout#rollout.progress + 1},
    {noreply, State#{rollouts => maps:put(RolloutId, NewRollout, Rs)}};

handle_cast({device_ack, RolloutId, DeviceId, Error}, #{rollouts := Rs} = State) ->
    Rollout = maps:get(RolloutId, Rs),
    NewFails = [{DeviceId, Error} | Rollout#rollout.failures],
    NewRollout = Rollout#rollout{
        progress = Rollout#rollout.progress + 1,
        failures = NewFails
    },
    {noreply, State#{rollouts => maps:put(RolloutId, NewRollout, Rs)}}.

handle_info({next_batch, RolloutId, Devices}, #{rollouts := Rs} = State) ->
    case maps:get(RolloutId, Rs, undefined) of
        #rollout{status = running, firmware_url = Url} ->
            {Batch, Remaining} = split_batch(Devices, ?BATCH_SIZE),
            send_ota_commands(Batch, Url, RolloutId),
            case Remaining of
                [] -> ok;
                _  -> schedule_next_batch(RolloutId, Remaining)
            end;
        _ -> ok
    end,
    {noreply, State}.

send_ota_commands(Devices, FirmwareUrl, RolloutId) ->
    lists:foreach(fun(Device) ->
        DeviceId = element(#device.id, Device),
        Command = #{type => ota_update, url => FirmwareUrl,
                    rollout_id => RolloutId},
        command_dispatcher:send(DeviceId, Command)
    end, Devices).

schedule_next_batch(RolloutId, Devices) ->
    erlang:send_after(?BATCH_INTERVAL_MS, self(), {next_batch, RolloutId, Devices}).

split_batch(List, N) ->
    {lists:sublist(List, N), lists:nthtail(min(N, length(List)), List)}.

rollout_to_map(#rollout{id=Id, status=S, progress=P, total=T, failures=F}) ->
    #{id => Id, status => S, progress => P, total => T,
      success_rate => P / max(1, T), failures => F}.

-record(device, {id, type, group_id, firmware_version,
                 status, last_seen, shadow_pid, metadata}).
```

---

## 7. แบบฝึกหัด

1. Implement device alerting: threshold-based alerts when sensor value exceeds limit
2. สร้าง device group hierarchy: site → building → floor → room → device
3. เพิ่ม binary protocol support นอกจาก JSON (เช่น CBOR หรือ msgpack)
4. Implement rollback: ถ้า OTA update fail rate > 10% ให้ pause rollout อัตโนมัติ

---

## สรุป Part 92

✅ IoT architecture: MQTT broker + shadow + registry + telemetry + OTA  
✅ Device registry: ETS-backed with shadow process per device  
✅ MQTT handler: broker hooks for publish/connect/disconnect events  
✅ Device shadow: desired vs reported state, delta computation and push  
✅ Telemetry ingestion: batched buffering, Kafka flush at 500ms or 1000 records  
✅ OTA firmware: batched rollout, per-device ack tracking, cancellation  

---

*Part 92/100 | [← ก่อนหน้า](../part91/README.md) | [ถัดไป →](../part93/README.md)*
