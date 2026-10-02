# Part 73: Zero-Downtime Deployment Patterns

> **"Every deploy should be boring. Excitement is a sign of poor engineering"**  
> ทุก deploy ควรน่าเบื่อ ความตื่นเต้นเป็นสัญญาณของ engineering ที่ไม่ดี

---

## สารบัญ

1. [Hot Code Upgrades with Relups](#1-hot-code-upgrades-with-relups)
2. [Blue-Green Deployment](#2-blue-green-deployment)
3. [Rolling Restarts](#3-rolling-restarts)
4. [Feature Flags](#4-feature-flags)
5. [Database Migration Without Downtime](#5-database-migration-without-downtime)
6. [แบบฝึกหัด](#6-แบบฝึกหัด)

---

## 1. Hot Code Upgrades with Relups

```erlang
%% my_server.erl — demonstrates code_change for hot upgrades
-module(my_server).
-behaviour(gen_server).

-export([start_link/0, get_state/0]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2,
         code_change/3, terminate/2]).

%% Version 1 state — flat record
-record(state_v1, {
    cache    = #{} :: map(),
    hits     = 0   :: integer(),
    misses   = 0   :: integer()
}).

%% Version 2 state — enhanced with metadata
-record(state, {
    cache     = #{} :: map(),
    hits      = 0   :: integer(),
    misses    = 0   :: integer(),
    version   = 2   :: integer(),
    started_at      :: integer()
}).

start_link() -> gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).
get_state()  -> gen_server:call(?MODULE, get_state).

init([]) ->
    {ok, #state{started_at = erlang:system_time(second)}}.

handle_call(get_state, _From, State) ->
    Info = #{
        cache_size => map_size(State#state.cache),
        hits       => State#state.hits,
        misses     => State#state.misses,
        version    => State#state.version
    },
    {reply, Info, State};

handle_call({get, Key}, _From, State) ->
    case maps:get(Key, State#state.cache, miss) of
        miss ->
            {reply, undefined, State#state{misses = State#state.misses + 1}};
        Value ->
            {reply, Value, State#state{hits = State#state.hits + 1}}
    end;

handle_call({put, Key, Value}, _From, State) ->
    NewCache = maps:put(Key, Value, State#state.cache),
    {reply, ok, State#state{cache = NewCache}}.

handle_cast(_Msg, State) -> {noreply, State}.
handle_info(_Info, State) -> {noreply, State}.
terminate(_Reason, _State) -> ok.

%% THE KEY FUNCTION: called during hot code upgrade
code_change("1.0.0", OldState, _Extra) when is_record(OldState, state_v1) ->
    %% Migrate from v1 to v2: add version and started_at fields
    NewState = #state{
        cache      = OldState#state_v1.cache,
        hits       = OldState#state_v1.hits,
        misses     = OldState#state_v1.misses,
        version    = 2,
        started_at = erlang:system_time(second)
    },
    {ok, NewState};

code_change(_OldVsn, State, _Extra) ->
    %% No migration needed
    {ok, State}.
```

```erlang
%% my_app.appup — release upgrade instructions
%% File: my_app-2.0.0/ebin/my_app.appup
%% Format: {NewVsn, [{OldVsn, Instructions}], [{NewVsn, DowngradeInstructions}]}

{"2.0.0",
 [{"1.0.0",
   [
     %% Load new beam files
     {load_module, my_server},
     {load_module, my_api_handler},

     %% Update supervisor child spec if needed
     {update, my_server, {advanced, []}},

     %% Add a new child process
     {add_module, new_feature_module}
   ]}
 ],
 [{"1.0.0",
   [
     %% Downgrade instructions
     {load_module, my_server},
     {update, my_server, {advanced, [downgrade]}}
   ]}
 ]
}.
```

```bash
# Generate relup from appup files using rebar3
# $ rebar3 relup  (reads all .appup files, creates relup)

# Apply the upgrade to a running node
# 1. Build new release
rebar3 release

# 2. Create upgrade tarball
rebar3 tar

# 3. Copy to running system and apply
# In Erlang shell or via rpc:
# release_handler:unpack_release("my_app-2.0.0")
# release_handler:install_release("2.0.0")
# release_handler:make_permanent("2.0.0")
```

---

## 2. Blue-Green Deployment

```erlang
%% blue_green.erl — manage blue-green deploy via load balancer
-module(blue_green).
-export([
    deploy_green/1,
    switch_traffic/1,
    rollback/0,
    verify_green/0
]).

-define(LB_TABLE, lb_routing).

deploy_green(Version) ->
    %% 1. Start new 'green' application on separate port
    GreenPort = 8081,
    ok = start_application(green, Version, GreenPort),

    %% 2. Wait for green to be healthy
    ok = wait_for_health(green, GreenPort, 30),

    %% 3. Run smoke tests against green
    ok = run_smoke_tests(GreenPort),

    io:format("Green v~s is ready on port ~p~n", [Version, GreenPort]),
    ok.

switch_traffic(Percentage) when Percentage =:= 100 ->
    %% Route all traffic to green
    ets:insert(?LB_TABLE, {active, green}),
    io:format("Traffic switched 100% to green~n"),
    ok;
switch_traffic(Percentage) ->
    %% Gradual traffic shift (canary approach)
    ets:insert(?LB_TABLE, {canary_pct, Percentage}),
    io:format("~p% traffic routed to green (canary)~n", [Percentage]),
    ok.

rollback() ->
    ets:insert(?LB_TABLE, {active, blue}),
    io:format("Rolled back to blue~n"),
    ok.

verify_green() ->
    %% Check error rates on green before full switch
    {ok, ErrorRate} = metrics:get_error_rate(green, 60),
    BlueErrorRate   = metrics:get_error_rate(blue, 60),
    case ErrorRate =< BlueErrorRate * 1.1 of
        true  -> {ok, comparable};
        false -> {error, {green_error_rate_higher, ErrorRate}}
    end.

%% Load balancer router — called per request
route_request(Req) ->
    case ets:lookup(?LB_TABLE, active) of
        [{active, green}] -> green;
        [{active, blue}]  -> blue;
        [] ->
            %% Canary routing
            Pct = case ets:lookup(?LB_TABLE, canary_pct) of
                [{canary_pct, P}] -> P;
                []                -> 0
            end,
            case rand:uniform(100) =< Pct of
                true  -> green;
                false -> blue
            end
    end.

start_application(_Color, _Version, _Port) -> ok.
wait_for_health(_Color, _Port, _Timeout)    -> ok.
run_smoke_tests(_Port)                      -> ok.
metrics_placeholder(_) -> ok.
metrics() -> metrics.
metrics:get_error_rate(_Color, _Window) -> {ok, 0.01}.
```

---

## 3. Rolling Restarts

```erlang
%% rolling_restart.erl
%% Restart cluster nodes one at a time to avoid downtime
-module(rolling_restart).
-export([restart_cluster/1, restart_node/2]).

-define(HEALTH_WAIT_TIMEOUT, 30000).
-define(DRAIN_TIMEOUT,       10000).

restart_cluster(Nodes) ->
    io:format("Starting rolling restart of ~p nodes~n", [length(Nodes)]),
    restart_nodes(Nodes, []).

restart_nodes([], Done) ->
    io:format("Rolling restart complete. Restarted: ~p~n", [Done]),
    ok;
restart_nodes([Node | Rest], Done) ->
    io:format("Restarting node: ~p~n", [Node]),
    case restart_node(Node, #{drain_timeout => ?DRAIN_TIMEOUT}) of
        ok ->
            io:format("Node ~p restarted successfully~n", [Node]),
            restart_nodes(Rest, [Node | Done]);
        {error, Reason} ->
            io:format("Node ~p failed: ~p — aborting rollout~n",
                      [Node, Reason]),
            {error, {node_failed, Node, Reason}}
    end.

restart_node(Node, Opts) ->
    DrainTimeout = maps:get(drain_timeout, Opts, ?DRAIN_TIMEOUT),

    %% Step 1: Remove from load balancer
    ok = lb:remove_backend(Node),
    io:format("  [~p] Removed from LB~n", [Node]),

    %% Step 2: Drain in-flight requests
    ok = drain_requests(Node, DrainTimeout),
    io:format("  [~p] Drained~n", [Node]),

    %% Step 3: Restart the application (not the Erlang node)
    ok = rpc:call(Node, application, stop, [myapp]),
    ok = rpc:call(Node, application, start, [myapp]),
    io:format("  [~p] Application restarted~n", [Node]),

    %% Step 4: Wait for node to be healthy
    case wait_healthy(Node, ?HEALTH_WAIT_TIMEOUT) of
        ok ->
            %% Step 5: Re-add to load balancer
            ok = lb:add_backend(Node),
            io:format("  [~p] Back in LB~n", [Node]),
            ok;
        {error, _} = Err ->
            Err
    end.

drain_requests(Node, Timeout) ->
    %% Ask the node to stop accepting new requests and wait
    rpc:call(Node, myapp, set_draining, [true]),
    wait_for_empty_mailboxes(Node, Timeout).

wait_for_empty_mailboxes(_Node, Timeout) when Timeout =< 0 ->
    {error, drain_timeout};
wait_for_empty_mailboxes(Node, Timeout) ->
    QueueLen = rpc:call(Node, myapp, active_request_count, []),
    case QueueLen of
        0 -> ok;
        N ->
            io:format("  Waiting for ~p active requests to complete~n", [N]),
            timer:sleep(500),
            wait_for_empty_mailboxes(Node, Timeout - 500)
    end.

wait_healthy(_Node, Timeout) when Timeout =< 0 ->
    {error, health_check_timeout};
wait_healthy(Node, Timeout) ->
    case rpc:call(Node, health_checker, get_health, []) of
        #{status := healthy} -> ok;
        _ ->
            timer:sleep(1000),
            wait_healthy(Node, Timeout - 1000)
    end.

%% Stubs
lb() -> lb.
lb:remove_backend(_) -> ok.
lb:add_backend(_) -> ok.
```

---

## 4. Feature Flags

```erlang
%% feature_flags.erl — runtime feature flag management
-module(feature_flags).
-behaviour(gen_server).

-export([start_link/0, is_enabled/1, is_enabled/2,
         enable/1, disable/1, rollout/2]).
-export([init/1, handle_call/3, handle_cast/2]).

-record(flag, {
    name       :: atom(),
    enabled    :: boolean(),
    rollout_pct = 0   :: 0..100,    % percent of users who see this feature
    conditions = []   :: list()     % [{user_group, vip}, {country, <<"US">>}]
}).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

is_enabled(FlagName) ->
    is_enabled(FlagName, #{}).

is_enabled(FlagName, Context) ->
    gen_server:call(?MODULE, {is_enabled, FlagName, Context}).

enable(FlagName)  -> gen_server:call(?MODULE, {set, FlagName, true}).
disable(FlagName) -> gen_server:call(?MODULE, {set, FlagName, false}).
rollout(FlagName, Pct) ->
    gen_server:call(?MODULE, {rollout, FlagName, Pct}).

init([]) ->
    Flags = #{
        new_checkout_flow => #flag{name = new_checkout_flow, enabled = false},
        dark_mode         => #flag{name = dark_mode, enabled = true},
        ai_recommendations=> #flag{name = ai_recommendations,
                                   enabled = true, rollout_pct = 20}
    },
    {ok, Flags}.

handle_call({is_enabled, Name, Context}, _From, Flags) ->
    Result = case maps:get(Name, Flags, undefined) of
        undefined -> false;
        Flag      -> evaluate_flag(Flag, Context)
    end,
    {reply, Result, Flags};

handle_call({set, Name, Value}, _From, Flags) ->
    NewFlags = maps:update_with(Name,
        fun(F) -> F#flag{enabled = Value} end,
        #flag{name = Name, enabled = Value},
        Flags),
    {reply, ok, NewFlags};

handle_call({rollout, Name, Pct}, _From, Flags) ->
    NewFlags = maps:update_with(Name,
        fun(F) -> F#flag{rollout_pct = Pct, enabled = Pct > 0} end,
        #flag{name = Name, enabled = Pct > 0, rollout_pct = Pct},
        Flags),
    {reply, ok, NewFlags}.

evaluate_flag(#flag{enabled = false}, _Context) -> false;
evaluate_flag(#flag{rollout_pct = 100}, _Context) -> true;
evaluate_flag(#flag{rollout_pct = 0}, _Context) -> true;  % enabled, no rollout
evaluate_flag(#flag{rollout_pct = Pct}, Context) ->
    %% Stable rollout: hash user ID to get consistent results
    UserId  = maps:get(user_id, Context, <<"">>),
    Hash    = erlang:phash2(UserId, 100),
    Hash < Pct.
```

---

## 5. Database Migration Without Downtime

```
Strategy: Expand-Contract (4-phase migration)
═══════════════════════════════════════════════════════════════

EXAMPLE: Rename column 'user_name' → 'username' in users table

PHASE 1 — EXPAND (deploy v1.1)
  Add new column: ALTER TABLE users ADD COLUMN username VARCHAR(255);
  Code v1.1 writes to BOTH old and new columns
  Reads from old column only

PHASE 2 — BACKFILL (background job)
  UPDATE users SET username = user_name WHERE username IS NULL;
  Run in small batches to avoid locking:
    UPDATE users SET username = user_name
    WHERE id IN (SELECT id FROM users WHERE username IS NULL LIMIT 1000);

PHASE 3 — SWITCH READ (deploy v1.2)
  Code reads from NEW column (username)
  Writes to BOTH columns still

PHASE 4 — CONTRACT (deploy v1.3 + DB cleanup)
  Code writes to new column only
  Then: ALTER TABLE users DROP COLUMN user_name;
```

```erlang
%% db_migration.erl — safe batched migration
-module(db_migration).
-export([run_migration/2, backfill/3]).

run_migration(Name, Fun) ->
    case is_migrated(Name) of
        true  ->
            io:format("Migration ~p already applied~n", [Name]);
        false ->
            io:format("Running migration: ~p~n", [Name]),
            ok = Fun(),
            mark_migrated(Name),
            io:format("Migration ~p complete~n", [Name])
    end.

%% Backfill in batches with configurable delay
backfill(Table, UpdateSql, Opts) ->
    BatchSize  = maps:get(batch_size, Opts, 1000),
    DelayMs    = maps:get(delay_ms, Opts, 100),
    TotalRows  = count_rows(Table),
    do_backfill(Table, UpdateSql, BatchSize, DelayMs, 0, TotalRows).

do_backfill(_Table, _SQL, _Batch, _Delay, Done, Total)
  when Done >= Total ->
    io:format("Backfill complete: ~p rows~n", [Total]),
    ok;
do_backfill(Table, Sql, BatchSize, DelayMs, Done, Total) ->
    {ok, Affected} = db:execute(Sql, [BatchSize]),
    NewDone = Done + Affected,
    Progress = trunc(NewDone / Total * 100),
    io:format("Backfill progress: ~p% (~p/~p)~n", [Progress, NewDone, Total]),
    case Affected > 0 of
        true ->
            timer:sleep(DelayMs),
            do_backfill(Table, Sql, BatchSize, DelayMs, NewDone, Total);
        false ->
            ok
    end.

is_migrated(Name) ->
    case db:query("SELECT 1 FROM schema_migrations WHERE name = $1", [Name]) of
        {ok, [_|_]} -> true;
        {ok, []}    -> false
    end.

mark_migrated(Name) ->
    db:execute("INSERT INTO schema_migrations (name, applied_at) VALUES ($1, NOW())",
               [Name]).

count_rows(Table) ->
    {ok, [{Count}]} = db:query(["SELECT COUNT(*) FROM ", Table], []),
    Count.
```

---

## 6. แบบฝึกหัด

1. เขียน `appup` file สำหรับ upgrade จาก version 1.0.0 → 2.0.0 ที่เพิ่ม field ใน state
2. Implement canary deployment ที่ monitor error rate แล้ว auto-rollback ถ้า >1%
3. สร้าง feature flag system ที่ load config จาก environment variable
4. เขียน database backfill script ที่ track progress และ resume จากจุดที่หยุด

---

## สรุป Part 73

✅ Hot code upgrades: appup/relup, code_change/3 state migration  
✅ Blue-green deployment: traffic switching, canary percentage  
✅ Rolling restarts: drain → restart → health check → re-add to LB  
✅ Feature flags: rollout percentage, stable hash-based assignment  
✅ Database migrations: expand-contract pattern, batched backfill  

---

*Part 73/100 | [← ก่อนหน้า](../part72/README.md) | [ถัดไป →](../part74/README.md)*
