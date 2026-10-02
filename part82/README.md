# Part 82: Multi-Tenancy Architecture

> **"Multi-tenancy: serve many masters from a single codebase"**  
> Multi-tenancy: ให้บริการหลายลูกค้าจาก codebase เดียว

---

## สารบัญ

1. [Multi-Tenancy Models](#1-multi-tenancy-models)
2. [Tenant Isolation](#2-tenant-isolation)
3. [Tenant-Aware Database](#3-tenant-aware-database)
4. [Per-Tenant Resource Quotas](#4-per-tenant-resource-quotas)
5. [Tenant Onboarding and Offboarding](#5-tenant-onboarding-and-offboarding)
6. [Cross-Tenant Analytics](#6-cross-tenant-analytics)
7. [แบบฝึกหัด](#7-แบบฝึกหัด)

---

## 1. Multi-Tenancy Models

```
Three Models of Multi-Tenancy
════════════════════════════════════════════════════════

MODEL 1: SHARED DATABASE, SHARED SCHEMA
  All tenants in same tables, filtered by tenant_id column
  
  Pros: Simple, low overhead, easy to maintain
  Cons: Tenant data co-mingled, harder to isolate, harder to move data

  Example:
    SELECT * FROM orders WHERE tenant_id = $1 AND user_id = $2

MODEL 2: SHARED DATABASE, SEPARATE SCHEMAS
  Each tenant has their own PostgreSQL schema
  
  Pros: Better isolation, easy per-tenant backup
  Cons: More connection overhead, schema migration complexity

  Example:
    SELECT * FROM tenant_acme.orders WHERE user_id = $1
    SELECT * FROM tenant_globex.orders WHERE user_id = $1

MODEL 3: SEPARATE DATABASES
  Each tenant has their own database/server
  
  Pros: Maximum isolation, compliance-friendly
  Cons: High operational overhead, expensive

  RECOMMENDATION for most SaaS:
  Start with Model 1, migrate to Model 2 as compliance needs arise.
  
ERLANG IMPLEMENTATION:
  Tenant context is stored in process dictionary per request.
  All DB queries automatically scoped to current tenant.
  ETS tables partitioned by tenant_id key prefix.
```

---

## 2. Tenant Isolation

```erlang
%% tenant_context.erl — per-request tenant context
-module(tenant_context).
-export([set/1, get/0, require/0, clear/0]).
-export([with_tenant/2]).

-define(TENANT_KEY, '$tenant_id').

set(TenantId) ->
    put(?TENANT_KEY, TenantId).

get() ->
    erlang:get(?TENANT_KEY).

require() ->
    case get() of
        undefined -> error(no_tenant_context);
        TenantId  -> TenantId
    end.

clear() ->
    erase(?TENANT_KEY).

%% Run a function in the context of a specific tenant
with_tenant(TenantId, Fun) ->
    Previous = get(),
    set(TenantId),
    try
        Fun()
    after
        case Previous of
            undefined -> clear();
            _         -> set(Previous)
        end
    end.
```

```erlang
%% tenant_middleware.erl — extract tenant from request
-module(tenant_middleware).
-behaviour(cowboy_middleware).
-export([execute/2]).

execute(Req, Env) ->
    case resolve_tenant(Req) of
        {ok, TenantId} ->
            tenant_context:set(TenantId),
            Req1 = cowboy_req:set_meta(tenant_id, TenantId, Req),
            {ok, Req1, Env};
        {error, not_found} ->
            Response = cowboy_req:reply(404,
                #{<<"content-type">> => <<"application/json">>},
                json:encode(#{error => tenant_not_found}),
                Req),
            {stop, Response}
    end.

resolve_tenant(Req) ->
    %% Strategy 1: Subdomain (acme.myapp.com → tenant "acme")
    Host = cowboy_req:host(Req),
    case extract_subdomain(Host) of
        {ok, Subdomain} ->
            lookup_tenant_by_subdomain(Subdomain);
        not_found ->
            %% Strategy 2: Header (X-Tenant-Id: acme)
            case cowboy_req:header(<<"x-tenant-id">>, Req) of
                undefined -> {error, not_found};
                TenantId  -> verify_tenant(TenantId)
            end
    end.

extract_subdomain(Host) ->
    %% Strip base domain to get subdomain
    BaseDomain = <<"myapp.com">>,
    Suffix = <<".", BaseDomain/binary>>,
    case binary:longest_common_suffix([Host, Suffix]) of
        N when N =:= byte_size(Suffix) ->
            Prefix = binary:part(Host, 0, byte_size(Host) - N),
            case Prefix of
                <<>> -> not_found;
                Sub  -> {ok, Sub}
            end;
        _ -> not_found
    end.

lookup_tenant_by_subdomain(Subdomain) ->
    %% Cache in ETS
    case ets:lookup(tenant_cache, {subdomain, Subdomain}) of
        [{_, TenantId, Exp}] when Exp > erlang:system_time(second) ->
            {ok, TenantId};
        _ ->
            case db:query("SELECT id FROM tenants WHERE subdomain = $1 AND active = TRUE",
                          [Subdomain]) of
                {ok, [{TenantId}]} ->
                    ets:insert(tenant_cache,
                               {{subdomain, Subdomain}, TenantId,
                                erlang:system_time(second) + 300}),
                    {ok, TenantId};
                {ok, []} ->
                    {error, not_found}
            end
    end.

verify_tenant(TenantId) ->
    case db:query("SELECT id FROM tenants WHERE id = $1 AND active = TRUE",
                  [TenantId]) of
        {ok, [{TenantId}]} -> {ok, TenantId};
        {ok, []}           -> {error, not_found}
    end.
```

---

## 3. Tenant-Aware Database

```erlang
%% tenant_db.erl — automatic tenant_id injection in all queries
-module(tenant_db).
-export([query/2, execute/2]).

%% All queries automatically scoped to current tenant
query(Sql, Params) ->
    TenantId = tenant_context:require(),
    %% Inject tenant_id as first parameter
    FullSql  = inject_tenant_filter(Sql),
    db:query(FullSql, [TenantId | Params]).

execute(Sql, Params) ->
    TenantId = tenant_context:require(),
    FullSql  = inject_tenant_filter(Sql),
    db:execute(FullSql, [TenantId | Params]).

%% Re-number parameters after injecting tenant_id as $1
%% Original: "SELECT * FROM orders WHERE user_id = $1"
%% Result:   "SELECT * FROM orders WHERE tenant_id = $1 AND user_id = $2"
inject_tenant_filter(Sql) ->
    %% Simple approach: append AND tenant_id = $1 to WHERE clause
    %% and renumber existing parameters
    SqlBin = iolist_to_binary(Sql),
    case binary:match(SqlBin, <<"WHERE">>) of
        {Pos, Len} ->
            Before = binary:part(SqlBin, 0, Pos + Len),
            After  = binary:part(SqlBin, Pos + Len, byte_size(SqlBin) - Pos - Len),
            Renumbered = renumber_params(After, 2),
            <<Before/binary, " tenant_id = $1 AND", Renumbered/binary>>;
        nomatch ->
            %% No WHERE clause - add one
            renumber_params(<<SqlBin/binary, " WHERE tenant_id = $1">>, 2)
    end.

renumber_params(Sql, StartAt) ->
    %% Replace $1 → $StartAt, $2 → $StartAt+1, etc.
    {Result, _} = re:replace(
        Sql,
        <<"\\$(\\d+)">>,
        fun([_, Num]) ->
            N = binary_to_integer(Num),
            <<"$", (integer_to_binary(N + StartAt - 1))/binary>>
        end,
        [global, {return, binary}]
    ),
    Result.
```

---

## 4. Per-Tenant Resource Quotas

```erlang
%% quota_manager.erl — enforce resource limits per tenant
-module(quota_manager).
-behaviour(gen_server).

-export([start_link/0, check_quota/2, consume/3, get_usage/1]).
-export([init/1, handle_call/3, handle_cast/2]).

-define(DEFAULT_QUOTAS, #{
    api_calls_per_hour => 10000,
    storage_gb         => 100,
    users              => 500,
    concurrent_jobs    => 10
}).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

check_quota(TenantId, Resource) ->
    gen_server:call(?MODULE, {check, TenantId, Resource}).

consume(TenantId, Resource, Amount) ->
    gen_server:call(?MODULE, {consume, TenantId, Resource, Amount}).

get_usage(TenantId) ->
    gen_server:call(?MODULE, {usage, TenantId}).

init([]) ->
    ets:new(quotas, [named_table, public, {write_concurrency, true}]),
    ets:new(tenant_limits, [named_table, public]),
    %% Load tenant limits from DB
    load_tenant_limits(),
    %% Reset hourly quotas
    schedule_reset(),
    {ok, #{}}.

handle_call({check, TenantId, Resource}, _From, State) ->
    Limit = get_limit(TenantId, Resource),
    Used  = get_used(TenantId, Resource),
    case Used < Limit of
        true  -> {reply, {ok, Limit - Used}, State};
        false -> {reply, {error, quota_exceeded}, State}
    end;

handle_call({consume, TenantId, Resource, Amount}, _From, State) ->
    Limit = get_limit(TenantId, Resource),
    Key   = {TenantId, Resource},
    NewUsed = ets:update_counter(quotas, Key, Amount, {Key, 0}),
    case NewUsed =< Limit of
        true  -> {reply, ok, State};
        false ->
            %% Undo the consumption
            ets:update_counter(quotas, Key, -Amount),
            {reply, {error, quota_exceeded}, State}
    end;

handle_call({usage, TenantId}, _From, State) ->
    Limits = case ets:lookup(tenant_limits, TenantId) of
        [{_, L}] -> L;
        []       -> ?DEFAULT_QUOTAS
    end,
    Usage = maps:map(fun(Resource, _Limit) ->
        get_used(TenantId, Resource)
    end, Limits),
    {reply, {ok, #{limits => Limits, usage => Usage}}, State}.

handle_cast(_Msg, State) -> {noreply, State}.

get_limit(TenantId, Resource) ->
    case ets:lookup(tenant_limits, TenantId) of
        [{_, Limits}] -> maps:get(Resource, Limits, 0);
        []            -> maps:get(Resource, ?DEFAULT_QUOTAS, 0)
    end.

get_used(TenantId, Resource) ->
    Key = {TenantId, Resource},
    case ets:lookup(quotas, Key) of
        [{_, N}] -> N;
        []       -> 0
    end.

load_tenant_limits() ->
    {ok, Rows} = db:query(
        "SELECT id, quota_config FROM tenants WHERE active = TRUE", []),
    lists:foreach(fun({TenantId, ConfigJson}) ->
        Config = json:decode(ConfigJson),
        ets:insert(tenant_limits, {TenantId, Config})
    end, Rows).

schedule_reset() ->
    %% Reset API call counter every hour
    erlang:send_after(3600000, self(), reset_hourly).
```

---

## 5. Tenant Onboarding and Offboarding

```erlang
%% tenant_lifecycle.erl — create and deactivate tenants
-module(tenant_lifecycle).
-export([onboard/1, offboard/2, suspend/2, resume/2]).

onboard(Params) ->
    %% Validate plan and billing
    Name      = maps:get(name, Params),
    Subdomain = maps:get(subdomain, Params),
    Plan      = maps:get(plan, Params, starter),

    %% Check subdomain availability
    case db:query("SELECT 1 FROM tenants WHERE subdomain = $1", [Subdomain]) of
        {ok, [_]} ->
            {error, subdomain_taken};
        {ok, []} ->
            %% Create tenant
            TenantId = generate_tenant_id(),
            Sql = "INSERT INTO tenants (id, name, subdomain, plan, active, created_at)
                   VALUES ($1, $2, $3, $4, TRUE, NOW())",
            {ok, _} = db:execute(Sql, [TenantId, Name, Subdomain,
                                       atom_to_binary(Plan)]),

            %% Run per-tenant setup
            setup_tenant_schema(TenantId),
            setup_default_roles(TenantId),
            setup_quota_config(TenantId, Plan),

            %% Send welcome email
            email_service:send_welcome(maps:get(admin_email, Params), TenantId),

            {ok, TenantId}
    end.

offboard(TenantId, Reason) ->
    %% Graceful shutdown: cancel subscriptions, export data
    billing_service:cancel_subscription(TenantId),
    export_tenant_data(TenantId),

    %% Soft delete — keep data for 30 days for recovery
    db:execute("UPDATE tenants SET active = FALSE, deleted_at = NOW(),
                deletion_reason = $2 WHERE id = $1",
               [TenantId, Reason]),

    %% Schedule hard delete
    schedule_hard_delete(TenantId, 30 * 24 * 3600).

suspend(TenantId, Reason) ->
    db:execute("UPDATE tenants SET suspended = TRUE, suspend_reason = $2
                WHERE id = $1", [TenantId, Reason]),
    %% Invalidate all sessions for this tenant
    session_store:invalidate_tenant_sessions(TenantId),
    ok.

resume(TenantId, _Approver) ->
    db:execute("UPDATE tenants SET suspended = FALSE WHERE id = $1",
               [TenantId]).

setup_tenant_schema(_TenantId) -> ok.
setup_default_roles(_TenantId) -> ok.
setup_quota_config(_TenantId, _Plan) -> ok.
export_tenant_data(_TenantId) -> ok.
schedule_hard_delete(_TenantId, _DelaySeconds) -> ok.
generate_tenant_id() -> base64:encode(crypto:strong_rand_bytes(12)).
```

---

## 6. Cross-Tenant Analytics

```erlang
%% platform_analytics.erl — aggregate metrics across all tenants (admin only)
-module(platform_analytics).
-export([tenant_health_report/0, usage_breakdown/1, churn_risk/0]).

tenant_health_report() ->
    Sql = "
        SELECT t.id, t.name, t.plan,
               COUNT(DISTINCT u.id) as active_users,
               COUNT(DISTINCT o.id) as orders_30d,
               t.created_at
        FROM tenants t
        LEFT JOIN users u ON u.tenant_id = t.id
                          AND u.last_seen_at > NOW() - INTERVAL '30 days'
        LEFT JOIN orders o ON o.tenant_id = t.id
                           AND o.created_at > NOW() - INTERVAL '30 days'
        WHERE t.active = TRUE
        GROUP BY t.id, t.name, t.plan, t.created_at
        ORDER BY active_users DESC
    ",
    {ok, Rows} = db:query(Sql, []),
    [#{id => Id, name => Name, plan => Plan,
       active_users => Users, orders_30d => Orders,
       created_at => CreatedAt}
     || {Id, Name, Plan, Users, Orders, CreatedAt} <- Rows].

usage_breakdown(TenantId) ->
    %% Must be admin or the tenant themselves
    Sql = "
        SELECT
            (SELECT COUNT(*) FROM users WHERE tenant_id = $1) as total_users,
            (SELECT COUNT(*) FROM orders WHERE tenant_id = $1) as total_orders,
            (SELECT COALESCE(SUM(amount), 0) FROM orders
             WHERE tenant_id = $1 AND created_at > NOW() - INTERVAL '30 days')
             as revenue_30d
    ",
    {ok, [{Users, Orders, Revenue}]} = db:query(Sql, [TenantId]),
    #{total_users => Users, total_orders => Orders, revenue_30d => Revenue}.

churn_risk() ->
    %% Identify tenants at risk of churning
    Sql = "
        SELECT t.id, t.name,
               EXTRACT(DAY FROM NOW() - MAX(u.last_seen_at)) as days_since_activity,
               t.plan
        FROM tenants t
        JOIN users u ON u.tenant_id = t.id
        WHERE t.active = TRUE
        GROUP BY t.id, t.name, t.plan
        HAVING MAX(u.last_seen_at) < NOW() - INTERVAL '14 days'
        ORDER BY days_since_activity DESC
        LIMIT 50
    ",
    {ok, Rows} = db:query(Sql, []),
    [#{id => Id, name => Name, days_inactive => Days, plan => Plan}
     || {Id, Name, Days, Plan} <- Rows].
```

---

## 7. แบบฝึกหัด

1. Implement tenant data export ที่ export ข้อมูลทั้งหมดเป็น ZIP file
2. สร้าง plan upgrade flow ที่ เพิ่ม quota ทันทีเมื่อ payment สำเร็จ
3. เพิ่ม tenant-specific custom domains (e.g., app.customer-domain.com)
4. Implement data residency: บาง tenants ต้องการ data ใน EU เท่านั้น

---

## สรุป Part 82

✅ Multi-tenancy models: shared schema vs. separate schemas tradeoffs  
✅ Tenant isolation: process dictionary context, automatic middleware  
✅ Tenant-aware DB: parameter injection, automatic tenant_id scoping  
✅ Resource quotas: per-tenant ETS counters with atomic enforcement  
✅ Tenant lifecycle: onboarding, suspension, graceful offboarding  
✅ Cross-tenant analytics: admin-level reporting across all tenants  

---

*Part 82/100 | [← ก่อนหน้า](../part81/README.md) | [ถัดไป →](../part83/README.md)*
