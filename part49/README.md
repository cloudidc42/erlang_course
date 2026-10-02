# Part 49: Production Operations

> **"Shipping is a feature — but operating reliably is the product"**  
> Shipping เป็น feature — แต่ operate อย่างเชื่อถือได้คือตัวสินค้า

---

## สารบัญ

1. [Operations Runbook](#1-operations-runbook)
2. [Deployment Strategies](#2-deployment-strategies)
3. [Configuration Management](#3-configuration-management)
4. [Monitoring and Alerting](#4-monitoring-and-alerting)
5. [Incident Response](#5-incident-response)
6. [Capacity Planning](#6-capacity-planning)
7. [Runbooks as Code](#7-runbooks-as-code)
8. [แบบฝึกหัด](#8-แบบฝึกหัด)

---

## 1. Operations Runbook

```
Production Erlang Runbook:

Startup Sequence:
  1. Start infrastructure (DB, Redis, etc.)
  2. Apply DB migrations: rebar3 run_migration
  3. Start application: bin/myapp start
  4. Verify health: curl http://localhost:8080/health
  5. Smoke test: run/smoke_tests
  6. Enable traffic: update load balancer

Daily Operations:
  - Monitor: Prometheus/Grafana dashboards
  - Log review: grep ERROR /var/log/myapp/app.log
  - Backup: pg_dump myapp > backup_$(date +%Y%m%d).sql

Common Commands:
  bin/myapp console        # Erlang shell
  bin/myapp remote_console # Attach to running node
  bin/myapp eval "myapp:status()."
  bin/myapp rpc myapp status
```

---

## 2. Deployment Strategies

```bash
#!/bin/bash
# deploy.sh — blue-green deployment

APP=myapp
CURRENT=$(cat /etc/nginx/current_backend)
NEW=$([ "$CURRENT" = "blue" ] && echo "green" || echo "blue")

echo "Deploying to $NEW (current: $CURRENT)"

# Build release
rebar3 as prod release
rebar3 as prod tar

# Copy to new environment
scp _build/prod/rel/$APP/$APP-*.tar.gz $NEW-host:/opt/$APP/

# Deploy on new host
ssh $NEW-host "
  cd /opt/$APP
  tar xf $APP-*.tar.gz
  ./bin/$APP stop || true
  ./bin/$APP start
  sleep 5
  ./bin/$APP ping || exit 1
"

# Run smoke tests against new env
./smoke_tests.sh http://$NEW-host:8080
if [ $? -ne 0 ]; then
  echo "Smoke tests FAILED, aborting"
  exit 1
fi

# Switch traffic
nginx_switch() {
  sed -i "s/backend_$CURRENT/backend_$NEW/" /etc/nginx/nginx.conf
  nginx -s reload
  echo $NEW > /etc/nginx/current_backend
}
nginx_switch

echo "Deployment complete! Traffic now on $NEW"
echo "Old $CURRENT still running — stop with: ssh $CURRENT-host bin/$APP stop"
```

---

## 3. Configuration Management

```erlang
%% config.erl — runtime configuration management
-module(config).
-export([get/1, get/2, set/2, reload/0, validate/0]).

-define(APP, myapp).

get(Key) ->
    application:get_env(?APP, Key).

get(Key, Default) ->
    case application:get_env(?APP, Key) of
        {ok, V} -> V;
        undefined -> Default
    end.

set(Key, Value) ->
    application:set_env(?APP, Key, Value).

reload() ->
    %% Re-read sys.config without restart
    File = code:root_dir() ++ "/releases/" ++
           release_handler:which_releases(current) ++ "/sys.config",
    case file:consult(File) of
        {ok, [{Env}]} ->
            [application:set_env(App, Key, Val)
             || {App, Configs} <- Env,
                {Key, Val} <- Configs],
            ok;
        {error, Reason} ->
            {error, Reason}
    end.

validate() ->
    Required = [
        {db_host,       fun is_list/1},
        {db_port,       fun is_integer/1},
        {jwt_secret,    fun(V) -> is_binary(V) andalso byte_size(V) >= 32 end},
        {http_port,     fun is_integer/1}
    ],
    Errors = lists:filtermap(fun({Key, Validator}) ->
        case get(Key) of
            {ok, V} ->
                case Validator(V) of
                    true  -> false;
                    false -> {true, {Key, invalid_value}}
                end;
            undefined ->
                {true, {Key, missing}}
        end
    end, Required),
    case Errors of
        [] -> ok;
        _  -> {error, Errors}
    end.
```

---

## 4. Monitoring and Alerting

```erlang
%% ops_monitor.erl — production monitoring
-module(ops_monitor).
-behaviour(gen_server).
-export([start_link/0]).
-export([init/1, handle_info/2, handle_call/3, handle_cast/2]).

-define(CHECK_INTERVAL, 30_000).

-record(baseline, {
    memory_mb = 0,
    process_count = 0,
    message_queue_threshold = 1000
}).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

init([]) ->
    erlang:send_after(?CHECK_INTERVAL, self(), check),
    {ok, #baseline{}}.

handle_info(check, Baseline) ->
    erlang:send_after(?CHECK_INTERVAL, self(), check),
    Checks = [
        check_memory(Baseline),
        check_process_count(Baseline),
        check_large_mailboxes(Baseline),
        check_scheduler_utilization(),
        check_gc_pressure(),
        check_ets_memory()
    ],
    [alert(A) || {alert, A} <- Checks],
    {noreply, Baseline}.

check_memory(_B) ->
    MemMb = erlang:memory(total) div (1024*1024),
    metrics:gauge(beam_memory_mb, MemMb, #{}),
    case MemMb > 4096 of
        true  -> {alert, #{type => high_memory, value => MemMb}};
        false -> ok
    end.

check_process_count(_B) ->
    Count = erlang:system_info(process_count),
    Limit = erlang:system_info(process_limit),
    metrics:gauge(beam_process_count, Count, #{}),
    case Count > Limit * 0.8 of
        true  -> {alert, #{type => process_limit_near, count => Count}};
        false -> ok
    end.

check_large_mailboxes(_B) ->
    LargeMailboxes = [
        {P, Len} || P <- processes(),
        {message_queue_len, Len} <- [process_info(P, message_queue_len)],
        Len > 1000
    ],
    case LargeMailboxes of
        [] -> ok;
        _  -> {alert, #{type => large_mailboxes, processes => LargeMailboxes}}
    end.

check_scheduler_utilization() ->
    case erlang:statistics(scheduler_wall_time) of
        undefined -> ok;
        Stats ->
            Util = compute_util(Stats),
            metrics:gauge(scheduler_utilization, Util, #{}),
            ok
    end.

check_gc_pressure() ->
    {GCs, Words, _} = erlang:statistics(garbage_collection),
    metrics:counter_inc(gc_total, #{}),
    ok.

check_ets_memory() ->
    EtsMem = erlang:memory(ets) div (1024*1024),
    metrics:gauge(ets_memory_mb, EtsMem, #{}),
    ok.

compute_util(Stats) ->
    Total = lists:sum([T || {_, T, _} <- Stats]),
    Active = lists:sum([A || {_, _, A} <- Stats]),
    case Total of
        0 -> 0;
        _ -> round(Active * 100 / Total)
    end.

alert(#{type := Type} = Info) ->
    logger:error("OPS ALERT: ~p ~p", [Type, Info]),
    %% Send to PagerDuty/Slack/etc
    notification_service:send_alert(Info).

handle_call(_, _, S) -> {reply, ok, S}.
handle_cast(_, S)    -> {noreply, S}.
```

---

## 5. Incident Response

```erlang
%% incident.erl — tools for incident investigation
-module(incident).
-export([diagnose/0, heap_dump/1, message_queue_dump/1,
         kill_stuck_process/1, throttle_traffic/1]).

diagnose() ->
    #{
        memory     => erlang:memory(),
        processes  => erlang:system_info(process_count),
        schedulers => erlang:system_info(scheduler_id),
        run_queue  => erlang:statistics(run_queue),
        top_procs  => top_processes(10)
    }.

top_processes(N) ->
    All = processes(),
    Ranked = lists:sort(
        fun({_, H1, _, _}, {_, H2, _, _}) -> H1 > H2 end,
        [{P,
          element(2, process_info(P, heap_size)),
          element(2, process_info(P, message_queue_len)),
          element(2, process_info(P, current_function))}
         || P <- All,
            process_info(P) =/= undefined]
    ),
    lists:sublist(Ranked, N).

heap_dump(Pid) ->
    process_info(Pid, [
        heap_size, total_heap_size, stack_size,
        garbage_collection, reductions,
        current_function, registered_name,
        links, monitors, monitored_by
    ]).

message_queue_dump(Pid) ->
    {messages, Msgs} = process_info(Pid, messages),
    {length(Msgs), lists:sublist(Msgs, 20)}.

kill_stuck_process(Pid) ->
    logger:warning("Killing stuck process: ~p info=~p",
                   [Pid, heap_dump(Pid)]),
    exit(Pid, kill).

throttle_traffic(RatePerSec) ->
    %% Update rate limiter in real time
    application:set_env(myapp, max_rps, RatePerSec),
    logger:info("Traffic throttled to ~p req/s", [RatePerSec]).
```

---

## 6. Capacity Planning

```erlang
%% capacity.erl — estimate capacity and scaling needs
-module(capacity).
-export([current_load/0, estimate_headroom/0, recommend_scaling/0]).

current_load() ->
    #{
        rps           => metrics:get_counter(http_requests_total),
        p99_latency   => metrics:get_histogram_p99(http_request_duration_ms),
        memory_mb     => erlang:memory(total) div (1024*1024),
        cpu_util      => scheduler_utilization(),
        db_pool_util  => pool_utilization()
    }.

estimate_headroom() ->
    Load = current_load(),
    #{
        memory_headroom_pct =>
            round((4096 - maps:get(memory_mb, Load)) * 100 / 4096),
        cpu_headroom_pct =>
            100 - maps:get(cpu_util, Load),
        db_pool_headroom_pct =>
            100 - maps:get(db_pool_util, Load)
    }.

recommend_scaling() ->
    Headroom = estimate_headroom(),
    Recs = [
        case maps:get(memory_headroom_pct, Headroom) < 20 of
            true  -> "Increase memory or add nodes";
            false -> "Memory OK"
        end,
        case maps:get(cpu_headroom_pct, Headroom) < 20 of
            true  -> "Add CPU/nodes or optimize hot paths";
            false -> "CPU OK"
        end,
        case maps:get(db_pool_headroom_pct, Headroom) < 20 of
            true  -> "Increase DB pool size or add read replica";
            false -> "DB pool OK"
        end
    ],
    Recs.

scheduler_utilization() ->
    case erlang:statistics(scheduler_wall_time) of
        undefined -> 0;
        Stats ->
            Total  = lists:sum([T || {_, T, _} <- Stats]),
            Active = lists:sum([A || {_, _, A} <- Stats]),
            case Total of 0 -> 0; _ -> round(Active * 100 / Total) end
    end.

pool_utilization() ->
    #{workers := W, overflow := O} = db_router:pool_stats(),
    Max = application:get_env(myapp, db_pool_size, 10),
    Used = Max - W + O,
    round(Used * 100 / Max).
```

---

## 7. Runbooks as Code

```erlang
%% runbook.erl — automated operational procedures
-module(runbook).
-export([run/1]).

run(restart_stuck_workers) ->
    Workers = [P || P <- processes(),
               {current_function, {job_worker, _, _}} <-
                   [process_info(P, current_function)],
               {message_queue_len, N} <-
                   [process_info(P, message_queue_len)],
               N > 100],
    lists:foreach(fun(P) ->
        logger:warning("Runbook: restarting stuck worker ~p", [P]),
        supervisor:terminate_child(job_worker_sup, P)
    end, Workers),
    {ok, length(Workers)};

run(clear_expired_sessions) ->
    N = session_store:clear_expired(),
    logger:info("Runbook: cleared ~p expired sessions", [N]),
    {ok, N};

run(compact_ets) ->
    Tables = ets:all(),
    [ets:delete_all_objects(T)
     || T <- Tables,
        lists:member(T, [expired_cache, old_metrics])],
    {ok, compacted};

run(rotate_logs) ->
    logger:info("Runbook: rotating logs"),
    logger_handler:filesync(default),
    {ok, rotated};

run(Unknown) ->
    {error, {unknown_runbook, Unknown}}.
```

---

## 8. แบบฝึกหัด

1. สร้าง deployment script พร้อม rollback อัตโนมัติเมื่อ health check fail
2. เพิ่ม metric: `beam_run_queue_length` สำหรับ scheduler run queue depth
3. Implement `runbook:run(drain_traffic)` — ลด max_connections ลงทีละน้อยก่อน deploy
4. สร้าง capacity forecast: ประมาณว่าจะถึง limit ในกี่ชั่วโมง จาก trend ปัจจุบัน

---

## สรุป Part 49

✅ Production runbook  
✅ Blue-green deployment script  
✅ Runtime configuration management  
✅ Process and memory monitoring  
✅ Incident investigation tools  
✅ Capacity planning  
✅ Automated runbooks  

---

*Part 49/100 | [← ก่อนหน้า](../part48/README.md) | [ถัดไป →](../part50/README.md)*
