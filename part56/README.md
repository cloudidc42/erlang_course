# Part 56: Elixir Interoperability

> **"Erlang and Elixir are siblings — they share the same powerful BEAM"**  
> Erlang และ Elixir เป็นพี่น้องกัน — ใช้ BEAM อันทรงพลังร่วมกัน

---

## สารบัญ

1. [Calling Erlang from Elixir](#1-calling-erlang-from-elixir)
2. [Calling Elixir from Erlang](#2-calling-elixir-from-erlang)
3. [Shared BEAM Node](#3-shared-beam-node)
4. [Mix vs rebar3](#4-mix-vs-rebar3)
5. [Phoenix Channels from Erlang](#5-phoenix-channels-from-erlang)
6. [แบบฝึกหัด](#6-แบบฝึกหัด)

---

## 1. Calling Erlang from Elixir

```elixir
# Calling Erlang modules directly from Elixir
# Erlang module names become atoms prefixed with :

# Call :crypto (Erlang crypto module)
:crypto.hash(:sha256, "hello")

# Call :ets
:ets.new(:my_table, [:named_table, :set])
:ets.insert(:my_table, {:key, :value})
:ets.lookup(:my_table, :key)

# Call custom Erlang module
# If you have Erlang module 'my_module.erl':
:my_module.some_function(arg1, arg2)

# Erlang records in Elixir (must use record_info)
# In Erlang: -record(user, {id, name}).
# In Elixir, access as tuple:
user_tuple = {:user, 1, "Alice"}
# Or use Erlang record accessors via :erlang.element/2
```

---

## 2. Calling Elixir from Erlang

```erlang
%% Elixir module names are Atoms that start with 'Elixir.'
%% Module Enum → 'Elixir.Enum'
%% Module MyApp.Worker → 'Elixir.MyApp.Worker'

call_elixir_enum() ->
    List = [3, 1, 4, 1, 5, 9, 2, 6],
    Sorted = 'Elixir.Enum':sort(List),
    Filtered = 'Elixir.Enum':filter(List, fun(X) -> X > 3 end),
    #{sorted => Sorted, filtered => Filtered}.

call_phoenix_pubsub(Topic, Message) ->
    'Elixir.Phoenix.PubSub':broadcast(
        'Elixir.MyApp.PubSub',
        Topic,
        Message
    ).

%% Note: Elixir strings are UTF-8 binaries (same as Erlang binaries)
%% Elixir atoms with special chars: already compatible
%% Elixir structs: maps with __struct__ key

decode_elixir_struct(Struct) ->
    case maps:find('__struct__', Struct) of
        {ok, ModuleName} ->
            io:format("Elixir struct of type: ~p~n", [ModuleName]),
            maps:remove('__struct__', Struct);
        error ->
            Struct  %% plain map
    end.
```

---

## 3. Shared BEAM Node

```erlang
%% Running Erlang and Elixir code on the same BEAM node

%% Start a node that can load both:
%% erl -name mixed@127.0.0.1 -pa _build/dev/lib/*/ebin

%% From Erlang, call Elixir's Logger
log_from_erlang(Level, Message) ->
    'Elixir.Logger':log(Level, Message).

%% From Erlang, use Elixir's String module
capitalize(Str) when is_binary(Str) ->
    'Elixir.String':capitalize(Str).

%% Shared ETS tables: both Erlang and Elixir processes can access
%% ETS tables are process-independent — any process on the node can use them

%% Connecting an Erlang node to a running Elixir/Phoenix app
connect_to_phoenix() ->
    Node = 'myapp@hostname',
    case net_kernel:connect_node(Node) of
        true ->
            %% Can now call Phoenix modules
            'Elixir.Phoenix.PubSub':broadcast(
                {via, 'Elixir.Registry', {'Elixir.MyApp.PubSub', :unknown}},
                <<"topic">>,
                #{event => <<"hello">>}
            );
        false ->
            {error, connection_failed}
    end.
```

---

## 4. Mix vs rebar3

```
Mix (Elixir):                    rebar3 (Erlang):
─────────────────────────────────────────────────
mix new myapp                    rebar3 new app myapp
mix deps.get                     rebar3 deps
mix compile                      rebar3 compile
mix test                         rebar3 eunit / ct
mix release                      rebar3 release
mix run                          rebar3 shell

Both compile to BEAM bytecode (.beam files)
Both use hex.pm for packages
Both support hot code upgrades
```

```elixir
# mix.exs — using Erlang library in Elixir project
defmodule MyApp.MixProject do
  use Mix.Project

  def project do
    [
      app: :my_app,
      version: "1.0.0",
      elixir: "~> 1.15",
      deps: deps()
    ]
  end

  defp deps do
    [
      {:cowboy, "~> 2.10"},   # Erlang lib works in Elixir too!
      {:poolboy, "~> 1.5"},
      {:phoenix, "~> 1.7"}
    ]
  end
end
```

---

## 5. Phoenix Channels from Erlang

```erlang
%% Connect to Phoenix WebSocket channels from Erlang
%% Using gun HTTP/2 + WebSocket client

-module(phoenix_client).
-export([connect/2, join_channel/3, push/4]).

connect(Host, Port) ->
    {ok, Conn} = gun:open(Host, Port, #{protocols => [http]}),
    {ok, http} = gun:await_up(Conn),
    gun:ws_upgrade(Conn, "/socket/websocket"),
    receive
        {gun_upgrade, Conn, StreamRef, [<<"websocket">>], _} ->
            {ok, Conn, StreamRef}
    after 5000 ->
        {error, timeout}
    end.

join_channel(Conn, StreamRef, Topic) ->
    Msg = jsx:encode([
        null,           %% join_ref
        <<"1">>,        %% ref
        Topic,
        <<"phx_join">>,
        #{}             %% payload
    ]),
    gun:ws_send(Conn, StreamRef, {text, Msg}).

push(Conn, StreamRef, Topic, Payload) ->
    Msg = jsx:encode([
        null,
        <<"2">>,
        Topic,
        <<"new_msg">>,
        Payload
    ]),
    gun:ws_send(Conn, StreamRef, {text, Msg}).

receive_messages(Conn, StreamRef) ->
    receive
        {gun_ws, Conn, StreamRef, {text, Data}} ->
            case jsx:decode(Data, [return_maps]) of
                [_, _, Topic, <<"phx_reply">>, Payload] ->
                    {reply, Topic, Payload};
                [_, _, Topic, Event, Payload] ->
                    {event, Topic, Event, Payload}
            end
    after 5000 ->
        timeout
    end.
```

---

## 6. แบบฝึกหัด

1. สร้าง Erlang module ที่ call Elixir's `Enum.group_by/2`
2. Share an ETS table ระหว่าง Erlang GenServer และ Elixir GenServer บน node เดียวกัน
3. เชื่อมต่อ Erlang node ไปยัง Phoenix PubSub และ broadcast message
4. เขียน mix task ที่ generate Erlang record definitions จาก Elixir struct

---

## สรุป Part 56

✅ เรียก Erlang จาก Elixir ด้วย atom prefix  
✅ เรียก Elixir จาก Erlang ด้วย 'Elixir.Module'  
✅ Shared BEAM node  
✅ Mix vs rebar3 comparison  
✅ Phoenix Channels client จาก Erlang  

---

*Part 56/100 | [← ก่อนหน้า](../part55/README.md) | [ถัดไป →](../part57/README.md)*
