# Part 71: Production Monitoring and Observability

> **"You cannot improve what you cannot measure"**  
> คุณไม่สามารถปรับปรุงสิ่งที่วัดไม่ได้

---

## สารบัญ

1. [Telemetry Pipeline](#1-telemetry-pipeline)
2. [Custom Metrics Exporter](#2-custom-metrics-exporter)
3. [Distributed Tracing](#3-distributed-tracing)
4. [Health Check Framework](#4-health-check-framework)
5. [Alerting and Anomaly Detection](#5-alerting-and-anomaly-detection)
6. [SLO/SLA Tracking](#6-slaslo-tracking)
7. [แบบฝึกหัด](#7-แบบฝึกหัด)

---

## 1. Telemetry Pipeline

```erlang
%% telemetry_pipeline.erl
%% Collects telemetry events and routes to multiple backends
-module(telemetry_pipeline).
-behaviour(gen_server).

-export([start_link/0, attach_handlers/0]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

-define(FLUSH_INTERVAL, 5000).
-define(BATCH_SIZE, 100).

-record(state, {
    buffer  = [] :: list(),
    backends    :: list(),
    flush_timer :: reference() | undefined
}).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

attach_handlers() ->
    %% Attach to all application telemetry events
    Events = [
        [myapp, request, stop],
        [myapp, db, query, stop],
        [myapp, cache, lookup, stop],
        [myapp, job, execute, stop],
        [myapp, circuit_breaker, state_change]
    ],
    telemetry:attach_many(
        telemetry_pipeline,
        Events,
        fun handle_event/4,
        []
    ).

handle_event(Event, Measurements, Metadata, _Config) ->
    gen_server:cast(?MODULE, {event, Event, Measurements, Metadata}).

init([]) ->
    Backends = [
        {prometheus, prometheus_backend},
        {statsd,     statsd_backend},
        {logger,     log_backend}
    ],
    TRef = schedule_flush(),
    {ok, #state{backends = Backends, flush_timer = TRef}}.

handle_cast({event, Event, Measurements, Metadata}, State) ->
    Entry = #{
        event        => Event,
        measurements => Measurements,
        metadata     => Metadata,
        timestamp    => erlang:system_time(millisecond)
    },
    Buffer = [Entry | State#state.buffer],
    case length(Buffer) >= ?BATCH_SIZE of
        true ->
            flush_buffer(Buffer, State#state.backends),
            {noreply, State#state{buffer = []}};
        false ->
            {noreply, State#state{buffer = Buffer}}
    end.

handle_info(flush, State) ->
    flush_buffer(State#state.buffer, State#state.backends),
    TRef = schedule_flush(),
    {noreply, State#state{buffer = [], flush_timer = TRef}}.

handle_call(_Req, _From, State) -> {reply, ok, State}.

flush_buffer([], _Backends) -> ok;
flush_buffer(Buffer, Backends) ->
    [Backend:send_batch(lists:reverse(Buffer)) || {_Name, Backend} <- Backends],
    ok.

schedule_flush() ->
    erlang:send_after(?FLUSH_INTERVAL, self(), flush).
```

---

## 2. Custom Metrics Exporter

```erlang
%% prometheus_backend.erl
-module(prometheus_backend).
-export([send_batch/1, format_metrics/0]).

send_batch(Events) ->
    lists:foreach(fun record_event/1, Events).

record_event(#{event := [myapp, request, stop],
               measurements := #{duration := Duration},
               metadata := #{status_code := Code, path := Path}}) ->
    %% Update histogram and counter
    prometheus_histogram:observe(
        http_request_duration_ms,
        [Path, integer_to_list(Code)],
        Duration / 1000
    ),
    prometheus_counter:inc(
        http_requests_total,
        [Path, integer_to_list(Code)]
    );

record_event(#{event := [myapp, db, query, stop],
               measurements := #{duration := Duration},
               metadata := #{query_type := Type}}) ->
    prometheus_histogram:observe(
        db_query_duration_ms,
        [atom_to_list(Type)],
        Duration / 1000
    );

record_event(#{event := [myapp, circuit_breaker, state_change],
               metadata := #{name := Name, state := State}}) ->
    prometheus_gauge:set(
        circuit_breaker_state,
        [atom_to_list(Name)],
        case State of open -> 1; _ -> 0 end
    );

record_event(_) -> ok.

%% Format all metrics in Prometheus text format
format_metrics() ->
    iolist_to_binary([
        format_counter(http_requests_total),
        format_histogram(http_request_duration_ms),
        format_histogram(db_query_duration_ms),
        format_gauge(circuit_breaker_state)
    ]).

format_counter(Name) ->
    %% In real code use prometheus.erl library
    io_lib:format("# TYPE ~p counter\n", [Name]).

format_histogram(Name) ->
    io_lib:format("# TYPE ~p histogram\n", [Name]).

format_gauge(Name) ->
    io_lib:format("# TYPE ~p gauge\n", [Name]).
```

```erlang
%% metrics_handler.erl — /metrics HTTP endpoint for Prometheus scraping
-module(metrics_handler).
-behaviour(cowboy_handler).
-export([init/2]).

init(Req, State) ->
    Body = prometheus_backend:format_metrics(),
    Resp = cowboy_req:reply(200,
        #{<<"content-type">> => <<"text/plain; version=0.0.4">>},
        Body,
        Req),
    {ok, Resp, State}.
```

---

## 3. Distributed Tracing

```erlang
%% tracer.erl — lightweight distributed tracing
-module(tracer).
-export([start_span/2, finish_span/1, add_tag/3, propagate/1, extract/1]).

-record(span, {
    trace_id  :: binary(),
    span_id   :: binary(),
    parent_id :: binary() | undefined,
    operation :: atom(),
    start_us  :: integer(),
    tags = #{} :: map(),
    logs = []  :: list()
}).

start_span(Operation, ParentSpan) ->
    TraceId = case ParentSpan of
        undefined -> generate_id();
        #span{trace_id = T} -> T
    end,
    ParentId = case ParentSpan of
        undefined -> undefined;
        #span{span_id = S} -> S
    end,
    #span{
        trace_id  = TraceId,
        span_id   = generate_id(),
        parent_id = ParentId,
        operation = Operation,
        start_us  = erlang:monotonic_time(microsecond)
    }.

finish_span(Span) ->
    EndUs    = erlang:monotonic_time(microsecond),
    Duration = EndUs - Span#span.start_us,
    report_span(Span#span{tags = maps:put(duration_us, Duration, Span#span.tags)}).

add_tag(Span, Key, Value) ->
    Span#span{tags = maps:put(Key, Value, Span#span.tags)}.

%% Propagate trace context via HTTP headers (W3C Trace Context)
propagate(#span{trace_id = T, span_id = S}) ->
    TraceParent = iolist_to_binary(
        io_lib:format("00-~s-~s-01", [T, S])
    ),
    #{<<"traceparent">> => TraceParent}.

extract(Headers) ->
    case maps:get(<<"traceparent">>, Headers, undefined) of
        undefined -> undefined;
        TP ->
            case binary:split(TP, <<"-">>, [global]) of
                [_Ver, TraceId, SpanId, _Flags] ->
                    #span{
                        trace_id  = TraceId,
                        span_id   = SpanId,
                        operation = incoming
                    };
                _ -> undefined
            end
    end.

report_span(Span) ->
    %% Send to Jaeger/Zipkin collector
    logger:debug("SPAN", #{
        trace_id  => Span#span.trace_id,
        span_id   => Span#span.span_id,
        parent_id => Span#span.parent_id,
        operation => Span#span.operation,
        tags      => Span#span.tags
    }).

generate_id() ->
    <<A:64>> = crypto:strong_rand_bytes(8),
    integer_to_binary(A, 16).
```

```erlang
%% Middleware that creates a span per request
traced_handler_middleware(Req, State) ->
    ParentSpan = tracer:extract(cowboy_req:headers(Req)),
    Span = tracer:start_span(http_request, ParentSpan),
    Span1 = tracer:add_tag(Span, path, cowboy_req:path(Req)),
    Span1 = tracer:add_tag(Span1, method, cowboy_req:method(Req)),
    %% Store span in process dictionary for downstream use
    put(current_span, Span1),
    %% Execute handler
    {ok, Req1, State1} = cowboy_handler:execute(Req, State),
    %% Finish span
    tracer:finish_span(Span1),
    {ok, Req1, State1}.
```

---

## 4. Health Check Framework

```erlang
%% health_checker.erl
-module(health_checker).
-behaviour(gen_server).

-export([start_link/0, register_check/3, get_health/0]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

-record(check, {
    name     :: atom(),
    fun_     :: fun(),
    interval :: integer(),    % ms
    last_result  = unknown :: ok | {error, term()} | unknown,
    last_checked = 0       :: integer()
}).

-record(state, {
    checks :: #{atom() => #check{}}
}).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

register_check(Name, Fun, IntervalMs) ->
    gen_server:call(?MODULE, {register, Name, Fun, IntervalMs}).

get_health() ->
    gen_server:call(?MODULE, get_health).

init([]) ->
    %% Register built-in checks
    self() ! tick,
    {ok, #state{checks = #{}}}.

handle_call({register, Name, Fun, Interval}, _From, State) ->
    Check = #check{name = Name, fun_ = Fun, interval = Interval},
    Checks = maps:put(Name, Check, State#state.checks),
    {reply, ok, State#state{checks = Checks}};

handle_call(get_health, _From, State) ->
    Results = maps:map(fun(_Name, C) ->
        #{result => C#check.last_result,
          last_checked => C#check.last_checked}
    end, State#state.checks),
    Overall = case maps:values(Results) of
        [] -> healthy;
        Rs ->
            case lists:all(fun(R) -> maps:get(result, R) =:= ok end, Rs) of
                true  -> healthy;
                false -> degraded
            end
    end,
    {reply, #{status => Overall, checks => Results}, State}.

handle_info(tick, State) ->
    Now = erlang:system_time(millisecond),
    UpdatedChecks = maps:map(fun(_Name, Check) ->
        case Now - Check#check.last_checked >= Check#check.interval of
            true ->
                Result = try Check#check.fun_() catch _:E -> {error, E} end,
                Check#check{last_result = Result, last_checked = Now};
            false ->
                Check
        end
    end, State#state.checks),
    erlang:send_after(1000, self(), tick),
    {noreply, State#state{checks = UpdatedChecks}}.

handle_cast(_Msg, State) -> {noreply, State}.
```

```erlang
%% health_handler.erl — /health and /ready endpoints
-module(health_handler).
-behaviour(cowboy_handler).
-export([init/2]).

init(Req, State) ->
    Path = cowboy_req:path(Req),
    case Path of
        <<"/health">> ->
            %% Liveness: is the process alive?
            reply(200, #{status => <<"ok">>}, Req, State);
        <<"/ready">> ->
            %% Readiness: can we serve traffic?
            Health = health_checker:get_health(),
            Code = case maps:get(status, Health) of
                healthy  -> 200;
                degraded -> 503
            end,
            reply(Code, Health, Req, State)
    end.

reply(Code, Body, Req, State) ->
    Resp = cowboy_req:reply(Code,
        #{<<"content-type">> => <<"application/json">>},
        json:encode(Body),
        Req),
    {ok, Resp, State}.
```

---

## 5. Alerting and Anomaly Detection

```erlang
%% anomaly_detector.erl
%% Simple statistical anomaly detection using rolling window
-module(anomaly_detector).
-behaviour(gen_server).

-export([start_link/0, record/2, get_stats/1]).
-export([init/1, handle_call/3, handle_cast/2]).

-define(WINDOW_SIZE, 60).       % data points
-define(ANOMALY_THRESHOLD, 3).  % standard deviations

-record(metric, {
    name    :: atom(),
    window  :: queue:queue(),
    count   :: integer(),
    sum     :: float(),
    sum_sq  :: float()
}).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

record(MetricName, Value) ->
    gen_server:cast(?MODULE, {record, MetricName, Value}).

get_stats(MetricName) ->
    gen_server:call(?MODULE, {get_stats, MetricName}).

init([]) ->
    {ok, #{}}.

handle_cast({record, Name, Value}, State) ->
    Metric = maps:get(Name, State, empty_metric(Name)),
    {Updated, IsAnomaly} = update_metric(Metric, Value),
    case IsAnomaly of
        true ->
            %% Trigger alert
            Stats = compute_stats(Updated),
            alert(Name, Value, Stats);
        false -> ok
    end,
    {noreply, maps:put(Name, Updated, State)}.

handle_call({get_stats, Name}, _From, State) ->
    case maps:get(Name, State, undefined) of
        undefined -> {reply, undefined, State};
        Metric    -> {reply, compute_stats(Metric), State}
    end.

empty_metric(Name) ->
    #metric{name = Name, window = queue:new(),
            count = 0, sum = 0.0, sum_sq = 0.0}.

update_metric(Metric, Value) ->
    %% Add new value
    Window0 = queue:in(Value, Metric#metric.window),
    Count0  = Metric#metric.count + 1,
    Sum0    = Metric#metric.sum + Value,
    SumSq0  = Metric#metric.sum_sq + Value * Value,

    %% Remove oldest value if window full
    {Window, Count, Sum, SumSq} =
        case Count0 > ?WINDOW_SIZE of
            true ->
                {{value, Oldest}, W} = queue:out(Window0),
                {W, Count0 - 1, Sum0 - Oldest, SumSq0 - Oldest * Oldest};
            false ->
                {Window0, Count0, Sum0, SumSq0}
        end,

    Updated = Metric#metric{window  = Window, count  = Count,
                            sum     = Sum,    sum_sq = SumSq},

    %% Check if anomaly (only after enough data)
    IsAnomaly = case Count >= 10 of
        false -> false;
        true  ->
            Mean = Sum / Count,
            Variance = (SumSq / Count) - (Mean * Mean),
            Stddev = math:sqrt(max(0.0, Variance)),
            case Stddev > 0 of
                true  -> abs(Value - Mean) > ?ANOMALY_THRESHOLD * Stddev;
                false -> false
            end
    end,
    {Updated, IsAnomaly}.

compute_stats(#metric{count = 0}) -> #{count => 0};
compute_stats(#metric{count = N, sum = Sum, sum_sq = SumSq}) ->
    Mean     = Sum / N,
    Variance = (SumSq / N) - (Mean * Mean),
    Stddev   = math:sqrt(max(0.0, Variance)),
    #{count => N, mean => Mean, stddev => Stddev}.

alert(Metric, Value, Stats) ->
    logger:warning("ANOMALY DETECTED", #{
        metric => Metric,
        value  => Value,
        stats  => Stats
    }),
    %% Could also: send to PagerDuty, Slack, etc.
    telemetry:execute([anomaly, detected], #{value => Value},
                      #{metric => Metric}).
```

---

## 6. SLO/SLA Tracking

```erlang
%% slo_tracker.erl
%% Service Level Objective tracking with error budget burn rate
-module(slo_tracker).
-behaviour(gen_server).

-export([start_link/1, record_request/2, get_slo_status/0]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

%% SLO configuration
-record(slo_config, {
    name             :: atom(),
    target_pct       :: float(),    % e.g. 99.9
    window_seconds   :: integer(),  % e.g. 86400 (1 day)
    burn_rate_warn   :: float()     % alert when budget burns at this rate
}).

-record(state, {
    configs  :: [#slo_config{}],
    buckets  :: ets:tid()
}).

start_link(SloConfigs) ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, SloConfigs, []).

record_request(SloName, good_request) ->
    gen_server:cast(?MODULE, {record, SloName, good});
record_request(SloName, bad_request) ->
    gen_server:cast(?MODULE, {record, SloName, bad}).

get_slo_status() ->
    gen_server:call(?MODULE, get_status).

init(Configs) ->
    Tid = ets:new(slo_buckets, [public, named_table, {write_concurrency, true}]),
    erlang:send_after(60000, self(), cleanup),
    {ok, #state{configs = Configs, buckets = Tid}}.

handle_cast({record, SloName, Type}, State) ->
    Bucket = current_minute_bucket(),
    Key = {SloName, Bucket, Type},
    ets:update_counter(State#state.buckets, Key, 1, {Key, 0}),
    {noreply, State}.

handle_call(get_status, _From, State) ->
    Status = [compute_slo_status(Config, State#state.buckets)
              || Config <- State#state.configs],
    {reply, Status, State}.

handle_info(cleanup, State) ->
    %% Remove buckets older than max window
    MaxWindow = lists:max([C#slo_config.window_seconds
                           || C <- State#state.configs]),
    CutoffBucket = current_minute_bucket() - (MaxWindow div 60),
    ets:select_delete(State#state.buckets,
        [{ {{'_', '$1', '_'}, '_'}, [{'<', '$1', CutoffBucket}], [true] }]),
    erlang:send_after(60000, self(), cleanup),
    {noreply, State}.

compute_slo_status(Config, Tid) ->
    WindowBuckets = Config#slo_config.window_seconds div 60,
    NowBucket     = current_minute_bucket(),
    FromBucket    = NowBucket - WindowBuckets,

    %% Sum good and bad requests in window
    Pattern = fun(Type) ->
        ets:select(Tid, [{ {{Config#slo_config.name, '$1', Type}, '$2'},
                           [{'>', '$1', FromBucket}], ['$2'] }])
    end,
    GoodList = Pattern(good),
    BadList  = Pattern(bad),
    Good = lists:sum(GoodList),
    Bad  = lists:sum(BadList),
    Total = Good + Bad,

    case Total of
        0 ->
            #{name => Config#slo_config.name, status => no_data};
        _ ->
            ErrorRate  = Bad / Total * 100,
            CurrentSlo = 100 - ErrorRate,
            ErrorBudgetPct = Config#slo_config.target_pct,
            AllowedErrors  = Total * (1 - ErrorBudgetPct / 100),
            BudgetUsed     = min(1.0, Bad / max(1, AllowedErrors)),
            BurnRate       = BudgetUsed / (WindowBuckets / (24 * 60)), % per day normalized
            #{
                name         => Config#slo_config.name,
                target_pct   => ErrorBudgetPct,
                current_pct  => CurrentSlo,
                status       => slo_status(CurrentSlo, ErrorBudgetPct),
                budget_used  => BudgetUsed,
                burn_rate    => BurnRate,
                total        => Total,
                good         => Good,
                bad          => Bad
            }
    end.

slo_status(Current, Target) when Current >= Target -> meeting;
slo_status(_, _)                                   -> breaching.

current_minute_bucket() ->
    erlang:system_time(second) div 60.
```

---

## 7. แบบฝึกหัด

1. เพิ่ม Jaeger exporter จริงใน `tracer.erl` โดยใช้ Thrift binary protocol
2. เขียน alerting rule ที่ส่ง notification ไปยัง Slack เมื่อ SLO breach
3. สร้าง dashboard handler ที่ return JSON ที่มี metrics ทั้งหมด
4. Implement percentile calculator (p95, p99) สำหรับ request latency

---

## สรุป Part 71

✅ Telemetry pipeline: batch buffer → multiple backends  
✅ Prometheus exporter: metrics format, HTTP /metrics endpoint  
✅ Distributed tracing: W3C traceparent, span propagation  
✅ Health checks: liveness /health and readiness /ready  
✅ Anomaly detection: rolling window + statistical z-score  
✅ SLO tracking: error budget, burn rate, compliance status  

---

*Part 71/100 | [← ก่อนหน้า](../part70/README.md) | [ถัดไป →](../part72/README.md)*
