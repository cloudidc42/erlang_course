# Part 18: Distributed Erlang พื้นฐาน

> **"Distribution is built into Erlang — not bolted on"**  
> Distribution ถูก built-in ไว้ใน Erlang ไม่ใช่เพิ่มเข้ามาทีหลัง

---

## สารบัญ

1. [Distributed Erlang คืออะไร?](#1-distributed-erlang-คืออะไร)
2. [Node Setup](#2-node-setup)
3. [Node Operations](#3-node-operations)
4. [Distributed Message Passing](#4-distributed-message-passing)
5. [Remote Process Management](#5-remote-process-management)
6. [Global Module](#6-global-module)
7. [net_kernel Module](#7-net_kernel-module)
8. [Distributed ETS](#8-distributed-ets)
9. [Network Partitions](#9-network-partitions)
10. [แบบฝึกหัด](#10-แบบฝึกหัด)

---

## 1. Distributed Erlang คืออะไร?

```
Distributed Erlang:
├── หลาย Erlang nodes ทำงานร่วมกัน
├── Transparent message passing ข้าม nodes
├── Shared cookie สำหรับ authentication
├── epmd (Erlang Port Mapper Daemon) สำหรับ discovery
└── Built on top of TCP/IP

Use cases:
- Scale-out: เพิ่ม capacity ด้วยการ add nodes
- Fault tolerance: ถ้า node หนึ่งตาย node อื่นรับช่วง
- Geographic distribution: nodes ต่าง datacenter
- Hot standby: primary/backup configuration
```

---

## 2. Node Setup

```bash
# Short name (เฉพาะ local network)
erl -sname myapp

# Long name (ข้าม network)
erl -name myapp@192.168.1.100

# กำหนด cookie
erl -name myapp@host -setcookie mycookie

# หรือใน ~/.erlang.cookie
# mycookie

# vm.args
# -name myapp@host
# -setcookie mycookie
# +K true
# +A 64
```

### เชื่อมต่อ nodes

```erlang
%% เชื่อมต่อ
net_kernel:connect_node('other@host').
%% true | false | ignored

%% ตรวจสอบ connectivity
net_adm:ping('other@host').
%% pong | pang

%% Node info
node().           %% 'myapp@host'
nodes().          %% ['other@host', ...]
nodes(connected). %% connected nodes
nodes(hidden).    %% hidden nodes

%% Disconnect
erlang:disconnect_node('other@host').
```

---

## 3. Node Operations

```erlang
%% เรียกใช้ code บน node อื่น
rpc:call('other@host', Module, Function, Args).
rpc:call('other@host', io, format, ["Hello from other node~n", []]).

%% Async call
rpc:cast('other@host', Module, Function, Args).
%% ไม่รอ return

%% Async call กับ callback
rpc:async_call('other@host', Module, Function, Args).
%% คืน Key
rpc:yield(Key).   %% block จน result พร้อม
rpc:nb_yield(Key). %% non-blocking

%% Multi-call (ทุก nodes)
{Replies, BadNodes} = rpc:multicall(
    nodes(),
    erlang, node, []
).

%% Parallel call
Calls = [{'node1@h', {M, F, A}}, {'node2@h', {M, F, A}}],
%% รัน ด้วย gen_server:multi_call หรือ custom
```

---

## 4. Distributed Message Passing

```erlang
%% ส่ง message ข้าม nodes — transparent!

%% ด้วย PID (remote PID)
RemotePid ! message.
%% Erlang จัดการ serialization/network ให้อัตโนมัติ

%% ด้วย registered name + node
{registered_name, 'other@host'} ! message.

%% spawn บน node อื่น
RemotePid = spawn('other@host', fun() -> do_work() end).
RemotePid = spawn('other@host', Module, Function, Args).

%% spawn_link ข้าม nodes
RemotePid = spawn_link('other@host', fun() -> do_work() end).

%% monitor ข้าม nodes
Ref = monitor(process, RemotePid).
%% หรือ
Ref = erlang:monitor(process, {RegName, 'other@host'}).
```

---

## 5. Remote Process Management

```erlang
%% erlang:processes() บน node อื่น
rpc:call('other@host', erlang, processes, []).

%% process_info บน remote process
rpc:call('other@host', erlang, process_info, [RemotePid]).

%% Exit remote process
exit(RemotePid, reason).  %% ทำงานข้าม nodes!

%% Group leader: redirect I/O ไปยัง node ที่สั่ง
%% ใช้สำหรับ interactive debugging

%% node/1 — ดูว่า PID อยู่ node ไหน
node(RemotePid).  %% 'other@host'
node().           %% local node name

%% ตรวจสอบว่า PID เป็น local หรือ remote
is_local(Pid) ->
    node(Pid) =:= node().
```

---

## 6. Global Module

```erlang
%% global: global name registry ข้าม cluster

%% Register name globally
global:register_name(my_server, self()).
%% ok | {error, no_agree} | {error, already_registered}

%% Unregister
global:unregister_name(my_server).

%% Lookup
global:whereis_name(my_server).
%% Pid | undefined

%% Send message to globally registered process
global:send(my_server, message).

%% Sync global names
global:sync().

%% ใช้กับ gen_server
gen_server:start_link({global, my_server}, ?MODULE, [], []).
gen_server:call({global, my_server}, request).

%% Conflict resolution callback (ถ้า 2 nodes register พร้อมกัน)
my_resolve(Name, Pid1, Pid2) ->
    %% เลือก Pid1 หรือ Pid2 หรือ none
    %% ฝั่ง loser จะได้รับ message: {'EXIT', winner, kill}
    Pid1.

global:register_name(my_server, self(), fun my_resolve/3).
```

---

## 7. net_kernel Module

```erlang
%% net_kernel: จัดการ distribution layer

%% เริ่ม distribution dynamically (จาก non-distributed node)
net_kernel:start(['mynode@host', longnames]).
net_kernel:start(['mynode', shortnames]).

%% หยุด distribution
net_kernel:stop().

%% Monitor nodes
net_kernel:monitor_nodes(true).
%% ตอนนี้จะได้รับ:
%% {nodeup, 'other@host'}
%% {nodedown, 'other@host'}

net_kernel:monitor_nodes(true, [nodedown_reason]).
%% {nodedown, 'other@host', [{nodedown_reason, connection_setup_failed}]}

%% Hidden nodes (ไม่แสดงใน nodes())
net_kernel:connect_node('hidden@host').

%% Bandwidth/latency
net_adm:ping('other@host').   %% ทดสอบ connection
```

---

## 8. Distributed ETS

```erlang
%% ETS ไม่ distributed โดยตรง
%% แต่ทำได้ด้วย:

%% 1. Replicate ผ่าน message passing
replicate_insert(Nodes, Tab, Object) ->
    ets:insert(Tab, Object),
    lists:foreach(fun(Node) ->
        rpc:cast(Node, ets, insert, [Tab, Object])
    end, Nodes).

%% 2. Master node เป็น owner, others query ผ่าน RPC
ets_get(MasterNode, Tab, Key) ->
    case node() of
        MasterNode -> ets:lookup(Tab, Key);
        _ -> rpc:call(MasterNode, ets, lookup, [Tab, Key])
    end.

%% 3. ใช้ Mnesia แทน (designed for distribution)

%% 4. ets:give_away — ส่ง ownership ไปยัง process อื่น
ets:give_away(Tab, NewOwner, GiftData).
%% NewOwner receives: {'ETS-TRANSFER', Tab, FromPid, GiftData}
```

---

## 9. Network Partitions

```erlang
%% Network Partition: nodes บางส่วนไม่สามารถ communicate กัน

%% Monitor nodes เพื่อตรวจจับ
handle_info({nodedown, Node}, State) ->
    logger:warning("Node ~p down", [Node]),
    handle_node_failure(Node, State);

handle_info({nodeup, Node}, State) ->
    logger:info("Node ~p up", [Node]),
    handle_node_recovery(Node, State).

%% Strategies:
%% 1. CP (Consistency + Partition tolerance): หยุดทำงานถ้า partition
%% 2. AP (Availability + Partition tolerance): ทำต่อแต่อาจ inconsistent

%% Quorum: ต้องการ majority ก่อน proceed
has_quorum(TotalNodes) ->
    ConnectedNodes = length(nodes()) + 1,  %% +1 สำหรับ self
    ConnectedNodes > TotalNodes div 2.

%% Rejoin cluster
rejoin_cluster(Node) ->
    case net_kernel:connect_node(Node) of
        true ->
            sync_state_after_rejoin(),
            ok;
        false ->
            {error, cannot_connect}
    end.
```

---

## 10. แบบฝึกหัด

### Exercise: Distributed Counter

```erlang
%% Distributed counter ที่ sync ระหว่าง nodes

-module(dist_counter).
-behaviour(gen_server).
-export([start_link/0, increment/0, get/0]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2, terminate/2]).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

increment() ->
    gen_server:cast({global, ?MODULE}, {increment, node()}).

get() ->
    gen_server:call({global, ?MODULE}, get).

init([]) ->
    net_kernel:monitor_nodes(true),
    global:register_name(?MODULE, self()),
    {ok, #{count => 0, nodes => [node()]}}.

handle_call(get, _From, #{count := C} = State) ->
    {reply, C, State}.

handle_cast({increment, _Node}, #{count := C} = State) ->
    NewState = State#{count => C + 1},
    broadcast_state(NewState),
    {noreply, NewState}.

handle_info({sync, #{count := C}}, State) ->
    %% receive sync from other node
    {noreply, State#{count => max(maps:get(count, State), C)}};

handle_info({nodeup, Node}, #{nodes := Ns} = State) ->
    logger:info("Node ~p joined", [Node]),
    {noreply, State#{nodes => [Node | Ns]}};

handle_info({nodedown, Node}, #{nodes := Ns} = State) ->
    logger:warning("Node ~p left", [Node]),
    {noreply, State#{nodes => lists:delete(Node, Ns)}}.

terminate(_Reason, _State) ->
    global:unregister_name(?MODULE),
    ok.

broadcast_state(State) ->
    lists:foreach(fun(Node) ->
        {?MODULE, Node} ! {sync, State}
    end, nodes()).
```

---

## สรุป Part 18

✅ Distributed Erlang concepts  
✅ Node setup: short name, long name, cookie  
✅ Node operations: connect, ping, disconnect  
✅ Remote code execution ด้วย rpc  
✅ Transparent message passing ข้าม nodes  
✅ Global name registry  
✅ net_kernel: monitor_nodes  
✅ Distributed ETS strategies  
✅ Network partitions handling

---

*Part 18/100 | [← ก่อนหน้า](../part17/README.md) | [ถัดไป →](../part19/README.md)*
