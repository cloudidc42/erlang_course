# Part 41: Advanced Distributed Systems

> **"A distributed system is one in which the failure of a computer you didn't even know existed can render your own computer unusable"**  
> — Leslie Lamport

---

## สารบัญ

1. [Consensus Algorithms](#1-consensus-algorithms)
2. [CRDTs — Conflict-free Replicated Data Types](#2-crdts)
3. [Vector Clocks](#3-vector-clocks)
4. [Gossip Protocol](#4-gossip-protocol)
5. [Consistent Hashing](#5-consistent-hashing)
6. [Leader Election](#6-leader-election)
7. [แบบฝึกหัด](#7-แบบฝึกหัด)

---

## 1. Consensus Algorithms

```erlang
%% raft_node.erl — simplified Raft consensus
-module(raft_node).
-behaviour(gen_statem).
-export([start_link/2, append_entry/2]).
-export([callback_mode/0, init/1,
         follower/3, candidate/3, leader/3]).

-record(data, {
    id,
    peers     = [],
    term      = 0,
    voted_for = undefined,
    log       = [],
    commit    = 0,
    votes     = 0
}).

callback_mode() -> state_functions.

start_link(Id, Peers) ->
    gen_statem:start_link({local, Id}, ?MODULE, {Id, Peers}, []).

append_entry(Node, Entry) ->
    gen_statem:call(Node, {append, Entry}).

init({Id, Peers}) ->
    {ok, follower, #data{id=Id, peers=Peers},
     [{state_timeout, election_timeout(), start_election}]}.

%% Follower state
follower({call, From}, {append, _Entry}, Data) ->
    %% Forward to leader if we know it
    {keep_state, Data, [{reply, From, {error, not_leader}}]};

follower(state_timeout, start_election, Data) ->
    %% Start an election
    NewTerm = Data#data.term + 1,
    NewData = Data#data{term=NewTerm, voted_for=Data#data.id, votes=1},
    [send_vote_request(P, Data#data.id, NewTerm) || P <- Data#data.peers],
    {next_state, candidate, NewData,
     [{state_timeout, election_timeout(), election_timeout}]};

follower(info, {request_vote, Term, CandidateId}, Data)
        when Term > Data#data.term ->
    send_vote(CandidateId, Term, true),
    {keep_state, Data#data{term=Term, voted_for=CandidateId},
     [{state_timeout, election_timeout(), start_election}]};

follower(info, {heartbeat, Term, LeaderId}, Data) when Term >= Data#data.term ->
    {keep_state, Data#data{term=Term},
     [{state_timeout, election_timeout(), start_election}]};

follower(EventType, Event, Data) ->
    handle_common(EventType, Event, Data, follower).

%% Candidate state
candidate(info, {vote_granted, Term}, Data)
        when Term =:= Data#data.term ->
    NewVotes = Data#data.votes + 1,
    Majority = (length(Data#data.peers) + 2) div 2,
    if NewVotes >= Majority ->
        send_heartbeats(Data#data.peers, Data#data.id, Data#data.term),
        {next_state, leader, Data#data{votes=NewVotes},
         [{state_timeout, 50, heartbeat}]};
    true ->
        {keep_state, Data#data{votes=NewVotes}}
    end;

candidate(state_timeout, election_timeout, Data) ->
    %% Start new election
    follower(state_timeout, start_election, Data);

candidate(EventType, Event, Data) ->
    handle_common(EventType, Event, Data, candidate).

%% Leader state
leader({call, From}, {append, Entry}, Data) ->
    NewLog = Data#data.log ++ [Entry],
    %% In real Raft: replicate to followers, then commit
    {keep_state, Data#data{log=NewLog}, [{reply, From, ok}]};

leader(state_timeout, heartbeat, Data) ->
    send_heartbeats(Data#data.peers, Data#data.id, Data#data.term),
    {keep_state, Data, [{state_timeout, 50, heartbeat}]};

leader(EventType, Event, Data) ->
    handle_common(EventType, Event, Data, leader).

handle_common(_Type, _Event, Data, _State) -> {keep_state, Data}.

election_timeout() -> 150 + rand:uniform(150).

send_vote_request(Peer, Id, Term) ->
    catch Peer ! {request_vote, Term, Id}.
send_vote(Candidate, Term, Granted) ->
    catch Candidate ! {vote_granted, Term, Granted}.
send_heartbeats(Peers, Id, Term) ->
    [catch P ! {heartbeat, Term, Id} || P <- Peers].
```

---

## 2. CRDTs

```erlang
%% crdt.erl — G-Counter (Grow-only Counter) CRDT
-module(crdt).
-export([new_gcounter/1, increment/2, value/1, merge/2]).
-export([new_pncounter/1, decrement/2]).

%% G-Counter: each node has its own counter slot
new_gcounter(NodeId) ->
    #{node_id => NodeId, counters => #{NodeId => 0}}.

increment(#{node_id := Me, counters := C} = State, By) ->
    State#{counters := maps:update_with(Me, fun(V) -> V + By end, By, C)}.

value(#{counters := C}) ->
    maps:fold(fun(_, V, Acc) -> Acc + V end, 0, C).

merge(#{counters := C1} = S1, #{counters := C2}) ->
    AllKeys = maps:keys(C1) ++ maps:keys(C2),
    Merged  = lists:foldl(fun(K, Acc) ->
        V = max(maps:get(K, C1, 0), maps:get(K, C2, 0)),
        maps:put(K, V, Acc)
    end, #{}, AllKeys),
    S1#{counters := Merged}.

%% PN-Counter: P (positive) + N (negative) G-counters
new_pncounter(NodeId) ->
    #{node_id => NodeId,
      p => new_gcounter(NodeId),
      n => new_gcounter(NodeId)}.

decrement(#{n := N} = State, By) ->
    State#{n := increment(N, By)}.

%% value for PN-Counter
%% value(#{p := P, n := N}) -> value(P) - value(N).

%% OR-Set (Observed-Remove Set)
new_orset() -> #{items => #{}}.

add_orset(#{items := Items}, Element) ->
    Tag = erlang:unique_integer([positive]),
    #{items := maps:update_with(Element,
        fun(Tags) -> sets:add_element(Tag, Tags) end,
        sets:from_list([Tag]), Items)}.

remove_orset(#{items := Items}, Element) ->
    #{items := maps:remove(Element, Items)}.

merge_orset(#{items := A}, #{items := B}) ->
    AllKeys = lists:usort(maps:keys(A) ++ maps:keys(B)),
    Merged  = lists:foldl(fun(K, Acc) ->
        TA  = maps:get(K, A, sets:new()),
        TB  = maps:get(K, B, sets:new()),
        Combined = sets:union(TA, TB),
        case sets:size(Combined) > 0 of
            true  -> maps:put(K, Combined, Acc);
            false -> Acc
        end
    end, #{}, AllKeys),
    #{items => Merged}.

members_orset(#{items := Items}) -> maps:keys(Items).
```

---

## 3. Vector Clocks

```erlang
%% vclock.erl — vector clocks for causality tracking
-module(vclock).
-export([new/0, increment/2, compare/2, merge/2, descends/2]).

new() -> #{}.

increment(Clock, NodeId) ->
    maps:update_with(NodeId, fun(V) -> V + 1 end, 1, Clock).

%% Returns: equal | before | after | concurrent
compare(C1, C2) ->
    AllKeys  = lists:usort(maps:keys(C1) ++ maps:keys(C2)),
    {Before, After} = lists:foldl(fun(K, {B, A}) ->
        V1 = maps:get(K, C1, 0),
        V2 = maps:get(K, C2, 0),
        {B or (V1 < V2), A or (V1 > V2)}
    end, {false, false}, AllKeys),
    case {Before, After} of
        {false, false} -> equal;
        {true,  false} -> before;
        {false, true}  -> after;
        {true,  true}  -> concurrent
    end.

merge(C1, C2) ->
    AllKeys = lists:usort(maps:keys(C1) ++ maps:keys(C2)),
    lists:foldl(fun(K, Acc) ->
        V = max(maps:get(K, C1, 0), maps:get(K, C2, 0)),
        maps:put(K, V, Acc)
    end, #{}, AllKeys).

descends(C1, C2) ->
    compare(C1, C2) =:= after.
```

---

## 4. Gossip Protocol

```erlang
%% gossip.erl — epidemic/gossip protocol for state dissemination
-module(gossip).
-behaviour(gen_server).
-export([start_link/2, update/2, get_state/0]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

-define(GOSSIP_INTERVAL, 1000).
-define(FANOUT, 3).

-record(state, {
    node_id,
    peers  = [],
    data   = #{},   %% Key => {Value, VClock}
    vclock = #{}
}).

start_link(NodeId, Peers) ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, {NodeId, Peers}, []).

update(Key, Value) ->
    gen_server:call(?MODULE, {update, Key, Value}).

get_state() ->
    gen_server:call(?MODULE, get_state).

init({NodeId, Peers}) ->
    erlang:send_after(?GOSSIP_INTERVAL, self(), gossip),
    {ok, #state{node_id=NodeId, peers=Peers}}.

handle_call({update, Key, Value}, _From, #state{node_id=Me, vclock=VC, data=D} = S) ->
    NewVC   = vclock:increment(VC, Me),
    NewData = maps:put(Key, {Value, NewVC}, D),
    {reply, ok, S#state{vclock=NewVC, data=NewData}};

handle_call(get_state, _From, #state{data=D} = S) ->
    {reply, D, S}.

handle_info(gossip, #state{peers=Peers, data=Data} = S) ->
    erlang:send_after(?GOSSIP_INTERVAL, self(), gossip),
    ToContact = choose_peers(Peers, ?FANOUT),
    [send_digest(P, Data) || P <- ToContact],
    {noreply, S};

handle_info({gossip_digest, RemoteData}, #state{data=LocalData} = S) ->
    Merged = merge_data(LocalData, RemoteData),
    {noreply, S#state{data=Merged}};

handle_info(_, S) -> {noreply, S}.
handle_cast(_, S) -> {noreply, S}.

choose_peers(Peers, N) ->
    Shuffled = [X || {_, X} <- lists:sort([{rand:uniform(), P} || P <- Peers])],
    lists:sublist(Shuffled, N).

send_digest(Peer, Data) ->
    catch Peer ! {gossip_digest, Data}.

merge_data(Local, Remote) ->
    maps:fold(fun(K, {RV, RVC}, Acc) ->
        case maps:find(K, Acc) of
            {ok, {_LV, LVC}} ->
                case vclock:compare(RVC, LVC) of
                    after  -> maps:put(K, {RV, RVC}, Acc);
                    _      -> Acc
                end;
            error ->
                maps:put(K, {RV, RVC}, Acc)
        end
    end, Local, Remote).
```

---

## 5. Consistent Hashing

```erlang
%% consistent_hash.erl — consistent hashing ring
-module(consistent_hash).
-export([new/1, add_node/2, remove_node/2, get_node/2]).

-define(VNODES, 150).

new(Nodes) ->
    Ring = lists:foldl(fun(N, R) -> add_node(R, N) end, #{}, Nodes),
    Ring.

add_node(Ring, NodeId) ->
    lists:foldl(fun(VN, R) ->
        Hash = hash(<<NodeId/binary, VN:32>>),
        maps:put(Hash, NodeId, R)
    end, Ring, lists:seq(0, ?VNODES - 1)).

remove_node(Ring, NodeId) ->
    maps:filter(fun(_Hash, N) -> N =/= NodeId end, Ring).

get_node(Ring, Key) ->
    Hash = hash(Key),
    Sorted = lists:sort(maps:keys(Ring)),
    case find_successor(Hash, Sorted) of
        not_found  -> maps:get(hd(Sorted), Ring);  %% wrap around
        SuccHash   -> maps:get(SuccHash, Ring)
    end.

find_successor(Hash, []) -> not_found;
find_successor(Hash, [H | _]) when H >= Hash -> H;
find_successor(Hash, [_ | Rest]) -> find_successor(Hash, Rest).

hash(Key) ->
    <<H:32, _/binary>> = crypto:hash(sha256, Key),
    H.
```

---

## 6. Leader Election

```erlang
%% leader_election.erl — simple bully algorithm
-module(leader_election).
-behaviour(gen_server).
-export([start_link/2, get_leader/0, am_i_leader/0]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

-record(state, {id, peers, leader = undefined}).

start_link(Id, Peers) ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, {Id, Peers}, []).

get_leader()  -> gen_server:call(?MODULE, get_leader).
am_i_leader() -> gen_server:call(?MODULE, am_i_leader).

init({Id, Peers}) ->
    self() ! start_election,
    {ok, #state{id=Id, peers=Peers}}.

handle_info(start_election, #state{id=Id, peers=Peers} = S) ->
    HigherPeers = [P || P <- Peers, peer_id(P) > Id],
    case HigherPeers of
        [] ->
            %% I have the highest ID, declare myself leader
            announce_leader(Peers, Id),
            {noreply, S#state{leader=Id}};
        _ ->
            [P ! {election, Id} || P <- HigherPeers],
            erlang:send_after(3000, self(), election_timeout),
            {noreply, S}
    end;

handle_info({election, CandidateId}, #state{id=Id, peers=Peers} = S) ->
    case Id > CandidateId of
        true ->
            %% I'm higher, start my own election
            self() ! start_election,
            {noreply, S};
        false ->
            %% Let higher ID win
            {noreply, S}
    end;

handle_info({leader, LeaderId}, S) ->
    {noreply, S#state{leader=LeaderId}};

handle_info(election_timeout, #state{id=Id, peers=Peers} = S) ->
    %% No response from higher nodes, declare myself leader
    announce_leader(Peers, Id),
    {noreply, S#state{leader=Id}};

handle_info(_, S) -> {noreply, S}.

handle_call(get_leader, _, S) -> {reply, S#state.leader, S};
handle_call(am_i_leader, _, S) -> {reply, S#state.leader =:= S#state.id, S}.
handle_cast(_, S) -> {noreply, S}.

announce_leader(Peers, LeaderId) ->
    [catch P ! {leader, LeaderId} || P <- Peers].

peer_id(Peer) -> Peer.  %% simplified: peer IS the id
```

---

## 7. แบบฝึกหัด

1. ทดสอบ G-Counter: สร้าง 3 nodes, increment แต่ละ node, แล้ว merge
2. เพิ่ม `LWW-Register` (Last-Write-Wins) CRDT โดยใช้ timestamp เป็น tiebreaker
3. ทดสอบ consistent hashing ด้วย 5 nodes และ 1000 keys — นับว่า distribute สม่ำเสมอแค่ไหน
4. เพิ่ม log replication ใน raft_node (replicate to majority before commit)

---

## สรุป Part 41

✅ Raft consensus algorithm  
✅ CRDTs: G-Counter, PN-Counter, OR-Set  
✅ Vector clocks สำหรับ causality  
✅ Gossip protocol  
✅ Consistent hashing ring  
✅ Bully leader election  

---

*Part 41/100 | [← ก่อนหน้า](../part40/README.md) | [ถัดไป →](../part42/README.md)*
