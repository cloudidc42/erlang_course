# Part 91: Game Server — Low-Latency Real-Time

> **"A game server is a physics engine with opinions about cheating"**  
> Game server คือ physics engine ที่มีความคิดเห็นเกี่ยวกับการโกง

---

## สารบัญ

1. [Game Server Architecture](#1-game-server-architecture)
2. [Game Loop (Tick Engine)](#2-game-loop-tick-engine)
3. [Player Session Management](#3-player-session-management)
4. [World State and Entity System](#4-world-state-and-entity-system)
5. [Input Processing and Anti-Cheat](#5-input-processing-and-anti-cheat)
6. [Room Management](#6-room-management)
7. [แบบฝึกหัด](#7-แบบฝึกหัด)

---

## 1. Game Server Architecture

```
Game Server Process Model
═══════════════════════════════════════════════════════

game_sup (one_for_one)
  ├── room_registry (gen_server)     — maps room_id → room_pid
  ├── matchmaker (gen_server)        — queues and matches players
  ├── tick_scheduler (gen_server)    — drives all room ticks
  └── room_sup (simple_one_for_one)
        └── room_N (gen_statem)      — one per active game room
              └── player_1..N        — one websocket process per player

Tick rate: 20 ticks/second (50ms per tick)
  Each tick:
    1. Collect all pending player inputs
    2. Run physics simulation
    3. Apply game logic (collisions, scoring)
    4. Broadcast authoritative state to all players
    5. Check win/lose conditions

Network model:
  Player → WebSocket → player process → room (cast input)
  Room   → broadcast → all player processes → WebSocket → players
  Delta compression: only changed entity positions broadcast
```

---

## 2. Game Loop (Tick Engine)

```erlang
%% tick_engine.erl — 20Hz authoritative game loop
-module(tick_engine).
-behaviour(gen_statem).

-export([start_link/1, player_input/2]).
-export([init/1, callback_mode/0, running/3]).

-define(TICK_RATE_HZ, 20).
-define(TICK_MS, 1000 div ?TICK_RATE_HZ).  %% 50ms

-record(data, {
    room_id,
    tick  = 0      :: non_neg_integer(),
    world          :: map(),        %% entity_id -> entity state
    pending_inputs = #{} :: map()  %% player_id -> [input]
}).

start_link(RoomId) ->
    gen_statem:start_link(?MODULE, RoomId, []).

player_input(TickPid, {PlayerId, Input}) ->
    gen_statem:cast(TickPid, {input, PlayerId, Input}).

callback_mode() -> [handle_event_function].

init(RoomId) ->
    %% First tick fires immediately
    schedule_tick(),
    {ok, running, #data{room_id = RoomId, world = initial_world()}}.

running(info, tick, #data{tick = T, world = World,
                           pending_inputs = Inputs} = Data) ->
    %% 1. Process all queued player inputs
    World1 = apply_inputs(Inputs, World),

    %% 2. Physics step
    World2 = physics_step(World1, ?TICK_MS / 1000.0),

    %% 3. Game logic
    World3 = game_logic(World2),

    %% 4. Broadcast state delta
    Delta = compute_delta(World, World3),
    broadcast_delta(Data#data.room_id, T, Delta),

    %% 5. Check end conditions
    case check_game_over(World3) of
        {over, Winner} ->
            finish_game(Data#data.room_id, Winner),
            {stop, normal, Data};
        playing ->
            schedule_tick(),
            NewData = Data#data{
                tick           = T + 1,
                world          = World3,
                pending_inputs = #{}
            },
            {keep_state, NewData}
    end;

running(cast, {input, PlayerId, Input}, #data{pending_inputs = Inputs} = Data) ->
    %% Buffer input for next tick
    PlayerInputs = maps:get(PlayerId, Inputs, []),
    NewInputs    = maps:put(PlayerId, [Input | PlayerInputs], Inputs),
    {keep_state, Data#data{pending_inputs = NewInputs}}.

apply_inputs(Inputs, World) ->
    maps:fold(fun(PlayerId, PlayerInputs, W) ->
        lists:foldl(fun(Input, WInner) ->
            apply_player_input(PlayerId, Input, WInner)
        end, W, lists:reverse(PlayerInputs))
    end, World, Inputs).

apply_player_input(PlayerId, #{direction := Dir, action := Action}, World) ->
    case maps:get(PlayerId, World, undefined) of
        undefined -> World;
        Player ->
            Moved = move_player(Player, Dir, ?TICK_MS / 1000.0),
            After = case Action of
                jump  -> jump(Moved);
                shoot -> shoot(Moved, World);
                _     -> Moved
            end,
            maps:put(PlayerId, After, World)
    end.

physics_step(World, DtSec) ->
    maps:map(fun(_Id, Entity) ->
        apply_gravity(apply_velocity(Entity, DtSec))
    end, World).

apply_velocity(#{pos := {X, Y}, vel := {Vx, Vy}} = E, Dt) ->
    E#{pos => {X + Vx * Dt, Y + Vy * Dt}}.

apply_gravity(#{vel := {Vx, Vy}} = E) ->
    E#{vel => {Vx, Vy - 9.8 * (?TICK_MS / 1000.0)}}.

compute_delta(Old, New) ->
    maps:fold(fun(Id, NewState, Acc) ->
        case maps:get(Id, Old, undefined) of
            NewState -> Acc;  %% unchanged
            _        -> maps:put(Id, NewState, Acc)
        end
    end, #{}, New).

broadcast_delta(RoomId, Tick, Delta) ->
    room_registry:broadcast(RoomId, #{tick => Tick, delta => Delta}).

schedule_tick() ->
    erlang:send_after(?TICK_MS, self(), tick).

initial_world() -> #{}.
game_logic(W) -> W.
check_game_over(_W) -> playing.
finish_game(_RoomId, _Winner) -> ok.
move_player(P, _Dir, _Dt) -> P.
jump(P) -> P.
shoot(P, _W) -> P.
```

---

## 3. Player Session Management

```erlang
%% player_session.erl — WebSocket process per player
-module(player_session).
-behaviour(cowboy_websocket).

-export([init/2, websocket_init/1, websocket_handle/2,
         websocket_info/2, terminate/3]).

-define(PING_INTERVAL_MS, 5000).
-define(TIMEOUT_MS, 30000).

-record(state, {
    player_id,
    room_pid,
    last_ping_at
}).

init(Req, _Opts) ->
    Token = cowboy_req:binding(token, Req),
    case player_auth:verify_token(Token) of
        {ok, PlayerId} ->
            {cowboy_websocket, Req, PlayerId,
             #{idle_timeout => ?TIMEOUT_MS}};
        {error, _} ->
            {ok, cowboy_req:reply(401, #{}, <<>>, Req), undefined}
    end.

websocket_init(PlayerId) ->
    {ok, RoomPid} = matchmaker:find_or_create_room(PlayerId),
    room:player_joined(RoomPid, PlayerId, self()),
    schedule_ping(),
    {ok, #state{player_id = PlayerId, room_pid = RoomPid,
                last_ping_at = erlang:monotonic_time(second)}}.

%% Message from player (browser → server)
websocket_handle({binary, Data}, #state{room_pid = RoomPid,
                                         player_id = PlayerId} = State) ->
    case decode_input(Data) of
        {ok, Input} ->
            tick_engine:player_input(RoomPid, {PlayerId, Input}),
            {ok, State};
        {error, _} ->
            {ok, State}
    end;

websocket_handle(ping, State) ->
    {reply, pong, State#state{last_ping_at = erlang:monotonic_time(second)}}.

%% Message from server (game loop → this process → browser)
websocket_info({game_state, DeltaBin}, State) ->
    {reply, {binary, DeltaBin}, State};

websocket_info(send_ping, State) ->
    schedule_ping(),
    {reply, ping, State}.

terminate(_Reason, _Req, #state{room_pid = RoomPid, player_id = PlayerId}) ->
    room:player_left(RoomPid, PlayerId),
    ok.

decode_input(Binary) ->
    try
        Input = binary_to_term(Binary, [safe]),
        {ok, Input}
    catch _:_ ->
        {error, invalid}
    end.

schedule_ping() ->
    erlang:send_after(?PING_INTERVAL_MS, self(), send_ping).
```

---

## 4. World State and Entity System

```erlang
%% entities.erl — entity component system for game objects
-module(entities).
-export([new_player/2, new_bullet/3, new_wall/3, tick/2]).

-define(PLAYER_SPEED, 200.0).  %% pixels per second
-define(BULLET_SPEED, 800.0).
-define(GRAVITY, 980.0).       %% pixels per second^2

new_player(Id, #{x := X, y := Y}) ->
    #{type => player, id => Id,
      pos  => {X, Y}, vel => {0.0, 0.0},
      hp   => 100, alive => true,
      facing => right}.

new_bullet(OwnerId, {X, Y}, Direction) ->
    {Vx, Vy} = direction_to_vel(Direction, ?BULLET_SPEED),
    #{type => bullet, id => make_ref(),
      owner_id => OwnerId,
      pos  => {X, Y}, vel => {Vx, Vy},
      damage => 25, lifetime_ticks => 60}.

new_wall(Id, {X, Y}, {W, H}) ->
    #{type => wall, id => Id, pos => {X, Y},
      size => {W, H}, solid => true}.

tick(Entities, DtSec) ->
    %% Move everything
    Moved = maps:map(fun(_Id, E) -> move(E, DtSec) end, Entities),
    %% Detect collisions
    detect_collisions(Moved).

move(#{type := wall} = E, _Dt) -> E;
move(#{pos := {X, Y}, vel := {Vx, Vy}} = E, Dt) ->
    Vy2 = Vy - ?GRAVITY * Dt,  %% apply gravity
    E#{pos => {X + Vx * Dt, Y + Vy2 * Dt}, vel => {Vx, Vy2}}.

detect_collisions(Entities) ->
    Walls    = [E || {_, E} <- maps:to_list(Entities), maps:get(type, E) =:= wall],
    Bullets  = [E || {_, E} <- maps:to_list(Entities), maps:get(type, E) =:= bullet],
    Entities2 = resolve_bullet_hits(Bullets, Entities),
    resolve_wall_collisions(Walls, Entities2).

resolve_bullet_hits(Bullets, Entities) ->
    lists:foldl(fun(Bullet, Es) ->
        #{pos := BPos, owner_id := OwnerId, damage := Dmg} = Bullet,
        maps:map(fun(Id, E) ->
            case Id =/= OwnerId andalso maps:get(type, E) =:= player
                 andalso within_range(BPos, maps:get(pos, E), 20) of
                true  -> apply_damage(E, Dmg);
                false -> E
            end
        end, Es)
    end, Entities, Bullets).

resolve_wall_collisions(_Walls, Entities) -> Entities.  %% simplified

apply_damage(#{hp := Hp} = E, Dmg) ->
    NewHp = max(0, Hp - Dmg),
    E#{hp => NewHp, alive => NewHp > 0}.

within_range({X1, Y1}, {X2, Y2}, Dist) ->
    math:sqrt(math:pow(X2-X1, 2) + math:pow(Y2-Y1, 2)) < Dist.

direction_to_vel(right, Speed) -> {Speed, 0.0};
direction_to_vel(left, Speed)  -> {-Speed, 0.0};
direction_to_vel(up, Speed)    -> {0.0, Speed};
direction_to_vel(down, Speed)  -> {0.0, -Speed}.
```

---

## 5. Input Processing and Anti-Cheat

```erlang
%% anticheat.erl — server-side input validation
-module(anticheat).
-export([validate_input/3, check_movement/3]).

-define(MAX_INPUTS_PER_TICK, 5).
-define(MAX_SPEED_PX_SEC, 300.0).     %% slightly above PLAYER_SPEED for tolerance
-define(MAX_POSITION_JUMP_PX, 50.0).  %% max teleport distance per tick

validate_input(PlayerId, Input, #{tick := Tick} = _Context) ->
    case Input of
        #{type := move, direction := Dir}
          when Dir =:= up; Dir =:= down; Dir =:= left; Dir =:= right ->
            {ok, Input};
        #{type := shoot, target := {X, Y}}
          when is_number(X), is_number(Y) ->
            {ok, Input};
        #{type := jump} ->
            {ok, Input};
        _ ->
            log_suspicious(PlayerId, Tick, {invalid_input_type, Input}),
            {error, invalid_input}
    end.

%% Validate that player didn't move impossibly fast
check_movement(PlayerId, OldPos, NewPos) ->
    Distance = position_distance(OldPos, NewPos),
    MaxAllowed = ?MAX_SPEED_PX_SEC * (?TICK_MS / 1000.0) + ?MAX_POSITION_JUMP_PX,
    case Distance > MaxAllowed of
        true ->
            log_suspicious(PlayerId, erlang:system_time(), {teleport, Distance}),
            {reject, OldPos};  %% snap back to old position
        false ->
            {ok, NewPos}
    end.

position_distance({X1, Y1}, {X2, Y2}) ->
    math:sqrt(math:pow(X2-X1, 2) + math:pow(Y2-Y1, 2)).

log_suspicious(PlayerId, Tick, Event) ->
    logger:warning("Anti-cheat: player ~p tick ~p event ~p",
                   [PlayerId, Tick, Event]).

-define(TICK_MS, 50).
```

---

## 6. Room Management

```erlang
%% room_registry.erl — room discovery and broadcast
-module(room_registry).
-behaviour(gen_server).

-export([start_link/0, register_room/2, broadcast/2, list_rooms/0]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

register_room(RoomId, Pid) ->
    gen_server:cast(?MODULE, {register, RoomId, Pid}).

broadcast(RoomId, Message) ->
    gen_server:cast(?MODULE, {broadcast, RoomId, Message}).

list_rooms() ->
    gen_server:call(?MODULE, list).

init([]) ->
    {ok, #{rooms => #{}, players => #{}}}.

handle_cast({register, RoomId, Pid}, #{rooms := Rooms} = State) ->
    monitor(process, Pid),
    {noreply, State#{rooms => maps:put(RoomId, #{pid => Pid, players => []}, Rooms)}};

handle_cast({broadcast, RoomId, Message}, #{rooms := Rooms} = State) ->
    case maps:get(RoomId, Rooms, undefined) of
        undefined -> ok;
        #{players := Players} ->
            Encoded = term_to_binary(Message),
            lists:foreach(fun(PlayerPid) ->
                PlayerPid ! {game_state, Encoded}
            end, Players)
    end,
    {noreply, State};

handle_cast({player_join, RoomId, PlayerPid}, #{rooms := Rooms} = State) ->
    Room    = maps:get(RoomId, Rooms, #{players => []}),
    Players = maps:get(players, Room, []),
    NewRoom = Room#{players => [PlayerPid | Players]},
    {noreply, State#{rooms => maps:put(RoomId, NewRoom, Rooms)}}.

handle_call(list, _From, #{rooms := Rooms} = State) ->
    Info = [{Id, maps:get(pid, R)} || {Id, R} <- maps:to_list(Rooms)],
    {reply, Info, State}.

handle_info({'DOWN', _Ref, process, Pid, _Reason}, #{rooms := Rooms} = State) ->
    NewRooms = maps:filter(fun(_, #{pid := P}) -> P =/= Pid end, Rooms),
    {noreply, State#{rooms => NewRooms}}.
```

---

## 7. แบบฝึกหัด

1. Implement client-side prediction reconciliation: client predicts movement locally, corrects when server responds
2. สร้าง spectator mode: player ที่ตายแล้วยัง watch ห้องได้
3. เพิ่ม lag compensation: เก็บ world state history 500ms เพื่อตรวจ hit detection ย้อนหลัง
4. Implement matchmaking ELO rating: จับคู่ผู้เล่นที่มีระดับ skill ใกล้เคียงกัน

---

## สรุป Part 91

✅ Game server architecture: tick engine, room per process, player websocket  
✅ 20Hz game loop: input collection → physics → game logic → broadcast delta  
✅ Player session: WebSocket process, ping/pong, room lifecycle  
✅ Entity system: player, bullet, wall; gravity, velocity, collision  
✅ Anti-cheat: input type validation, movement speed clamping  
✅ Room registry: broadcast to all players, monitor for cleanup  

---

*Part 91/100 | [← ก่อนหน้า](../part90/README.md) | [ถัดไป →](../part92/README.md)*
