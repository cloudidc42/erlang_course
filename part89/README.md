# Part 89: Contributing to OTP Internals

> **"The best way to understand a system is to change it — carefully"**  
> วิธีที่ดีที่สุดในการเข้าใจระบบคือการเปลี่ยนมัน — อย่างระมัดระวัง

---

## สารบัญ

1. [OTP Source Structure](#1-otp-source-structure)
2. [Understanding BEAM Internals](#2-understanding-beam-internals)
3. [Reading gen_server Source](#3-reading-gen_server-source)
4. [Writing OTP Documentation](#4-writing-otp-documentation)
5. [Submitting a Bug Report](#5-submitting-a-bug-report)
6. [Writing an EEP (Erlang Enhancement Proposal)](#6-writing-an-eep-erlang-enhancement-proposal)
7. [แบบฝึกหัด](#7-แบบฝึกหัด)

---

## 1. OTP Source Structure

```
OTP Repository Layout
════════════════════════════════════════════════════

github.com/erlang/otp
├── erts/              BEAM virtual machine (C source)
│   ├── emulator/      Core VM: process scheduling, GC, port drivers
│   │   ├── beam/      Bytecode interpreter
│   │   └── sys/       Platform-specific: unix/, win32/
│   └── epmd/          Erlang Port Mapper Daemon
│
├── lib/               OTP standard library (Erlang source)
│   ├── stdlib/        lists, maps, queue, ets, etc.
│   ├── kernel/        Application, net_kernel, logger
│   ├── sasl/          System Architecture Support (releases)
│   ├── compiler/      Erlang → bytecode compiler
│   ├── crypto/        NIF wrapper around OpenSSL
│   └── ssl/           TLS/DTLS implementation
│
├── system/            Documentation infrastructure
│   └── doc/           Reference manual Markdown sources
│
└── make/              Build system, cross-compilation

Key files for OTP behavior understanding:
  lib/stdlib/src/gen_server.erl     gen_server implementation
  lib/stdlib/src/supervisor.erl     supervisor strategy logic
  lib/kernel/src/application.erl   application controller
  erts/emulator/beam/erl_process.c  process scheduler (C)
  erts/emulator/beam/erl_gc.c       garbage collector (C)
```

---

## 2. Understanding BEAM Internals

```erlang
%% beam_concepts.erl — code that reveals BEAM internals
-module(beam_concepts).
-export([process_info_deep/0, scheduler_info/0, gc_stats/0]).

%% Explore what BEAM tracks per-process
process_info_deep() ->
    Pid = self(),
    Keys = [
        status,          %% running | waiting | suspended
        current_function,%% {M, F, A}
        initial_call,    %% how the process was started
        message_queue_len,
        messages,
        links,
        monitors,
        memory,          %% total memory in bytes
        heap_size,       %% words in heap (not bytes)
        stack_size,      %% words in stack
        reductions,      %% work units completed
        garbage_collection %% GC settings and stats
    ],
    [{K, process_info(Pid, K)} || K <- Keys].

%% Scheduler utilization (how busy each scheduler thread is)
scheduler_info() ->
    %% Enable scheduler wall time tracking
    erlang:system_flag(scheduler_wall_time, true),
    T1 = erlang:statistics(scheduler_wall_time),
    timer:sleep(1000),
    T2 = erlang:statistics(scheduler_wall_time),
    %% Calculate utilization per scheduler
    [{SchedId,
      ActiveUs / (ActiveUs + IdleUs)}
     || {{SchedId, A1, T1_total}, {SchedId, A2, T2_total}} <-
            lists:zip(lists:sort(T1), lists:sort(T2)),
        ActiveUs = A2 - A1,
        IdleUs = T2_total - T1_total - ActiveUs,
        IdleUs + ActiveUs > 0].

%% GC statistics for a process
gc_stats() ->
    {GcCount, WordsReclaimed, 0} = erlang:statistics(garbage_collection),
    #{
        gc_count         => GcCount,
        words_reclaimed  => WordsReclaimed,
        process_memory   => erlang:memory(processes),
        atom_memory      => erlang:memory(atom),
        binary_memory    => erlang:memory(binary),
        ets_memory       => erlang:memory(ets)
    }.

%% Trace a process to see message passing and scheduling
trace_process(Pid) ->
    erlang:trace(Pid, true, [
        send,           %% when this process sends a message
        'receive',      %% when this process receives a message
        procs,          %% spawn, exit, link, unlink events
        running,        %% when scheduled on/off CPU
        garbage_collection  %% GC events
    ]).
```

---

## 3. Reading gen_server Source

```erlang
%% Simplified version of gen_server:loop/7 to understand the core loop
%% This is educational — the real source is in lib/stdlib/src/gen_server.erl

%% The main receive loop that every gen_server runs
loop(Parent, Name, State, Mod, hibernate, _Dbg, Hib) ->
    %% Hibernate: release heap memory while waiting for message
    proc_lib:hibernate(?MODULE, wake_hib, [Parent, Name, State, Mod, _Dbg, Hib]);

loop(Parent, Name, State, Mod, infinity, Dbg, Hib) ->
    receive
        Msg ->
            decode_msg(Msg, Parent, Name, State, Mod, infinity, Dbg, Hib)
    end;

loop(Parent, Name, State, Mod, Time, Dbg, Hib) ->
    Msg = receive
        Input -> Input
    after Time ->
        timeout
    end,
    decode_msg(Msg, Parent, Name, State, Mod, Time, Dbg, Hib).

%% Decode the message type and dispatch to appropriate callback
decode_msg(Msg, Parent, Name, State, Mod, Time, Dbg, Hib) ->
    case Msg of
        {system, From, Req} ->
            %% System messages: suspend, resume, get_state, change_code
            sys:handle_system_msg(Req, From, Parent, ?MODULE,
                                  Dbg, [Name, State, Mod, Time, Hib]);
        {'EXIT', Parent, Reason} ->
            %% Parent died: call terminate and exit
            terminate(Reason, Name, undefined, Mod, State, Dbg);
        _Msg ->
            handle_msg(Msg, Parent, Name, State, Mod, Time, Dbg, Hib)
    end.

handle_msg({call, From, Msg}, Parent, Name, State, Mod, _Time, Dbg, Hib) ->
    %% Synchronous call
    Result = try Mod:handle_call(Msg, From, State) catch C:R -> {error, {C, R}} end,
    case Result of
        {reply, Reply, NState} ->
            reply(From, Reply),
            loop(Parent, Name, NState, Mod, infinity, Dbg, Hib);
        {reply, Reply, NState, TimeoutOrHib} ->
            reply(From, Reply),
            loop(Parent, Name, NState, Mod, TimeoutOrHib, Dbg, Hib);
        {noreply, NState} ->
            loop(Parent, Name, NState, Mod, infinity, Dbg, Hib);
        {stop, Reason, Reply, NState} ->
            reply(From, Reply),
            terminate(Reason, Name, Msg, Mod, NState, Dbg)
    end.
```

---

## 4. Writing OTP Documentation

```erlang
%% Example of EDoc-style documentation that meets OTP standards
%% OTP docs use .xml format, but here we show the intent

-module(my_counter).

%% @doc A simple counter process.
%%
%% Provides a persistent counter that survives process restarts.
%% The counter value is stored in ETS for crash resilience.
%%
%% == Example ==
%% ```
%% {ok, _} = my_counter:start_link(my_count, 0),
%% ok = my_counter:increment(my_count),
%% {ok, 1} = my_counter:get(my_count).
%% '''
%%
%% @end

-behaviour(gen_server).

-export([start_link/2, increment/1, increment_by/2, get/1, reset/1]).
-export([init/1, handle_call/3, handle_cast/2]).

%% @doc Start a named counter with an initial value.
%%
%% Returns `{ok, Pid}' if successful, or
%% `{error, {already_started, Pid}}' if a counter with this name exists.
-spec start_link(Name :: atom(), Initial :: non_neg_integer()) ->
    {ok, pid()} | {error, term()}.
start_link(Name, Initial) ->
    gen_server:start_link({local, Name}, ?MODULE, Initial, []).

%% @doc Increment the counter by 1.
-spec increment(Name :: atom()) -> ok.
increment(Name) -> increment_by(Name, 1).

%% @doc Increment the counter by N.
-spec increment_by(Name :: atom(), N :: integer()) -> ok.
increment_by(Name, N) ->
    gen_server:cast(Name, {increment, N}).

%% @doc Get the current counter value.
-spec get(Name :: atom()) -> {ok, integer()}.
get(Name) ->
    gen_server:call(Name, get).

%% @doc Reset the counter to 0.
-spec reset(Name :: atom()) -> ok.
reset(Name) ->
    gen_server:cast(Name, reset).

init(Initial) ->
    {ok, Initial}.

handle_call(get, _From, Value) ->
    {reply, {ok, Value}, Value}.

handle_cast({increment, N}, Value) ->
    {noreply, Value + N};
handle_cast(reset, _Value) ->
    {noreply, 0}.
```

---

## 5. Submitting a Bug Report

```
How to write a good Erlang/OTP bug report (bugs.erlang.org / GitHub Issues)
════════════════════════════════════════════════════════════════════════════

REQUIRED INFORMATION:

1. Erlang/OTP version:
   $ erl -eval 'erlang:display(erlang:system_info(otp_release)).' -noshell -s init stop
   → 27

2. Operating system and architecture:
   Linux x86_64, macOS arm64, Windows x64...

3. Minimal reproducible example:
   GOOD: 10-line script that reliably shows the bug
   BAD:  "our production system crashes sometimes"

4. Expected vs actual behavior:
   Expected: lists:sort([3,1,2]) returns [1,2,3]
   Actual:   crash with badarg

5. Crash dump or error output:
   Include full stack trace from error_logger or crash dump

MINIMAL REPRODUCTION SCRIPT TEMPLATE:
```

```erlang
%% bug_repro.erl — always provide a self-contained reproduction
-module(bug_repro).
-export([reproduce/0]).

reproduce() ->
    %% Setup state that triggers the bug
    {ok, Pid} = gen_server:start(some_module, [], []),

    %% Action that triggers the bug
    Result = gen_server:call(Pid, some_request),

    %% What we expected vs what we got
    Expected = {ok, 42},
    case Result =:= Expected of
        true  -> io:format("OK~n");
        false -> io:format("FAIL: expected ~p, got ~p~n", [Expected, Result])
    end.
```

---

## 6. Writing an EEP (Erlang Enhancement Proposal)

```
EEP Format (based on PEP / RFC conventions)
════════════════════════════════════════════════════

EEP: <number>
Title: <descriptive title>
Author: Your Name <email>
Status: Draft | Accepted | Final | Rejected | Withdrawn
Type: Standards Track | Informational | Process
Created: YYYY-MM-DD
Erlang-Version: 27+
Post-History: <links to mailing list discussion>

Abstract:
  One paragraph summary of the proposed change.

Rationale:
  Why is this change needed? What problem does it solve?
  Show concrete examples of current pain points.

Specification:
  Precise, technical description of the proposed change.
  Include new syntax, semantic rules, error conditions.

Backwards Compatibility:
  Does this break existing code? How can users migrate?
  "Fully backwards compatible" or "Requires code change when..."

Reference Implementation:
  Link to prototype implementation or proof-of-concept.
  Not required for Draft status, required for Accepted.

Example EEP concept — process labels:

  %% Currently: processes are identified by PID or registered name
  Pid = spawn(fun() -> ... end),
  erlang:register(my_worker, Pid),  %% global name, collision risk

  %% Proposed: arbitrary key-value labels attached to processes
  Pid = spawn(fun() -> ... end),
  erlang:set_label(Pid, #{service => payment, tenant => <<"acme">>}),

  %% Query: find all processes with a given label
  Workers = erlang:find_labeled(#{service => payment}),

  %% This would help debugging, monitoring, and structured logging
  %% Status: EEP draft (not yet submitted)
```

---

## 7. แบบฝึกหัด

1. Clone OTP repo, find `supervisor.erl`, trace how `one_for_one` restart works
2. เพิ่ม debug logging ไปยัง `gen_server` ของตัวเอง เพื่อจำลอง sys:trace ที่มีอยู่
3. เขียน minimal bug report สำหรับ edge case ที่คุณพบในโปรเจกต์ของตัวเอง
4. ร่าง EEP สั้นๆ สำหรับ Erlang feature ที่คุณอยากให้มี

---

## สรุป Part 89

✅ OTP repository structure: erts (C), lib (Erlang), doc layout  
✅ BEAM internals: process_info, scheduler_wall_time, GC statistics  
✅ gen_server source: decode_msg and handle_msg loop mechanics  
✅ EDoc-style documentation: specs, examples, returns  
✅ Bug reporting: version, minimal reproduction, expected vs actual  
✅ EEP format: abstract, rationale, spec, backwards compat  

---

*Part 89/100 | [← ก่อนหน้า](../part88/README.md) | [ถัดไป →](../part90/README.md)*
