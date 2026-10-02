# Part 62: Real-Time Collaboration System

> **"Build systems where many users work together without conflict"**  
> สร้างระบบที่ผู้ใช้หลายคนทำงานร่วมกันได้โดยไม่ขัดแย้งกัน

---

## สารบัญ

1. [Operational Transformation](#1-operational-transformation)
2. [Document State Management](#2-document-state-management)
3. [Presence System](#3-presence-system)
4. [Conflict Resolution](#4-conflict-resolution)
5. [Collaborative Editor Backend](#5-collaborative-editor-backend)
6. [Cursor and Selection Sync](#6-cursor-and-selection-sync)
7. [แบบฝึกหัด](#7-แบบฝึกหัด)

---

## 1. Operational Transformation

```erlang
%% ot.erl — Operational Transformation for text editing
-module(ot).
-export([apply_op/2, compose/2, transform/2]).

%% Operations:
%% {retain, N}   — keep N characters
%% {insert, Str} — insert string at cursor
%% {delete, N}   — delete N characters

-type op() :: {retain, pos_integer()}
            | {insert, binary()}
            | {delete, pos_integer()}.

-spec apply_op(binary(), [op()]) -> binary().
apply_op(Doc, Ops) ->
    apply_op(Doc, Ops, <<>>).

apply_op(<<>>, [], Acc) -> Acc;
apply_op(Doc, [], Acc)  -> <<Acc/binary, Doc/binary>>;
apply_op(Doc, [{retain, N} | Rest], Acc) ->
    <<Chunk:N/binary, Remaining/binary>> = Doc,
    apply_op(Remaining, Rest, <<Acc/binary, Chunk/binary>>);
apply_op(Doc, [{insert, Str} | Rest], Acc) ->
    apply_op(Doc, Rest, <<Acc/binary, Str/binary>>);
apply_op(Doc, [{delete, N} | Rest], Acc) ->
    <<_:N/binary, Remaining/binary>> = Doc,
    apply_op(Remaining, Rest, Acc).

%% Compose two sequential operations into one
-spec compose([op()], [op()]) -> [op()].
compose(Ops1, Ops2) ->
    compose(Ops1, Ops2, []).

compose([], [], Acc) ->
    normalize(lists:reverse(Acc));
compose([], [{insert, S} | Rest], Acc) ->
    compose([], Rest, [{insert, S} | Acc]);
compose(Ops1, [], Acc) ->
    lists:reverse(Acc) ++ Ops1;
compose([{retain, N1} | R1], [{retain, N2} | R2], Acc) when N1 =:= N2 ->
    compose(R1, R2, [{retain, N1} | Acc]);
compose([{retain, N1} | R1], [{retain, N2} | R2], Acc) when N1 > N2 ->
    compose([{retain, N1-N2} | R1], R2, [{retain, N2} | Acc]);
compose([{retain, N1} | R1], [{retain, N2} | R2], Acc) ->
    compose(R1, [{retain, N2-N1} | R2], [{retain, N1} | Acc]);
compose([{insert, S} | R1], [{retain, N} | R2], Acc) ->
    Len = byte_size(S),
    case Len =< N of
        true  -> compose(R1, [{retain, N-Len} | R2], [{insert, S} | Acc]);
        false ->
            <<Keep:N/binary, _/binary>> = S,
            compose([{insert, binary:part(S, N, Len-N)} | R1], R2,
                    [{insert, Keep} | Acc])
    end;
compose([{insert, S} | R1], [{delete, N} | R2], Acc) ->
    Len = byte_size(S),
    case Len =< N of
        true  -> compose(R1, [{delete, N-Len} | R2], Acc);
        false -> compose([{insert, binary:part(S, N, Len-N)} | R1], R2, Acc)
    end;
compose([{delete, N} | R1], Ops2, Acc) ->
    compose(R1, Ops2, [{delete, N} | Acc]).

%% Transform op1 against op2 (concurrent operations)
%% Returns {Op1', Op2'} where both can be applied in any order
-spec transform([op()], [op()]) -> {[op()], [op()]}.
transform(Ops1, Ops2) ->
    transform(Ops1, Ops2, [], []).

transform([], [], Acc1, Acc2) ->
    {normalize(lists:reverse(Acc1)), normalize(lists:reverse(Acc2))};
transform(Ops1, [{insert, S} | Rest2], Acc1, Acc2) ->
    Len = byte_size(S),
    transform([{retain, Len} | Ops1], Rest2,
              [{insert, S} | Acc1], Acc2);
transform([{insert, S} | Rest1], Ops2, Acc1, Acc2) ->
    Len = byte_size(S),
    transform(Rest1, [{retain, Len} | Ops2],
              Acc1, [{insert, S} | Acc2]);
transform([{retain, N1} | R1], [{retain, N2} | R2], Acc1, Acc2) when N1 =:= N2 ->
    transform(R1, R2, [{retain, N1} | Acc1], [{retain, N2} | Acc2]);
transform([{retain, N1} | R1], [{delete, N2} | R2], Acc1, Acc2) when N1 =:= N2 ->
    transform(R1, R2, Acc1, [{delete, N2} | Acc2]);
transform(_, _, Acc1, Acc2) ->
    {normalize(lists:reverse(Acc1)), normalize(lists:reverse(Acc2))}.

normalize([]) -> [];
normalize([{retain, 0} | Rest]) -> normalize(Rest);
normalize([{delete, 0} | Rest]) -> normalize(Rest);
normalize([{insert, <<>>} | Rest]) -> normalize(Rest);
normalize([H | Rest]) -> [H | normalize(Rest)].
```

---

## 2. Document State Management

```erlang
%% collab_doc.erl — collaborative document GenServer
-module(collab_doc).
-behaviour(gen_server).
-export([start_link/2, apply_op/3, get_state/1, subscribe/2]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

-record(state, {
    doc_id    :: binary(),
    content   :: binary(),
    revision  :: non_neg_integer(),
    history   :: [{rev(), client_id(), [ot:op()]}],
    clients   :: #{client_id() => pid()}
}).

-type rev() :: non_neg_integer().
-type client_id() :: binary().

start_link(DocId, InitialContent) ->
    gen_server:start_link({via, gproc, {n, l, {doc, DocId}}},
                          ?MODULE, {DocId, InitialContent}, []).

apply_op(DocId, ClientId, {ClientRev, Ops}) ->
    gen_server:call({via, gproc, {n, l, {doc, DocId}}},
                    {apply_op, ClientId, ClientRev, Ops}).

get_state(DocId) ->
    gen_server:call({via, gproc, {n, l, {doc, DocId}}}, get_state).

subscribe(DocId, Pid) ->
    gen_server:call({via, gproc, {n, l, {doc, DocId}}}, {subscribe, Pid}).

init({DocId, Content}) ->
    {ok, #state{doc_id=DocId, content=Content, revision=0,
                history=[], clients=#{}}}.

handle_call({apply_op, ClientId, ClientRev, Ops}, _From, State) ->
    #state{content=Doc, revision=ServerRev, history=History} = State,
    case ClientRev > ServerRev of
        true ->
            {reply, {error, invalid_revision}, State};
        false ->
            %% Transform ops against concurrent operations
            ConcurrentOps = ops_since(History, ClientRev),
            TransformedOps = transform_against(Ops, ConcurrentOps),
            NewDoc      = ot:apply_op(Doc, TransformedOps),
            NewRev      = ServerRev + 1,
            NewHistory  = [{NewRev, ClientId, TransformedOps} | History],
            State1 = State#state{content=NewDoc, revision=NewRev,
                                 history=NewHistory},
            %% Broadcast to other clients
            broadcast(State1, ClientId, NewRev, TransformedOps),
            {reply, {ok, NewRev, TransformedOps}, State1}
    end;
handle_call(get_state, _From, #state{content=C, revision=R} = State) ->
    {reply, {ok, C, R}, State};
handle_call({subscribe, Pid}, _From, #state{clients=Clients} = State) ->
    monitor(process, Pid),
    {reply, ok, State#state{clients=Clients#{Pid => Pid}}}.

handle_cast(_, State) -> {noreply, State}.

handle_info({'DOWN', _, process, Pid, _}, #state{clients=Clients} = State) ->
    {noreply, State#state{clients=maps:remove(Pid, Clients)}}.

ops_since(History, SinceRev) ->
    [Ops || {Rev, _, Ops} <- History, Rev > SinceRev].

transform_against(Ops, []) -> Ops;
transform_against(Ops, [Concurrent | Rest]) ->
    {Ops1, _} = ot:transform(Ops, Concurrent),
    transform_against(Ops1, Rest).

broadcast(#state{clients=Clients}, FromPid, Rev, Ops) ->
    [Pid ! {doc_op, Rev, Ops}
     || Pid <- maps:keys(Clients), Pid =/= FromPid].
```

---

## 3. Presence System

```erlang
%% presence.erl — user presence tracking
-module(presence).
-behaviour(gen_server).
-export([start_link/0, join/3, leave/2, get_users/1, update_cursor/3]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

-record(user_presence, {
    user_id  :: integer(),
    name     :: binary(),
    color    :: binary(),
    cursor   :: {integer(), integer()} | undefined,
    joined_at :: integer()
}).

-record(state, {
    docs :: #{binary() => #{integer() => #user_presence{}}}
}).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

join(DocId, UserId, UserInfo) ->
    gen_server:call(?MODULE, {join, DocId, UserId, UserInfo}).

leave(DocId, UserId) ->
    gen_server:call(?MODULE, {leave, DocId, UserId}).

get_users(DocId) ->
    gen_server:call(?MODULE, {get_users, DocId}).

update_cursor(DocId, UserId, Cursor) ->
    gen_server:cast(?MODULE, {update_cursor, DocId, UserId, Cursor}).

init([]) ->
    {ok, #state{docs=#{}}}.

handle_call({join, DocId, UserId, #{name := Name}}, _From, #state{docs=Docs} = State) ->
    DocUsers = maps:get(DocId, Docs, #{}),
    Color = pick_color(map_size(DocUsers)),
    Presence = #user_presence{
        user_id   = UserId,
        name      = Name,
        color     = Color,
        cursor    = undefined,
        joined_at = os:system_time(millisecond)
    },
    NewDocUsers = DocUsers#{UserId => Presence},
    NewDocs = Docs#{DocId => NewDocUsers},
    broadcast_presence(DocId, NewDocUsers),
    {reply, {ok, Color}, State#state{docs=NewDocs}};

handle_call({leave, DocId, UserId}, _From, #state{docs=Docs} = State) ->
    DocUsers = maps:get(DocId, Docs, #{}),
    NewDocUsers = maps:remove(UserId, DocUsers),
    NewDocs = case map_size(NewDocUsers) of
        0 -> maps:remove(DocId, Docs);
        _ -> Docs#{DocId => NewDocUsers}
    end,
    broadcast_presence(DocId, NewDocUsers),
    {reply, ok, State#state{docs=NewDocs}};

handle_call({get_users, DocId}, _From, #state{docs=Docs} = State) ->
    DocUsers = maps:get(DocId, Docs, #{}),
    Users = [presence_to_map(P) || P <- maps:values(DocUsers)],
    {reply, {ok, Users}, State}.

handle_cast({update_cursor, DocId, UserId, Cursor}, #state{docs=Docs} = State) ->
    case maps:find(DocId, Docs) of
        {ok, DocUsers} ->
            case maps:find(UserId, DocUsers) of
                {ok, Presence} ->
                    Updated = Presence#user_presence{cursor=Cursor},
                    NewDocUsers = DocUsers#{UserId => Updated},
                    %% Broadcast cursor only (lightweight)
                    broadcast_cursor(DocId, UserId, Cursor),
                    {noreply, State#state{docs=Docs#{DocId => NewDocUsers}}};
                error -> {noreply, State}
            end;
        error -> {noreply, State}
    end.

handle_info(_, State) -> {noreply, State}.

presence_to_map(#user_presence{user_id=Id, name=N, color=C, cursor=Cu}) ->
    #{user_id => Id, name => N, color => C, cursor => Cu}.

pick_color(Index) ->
    Colors = [<<"#e74c3c">>, <<"#3498db">>, <<"#2ecc71">>,
              <<"#f39c12">>, <<"#9b59b6">>, <<"#1abc9c">>],
    lists:nth((Index rem length(Colors)) + 1, Colors).

broadcast_presence(DocId, Users) ->
    pg:send(DocId, {presence_update, Users}).

broadcast_cursor(DocId, UserId, Cursor) ->
    pg:send(DocId, {cursor_update, UserId, Cursor}).
```

---

## 4. Conflict Resolution

```erlang
%% conflict_resolver.erl — detect and handle edit conflicts
-module(conflict_resolver).
-export([detect/2, resolve/3, merge_text/3]).

%% Detect conflict: two ops modify the same region
detect(Ops1, Ops2) ->
    Ranges1 = affected_ranges(Ops1, 0, []),
    Ranges2 = affected_ranges(Ops2, 0, []),
    overlapping(Ranges1, Ranges2).

affected_ranges([], _, Acc) -> lists:reverse(Acc);
affected_ranges([{retain, N} | Rest], Pos, Acc) ->
    affected_ranges(Rest, Pos+N, Acc);
affected_ranges([{insert, S} | Rest], Pos, Acc) ->
    Len = byte_size(S),
    affected_ranges(Rest, Pos+Len, [{Pos, Pos+Len, insert} | Acc]);
affected_ranges([{delete, N} | Rest], Pos, Acc) ->
    affected_ranges(Rest, Pos+N, [{Pos, Pos+N, delete} | Acc]).

overlapping(Ranges1, Ranges2) ->
    [{R1, R2}
     || {Start1, End1, _} = R1 <- Ranges1,
        {Start2, End2, _} = R2 <- Ranges2,
        Start1 < End2, Start2 < End1].

%% Resolve: prefer last-writer-wins for most conflicts
resolve(Ops1, Ops2, Strategy) ->
    case Strategy of
        last_write_wins ->
            %% Apply both; op2 wins on conflict
            {_, Ops2Prime} = ot:transform(Ops1, Ops2),
            Ops2Prime;
        first_write_wins ->
            {Ops1Prime, _} = ot:transform(Ops1, Ops2),
            Ops1Prime;
        merge ->
            merge_ops(Ops1, Ops2)
    end.

%% Three-way merge using base, op1, op2
merge_text(Base, Text1, Text2) ->
    %% Simple diff3-like approach
    Lines0 = binary:split(Base, <<"\n">>, [global]),
    Lines1 = binary:split(Text1, <<"\n">>, [global]),
    Lines2 = binary:split(Text2, <<"\n">>, [global]),
    merge_lines(Lines0, Lines1, Lines2, []).

merge_lines([], [], [], Acc) ->
    {ok, lists:join(<<"\n">>, lists:reverse(Acc))};
merge_lines([L|R0], [L|R1], [L|R2], Acc) ->
    %% All agree
    merge_lines(R0, R1, R2, [L | Acc]);
merge_lines([_|R0], [L1|R1], [L1|R2], Acc) ->
    %% Both changed same way
    merge_lines(R0, R1, R2, [L1 | Acc]);
merge_lines([L0|R0], [L0|R1], [L2|R2], Acc) ->
    %% Only side 2 changed
    merge_lines(R0, R1, R2, [L2 | Acc]);
merge_lines([L0|R0], [L1|R1], [L0|R2], Acc) ->
    %% Only side 1 changed
    merge_lines(R0, R1, R2, [L1 | Acc]);
merge_lines([_|R0], [L1|R1], [L2|R2], Acc) ->
    %% Both changed differently — conflict marker
    Conflict = <<"<<<<<<\n", L1/binary, "\n======\n", L2/binary, "\n>>>>>>">>,
    merge_lines(R0, R1, R2, [Conflict | Acc]);
merge_lines(_, _, _, Acc) ->
    {conflict, lists:reverse(Acc)}.

merge_ops(Ops1, Ops2) ->
    {Ops1T, Ops2T} = ot:transform(Ops1, Ops2),
    ot:compose(Ops1T, Ops2T).
```

---

## 5. Collaborative Editor Backend

```erlang
%% collab_ws_handler.erl — WebSocket handler for collaborative editing
-module(collab_ws_handler).
-behaviour(cowboy_websocket).
-export([init/2, websocket_init/1, websocket_handle/2,
         websocket_info/2, terminate/3]).

-record(state, {
    doc_id  :: binary(),
    user_id :: integer(),
    user    :: map()
}).

init(Req, _) ->
    DocId  = cowboy_req:binding(doc_id, Req),
    Token  = cowboy_req:header(<<"authorization">>, Req, <<>>),
    case jwt_auth:verify(Token) of
        {ok, Claims} ->
            UserId = maps:get(<<"sub">>, Claims),
            User   = #{id => UserId, name => maps:get(<<"name">>, Claims)},
            {cowboy_websocket, Req, #{doc_id => DocId, user_id => UserId, user => User},
             #{idle_timeout => 120000}};
        _ ->
            {ok, cowboy_req:reply(401, Req), #{}}
    end.

websocket_init(#{doc_id := DocId, user_id := UserId, user := User}) ->
    %% Ensure document process is running
    case collab_doc_sup:ensure_doc(DocId) of
        ok -> ok;
        {error, _} -> throw(doc_not_found)
    end,
    %% Subscribe to document events
    collab_doc:subscribe(DocId, self()),
    %% Join presence
    {ok, Color} = presence:join(DocId, UserId, User),
    %% Fetch current document state
    {ok, Content, Rev} = collab_doc:get_state(DocId),
    %% Send initial state
    InitMsg = jsx:encode(#{
        type     => <<"init">>,
        content  => Content,
        revision => Rev,
        color    => Color
    }),
    {[{text, InitMsg}],
     #state{doc_id=DocId, user_id=UserId, user=User#{color => Color}}}.

websocket_handle({text, Data}, #state{doc_id=DocId, user_id=UserId} = State) ->
    case jsx:decode(Data, [return_maps]) of
        #{<<"type">> := <<"op">>, <<"rev">> := Rev, <<"ops">> := RawOps} ->
            Ops = decode_ops(RawOps),
            case collab_doc:apply_op(DocId, UserId, {Rev, Ops}) of
                {ok, ServerRev, TransformedOps} ->
                    Ack = jsx:encode(#{
                        type  => <<"ack">>,
                        rev   => ServerRev,
                        ops   => encode_ops(TransformedOps)
                    }),
                    {[{text, Ack}], State};
                {error, Reason} ->
                    Err = jsx:encode(#{type => <<"error">>, reason => Reason}),
                    {[{text, Err}], State}
            end;
        #{<<"type">> := <<"cursor">>, <<"pos">> := Pos} ->
            presence:update_cursor(DocId, UserId, Pos),
            {[], State};
        _ ->
            {[], State}
    end.

websocket_info({doc_op, Rev, Ops}, State) ->
    Msg = jsx:encode(#{type => <<"op">>, rev => Rev, ops => encode_ops(Ops)}),
    {[{text, Msg}], State};
websocket_info({presence_update, Users}, State) ->
    Msg = jsx:encode(#{type => <<"presence">>, users => Users}),
    {[{text, Msg}], State};
websocket_info({cursor_update, UserId, Pos}, State) ->
    Msg = jsx:encode(#{type => <<"cursor">>, user_id => UserId, pos => Pos}),
    {[{text, Msg}], State};
websocket_info(_, State) ->
    {[], State}.

terminate(_, _, #state{doc_id=DocId, user_id=UserId}) ->
    presence:leave(DocId, UserId),
    ok.

decode_ops(RawOps) ->
    [decode_op(Op) || Op <- RawOps].

decode_op(#{<<"retain">> := N}) -> {retain, N};
decode_op(#{<<"insert">> := S}) -> {insert, S};
decode_op(#{<<"delete">> := N}) -> {delete, N}.

encode_ops(Ops) ->
    [encode_op(Op) || Op <- Ops].

encode_op({retain, N}) -> #{<<"retain">> => N};
encode_op({insert, S}) -> #{<<"insert">> => S};
encode_op({delete, N}) -> #{<<"delete">> => N}.
```

---

## 6. Cursor and Selection Sync

```erlang
%% cursor_tracker.erl — track multiple user cursors
-module(cursor_tracker).
-export([new/0, update/4, get_all/1, remove/2]).

%% State: #{doc_id => #{user_id => cursor_info}}
new() -> #{}.

update(DocId, UserId, Position, State) ->
    DocCursors = maps:get(DocId, State, #{}),
    CursorInfo = #{
        position  => Position,
        updated   => os:system_time(millisecond)
    },
    State#{DocId => DocCursors#{UserId => CursorInfo}}.

get_all(DocId) ->
    %% Return all cursors for a document
    gen_server:call(cursor_tracker_server, {get_all, DocId}).

remove(DocId, UserId) ->
    gen_server:cast(cursor_tracker_server, {remove, DocId, UserId}).

%% Transform cursor position when document is modified
transform_cursor(CursorPos, Ops) ->
    transform_cursor(CursorPos, Ops, 0).

transform_cursor(CursorPos, [], _Offset) ->
    CursorPos;
transform_cursor(CursorPos, [{retain, N} | Rest], Offset) when Offset + N =< CursorPos ->
    transform_cursor(CursorPos, Rest, Offset + N);
transform_cursor(CursorPos, [{insert, S} | Rest], Offset) when Offset =< CursorPos ->
    Len = byte_size(S),
    transform_cursor(CursorPos + Len, Rest, Offset + Len);
transform_cursor(CursorPos, [{delete, N} | Rest], Offset) ->
    NewPos = max(Offset, CursorPos - N),
    transform_cursor(NewPos, Rest, Offset);
transform_cursor(CursorPos, _, _) ->
    CursorPos.
```

---

## 7. แบบฝึกหัด

1. Implement undo/redo stack: เก็บ history ของ operations และ reverse them
2. เพิ่ม offline support: queue operations เมื่อ connection ขาด แล้ว sync เมื่อกลับมา
3. สร้าง commenting system ที่ anchor to document positions (คล้าย Google Docs)
4. Implement rich text operations: bold, italic, heading

---

## สรุป Part 62

✅ Operational Transformation: apply, compose, transform  
✅ Collaborative document GenServer ด้วย OT  
✅ Presence system: join/leave/cursor tracking  
✅ Three-way merge conflict resolution  
✅ WebSocket handler สำหรับ real-time editing  
✅ Cursor position transformation  

---

*Part 62/100 | [← ก่อนหน้า](../part61/README.md) | [ถัดไป →](../part63/README.md)*
