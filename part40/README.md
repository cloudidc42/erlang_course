# Part 40: Telemetry and Observability

> **"You can't improve what you can't measure"**  
> สิ่งที่วัดไม่ได้ ปรับปรุงไม่ได้

---

## สารบัญ

1. [Telemetry Library](#1-telemetry-library)
2. [Metrics Collection](#2-metrics-collection)
3. [Prometheus Export](#3-prometheus-export)
4. [Distributed Tracing](#4-distributed-tracing)
5. [Structured Logging](#5-structured-logging)
6. [Health Checks](#6-health-checks)
7. [Alerting](#7-alerting)
8. [แบบฝึกหัด](#8-แบบฝึกหัด)

---

## 1. Telemetry Library

```erlang
%% telemetry_setup.erl — wiring the :telemetry library
-module(telemetry_setup).
-export([attach_all/0]).

attach_all() ->
    %% HTTP request metrics
    telemetry:attach(
        <<"http-request-handler">>,
        [myapp, cowboy, request, stop],
        fun handle_http_request/4,
        #{}
    ),
    %% DB query metrics
    telemetry:attach(
        <<"db-query-handler">>,
        [myapp, db, query, stop],
        fun handle_db_query/4,
        #{}
    ),
    %% Job metrics
    telemetry:attach_many(
        <<"job-handler">>,
        [
            [myapp, job, start],
            [myapp, job, stop],
            [myapp, job, exception]
        ],
        fun handle_job/4,
        #{}
    ).

handle_http_request([myapp, cowboy, request, stop], Measurements, Meta, _Config) ->
    Duration = maps:get(duration, Measurements),
    Status   = maps:get(status, Meta, 0),
    Path     = maps:get(path, Meta, <<"unknown">>),
    Method   = maps:get(method, Meta, <<"UNKNOWN">>),
    metrics:histogram(http_request_duration_ms,
                      Duration div 1000,
                      #{path => Path, method => Method, status => Status}).

handle_db_query([myapp, db, query, stop], #{duration := D}, #{query := Q}, _) ->
    metrics:histogram(db_query_duration_ms, D div 1000, #{query_type => classify(Q)}).

handle_job([myapp, job, start], _, #{type := T}, _) ->
    metrics:counter_inc(jobs_started_total, #{type => T});
handle_job([myapp, job, stop], #{duration := D}, #{type := T}, _) ->
    metrics:histogram(job_duration_ms, D div 1000, #{type => T}),
    metrics:counter_inc(jobs_completed_total, #{type => T});
handle_job([myapp, job, exception], _, #{type := T, kind := K}, _) ->
    metrics:counter_inc(jobs_failed_total, #{type => T, kind => K}).

classify(Q) when is_binary(Q) ->
    Upper = string:uppercase(binary_to_list(Q)),
    case lists:prefix("SELECT", Upper) of true -> select; _ ->
    case lists:prefix("INSERT", Upper) of true -> insert; _ ->
    case lists:prefix("UPDATE", Upper) of true -> update; _ ->
    case lists:prefix("DELETE", Upper) of true -> delete; _ -> other
    end end end end.
```

---

## 2. Metrics Collection

```erlang
%% metrics.erl — in-memory metrics using ETS counters + histograms
-module(metrics).
-behaviour(gen_server).
-export([start_link/0, counter_inc/2, histogram/3, gauge/3, get_all/0]).
-export([init/1, handle_call/3, handle_cast/2]).

-define(COUNTERS, metrics_counters).
-define(HISTOGRAMS, metrics_histograms).
-define(GAUGES, metrics_gauges).

start_link() -> gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

counter_inc(Name, Labels) ->
    Key = {Name, Labels},
    try ets:update_counter(?COUNTERS, Key, 1)
    catch error:badarg ->
        ets:insert(?COUNTERS, {Key, 0}),
        ets:update_counter(?COUNTERS, Key, 1)
    end.

histogram(Name, Value, Labels) when is_number(Value) ->
    Key = {Name, Labels},
    gen_server:cast(?MODULE, {histogram, Key, Value}).

gauge(Name, Value, Labels) ->
    ets:insert(?GAUGES, {{Name, Labels}, Value}).

get_all() ->
    Counters   = ets:tab2list(?COUNTERS),
    Histograms = gen_server:call(?MODULE, get_histograms),
    Gauges     = ets:tab2list(?GAUGES),
    #{counters => Counters, histograms => Histograms, gauges => Gauges}.

init([]) ->
    ets:new(?COUNTERS,   [named_table, set, public, {write_concurrency, true}]),
    ets:new(?GAUGES,     [named_table, set, public, {write_concurrency, true}]),
    {ok, #{}}.

handle_cast({histogram, Key, Value}, State) ->
    Existing = maps:get(Key, State, []),
    {noreply, maps:put(Key, [Value | Existing], State)}.

handle_call(get_histograms, _From, State) ->
    Summary = maps:map(fun(_K, Values) ->
        Sorted = lists:sort(Values),
        N = length(Sorted),
        #{
            count  => N,
            sum    => lists:sum(Sorted),
            min    => hd(Sorted),
            max    => lists:last(Sorted),
            mean   => lists:sum(Sorted) div N,
            p50    => percentile(Sorted, 50),
            p95    => percentile(Sorted, 95),
            p99    => percentile(Sorted, 99)
        }
    end, State),
    {reply, Summary, State}.

percentile(Sorted, P) ->
    Idx = max(1, round(length(Sorted) * P / 100)),
    lists:nth(Idx, Sorted).
```

---

## 3. Prometheus Export

```erlang
%% prometheus_handler.erl — Cowboy handler exposing /metrics
-module(prometheus_handler).
-behaviour(cowboy_handler).
-export([init/2]).

init(Req, State) ->
    AllMetrics = metrics:get_all(),
    Body = format_prometheus(AllMetrics),
    Req2 = cowboy_req:reply(200,
        #{<<"content-type">> => <<"text/plain; version=0.0.4">>},
        Body, Req),
    {ok, Req2, State}.

format_prometheus(#{counters := Cs, histograms := Hs, gauges := Gs}) ->
    Parts = [
        format_counters(Cs),
        format_histograms(Hs),
        format_gauges(Gs)
    ],
    iolist_to_binary(Parts).

format_counters(Counters) ->
    [format_metric_line(Name, Labels, Value)
     || {{Name, Labels}, Value} <- Counters].

format_gauges(Gauges) ->
    [format_metric_line(Name, Labels, Value)
     || {{Name, Labels}, Value} <- Gauges].

format_histograms(Histograms) ->
    maps:fold(fun({Name, Labels}, Stats, Acc) ->
        NameStr = atom_to_list(Name),
        LabelStr = format_labels(Labels),
        [
            io_lib:format("~s_count~s ~p~n",
                         [NameStr, LabelStr, maps:get(count, Stats)]),
            io_lib:format("~s_sum~s ~p~n",
                         [NameStr, LabelStr, maps:get(sum, Stats)]),
            io_lib:format("~s_min~s ~p~n",
                         [NameStr, LabelStr, maps:get(min, Stats)]),
            io_lib:format("~s_p95~s ~p~n",
                         [NameStr, LabelStr, maps:get(p95, Stats)]),
            io_lib:format("~s_p99~s ~p~n",
                         [NameStr, LabelStr, maps:get(p99, Stats)])
            | Acc
        ]
    end, [], Histograms).

format_metric_line(Name, Labels, Value) ->
    LabelStr = format_labels(Labels),
    io_lib:format("~s~s ~p~n", [atom_to_list(Name), LabelStr, Value]).

format_labels(Labels) when map_size(Labels) =:= 0 -> "";
format_labels(Labels) ->
    Pairs = maps:fold(fun(K, V, Acc) ->
        [io_lib:format("~s=\"~s\"", [K, V]) | Acc]
    end, [], Labels),
    ["{", lists:join(",", Pairs), "}"].
```

---

## 4. Distributed Tracing

```erlang
%% trace.erl — lightweight trace context
-module(trace).
-export([start_span/1, finish_span/1, current_trace_id/0,
         with_span/2, inject_headers/1, extract_headers/1]).

-define(TRACE_KEY, '$trace_ctx').

start_span(Name) ->
    TraceId = case get(?TRACE_KEY) of
        undefined -> generate_id();
        #{trace_id := Id} -> Id
    end,
    SpanId = generate_id(),
    Ctx = #{trace_id => TraceId, span_id => SpanId, name => Name,
            start_ms => erlang:monotonic_time(millisecond)},
    put(?TRACE_KEY, Ctx),
    Ctx.

finish_span(#{start_ms := T0, name := Name, trace_id := TId, span_id := SId}) ->
    Duration = erlang:monotonic_time(millisecond) - T0,
    logger:info("TRACE trace_id=~s span_id=~s name=~s duration_ms=~p",
                [TId, SId, Name, Duration]);
finish_span(_) -> ok.

current_trace_id() ->
    case get(?TRACE_KEY) of
        #{trace_id := Id} -> {ok, Id};
        _ -> {error, no_trace}
    end.

with_span(Name, Fun) ->
    Span = start_span(Name),
    try
        Result = Fun(),
        finish_span(Span),
        Result
    catch
        Class:Reason:Stack ->
            finish_span(Span#{error => {Class, Reason}}),
            erlang:raise(Class, Reason, Stack)
    end.

inject_headers(Headers) ->
    case get(?TRACE_KEY) of
        #{trace_id := TId, span_id := SId} ->
            Headers#{
                <<"x-trace-id">> => TId,
                <<"x-span-id">>  => SId
            };
        _ -> Headers
    end.

extract_headers(Headers) ->
    case maps:find(<<"x-trace-id">>, Headers) of
        {ok, TraceId} ->
            SpanId = maps:get(<<"x-span-id">>, Headers, generate_id()),
            put(?TRACE_KEY, #{trace_id => TraceId, span_id => SpanId});
        error -> ok
    end.

generate_id() ->
    <<I:128>> = crypto:strong_rand_bytes(16),
    list_to_binary(io_lib:format("~32.16.0b", [I])).
```

---

## 5. Structured Logging

```erlang
%% log.erl — structured JSON logging wrapper
-module(log).
-export([info/2, warning/2, error/2, debug/2]).

info(Msg, Meta)    -> log(info, Msg, Meta).
warning(Msg, Meta) -> log(warning, Msg, Meta).
error(Msg, Meta)   -> log(error, Msg, Meta).
debug(Msg, Meta)   -> log(debug, Msg, Meta).

log(Level, Msg, Meta) ->
    TraceInfo = case trace:current_trace_id() of
        {ok, Id} -> #{trace_id => Id};
        _        -> #{}
    end,
    FullMeta = maps:merge(TraceInfo, Meta),
    logger:log(Level, Msg, FullMeta).

%% sys.config — configure logger to emit JSON
%% {logger, [
%%   {handler, default, logger_std_h, #{
%%     formatter => {logger_formatter, #{
%%       template => [msg, "\n"],
%%       single_line => true
%%     }}
%%   }}
%% ]}
%%
%% Or use a JSON formatter library
```

---

## 6. Health Checks

```erlang
%% health_handler.erl — /health endpoint
-module(health_handler).
-behaviour(cowboy_handler).
-export([init/2]).

init(Req, State) ->
    Checks = run_checks(),
    Status = case lists:all(fun({_, ok}) -> true; (_) -> false end, Checks) of
        true  -> 200;
        false -> 503
    end,
    Body = jsx:encode(#{
        status => if Status =:= 200 -> <<"ok">>; true -> <<"degraded">> end,
        checks => maps:from_list(Checks),
        timestamp => os:system_time(second)
    }),
    Req2 = cowboy_req:reply(Status,
        #{<<"content-type">> => <<"application/json">>},
        Body, Req),
    {ok, Req2, State}.

run_checks() ->
    [
        {db,    check_db()},
        {cache, check_cache()},
        {disk,  check_disk()}
    ].

check_db() ->
    case db:query("SELECT 1", []) of
        {ok, _} -> ok;
        _       -> {error, db_unreachable}
    end.

check_cache() ->
    case ets:info(cache_table) of
        undefined -> {error, cache_down};
        _         -> ok
    end.

check_disk() ->
    {ok, [{total, _}, {free, Free} | _]} = disksup:get_disk_data(),
    case Free > 100_000 of
        true  -> ok;
        false -> {error, low_disk_space}
    end.
```

---

## 7. Alerting

```erlang
%% alerting.erl — threshold-based alerting
-module(alerting).
-behaviour(gen_server).
-export([start_link/0, check/0]).
-export([init/1, handle_info/2, handle_call/3, handle_cast/2]).

-record(state, {last_alert = #{}}).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

init([]) ->
    erlang:send_after(30_000, self(), check),
    {ok, #state{}}.

handle_info(check, State) ->
    erlang:send_after(30_000, self(), check),
    NewState = run_checks(State),
    {noreply, NewState}.

handle_call(_, _, S) -> {reply, ok, S}.
handle_cast(_, S)    -> {noreply, S}.

run_checks(#state{last_alert = LastAlert} = State) ->
    Rules = [
        {error_rate,    fun check_error_rate/0,    5.0,  ">"},
        {p99_latency,   fun check_p99_latency/0,   1000, ">"},
        {queue_depth,   fun check_queue_depth/0,   1000, ">"},
        {memory_mb,     fun check_memory_mb/0,     3000, ">"}
    ],
    NewAlerts = lists:foldl(fun({Name, Check, Threshold, Op}, Acc) ->
        Value = Check(),
        Firing = case Op of
            ">" -> Value > Threshold;
            "<" -> Value < Threshold
        end,
        WasAlerted = maps:get(Name, LastAlert, false),
        case {Firing, WasAlerted} of
            {true, false} ->
                send_alert(Name, Value, Threshold),
                maps:put(Name, true, Acc);
            {false, true} ->
                send_recovery(Name, Value),
                maps:put(Name, false, Acc);
            _ ->
                Acc
        end
    end, LastAlert, Rules),
    State#state{last_alert = NewAlerts}.

send_alert(Name, Value, Threshold) ->
    logger:error("ALERT: ~p firing — value=~p threshold=~p",
                 [Name, Value, Threshold]).

send_recovery(Name, Value) ->
    logger:info("RECOVERY: ~p resolved — value=~p", [Name, Value]).

check_error_rate() -> 0.0.   %% TODO: compute from metrics
check_p99_latency() -> 0.    %% TODO: read from metrics histogram
check_queue_depth() -> 0.    %% TODO: job_queue:depth()
check_memory_mb() ->
    erlang:memory(total) div (1024 * 1024).
```

---

## 8. แบบฝึกหัด

1. เพิ่ม telemetry events สำหรับ WebSocket connections (connect/disconnect/message)
2. Export custom business metric: `active_users_total` gauge
3. สร้าง alert rule: เมื่อ DB query latency P95 > 500ms
4. เพิ่ม trace_id ลงใน error log ทุก line โดยอัตโนมัติ
5. สร้าง Grafana dashboard config สำหรับ metrics ที่ export

---

## สรุป Part 40

✅ telemetry library events  
✅ In-memory metrics (counter/histogram/gauge)  
✅ Prometheus text format export  
✅ Distributed tracing with span context  
✅ Structured JSON logging  
✅ Health check endpoint  
✅ Threshold-based alerting  

---

*Part 40/100 | [← ก่อนหน้า](../part39/README.md) | [ถัดไป →](../part41/README.md)*
