# Part 45: Protocol Design and Binary Parsing

> **"A well-designed protocol is the foundation of every distributed system"**  
> Protocol ที่ออกแบบดีคือรากฐานของทุก distributed system

---

## สารบัญ

1. [Binary Protocol Design](#1-binary-protocol-design)
2. [Erlang Binary Pattern Matching](#2-erlang-binary-pattern-matching)
3. [Custom TCP Protocol](#3-custom-tcp-protocol)
4. [Frame Encoding/Decoding](#4-frame-encodingdecoding)
5. [Protocol Versioning](#5-protocol-versioning)
6. [WebSocket Binary Messages](#6-websocket-binary-messages)
7. [แบบฝึกหัด](#7-แบบฝึกหัด)

---

## 1. Binary Protocol Design

```
Protocol Frame Layout:

  0                   1                   2                   3
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |  Magic (0xAB) |    Version    |     Type      |   Flags      |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |                        Sequence Number                        |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |                        Payload Length                         |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |                      Payload (variable)                       |
 |                            ...                                |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |                    CRC32 Checksum                             |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

Magic:    0xAB (protocol identifier)
Version:  1 byte
Type:     request (0x01), response (0x02), notify (0x03), error (0x04)
Flags:    compressed (bit 0), encrypted (bit 1), ack_required (bit 2)
Sequence: 4 bytes, big-endian
Length:   4 bytes, big-endian
Payload:  variable
CRC32:    4 bytes, big-endian
```

---

## 2. Erlang Binary Pattern Matching

```erlang
%% protocol.erl — encode/decode the custom protocol
-module(protocol).
-export([encode/1, decode/1, decode_stream/2]).

-define(MAGIC, 16#AB).
-define(HEADER_SIZE, 14).   %% magic(1)+vsn(1)+type(1)+flags(1)+seq(4)+len(4)+crc(4) = 16... adjust

%% Message types
-define(REQ, 1).
-define(RESP, 2).
-define(NOTIFY, 3).
-define(ERROR, 4).

-record(frame, {
    version  = 1,
    type,
    flags    = 0,
    sequence = 0,
    payload  = <<>>
}).

encode(#frame{version=V, type=T, flags=F, sequence=Seq, payload=P}) ->
    Len  = byte_size(P),
    Head = <<?MAGIC:8, V:8, T:8, F:8, Seq:32/big, Len:32/big>>,
    Body = <<Head/binary, P/binary>>,
    CRC  = erlang:crc32(Body),
    <<Body/binary, CRC:32/big>>.

decode(<<16#AB:8, V:8, T:8, F:8, Seq:32/big, Len:32/big,
         Payload:Len/binary, CRC:32/big, _Rest/binary>>) ->
    %% Verify checksum
    Head = <<16#AB:8, V:8, T:8, F:8, Seq:32/big, Len:32/big, Payload/binary>>,
    case erlang:crc32(Head) of
        CRC ->
            {ok, #frame{version=V, type=T, flags=F,
                        sequence=Seq, payload=Payload}};
        _ ->
            {error, checksum_mismatch}
    end;
decode(Bin) when byte_size(Bin) < ?HEADER_SIZE ->
    {more, Bin};
decode(_) ->
    {error, invalid_frame}.

%% Streaming decode: handle partial frames
decode_stream(Buffer, Acc) ->
    case decode(Buffer) of
        {ok, Frame} ->
            %% Calculate consumed bytes
            Consumed = ?HEADER_SIZE + byte_size(Frame#frame.payload),
            Rest = binary:part(Buffer, Consumed, byte_size(Buffer) - Consumed),
            decode_stream(Rest, [Frame | Acc]);
        {more, _} ->
            {lists:reverse(Acc), Buffer};
        {error, _} ->
            %% Skip one byte and try again (framing error recovery)
            <<_:8, Rest/binary>> = Buffer,
            decode_stream(Rest, Acc)
    end.
```

---

## 3. Custom TCP Protocol

```erlang
%% proto_server.erl — TCP server with custom protocol
-module(proto_server).
-behaviour(gen_server).
-export([start_link/1]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

-record(state, {
    socket,
    buffer = <<>>,
    seq    = 0
}).

start_link(Port) ->
    gen_server:start_link(?MODULE, Port, []).

init(Port) ->
    {ok, LSock} = gen_tcp:listen(Port, [
        binary,
        {packet, raw},   %% raw bytes, we handle framing
        {active, false},
        {reuseaddr, true},
        {backlog, 128}
    ]),
    %% Accept in a separate process
    spawn_link(fun() -> accept_loop(LSock) end),
    {ok, #state{}}.

accept_loop(LSock) ->
    case gen_tcp:accept(LSock) of
        {ok, Sock} ->
            {ok, Pid} = proto_conn_sup:start_child(Sock),
            gen_tcp:controlling_process(Sock, Pid),
            accept_loop(LSock);
        {error, closed} ->
            ok
    end.

handle_info({tcp, Sock, Data},
            #state{socket=Sock, buffer=Buf} = S) ->
    NewBuf = <<Buf/binary, Data/binary>>,
    {Frames, Remaining} = protocol:decode_stream(NewBuf, []),
    NewState = lists:foldl(fun(Frame, Acc) ->
        handle_frame(Frame, Acc)
    end, S#state{buffer=Remaining}, Frames),
    {noreply, NewState};

handle_info({tcp_closed, _}, S) ->
    {stop, normal, S}.

handle_frame(#frame{type=?REQ, sequence=Seq, payload=P}, S) ->
    Response = process_request(P),
    Frame    = #frame{type=?RESP, sequence=Seq, payload=Response},
    gen_tcp:send(S#state.socket, protocol:encode(Frame)),
    S.

process_request(Payload) ->
    jsx:encode(#{result => <<"ok">>, echo => Payload}).

handle_call(_, _, S) -> {reply, ok, S}.
handle_cast(_, S)    -> {noreply, S}.
```

---

## 4. Frame Encoding/Decoding

```erlang
%% Various binary encoding patterns in Erlang

%% Length-prefixed strings
encode_string(Str) when is_binary(Str) ->
    Len = byte_size(Str),
    <<Len:16/big, Str/binary>>.

decode_string(<<Len:16/big, Str:Len/binary, Rest/binary>>) ->
    {Str, Rest}.

%% Variable-length integer (protobuf-style varint)
encode_varint(N) when N < 128   -> <<N:8>>;
encode_varint(N) ->
    <<(128 + (N band 127)):8, (encode_varint(N bsr 7))/binary>>.

decode_varint(<<0:1, N:7, Rest/binary>>) ->
    {N, Rest};
decode_varint(<<1:1, N:7, Rest/binary>>) ->
    {Cont, Rest2} = decode_varint(Rest),
    {N + (Cont bsl 7), Rest2}.

%% Fixed-size struct encoding
-record(packet_header, {cmd, flags, length}).

encode_header(#packet_header{cmd=C, flags=F, length=L}) ->
    <<C:16/big, F:8, L:32/big>>.

decode_header(<<C:16/big, F:8, L:32/big, Rest/binary>>) ->
    {#packet_header{cmd=C, flags=F, length=L}, Rest}.

%% Bit flags
-define(FLAG_COMPRESSED, 1).
-define(FLAG_ENCRYPTED,  2).
-define(FLAG_URGENT,     4).

set_flag(Flags, Flag)   -> Flags bor Flag.
clear_flag(Flags, Flag) -> Flags band (bnot Flag).
has_flag(Flags, Flag)   -> (Flags band Flag) =/= 0.
```

---

## 5. Protocol Versioning

```erlang
%% proto_compat.erl — handle multiple protocol versions
-module(proto_compat).
-export([decode_message/2, encode_response/3]).

decode_message(1, Payload) ->
    %% v1: simple JSON
    jsx:decode(Payload, [return_maps]);

decode_message(2, Payload) ->
    %% v2: MessagePack (more compact)
    msgpack:unpack(Payload);

decode_message(Version, _) ->
    {error, {unsupported_version, Version}}.

encode_response(1, SeqNum, Data) ->
    Body = jsx:encode(Data),
    protocol:encode(#frame{version=1, type=?RESP,
                           sequence=SeqNum, payload=Body});

encode_response(2, SeqNum, Data) ->
    Body = msgpack:pack(Data),
    protocol:encode(#frame{version=2, type=?RESP,
                           sequence=SeqNum, payload=Body}).

%% Negotiate version during handshake
negotiate(ClientVersions, ServerVersions) ->
    Common = [V || V <- ClientVersions, lists:member(V, ServerVersions)],
    case lists:sort(fun(A, B) -> A > B end, Common) of
        [Best | _] -> {ok, Best};
        []         -> {error, no_common_version}
    end.
```

---

## 6. WebSocket Binary Messages

```erlang
%% ws_binary_handler.erl — efficient binary messages over WebSocket
-module(ws_binary_handler).
-behaviour(cowboy_websocket).
-export([init/2, websocket_handle/2, websocket_info/2]).

init(Req, State) ->
    {cowboy_websocket, Req, State,
     #{compress => true, max_frame_size => 65536}}.

%% Handle binary frame
websocket_handle({binary, Data}, State) ->
    case decode_binary_message(Data) of
        {ok, {Type, SeqNum, Payload}} ->
            Response = handle_message(Type, Payload),
            Reply = encode_binary_response(SeqNum, Response),
            {reply, {binary, Reply}, State};
        {error, Reason} ->
            Error = encode_error(Reason),
            {reply, {binary, Error}, State}
    end;

%% Text fallback
websocket_handle({text, Data}, State) ->
    Msg = jsx:decode(Data, [return_maps]),
    {reply, {text, jsx:encode(#{status => <<"ok">>})}, State}.

websocket_info(_, State) -> {ok, State}.

decode_binary_message(<<Type:8, SeqNum:32/big, Len:32/big,
                         Payload:Len/binary, _/binary>>) ->
    {ok, {Type, SeqNum, Payload}};
decode_binary_message(_) ->
    {error, invalid_message}.

encode_binary_response(SeqNum, Data) ->
    Payload = jsx:encode(Data),
    Len = byte_size(Payload),
    <<16#02:8, SeqNum:32/big, Len:32/big, Payload/binary>>.

encode_error(Reason) ->
    Msg = jsx:encode(#{error => atom_to_binary(Reason, utf8)}),
    Len = byte_size(Msg),
    <<16#FF:8, 0:32, Len:32/big, Msg/binary>>.

handle_message(1, Payload) ->
    Data = jsx:decode(Payload, [return_maps]),
    #{result => <<"processed">>, input => Data};
handle_message(_, _) ->
    #{error => <<"unknown_type">>}.
```

---

## 7. แบบฝึกหัด

1. Implement `encode_varint/1` และ test กับค่า: 0, 127, 128, 300, 16383, 16384
2. เพิ่ม compression flag: ถ้า payload > 1KB ให้ compress ด้วย zlib
3. สร้าง protocol fuzzer: generate random bytes และ ensure decode ไม่ crash
4. Implement heartbeat mechanism: client/server ส่ง PING/PONG ทุก 30 วิ

---

## สรุป Part 45

✅ Binary protocol frame design  
✅ Binary pattern matching สำหรับ parsing  
✅ Custom TCP server  
✅ Frame encoding/decoding patterns  
✅ Protocol versioning  
✅ WebSocket binary messages  

---

*Part 45/100 | [← ก่อนหน้า](../part44/README.md) | [ถัดไป →](../part46/README.md)*
