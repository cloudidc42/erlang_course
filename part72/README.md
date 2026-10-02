# Part 72: Advanced Distributed Consensus

> **"In distributed systems, consensus is the art of agreeing to disagree — then agreeing"**  
> ในระบบกระจาย ความเห็นพ้องคือศิลปะของการยอมรับความขัดแย้ง แล้วก็ตกลงกัน

---

## สารบัญ

1. [Raft Consensus Basics](#1-raft-consensus-basics)
2. [Leader Election](#2-leader-election)
3. [Log Replication](#3-log-replication)
4. [Distributed Key-Value Store](#4-distributed-key-value-store)
5. [Cluster Membership Changes](#5-cluster-membership-changes)
6. [แบบฝึกหัด](#6-แบบฝึกหัด)

---

## 1. Raft Consensus Basics

```
Raft Consensus Algorithm Overview
═══════════════════════════════════════════════════════════════

Roles:
  FOLLOWER  — default state; receives heartbeats from leader
  CANDIDATE — seeking election votes
  LEADER    — handles all client writes; replicates to followers

Key Properties:
  • Leader election: first server to timeout becomes candidate
  • Log replication: leader appends → replicates → commits
  • Safety: at most one leader per term
  • Liveness: a new leader is always elected if quorum available

Terms:
  • Monotonically increasing logical clock
  • Each election starts a new term
  • A server rejects messages from older terms

Quorum:
  • Cluster of N servers needs (N/2)+1 for decisions
  • 3 servers: need 2 votes (tolerates 1 failure)
  • 5 servers: need 3 votes (tolerates 2 failures)

Log Entry:
  { index, term, command }
  Committed = replicated to quorum of nodes
```

---

## 2. Leader Election

```erlang
%% raft_server.erl — Raft consensus implementation
-module(raft_server).
-behaviour(gen_statem).

-export([start_link/2, submit/2]).
-export([callback_mode/0, init/1]).
-export([follower/3, candidate/3, leader/3]).

-define(ELECTION_TIMEOUT_MIN, 150).
-define(ELECTION_TIMEOUT_MAX, 300).
-define(HEARTBEAT_INTERVAL,   50).

-record(data, {
    %% Persistent state (would be written to disk)
    current_term = 0 :: integer(),
    voted_for    = undefined :: atom() | undefined,
    log          = [] :: list(),

    %% Volatile state
    commit_index = 0 :: integer(),
    last_applied = 0 :: integer(),

    %% Leader-only volatile state
    next_index   = #{} :: map(),
    match_index  = #{} :: map(),

    %% Node configuration
    node_id   :: atom(),
    peers     :: [atom()],
    votes     = #{} :: map(),

    %% State machine
    state_machine :: module(),
    sm_state      :: term()
}).

start_link(NodeId, Peers) ->
    gen_statem:start_link({local, NodeId}, ?MODULE,
                          [NodeId, Peers], []).

submit(Node, Command) ->
    gen_statem:call(Node, {submit, Command}, 5000).

callback_mode() -> [state_functions, state_enter].

init([NodeId, Peers]) ->
    Data = #data{
        node_id       = NodeId,
        peers         = Peers,
        state_machine = raft_kv_sm,
        sm_state      = raft_kv_sm:init()
    },
    {ok, follower, Data, [{state_timeout, election_timeout(), election}]}.

election_timeout() ->
    ?ELECTION_TIMEOUT_MIN + rand:uniform(
        ?ELECTION_TIMEOUT_MAX - ?ELECTION_TIMEOUT_MIN
    ).

%% ─────────────────────────────────────────────
%%  FOLLOWER
%% ─────────────────────────────────────────────
follower(enter, _OldState, Data) ->
    {keep_state, Data, [{state_timeout, election_timeout(), election}]};

follower(state_timeout, election, Data) ->
    %% Didn't hear from leader, start election
    {next_state, candidate, Data};

follower(cast, {append_entries, Term, LeaderId, PrevIdx, PrevTerm,
                Entries, LeaderCommit}, Data) ->
    case Term >= Data#data.current_term of
        false ->
            send_to(LeaderId, {append_entries_reply, Data#data.node_id,
                               Data#data.current_term, false, 0}),
            {keep_state, Data};
        true ->
            %% Accept entries, reset election timer
            NewData = accept_entries(Data#data{current_term = Term},
                                     PrevIdx, PrevTerm, Entries, LeaderCommit),
            send_to(LeaderId, {append_entries_reply,
                               Data#data.node_id, Term, true,
                               length(NewData#data.log)}),
            {keep_state, NewData,
             [{state_timeout, election_timeout(), election}]}
    end;

follower(cast, {request_vote, Term, CandidateId, LastLogIdx, LastLogTerm}, Data) ->
    {GrantVote, NewData} = evaluate_vote(Data, Term, CandidateId,
                                         LastLogIdx, LastLogTerm),
    send_to(CandidateId, {vote_reply, Data#data.node_id, Term, GrantVote}),
    {keep_state, NewData};

follower({call, From}, {submit, _Cmd}, Data) ->
    %% Redirect to leader — for simplicity, return error
    {keep_state, Data, [{reply, From, {error, not_leader}}]}.

%% ─────────────────────────────────────────────
%%  CANDIDATE
%% ─────────────────────────────────────────────
candidate(enter, _OldState, Data) ->
    NewTerm = Data#data.current_term + 1,
    NewData = Data#data{
        current_term = NewTerm,
        voted_for    = Data#data.node_id,
        votes        = #{Data#data.node_id => true}
    },
    %% Request votes from all peers
    {LastIdx, LastTerm} = last_log_idx_term(NewData),
    lists:foreach(fun(Peer) ->
        send_to(Peer, {request_vote, NewTerm, NewData#data.node_id,
                       LastIdx, LastTerm})
    end, NewData#data.peers),
    {keep_state, NewData, [{state_timeout, election_timeout(), election}]};

candidate(state_timeout, election, Data) ->
    %% No quorum reached, start new election
    {repeat_state, Data};

candidate(cast, {vote_reply, _FromNode, Term, true}, Data) ->
    NewVotes = maps:put(node_id_placeholder, true, Data#data.votes),
    Quorum = (length(Data#data.peers) + 1) div 2 + 1,
    case map_size(NewVotes) >= Quorum of
        true  -> {next_state, leader, Data#data{votes = NewVotes}};
        false -> {keep_state, Data#data{votes = NewVotes}}
    end;

candidate(cast, {vote_reply, _FromNode, _Term, false}, Data) ->
    {keep_state, Data};

candidate(cast, {append_entries, Term, _LeaderId, _, _, _, _}, Data)
  when Term >= Data#data.current_term ->
    %% New leader emerged, become follower
    {next_state, follower, Data#data{current_term = Term}};

candidate({call, From}, {submit, _}, Data) ->
    {keep_state, Data, [{reply, From, {error, election_in_progress}}]}.

%% ─────────────────────────────────────────────
%%  LEADER
%% ─────────────────────────────────────────────
leader(enter, _OldState, Data) ->
    %% Initialize next_index for each follower
    NextIdx = length(Data#data.log) + 1,
    NextIndexMap  = maps:from_list([{P, NextIdx} || P <- Data#data.peers]),
    MatchIndexMap = maps:from_list([{P, 0}       || P <- Data#data.peers]),
    NewData = Data#data{next_index  = NextIndexMap,
                        match_index = MatchIndexMap},
    %% Send immediate heartbeat
    send_heartbeats(NewData),
    {keep_state, NewData, [{state_timeout, ?HEARTBEAT_INTERVAL, heartbeat}]};

leader(state_timeout, heartbeat, Data) ->
    send_heartbeats(Data),
    {keep_state, Data, [{state_timeout, ?HEARTBEAT_INTERVAL, heartbeat}]};

leader({call, From}, {submit, Command}, Data) ->
    %% Append to local log
    Entry    = {length(Data#data.log) + 1, Data#data.current_term, Command},
    NewLog   = Data#data.log ++ [Entry],
    NewData  = Data#data{log = NewLog},
    %% Replicate to followers (simplified: immediate, no waiting)
    replicate_log(NewData),
    {keep_state, NewData, [{reply, From, {ok, length(NewLog)}}]};

leader(cast, {append_entries_reply, _From, _Term, true, MatchIdx}, Data) ->
    %% Update match index, check if we can commit
    NewData = try_commit(Data, MatchIdx),
    {keep_state, NewData};

leader(cast, {append_entries_reply, _From, _Term, false, _}, Data) ->
    %% Decrease next_index for this follower (simplified)
    {keep_state, Data}.

%% ─────────────────────────────────────────────
%%  Helpers
%% ─────────────────────────────────────────────
send_to(Node, Msg) ->
    gen_statem:cast(Node, Msg).

send_heartbeats(Data) ->
    {PrevIdx, PrevTerm} = last_log_idx_term(Data),
    lists:foreach(fun(Peer) ->
        send_to(Peer, {append_entries,
                       Data#data.current_term,
                       Data#data.node_id,
                       PrevIdx, PrevTerm,
                       [],   % no entries = heartbeat
                       Data#data.commit_index})
    end, Data#data.peers).

replicate_log(Data) ->
    {PrevIdx, PrevTerm} = prev_log_idx_term(Data),
    LastEntry = lists:last(Data#data.log),
    lists:foreach(fun(Peer) ->
        send_to(Peer, {append_entries,
                       Data#data.current_term,
                       Data#data.node_id,
                       PrevIdx, PrevTerm,
                       [LastEntry],
                       Data#data.commit_index})
    end, Data#data.peers).

accept_entries(Data, _PrevIdx, _PrevTerm, [], LeaderCommit) ->
    CommitIdx = min(LeaderCommit, length(Data#data.log)),
    apply_committed(Data#data{commit_index = CommitIdx});
accept_entries(Data, _PrevIdx, _PrevTerm, Entries, LeaderCommit) ->
    NewLog = Data#data.log ++ Entries,
    CommitIdx = min(LeaderCommit, length(NewLog)),
    apply_committed(Data#data{log = NewLog, commit_index = CommitIdx}).

apply_committed(Data) ->
    case Data#data.commit_index > Data#data.last_applied of
        false -> Data;
        true  ->
            Entry = lists:nth(Data#data.last_applied + 1, Data#data.log),
            {_, _, Command} = Entry,
            NewSmState = Data#data.state_machine:apply(Command,
                                                       Data#data.sm_state),
            apply_committed(Data#data{
                last_applied  = Data#data.last_applied + 1,
                sm_state      = NewSmState
            })
    end.

try_commit(Data, MatchIdx) ->
    %% If majority have replicated up to MatchIdx, commit
    Quorum = (length(Data#data.peers) + 1) div 2 + 1,
    Counts = length([1 || P <- Data#data.peers,
                          maps:get(P, Data#data.match_index, 0) >= MatchIdx]),
    case Counts + 1 >= Quorum of  % +1 for self
        true  -> apply_committed(Data#data{commit_index = MatchIdx});
        false -> Data
    end.

evaluate_vote(Data, Term, CandidateId, _LastLogIdx, _LastLogTerm) ->
    case Term > Data#data.current_term of
        true ->
            %% Update term, grant vote
            NewData = Data#data{
                current_term = Term,
                voted_for    = CandidateId
            },
            {true, NewData};
        false ->
            case Data#data.voted_for of
                undefined ->
                    {true, Data#data{voted_for = CandidateId}};
                CandidateId ->
                    {true, Data};
                _ ->
                    {false, Data}
            end
    end.

last_log_idx_term(#data{log = []})  -> {0, 0};
last_log_idx_term(#data{log = Log}) ->
    {Idx, Term, _} = lists:last(Log),
    {Idx, Term}.

prev_log_idx_term(#data{log = []})     -> {0, 0};
prev_log_idx_term(#data{log = [_|_] = Log}) ->
    case length(Log) > 1 of
        true  ->
            {Idx, Term, _} = lists:nth(length(Log) - 1, Log),
            {Idx, Term};
        false -> {0, 0}
    end.

node_id_placeholder -> ok.
```

---

## 3. Log Replication

```erlang
%% raft_kv_sm.erl — key-value state machine applied by Raft
-module(raft_kv_sm).
-export([init/0, apply/2, read/2]).

init() -> #{}.

apply({put, Key, Value}, State) ->
    maps:put(Key, Value, State);
apply({delete, Key}, State) ->
    maps:remove(Key, State);
apply({compare_and_swap, Key, Expected, New}, State) ->
    case maps:get(Key, State, undefined) of
        Expected -> maps:put(Key, New, State);
        _        -> State  % No change if value doesn't match
    end.

read(Key, State) ->
    maps:get(Key, State, undefined).
```

---

## 4. Distributed Key-Value Store

```erlang
%% raft_kv.erl — client API for Raft-backed KV store
-module(raft_kv).
-export([start_cluster/1, put/3, get/2, delete/2]).

start_cluster(Nodes) ->
    %% Start Raft on each node with all others as peers
    lists:foreach(fun(Node) ->
        Peers = Nodes -- [Node],
        raft_server:start_link(Node, Peers)
    end, Nodes),
    %% Wait for leader election
    timer:sleep(500),
    ok.

put(Cluster, Key, Value) ->
    submit_to_leader(Cluster, {put, Key, Value}).

get(Cluster, Key) ->
    %% Read from leader for consistency
    Leader = find_leader(Cluster),
    raft_kv_sm:read(Key, get_sm_state(Leader)).

delete(Cluster, Key) ->
    submit_to_leader(Cluster, {delete, Key}).

submit_to_leader(Cluster, Command) ->
    Leader = find_leader(Cluster),
    raft_server:submit(Leader, Command).

find_leader(_Cluster) ->
    %% In practice: ping each node, find the one that accepts writes
    node1.  % simplified

get_sm_state(_Node) ->
    #{}.  % simplified

%% Usage:
%%   Nodes = [node1, node2, node3],
%%   raft_kv:start_cluster(Nodes),
%%   raft_kv:put(Nodes, <<"key">>, <<"value">>),
%%   raft_kv:get(Nodes, <<"key">>).
```

---

## 5. Cluster Membership Changes

```erlang
%% raft_membership.erl — joint consensus for cluster membership changes
-module(raft_membership).

%% Phase 1: Enter joint consensus (both old and new config must agree)
enter_joint_consensus(OldNodes, NewNodes) ->
    JointConfig = #{
        old_nodes => OldNodes,
        new_nodes => NewNodes,
        phase     => joint
    },
    %% Commit this config change as a log entry
    {joint_consensus, JointConfig}.

%% Phase 2: Commit new config
commit_new_config(JointConfig) ->
    NewNodes = maps:get(new_nodes, JointConfig),
    %% Log entry that finalizes the new cluster config
    {config_change, #{nodes => NewNodes, phase => final}}.

%% Quorum calculation during joint consensus
quorum_joint(Votes, #{old_nodes := Old, new_nodes := New}) ->
    %% Need quorum from BOTH old and new configurations
    OldQuorum = quorum(Votes, Old),
    NewQuorum = quorum(Votes, New),
    OldQuorum andalso NewQuorum.

quorum(Votes, Nodes) ->
    Count = length([N || N <- Nodes, maps:get(N, Votes, false)]),
    Count > length(Nodes) div 2.

%% Safe node removal: never remove nodes below quorum threshold
safe_to_remove(NodeToRemove, CurrentNodes) ->
    RemoveCount = length(CurrentNodes -- [NodeToRemove]),
    RemoveCount > length(CurrentNodes) div 2.
```

---

## 6. แบบฝึกหัด

1. เพิ่ม persistent log storage ใน `raft_server.erl` โดยเขียน entries ลงไฟล์
2. Implement log compaction (snapshotting) เพื่อลด memory usage
3. เพิ่ม read index optimization สำหรับ linearizable reads
4. สร้าง test cluster ด้วย 5 nodes และทดสอบ leader failover

---

## สรุป Part 72

✅ Raft algorithm: roles, terms, quorum fundamentals  
✅ Leader election: candidate state, vote requesting/granting  
✅ Log replication: append entries, commit when quorum confirms  
✅ Key-value state machine: apply commands to persistent state  
✅ Cluster membership: joint consensus for safe node add/remove  

---

*Part 72/100 | [← ก่อนหน้า](../part71/README.md) | [ถัดไป →](../part73/README.md)*
