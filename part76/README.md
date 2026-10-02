# Part 76: Machine Learning Integration

> **"Erlang's role: serve models reliably, not train them"**  
> บทบาทของ Erlang: ให้บริการ models อย่างเชื่อถือได้ — ไม่ใช่ฝึก models

---

## สารบัญ

1. [ML Service Architecture](#1-ml-service-architecture)
2. [Python/Port Integration](#2-pythonport-integration)
3. [Feature Engineering Pipeline](#3-feature-engineering-pipeline)
4. [Model Registry](#4-model-registry)
5. [A/B Testing for Models](#5-ab-testing-for-models)
6. [Prediction Caching and Batching](#6-prediction-caching-and-batching)
7. [แบบฝึกหัด](#7-แบบฝึกหัด)

---

## 1. ML Service Architecture

```
Erlang + ML Integration Architecture
═══════════════════════════════════════════════════════════════

ERLANG (Orchestration + Serving)
┌──────────────────────────────────────────────────────────────┐
│  API Handler                                                  │
│       │                                                       │
│  Feature Engineering (ETS cache, Erlang NIF)                 │
│       │                                                       │
│  ML Service (gen_server pool → Python port / HTTP client)    │
│       │                                                       │
│  Prediction Cache (ETS with TTL)                             │
│       │                                                       │
│  A/B Router (feature flags + experiment tracking)            │
└───────────────────────────────┬──────────────────────────────┘
                                │
        ┌───────────────────────┼───────────────────────┐
        ▼                       ▼                       ▼
   Python Port              ONNX Runtime          External API
   (sklearn, torch)       (native C, NIF)        (OpenAI, etc.)
   
DECISION: Which integration to use?
  • Port: complex Python models, libraries unavailable in Erlang
  • NIF: performance-critical, ONNX models, small C++ inference
  • HTTP: managed services, GPU inference, models from model hubs
  • Pure Erlang: simple rules, decision trees, logistic regression
```

---

## 2. Python/Port Integration

```erlang
%% ml_port.erl — communicate with Python ML model via stdin/stdout
-module(ml_port).
-behaviour(gen_server).

-export([start_link/1, predict/2, predict_batch/2]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2,
         terminate/2]).

-record(state, {
    port      :: port(),
    pending   = #{} :: #{reference() => {pid(), term()}},
    model_id  :: binary()
}).

start_link(ModelId) ->
    gen_server:start_link({local, model_name(ModelId)}, ?MODULE,
                          [ModelId], []).

predict(ModelId, Features) ->
    gen_server:call(model_name(ModelId), {predict, Features}, 5000).

predict_batch(ModelId, FeaturesList) ->
    gen_server:call(model_name(ModelId), {predict_batch, FeaturesList}, 30000).

init([ModelId]) ->
    %% Start Python subprocess
    Cmd = iolist_to_binary([
        "python3 /opt/models/serve.py --model-id ", ModelId
    ]),
    Port = open_port({spawn, binary_to_list(Cmd)},
                     [binary, {packet, 4}, exit_status]),
    {ok, #state{port = Port, model_id = ModelId}}.

handle_call({predict, Features}, From, State) ->
    Ref = make_ref(),
    Msg = json:encode(#{ref => term_to_binary(Ref, [compressed]),
                        type => predict,
                        features => Features}),
    port_command(State#state.port, Msg),
    Pending = maps:put(Ref, From, State#state.pending),
    {noreply, State#state{pending = Pending}};

handle_call({predict_batch, FeaturesList}, From, State) ->
    Ref = make_ref(),
    Msg = json:encode(#{ref => term_to_binary(Ref, [compressed]),
                        type => predict_batch,
                        features => FeaturesList}),
    port_command(State#state.port, Msg),
    Pending = maps:put(Ref, From, State#state.pending),
    {noreply, State#state{pending = Pending}}.

handle_info({Port, {data, Data}}, #state{port = Port} = State) ->
    Response = json:decode(Data),
    RefBin = maps:get(<<"ref">>, Response),
    Ref     = binary_to_term(RefBin),
    case maps:get(Ref, State#state.pending, undefined) of
        undefined ->
            logger:warning("Unknown ref in ML response"),
            {noreply, State};
        From ->
            Prediction = maps:get(<<"prediction">>, Response),
            gen_server:reply(From, {ok, Prediction}),
            {noreply, State#state{pending = maps:remove(Ref, State#state.pending)}}
    end;

handle_info({Port, {exit_status, Code}}, #state{port = Port} = State) ->
    logger:error("ML port died", #{model => State#state.model_id, code => Code}),
    {stop, port_died, State}.

terminate(_Reason, #state{port = Port}) ->
    catch port_close(Port),
    ok.

handle_cast(_Msg, State) -> {noreply, State}.

model_name(ModelId) ->
    binary_to_atom(<<"ml_port_", ModelId/binary>>).
```

```python
# serve.py — Python side of the port
import sys
import json
import struct
import numpy as np
import pickle

def read_message():
    length_bytes = sys.stdin.buffer.read(4)
    if len(length_bytes) < 4:
        return None
    length = struct.unpack('>I', length_bytes)[0]
    return json.loads(sys.stdin.buffer.read(length))

def write_message(msg):
    data = json.dumps(msg).encode()
    sys.stdout.buffer.write(struct.pack('>I', len(data)))
    sys.stdout.buffer.write(data)
    sys.stdout.buffer.flush()

def main():
    import argparse
    parser = argparse.ArgumentParser()
    parser.add_argument('--model-id')
    args = parser.parse_args()

    # Load the model
    with open(f'/opt/models/{args.model_id}.pkl', 'rb') as f:
        model = pickle.load(f)

    while True:
        msg = read_message()
        if msg is None:
            break

        ref = msg['ref']
        features = np.array(msg['features']).reshape(1, -1)

        if msg['type'] == 'predict':
            prediction = model.predict(features)[0]
            proba = model.predict_proba(features)[0].tolist()
            write_message({'ref': ref, 'prediction': float(prediction),
                           'probability': proba})
        elif msg['type'] == 'predict_batch':
            batch = np.array(msg['features'])
            predictions = model.predict(batch).tolist()
            write_message({'ref': ref, 'prediction': predictions})

if __name__ == '__main__':
    main()
```

---

## 3. Feature Engineering Pipeline

```erlang
%% feature_pipeline.erl — compute features for ML predictions
-module(feature_pipeline).
-export([compute/2, register_feature/3]).

-define(FEATURE_CACHE, feature_cache).

compute(UserId, FeatureSet) ->
    %% Check cache first
    CacheKey = {UserId, FeatureSet},
    case ets:lookup(?FEATURE_CACHE, CacheKey) of
        [{_, Features, ExpireAt}] when ExpireAt > erlang:system_time(second) ->
            {ok, Features};
        _ ->
            %% Compute features
            Features = compute_features(UserId, FeatureSet),
            %% Cache for 60 seconds
            ets:insert(?FEATURE_CACHE,
                       {CacheKey, Features, erlang:system_time(second) + 60}),
            {ok, Features}
    end.

compute_features(UserId, FeatureSet) ->
    maps:from_list([
        {FeatureName, compute_one(UserId, FeatureName)}
        || FeatureName <- FeatureSet
    ]).

compute_one(UserId, purchase_count_30d) ->
    {ok, Count} = db:query(
        "SELECT COUNT(*) FROM orders WHERE user_id = $1
         AND created_at > NOW() - INTERVAL '30 days'",
        [UserId]),
    element(1, hd(Count));

compute_one(UserId, avg_order_value) ->
    {ok, Rows} = db:query(
        "SELECT AVG(total_amount) FROM orders WHERE user_id = $1",
        [UserId]),
    case Rows of
        [{null}] -> 0.0;
        [{Avg}]  -> Avg
    end;

compute_one(UserId, days_since_last_purchase) ->
    {ok, Rows} = db:query(
        "SELECT EXTRACT(DAY FROM NOW() - MAX(created_at)) FROM orders
         WHERE user_id = $1",
        [UserId]),
    case Rows of
        [{null}] -> 999;
        [{Days}] -> trunc(Days)
    end;

compute_one(_UserId, _Feature) -> 0.

register_feature(Name, Fun, CacheTtl) ->
    %% Register a new feature computation function at runtime
    ets:insert(feature_registry, {Name, Fun, CacheTtl}).
```

---

## 4. Model Registry

```erlang
%% model_registry.erl — manage multiple model versions
-module(model_registry).
-behaviour(gen_server).

-export([start_link/0, register_model/3, get_model/2,
         list_models/1, promote/2, retire/2]).
-export([init/1, handle_call/3, handle_cast/2]).

-record(model_info, {
    id         :: binary(),
    name       :: binary(),
    version    :: binary(),
    status     :: staging | production | retired,
    started_at :: integer(),
    metrics    :: map()
}).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

register_model(Name, Version, Opts) ->
    gen_server:call(?MODULE, {register, Name, Version, Opts}).

get_model(Name, Stage) ->
    gen_server:call(?MODULE, {get, Name, Stage}).

list_models(Name) ->
    gen_server:call(?MODULE, {list, Name}).

promote(ModelId, production) ->
    gen_server:call(?MODULE, {promote, ModelId, production}).

retire(ModelId) ->
    gen_server:call(?MODULE, {retire, ModelId}).

init([]) ->
    Tid = ets:new(model_registry, [named_table, public]),
    {ok, Tid}.

handle_call({register, Name, Version, Opts}, _From, Tid) ->
    Id = generate_model_id(Name, Version),
    Model = #model_info{
        id         = Id,
        name       = Name,
        version    = Version,
        status     = staging,
        started_at = erlang:system_time(second),
        metrics    = maps:get(metrics, Opts, #{})
    },
    ets:insert(Tid, {Id, Model}),
    {reply, {ok, Id}, Tid};

handle_call({get, Name, Stage}, _From, Tid) ->
    Matches = ets:match_object(Tid, {'_', #model_info{
        name = Name, status = Stage, _ = '_'
    }}),
    case Matches of
        [] -> {reply, {error, not_found}, Tid};
        [{_, Model} | _] -> {reply, {ok, Model}, Tid}
    end;

handle_call({promote, ModelId, production}, _From, Tid) ->
    case ets:lookup(Tid, ModelId) of
        [{_, Model}] ->
            %% Retire current production model
            retire_current_production(Tid, Model#model_info.name),
            %% Promote this one
            ets:insert(Tid, {ModelId, Model#model_info{status = production}}),
            {reply, ok, Tid};
        [] ->
            {reply, {error, not_found}, Tid}
    end;

handle_call({retire, ModelId}, _From, Tid) ->
    case ets:lookup(Tid, ModelId) of
        [{_, Model}] ->
            ets:insert(Tid, {ModelId, Model#model_info{status = retired}}),
            {reply, ok, Tid};
        [] ->
            {reply, {error, not_found}, Tid}
    end.

handle_cast(_Msg, Tid) -> {noreply, Tid}.

retire_current_production(Tid, Name) ->
    Matches = ets:match_object(Tid, {'_', #model_info{
        name = Name, status = production, _ = '_'
    }}),
    lists:foreach(fun({Id, Model}) ->
        ets:insert(Tid, {Id, Model#model_info{status = retired}})
    end, Matches).

generate_model_id(Name, Version) ->
    iolist_to_binary([Name, "_v", Version]).
```

---

## 5. A/B Testing for Models

```erlang
%% model_ab.erl — route predictions between model versions for A/B testing
-module(model_ab).
-export([predict/3, log_outcome/4]).

-define(AB_TABLE, model_ab_config).

predict(ExperimentName, UserId, Features) ->
    {ModelId, Variant} = assign_variant(ExperimentName, UserId),
    {ok, Prediction} = ml_port:predict(ModelId, Features),

    %% Log for analysis
    log_prediction(ExperimentName, UserId, Variant, ModelId, Prediction),

    {ok, Prediction, #{variant => Variant, model_id => ModelId}}.

assign_variant(ExperimentName, UserId) ->
    case ets:lookup(?AB_TABLE, ExperimentName) of
        [{_, Config}] ->
            Hash = erlang:phash2({UserId, ExperimentName}, 100),
            select_variant(Hash, maps:get(variants, Config));
        [] ->
            {maps:get(default_model, get_default_config()), control}
    end.

select_variant(Hash, Variants) ->
    select_variant(Hash, Variants, 0).

select_variant(_Hash, [{ModelId, _Pct, Variant} | _], _Acc) ->
    {ModelId, Variant};
select_variant(Hash, [{ModelId, Pct, Variant} | Rest], Acc) ->
    case Hash < Acc + Pct of
        true  -> {ModelId, Variant};
        false -> select_variant(Hash, Rest, Acc + Pct)
    end;
select_variant(_Hash, [], _Acc) ->
    {default_model, control}.

log_prediction(ExperimentName, UserId, Variant, ModelId, Prediction) ->
    telemetry:execute([ml, prediction], #{value => 1}, #{
        experiment => ExperimentName,
        user_id    => UserId,
        variant    => Variant,
        model_id   => ModelId,
        prediction => Prediction
    }).

log_outcome(ExperimentName, UserId, Variant, Outcome) ->
    telemetry:execute([ml, outcome], #{value => 1}, #{
        experiment => ExperimentName,
        user_id    => UserId,
        variant    => Variant,
        outcome    => Outcome
    }).

get_default_config() -> #{default_model => <<"v1">>}.
```

---

## 6. Prediction Caching and Batching

```erlang
%% prediction_cache.erl — cache predictions with TTL
-module(prediction_cache).
-behaviour(gen_server).

-export([start_link/0, get_or_predict/4, invalidate/2]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

-define(DEFAULT_TTL, 300).   % 5 minutes
-define(CLEANUP_INTERVAL, 60000).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

get_or_predict(ModelId, Key, Features, Opts) ->
    TTL = maps:get(ttl, Opts, ?DEFAULT_TTL),
    CacheKey = {ModelId, Key},
    Now = erlang:system_time(second),
    case ets:lookup(prediction_cache, CacheKey) of
        [{_, Prediction, ExpireAt}] when ExpireAt > Now ->
            {ok, Prediction, cached};
        _ ->
            {ok, Prediction} = ml_port:predict(ModelId, Features),
            ets:insert(prediction_cache,
                       {CacheKey, Prediction, Now + TTL}),
            {ok, Prediction, computed}
    end.

invalidate(ModelId, Key) ->
    ets:delete(prediction_cache, {ModelId, Key}).

init([]) ->
    ets:new(prediction_cache, [named_table, public, {read_concurrency, true}]),
    schedule_cleanup(),
    {ok, #{}}.

handle_info(cleanup, State) ->
    Now = erlang:system_time(second),
    ets:select_delete(prediction_cache,
        [{ {'_', '_', '$1'}, [{'<', '$1', Now}], [true] }]),
    schedule_cleanup(),
    {noreply, State}.

handle_call(_Req, _From, State) -> {reply, ok, State}.
handle_cast(_Msg, State) -> {noreply, State}.

schedule_cleanup() ->
    erlang:send_after(?CLEANUP_INTERVAL, self(), cleanup).
```

---

## 7. แบบฝึกหัด

1. เพิ่ม model warm-up: เมื่อ Port start ครั้งแรก ให้ส่ง dummy prediction
2. Implement circuit breaker สำหรับ Python port ที่ตอบสนองช้า
3. สร้าง feature importance tracker ที่บันทึกว่า feature ใดมีผลต่อ prediction
4. เพิ่ม shadow mode: รัน model ใหม่ใน background โดยไม่ส่งผลไปยัง user

---

## สรุป Part 76

✅ ML service architecture: port/NIF/HTTP tradeoffs  
✅ Python port integration: JSON over stdin/stdout, async refs  
✅ Feature engineering: cached computation pipeline  
✅ Model registry: versioning, staging→production promotion  
✅ A/B testing: hash-based stable variant assignment  
✅ Prediction caching: TTL-based with automatic cleanup  

---

*Part 76/100 | [← ก่อนหน้า](../part75/README.md) | [ถัดไป →](../part77/README.md)*
