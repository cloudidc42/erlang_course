# Part 54: Erlang Cluster Management

> **"A cluster of Erlang nodes is not just many machines — it's one mind"**  
> cluster ของ Erlang nodes ไม่ใช่แค่หลายเครื่อง — มันคือจิตใจเดียวกัน

---

## สารบัญ

1. [Node Discovery และ Formation](#1-node-discovery-และ-formation)
2. [Global Process Registry](#2-global-process-registry)
3. [Cluster-Wide Tasks](#3-cluster-wide-tasks)
4. [Network Partitions](#4-network-partitions)
5. [Node Monitoring](#5-node-monitoring)
6. [Load Distribution](#6-load-distribution)
7. [แบบฝึกหัด](#7-แบบฝึกหัด)

---

## 1. Node Discovery และ Formation

```erlang
%% cluster.erl — node discovery and formation
-module(cluster).
-export([form/1, join/1, leave/0, members/0]).

%% Form cluster from a list of known nodes
form(Nodes) ->
    [try_connect(N) || N <- Nodes],
    Members = members(),
    logger:info("Cluster formed with ~p nodes: ~p",
                [length(Members), Members]).

join(SeedNode) ->
    case net_kernel:connect_node(SeedNode) of
        true ->
            %% Get current cluster members from seed
            RemoteMembers = rpc:call(SeedNode, cluster, members, []),
            [try_connect(N) || N <- RemoteMembers],
            logger:info("Joined cluster via ~p", [SeedNode]),
            ok;
        false ->
            {error, {cannot_connect, SeedNode}}
    end.

leave() ->
    Members = members(),
    %% Notify all members we're leaving
    [rpc:cast(N, cluster, node_leaving, [node()]) || N <- Members],
    erlang:disconnect_node(N) || N <- Members,
    ok.

members() ->
    [node() | nodes()].

try_connect(Node) ->
    case net_kernel:connect_node(Node) of
        true  -> logger:info("Connected to ~p", [Node]);
        false -> logger:warning("Failed to connect to ~p", [Node])
    end.

%% DNS-based discovery
discover_via_dns(ServiceName) ->
    case inet_res:lookup(ServiceName, in, a) of
        {ok, Addrs} ->
            [list_to_atom("myapp@" ++ inet:ntoa(A)) || A <- Addrs];
        _ -> []
    end.

%% Environment-based discovery
discover_via_env() ->
    case os:getenv("ERLANG_NODES") of
        false -> [];
        Nodes -> [list_to_atom(N) || N <- string:split(Nodes, ",", all)]
    end.
```

---

## 2. Global Process Registry

```erlang
%% global_registry.erl — cluster-wide process names
-module(global_registry).
-export([register/2, whereis/1, unregister/1, call/2, cast/2]).

%% global module: built-in cluster-wide registry
register(Name, Pid) ->
    case global:register_name(Name, Pid) of
        yes -> ok;
        no  -> {error, already_registered}
    end.

whereis(Name) ->
    case global:whereis_name(Name) of
        undefined -> {error, not_found};
        Pid       -> {ok, Pid}
    end.

unregister(Name) ->
    global:unregister_name(Name).

call(Name, Request) ->
    case global:whereis_name(Name) of
        undefined -> {error, not_found};
        Pid       -> gen_server:call(Pid, Request)
    end.

cast(Name, Message) ->
    case global:whereis_name(Name) of
        undefined -> ok;
        Pid       -> gen_server:cast(Pid, Message)
    end.

%% Re-register on node reconnect
handle_global_name_conflict(Name, Pid1, Pid2) ->
    %% Keep the process on the node that has lower name
    case node(Pid1) < node(Pid2) of
        true  -> exit(Pid2, kill), Pid1;
        false -> exit(Pid1, kill), Pid2
    end.
```

---

## 3. Cluster-Wide Tasks

```erlang
%% cluster_tasks.erl — execute tasks across all nodes
-module(cluster_tasks).
-export([broadcast/1, map_nodes/1, gather/2, elect_and_run/2]).

%% Broadcast: fire-and-forget to all nodes
broadcast(Fun) ->
    [rpc:cast(N, erlang, apply, [Fun, []]) || N <- [node() | nodes()]].

%% Map: run Fun on every node, collect results
map_nodes(Fun) ->
    Nodes   = [node() | nodes()],
    Results = rpc:multicall(Nodes, erlang, apply, [Fun, []]),
    lists:zip(Nodes, element(1, Results)).

%% Gather results with timeout
gather(Fun, TimeoutMs) ->
    Self    = self(),
    Nodes   = [node() | nodes()],
    Workers = [spawn(fun() ->
        R = try Fun(N) catch C:E -> {error, {C, E}} end,
        Self ! {result, N, R}
    end) || N <- Nodes],
    collect(length(Workers), TimeoutMs, []).

collect(0, _, Acc) -> Acc;
collect(N, T, Acc) ->
    receive
        {result, Node, R} -> collect(N-1, T, [{Node, R} | Acc])
    after T ->
        Acc
    end.

%% Run task only on the elected leader node
elect_and_run(TaskName, Fun) ->
    case leader_election:am_i_leader() of
        true ->
            logger:info("Running cluster task ~p on leader ~p",
                        [TaskName, node()]),
            Fun();
        false ->
            {skipped, not_leader}
    end.
```

---

## 4. Network Partitions

```erlang
%% partition_handler.erl — handle netsplits gracefully
-module(partition_handler).
-behaviour(gen_server).
-export([start_link/0]).
-export([init/1, handle_info/2, handle_call/3, handle_cast/2]).

-record(state, {known_nodes = [], partitioned = []}).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

init([]) ->
    %% Monitor all nodes
    net_kernel:monitor_nodes(true, [nodedown_reason]),
    {ok, #state{known_nodes = nodes()}}.

handle_info({nodedown, Node, #{nodedown_reason := Reason}}, State) ->
    logger:error("Node down: ~p reason=~p", [Node, Reason]),
    case Reason of
        connection_closed ->
            %% Network partition — take conservative action
            handle_partition(Node, State);
        _ ->
            %% Likely intentional shutdown
            {noreply, State#state{known_nodes = State#state.known_nodes -- [Node]}}
    end;

handle_info({nodeup, Node}, State) ->
    logger:info("Node reconnected: ~p", [Node]),
    %% Trigger reconciliation
    spawn(fun() -> reconcile_state(Node) end),
    {noreply, State#state{
        known_nodes = lists:usort([Node | State#state.known_nodes]),
        partitioned = State#state.partitioned -- [Node]
    }}.

handle_partition(Node, State) ->
    logger:warning("Possible partition with ~p — entering partition mode", [Node]),
    %% Conservative: stop accepting writes that affect Node's data
    %% Depends on consistency model (CP vs AP)
    {noreply, State#state{partitioned = [Node | State#state.partitioned]}}.

reconcile_state(Node) ->
    %% After partition heals: sync state
    logger:info("Reconciling state with ~p", [Node]),
    %% Strategy depends on data type:
    %% - CRDTs: merge automatically
    %% - Last-write-wins: compare timestamps
    %% - Raft/consensus: replay log from last common point
    ok.

handle_call(_, _, S) -> {reply, ok, S}.
handle_cast(_, S)    -> {noreply, S}.
```

---

## 5. Node Monitoring

```erlang
%% node_monitor.erl — track cluster health
-module(node_monitor).
-behaviour(gen_server).
-export([start_link/0, cluster_status/0]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

-record(node_info, {
    name,
    status,         %% up | down | degraded
    memory_mb   = 0,
    process_cnt = 0,
    last_seen
}).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

cluster_status() ->
    gen_server:call(?MODULE, status).

init([]) ->
    erlang:send_after(5000, self(), collect_stats),
    net_kernel:monitor_nodes(true),
    {ok, #{}}.

handle_info(collect_stats, NodeMap) ->
    erlang:send_after(5000, self(), collect_stats),
    NewMap = collect_all_stats(NodeMap),
    metrics:gauge(cluster_node_count, maps:size(NewMap), #{}),
    {noreply, NewMap};

handle_info({nodeup, Node}, NodeMap) ->
    Info = #node_info{name=Node, status=up, last_seen=os:system_time()},
    {noreply, maps:put(Node, Info, NodeMap)};

handle_info({nodedown, Node}, NodeMap) ->
    Info = case maps:find(Node, NodeMap) of
        {ok, I} -> I#node_info{status=down};
        error   -> #node_info{name=Node, status=down}
    end,
    {noreply, maps:put(Node, Info, NodeMap)}.

handle_call(status, _, NodeMap) ->
    {reply, maps:values(NodeMap), NodeMap}.

collect_all_stats(NodeMap) ->
    AllNodes = [node() | nodes()],
    lists:foldl(fun(N, Acc) ->
        Info = get_node_info(N),
        maps:put(N, Info, Acc)
    end, NodeMap, AllNodes).

get_node_info(Node) ->
    MemMb = case rpc:call(Node, erlang, memory, [total], 5000) of
        {badrpc, _} -> 0;
        M           -> M div (1024*1024)
    end,
    ProcCnt = case rpc:call(Node, erlang, system_info, [process_count], 5000) of
        {badrpc, _} -> 0;
        C           -> C
    end,
    #node_info{name=Node, status=up, memory_mb=MemMb,
               process_cnt=ProcCnt, last_seen=os:system_time()}.

handle_cast(_, S) -> {noreply, S}.
```

---

## 6. Load Distribution

```erlang
%% load_balancer.erl — distribute work across cluster nodes
-module(load_balancer).
-export([choose_node/0, choose_node/1, dispatch/2]).

%% Round-robin node selection
choose_node() ->
    Nodes = [node() | nodes()],
    Idx   = ets:update_counter(lb_state, rr_index, {2, 1, length(Nodes), 1}),
    lists:nth(Idx, Nodes).

%% Least-loaded node
choose_node(least_loaded) ->
    Nodes  = [node() | nodes()],
    Loads  = [{N, get_load(N)} || N <- Nodes],
    {Best, _} = lists:foldl(fun({N, L}, {BestN, BestL}) ->
        case L < BestL of
            true  -> {N, L};
            false -> {BestN, BestL}
        end
    end, hd(Loads), tl(Loads)),
    Best.

get_load(Node) ->
    case rpc:call(Node, erlang, statistics, [run_queue], 1000) of
        {badrpc, _} -> 999999;
        Load        -> Load
    end.

%% Dispatch work to chosen node
dispatch(Fun, Strategy) ->
    Node = choose_node(Strategy),
    case Node =:= node() of
        true  -> Fun();
        false -> rpc:call(Node, erlang, apply, [Fun, []])
    end.
```

---

## 7. แบบฝึกหัด

1. สร้าง cluster-wide rate limiter ด้วย global counter
2. Implement quorum write: require majority of nodes to confirm before reply
3. ทดสอบ partition tolerance: disconnect node mid-request, verify recovery
4. สร้าง auto-scaling: เพิ่ม node เมื่อ run queue > threshold

---

## สรุป Part 54

✅ Node discovery ด้วย DNS, environment variables  
✅ Global process registry  
✅ Cluster-wide task execution  
✅ Network partition handling  
✅ Node health monitoring  
✅ Load distribution strategies  

---

*Part 54/100 | [← ก่อนหน้า](../part53/README.md) | [ถัดไป →](../part55/README.md)*
