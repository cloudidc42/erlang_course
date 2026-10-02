# Part 52: Real-Time Gaming Backend

> **"Games demand microsecond consistency — Erlang was built for it"**  
> เกมต้องการ consistency ระดับ microsecond — Erlang ถูกสร้างมาสำหรับงานนี้

---

## สารบัญ

1. [Game Server Architecture](#1-game-server-architecture)
2. [Match Making](#2-match-making)
3. [Game Room Manager](#3-game-room-manager)
4. [Player State Sync](#4-player-state-sync)
5. [Leaderboard System](#5-leaderboard-system)
6. [Anti-Cheat](#6-anti-cheat)
7. [แบบฝึกหัด](#7-แบบฝึกหัด)

---

## 1. Game Server Architecture

```
Gaming Backend Architecture:

  [Game Client] — WebSocket/UDP
        |
  [Connection Manager]
        |
  [Matchmaking Queue]
        |
  [Game Room] (per-room GenServer)
        |
  [State Broadcaster] — push to all players
        |
  [Game State DB] — ETS + periodic Postgres
        |
  [Leaderboard] — sorted ETS + Redis
```

---

## 2. Match Making

```erlang
%% matchmaker.erl — skill-based matchmaking
-module(matchmaker).
-behaviour(gen_server).
-export([start_link/0, join_queue/2, leave_queue/1]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

-record(player, {id, skill, joined_at}).
-record(state, {queue = queue:new(), pending = #{}}).

-define(MATCH_SIZE, 2).
-define(SKILL_TOLERANCE, 200).
-define(WAIT_EXPAND_INTERVAL, 15_000).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

join_queue(PlayerId, Skill) ->
    gen_server:call(?MODULE, {join, PlayerId, Skill}).

leave_queue(PlayerId) ->
    gen_server:cast(?MODULE, {leave, PlayerId}).

init([]) ->
    erlang:send_after(1000, self(), try_match),
    {ok, #state{}}.

handle_call({join, PlayerId, Skill}, _From, #state{queue=Q, pending=P} = S) ->
    Player = #player{id=PlayerId, skill=Skill, joined_at=os:system_time(second)},
    NewQ   = queue:in(Player, Q),
    {reply, {ok, queued}, S#state{queue=NewQ, pending=maps:put(PlayerId, Player, P)}}.

handle_cast({leave, PlayerId}, #state{pending=P} = S) ->
    {noreply, S#state{pending=maps:remove(PlayerId, P)}}.

handle_info(try_match, #state{queue=Q} = S) ->
    erlang:send_after(1000, self(), try_match),
    {Matched, NewQ} = find_matches(queue:to_list(Q)),
    [game_room_sup:start_room(Players) || Players <- Matched],
    {noreply, S#state{queue=queue:from_list(NewQ)}}.

find_matches(Players) ->
    find_matches(Players, [], []).

find_matches([], Matched, Remaining) ->
    {Matched, Remaining};
find_matches([P | Rest], Matched, Remaining) ->
    Tolerance = skill_tolerance(P),
    Compatible = [Q || Q <- Rest,
                       abs(Q#player.skill - P#player.skill) =< Tolerance],
    case length(Compatible) >= ?MATCH_SIZE - 1 of
        true ->
            Group = [P | lists:sublist(Compatible, ?MATCH_SIZE - 1)],
            GroupIds = [G#player.id || G <- Group],
            Leftover = [X || X <- Rest, not lists:member(X#player.id, tl(GroupIds))],
            find_matches(Leftover, [Group | Matched], Remaining);
        false ->
            find_matches(Rest, Matched, [P | Remaining])
    end.

skill_tolerance(#player{joined_at=JoinedAt}) ->
    WaitSec = os:system_time(second) - JoinedAt,
    %% Expand tolerance the longer they wait
    ?SKILL_TOLERANCE + (WaitSec div 15) * 50.
```

---

## 3. Game Room Manager

```erlang
%% game_room.erl — manage game state for a room
-module(game_room).
-behaviour(gen_server).
-export([start_link/2, move/3, get_state/1, broadcast/2]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

-record(room, {
    id,
    players    = #{},   %% #{PlayerId => #{pid, pos, score, alive}}
    state      = lobby, %% lobby | playing | finished
    tick       = 0,
    start_time
}).

start_link(RoomId, Players) ->
    gen_server:start_link(?MODULE, {RoomId, Players}, []).

move(RoomPid, PlayerId, Direction) ->
    gen_server:cast(RoomPid, {move, PlayerId, Direction}).

get_state(RoomPid) ->
    gen_server:call(RoomPid, get_state).

broadcast(RoomPid, Message) ->
    gen_server:cast(RoomPid, {broadcast, Message}).

init({RoomId, Players}) ->
    PlayerMap = maps:from_list([
        {P#player.id, #{pid => P#player.id, pos => {0, 0},
                        score => 0, alive => true}}
        || P <- Players
    ]),
    erlang:send_after(100, self(), tick),  %% 10 ticks/second
    {ok, #room{id=RoomId, players=PlayerMap,
               state=playing, start_time=os:system_time(second)}}.

handle_info(tick, #room{state=playing, tick=T, players=Players} = R) ->
    erlang:send_after(100, self(), tick),
    NewTick = T + 1,
    %% Game logic tick
    NewPlayers = update_game_state(Players),
    %% Broadcast state to all players
    StateMsg = build_state_msg(NewPlayers, NewTick),
    [P ! {game_state, StateMsg} || #{pid := P} <- maps:values(NewPlayers)],
    %% Check win condition
    case check_winner(NewPlayers) of
        {winner, WinnerId} ->
            finish_game(R#room{players=NewPlayers, tick=NewTick}, WinnerId);
        no_winner ->
            {noreply, R#room{players=NewPlayers, tick=NewTick}}
    end;

handle_info(tick, R) -> {noreply, R}.

handle_cast({move, PlayerId, Direction}, #room{players=P} = R) ->
    NewPos = move_player(maps:get(PlayerId, P, #{}), Direction),
    NewP   = maps:update(PlayerId, NewPos, P),
    {noreply, R#room{players=NewP}};

handle_cast({broadcast, Msg}, #room{players=P} = R) ->
    [Pid ! Msg || #{pid := Pid} <- maps:values(P)],
    {noreply, R}.

handle_call(get_state, _, R) ->
    {reply, R, R}.

update_game_state(Players) -> Players.  %% game-specific logic
build_state_msg(P, T) -> #{tick => T, players => P}.
check_winner(_Players) -> no_winner.
finish_game(R, _Winner) -> {stop, normal, R}.

move_player(Player, {dx, DX}) ->
    {X, Y} = maps:get(pos, Player, {0, 0}),
    Player#{pos => {X + DX, Y}};
move_player(Player, {dy, DY}) ->
    {X, Y} = maps:get(pos, Player, {0, 0}),
    Player#{pos => {X, Y + DY}}.
```

---

## 4. Player State Sync

```erlang
%% game_ws_handler.erl — WebSocket for game clients
-module(game_ws_handler).
-behaviour(cowboy_websocket).
-export([init/2, websocket_init/1, websocket_handle/2,
         websocket_info/2, terminate/3]).

-record(ws, {player_id, room_pid}).

init(Req, State) ->
    PlayerId = cowboy_req:binding(player_id, Req),
    {cowboy_websocket, Req, #ws{player_id=PlayerId}}.

websocket_init(State) ->
    %% Find or join a room
    case matchmaker:join_queue(State#ws.player_id, 1000) of
        {ok, queued} -> {ok, State}
    end.

websocket_handle({binary, Data}, State) ->
    Msg = decode_game_msg(Data),
    handle_game_msg(Msg, State);

websocket_handle({text, Data}, State) ->
    Msg = jsx:decode(Data, [return_maps]),
    handle_game_msg(Msg, State).

websocket_info({room_assigned, RoomPid}, State) ->
    {reply, {binary, encode_msg(#{type => room_assigned})},
     State#ws{room_pid=RoomPid}};

websocket_info({game_state, State}, WsState) ->
    {reply, {binary, encode_msg(State)}, WsState};

websocket_info(_, State) -> {ok, State}.

terminate(_Reason, _Req, #ws{player_id=PId, room_pid=RoomPid}) ->
    matchmaker:leave_queue(PId),
    case RoomPid of
        undefined -> ok;
        _         -> game_room:broadcast(RoomPid, {player_left, PId})
    end.

handle_game_msg(#{<<"type">> := <<"move">>, <<"dir">> := Dir}, State) ->
    case State#ws.room_pid of
        undefined -> {ok, State};
        RoomPid ->
            game_room:move(RoomPid, State#ws.player_id, parse_dir(Dir)),
            {ok, State}
    end.

encode_msg(Msg) -> jsx:encode(Msg).
decode_game_msg(Data) -> jsx:decode(Data, [return_maps]).
parse_dir(<<"up">>)    -> {dy, -1};
parse_dir(<<"down">>)  -> {dy, 1};
parse_dir(<<"left">>)  -> {dx, -1};
parse_dir(<<"right">>) -> {dx, 1}.
```

---

## 5. Leaderboard System

```erlang
%% leaderboard.erl — real-time ranked leaderboard
-module(leaderboard).
-export([submit_score/3, top_n/2, rank_of/2, around_player/3]).

-define(LEADERBOARD_TABLE, leaderboard_scores).

submit_score(BoardId, PlayerId, Score) ->
    %% Sorted set: {-Score, PlayerId} so ets:select gives descending order
    Key = {BoardId, PlayerId},
    ets:insert(?LEADERBOARD_TABLE, {Key, Score, os:system_time(second)}).

top_n(BoardId, N) ->
    All = ets:select(?LEADERBOARD_TABLE, [
        {{{BoardId, '$1'}, '$2', '_'}, [], [{{score, '$2'}, {id, '$1'}}]}
    ]),
    Sorted = lists:sort(fun(#{score:=S1}, #{score:=S2}) -> S1 > S2 end,
                        [#{id=>Id, score=>Score} || {{score,Score},{id,Id}} <- All]),
    Ranked = lists:zip(lists:seq(1, length(Sorted)), Sorted),
    [maps:put(rank, R, P) || {R, P} <- lists:sublist(Ranked, N)].

rank_of(BoardId, PlayerId) ->
    Top = top_n(BoardId, 10_000),
    case lists:search(fun(#{id:=Id}) -> Id =:= PlayerId end, Top) of
        {value, #{rank := R}} -> {ok, R};
        false -> {error, not_found}
    end.

around_player(BoardId, PlayerId, N) ->
    {ok, Rank} = rank_of(BoardId, PlayerId),
    From = max(1, Rank - N),
    All  = top_n(BoardId, Rank + N),
    [P || #{rank := R} = P <- All, R >= From, R =< Rank + N].
```

---

## 6. Anti-Cheat

```erlang
%% anti_cheat.erl — server-side validation
-module(anti_cheat).
-export([validate_move/3, check_speed_hack/2]).

%% All game state on server — client is just a view
validate_move(PlayerId, NewPos, GameState) ->
    Players   = maps:get(players, GameState),
    Player    = maps:get(PlayerId, Players),
    CurrentPos = maps:get(pos, Player),
    case is_valid_move(CurrentPos, NewPos, GameState) of
        true  -> {ok, NewPos};
        false ->
            logger:warning("anti_cheat: invalid move player=~p from=~p to=~p",
                           [PlayerId, CurrentPos, NewPos]),
            {rejected, CurrentPos}   %% keep old position
    end.

is_valid_move({X1, Y1}, {X2, Y2}, _GameState) ->
    %% Max movement per tick
    DX = abs(X2 - X1),
    DY = abs(Y2 - Y1),
    DX =< 1 andalso DY =< 1.

check_speed_hack(PlayerId, ActionTimestamps) ->
    %% Check if player is sending too many actions
    Recent = [T || T <- ActionTimestamps,
                   T > os:system_time(millisecond) - 1000],
    case length(Recent) > 30 of   %% max 30 actions/second
        true ->
            logger:warning("speed_hack detected: player=~p rate=~p/s",
                           [PlayerId, length(Recent)]),
            {flagged, speed_hack};
        false ->
            ok
    end.
```

---

## 7. แบบฝึกหัด

1. เพิ่ม team matchmaking: จับคู่ 2v2 ด้วย average team skill
2. สร้าง spectator mode: subscribe to room state without playing
3. Implement game replay: store all moves, replay with ETS
4. เพิ่ม seasonal leaderboard: reset every month, keep historical

---

## สรุป Part 52

✅ Skill-based matchmaking  
✅ Per-room game state GenServer  
✅ WebSocket game client handler  
✅ Real-time leaderboard  
✅ Server-side anti-cheat validation  

---

*Part 52/100 | [← ก่อนหน้า](../part51/README.md) | [ถัดไป →](../part53/README.md)*
