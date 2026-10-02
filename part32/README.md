# Part 32: NIFs และ Ports — Calling C from Erlang

> **"When Erlang is not fast enough — bring C into the mix"**  
> เมื่อ Erlang ไม่เร็วพอ — นำ C เข้ามาช่วย

---

## สารบัญ

1. [NIF vs Port vs Port Driver](#1-nif-vs-port-vs-port-driver)
2. [Erlang Ports พื้นฐาน](#2-erlang-ports-พื้นฐาน)
3. [Port Commands และ Protocol](#3-port-commands-และ-protocol)
4. [Writing a NIF](#4-writing-a-nif)
5. [NIF Safety และ Scheduling](#5-nif-safety-และ-scheduling)
6. [Dirty NIFs](#6-dirty-nifs)
7. [enif API Overview](#7-enif-api-overview)
8. [รูปแบบที่นิยมใช้](#8-รูปแบบที่นิยมใช้)
9. [แบบฝึกหัด](#9-แบบฝึกหัด)

---

## 1. NIF vs Port vs Port Driver

```
NIF (Native Implemented Function):
  ├── C function runs IN the Erlang VM
  ├── เร็วมาก — no copy overhead
  ├── อันตราย: crash ทำให้ VM crash ทั้งหมด
  ├── ต้อง return ใน < 1ms (ไม่เช่นนั้น scheduler stall)
  └── ใช้สำหรับ: crypto, JSON parsing, image processing

Port:
  ├── Separate OS process
  ├── Communicate via stdin/stdout
  ├── ปลอดภัย: crash ไม่กระทบ VM
  ├── Slow: process + pipe overhead
  └── ใช้สำหรับ: legacy C programs, shell commands

Port Driver:
  ├── C code loaded as shared library
  ├── Runs in VM process (เร็วกว่า Port)
  ├── อันตรายน้อยกว่า NIF แต่ยังไม่ safe 100%
  └── ใช้ใน: database drivers รุ่นเก่า
```

---

## 2. Erlang Ports พื้นฐาน

```erlang
%% เปิด external program เป็น port
Port = open_port({spawn, "cat"}, [binary, {packet, 2}]).

%% ส่งข้อมูล
port_command(Port, <<"Hello">>).

%% รับข้อมูล
receive
    {Port, {data, Data}} ->
        io:format("Got: ~p~n", [Data])
end.

%% ปิด port
port_close(Port).

%% Port options:
%% binary          — data as binaries
%% {packet, N}     — length-prefixed packets (1,2,4 bytes)
%% stream          — no framing
%% {line, MaxLine} — line-based
%% use_stdio       — use stdin/stdout
%% nouse_stdio     — use FD 3,4 instead
%% exit_status     — notify when port program exits
%% {cd, Dir}       — working directory
%% {env, Env}      — environment variables

%% Port as GenServer
-module(port_server).
-behaviour(gen_server).

-export([start_link/0, call/1]).
-export([init/1, handle_call/3, handle_info/2]).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

call(Data) ->
    gen_server:call(?MODULE, {call, Data}).

init([]) ->
    Port = open_port({spawn, "my_c_program"}, [binary, {packet, 4}]),
    {ok, #{port => Port, pending => #{}}}.

handle_call({call, Data}, From, #{port := Port, pending := P} = S) ->
    Ref = make_ref(),
    port_command(Port, <<Ref:64, Data/binary>>),
    {noreply, S#{pending := P#{Ref => From}}};

handle_info({Port, {data, <<Ref:64, Result/binary>>}},
            #{port := Port, pending := P} = S) ->
    case maps:take(Ref, P) of
        {From, P2} ->
            gen_server:reply(From, {ok, Result}),
            {noreply, S#{pending := P2}};
        error ->
            {noreply, S}
    end.
```

---

## 3. Port Commands และ Protocol

```c
/* my_program.c — C side of an Erlang port */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdint.h>

/* Read N bytes, handling partial reads */
static int read_exact(unsigned char *buf, int len) {
    int got = 0;
    while (got < len) {
        int n = read(0, buf + got, len - got);
        if (n <= 0) return n;
        got += n;
    }
    return got;
}

/* Write N bytes with length prefix (4 bytes, big-endian) */
static int write_cmd(unsigned char *buf, int len) {
    unsigned char hd[4];
    hd[0] = (len >> 24) & 0xff;
    hd[1] = (len >> 16) & 0xff;
    hd[2] = (len >> 8)  & 0xff;
    hd[3] = (len)       & 0xff;
    write(1, hd, 4);
    write(1, buf, len);
    return len;
}

int main() {
    unsigned char hd[4];
    while (read_exact(hd, 4) == 4) {
        int len = (hd[0]<<24)|(hd[1]<<16)|(hd[2]<<8)|hd[3];
        unsigned char *buf = malloc(len);
        if (read_exact(buf, len) != len) break;

        /* Process: uppercase everything */
        for (int i = 0; i < len; i++) {
            if (buf[i] >= 'a' && buf[i] <= 'z')
                buf[i] -= 32;
        }

        write_cmd(buf, len);
        free(buf);
    }
    return 0;
}
```

---

## 4. Writing a NIF

```c
/* my_nif.c */
#include "erl_nif.h"
#include <string.h>

/* Simple NIF: reverse a binary */
static ERL_NIF_TERM reverse_binary(ErlNifEnv *env, int argc,
                                    const ERL_NIF_TERM argv[]) {
    ErlNifBinary bin, result;

    if (!enif_inspect_binary(env, argv[0], &bin))
        return enif_make_badarg(env);

    if (!enif_alloc_binary(bin.size, &result))
        return enif_make_atom(env, "error");

    for (size_t i = 0; i < bin.size; i++)
        result.data[i] = bin.data[bin.size - 1 - i];

    return enif_make_binary(env, &result);
}

/* NIF table */
static ErlNifFunc nif_funcs[] = {
    {"reverse_binary", 1, reverse_binary}
};

ERL_NIF_INIT(my_nif, nif_funcs, NULL, NULL, NULL, NULL)
```

```erlang
%% my_nif.erl — Erlang wrapper
-module(my_nif).
-export([reverse_binary/1]).
-on_load(init/0).

init() ->
    PrivDir = code:priv_dir(myapp),
    SoPath = filename:join(PrivDir, "my_nif"),
    erlang:load_nif(SoPath, 0).

%% Fallback if NIF not loaded
reverse_binary(_Bin) ->
    erlang:nif_error(nif_not_loaded).
```

```makefile
# Makefile
CFLAGS = $(shell erl -noshell -eval 'io:format("~s", [code:lib_dir(erl_interface)])' -s init stop)/include
EI_LIB = $(shell erl -noshell -eval 'io:format("~s", [code:lib_dir(erl_interface)])' -s init stop)/lib

priv/my_nif.so: c_src/my_nif.c
    $(CC) -shared -fPIC -O2 \
        -I $(CFLAGS) \
        -o $@ $<
```

---

## 5. NIF Safety และ Scheduling

```c
/* NIF ต้อง return ใน < 1ms
   ไม่เช่นนั้น scheduler stall และ soft-realtime guarantees หาย */

/* BAD: long-running NIF */
static ERL_NIF_TERM bad_nif(ErlNifEnv *env, int argc,
                             const ERL_NIF_TERM argv[]) {
    /* This takes 100ms — blocks scheduler! */
    slow_computation();
    return enif_make_atom(env, "ok");
}

/* GOOD: break work into chunks with yielding */
static ERL_NIF_TERM chunked_nif(ErlNifEnv *env, int argc,
                                 const ERL_NIF_TERM argv[]) {
    /* Process in small chunks */
    for (int i = 0; i < 1000; i++) {
        /* do small unit of work */
        process_chunk(i);
    }
    /* For truly long work, use dirty NIFs (see below) */
    return enif_make_atom(env, "ok");
}
```

---

## 6. Dirty NIFs

```c
/* Dirty NIFs สำหรับงานที่ใช้เวลานาน
   ไม่ block scheduler ปกติ */

#include "erl_nif.h"

/* Dirty CPU NIF: heavy computation */
static ERL_NIF_TERM heavy_cpu_work(ErlNifEnv *env, int argc,
                                    const ERL_NIF_TERM argv[]) {
    /* This can take seconds — runs on dirty CPU scheduler */
    perform_heavy_computation();
    return enif_make_atom(env, "ok");
}

/* Dirty IO NIF: blocking I/O */
static ERL_NIF_TERM blocking_read(ErlNifEnv *env, int argc,
                                   const ERL_NIF_TERM argv[]) {
    /* Blocking syscall — runs on dirty I/O scheduler */
    char buf[1024];
    ssize_t n = read(fd, buf, sizeof(buf));
    return enif_make_binary_from_data(env, (unsigned char*)buf, n);
}

/* Mark as dirty in NIF table */
static ErlNifFunc nif_funcs[] = {
    {"heavy_cpu_work", 0, heavy_cpu_work, ERL_NIF_DIRTY_JOB_CPU_BOUND},
    {"blocking_read",  1, blocking_read,  ERL_NIF_DIRTY_JOB_IO_BOUND}
};
```

---

## 7. enif API Overview

```c
/* Types */
ERL_NIF_TERM   — Erlang term opaque handle
ErlNifEnv     — Environment (memory pool per call)
ErlNifBinary  — Binary data {size, data}

/* Inspect (decode) */
enif_inspect_binary(env, term, &bin)  — binary
enif_get_int(env, term, &i)           — integer
enif_get_long(env, term, &l)          — long
enif_get_double(env, term, &d)        — double
enif_get_atom(env, term, buf, size, ERL_NIF_LATIN1)
enif_get_list_cell(env, list, &head, &tail)
enif_get_map_value(env, map, key, &value)

/* Make (encode) */
enif_make_atom(env, "ok")
enif_make_int(env, 42)
enif_make_double(env, 3.14)
enif_make_binary(env, &bin)
enif_make_tuple2(env, a, b)
enif_make_list3(env, a, b, c)
enif_make_badarg(env)  — raise badarg

/* Memory */
enif_alloc(size)
enif_free(ptr)
enif_alloc_binary(size, &bin)
enif_release_binary(&bin)

/* Resources (managed C objects) */
ErlNifResourceType *rt = enif_open_resource_type(
    env, NULL, "my_resource", destructor_fn,
    ERL_NIF_RT_CREATE, NULL);
void *obj = enif_alloc_resource(rt, sizeof(MyStruct));
ERL_NIF_TERM term = enif_make_resource(env, obj);
enif_release_resource(obj);
```

---

## 8. รูปแบบที่นิยมใช้

```erlang
%% Pattern: NIF with fallback
-module(fast_crypto).
-export([hash/1, encrypt/2]).
-on_load(init/0).

init() ->
    case erlang:load_nif(nif_path(), 0) of
        ok -> ok;
        {error, Reason} ->
            logger:warning("NIF not available: ~p, using fallback", [Reason]),
            ok
    end.

%% NIF version (fast)
hash(_Data) ->
    erlang:nif_error(not_loaded).

%% Pattern: async NIF via message passing
async_hash(Data) ->
    Self = self(),
    spawn_link(fun() ->
        Result = hash(Data),
        Self ! {hash_result, Result}
    end),
    receive
        {hash_result, Result} -> {ok, Result}
    after 5000 ->
        {error, timeout}
    end.

nif_path() ->
    filename:join(code:priv_dir(myapp), "fast_crypto").
```

---

## 9. แบบฝึกหัด

### Exercise: Port-based JSON Parser

```erlang
%% ใช้ port เพื่อเรียก jq สำหรับ JSON processing

-module(jq_port).
-behaviour(gen_server).

-export([start_link/0, query/2]).
-export([init/1, handle_call/3, handle_info/2]).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

query(Json, Filter) ->
    gen_server:call(?MODULE, {query, Json, Filter}).

init([]) ->
    Port = open_port({spawn, "jq -c -M ."}, [
        binary, {packet, 4}, use_stdio, exit_status
    ]),
    {ok, #{port => Port, pending => #{}, seq => 0}}.

handle_call({query, Json, Filter}, From,
            #{port := Port, pending := P, seq := N} = S) ->
    Cmd = jsx:encode(#{seq => N, json => Json, filter => Filter}),
    port_command(Port, Cmd),
    {noreply, S#{pending := P#{N => From}, seq := N + 1}};

handle_info({Port, {data, Data}}, #{port:=Port, pending:=P}=S) ->
    #{<<"seq">> := N, <<"result">> := Result} =
        jsx:decode(Data, [return_maps]),
    case maps:take(N, P) of
        {From, P2} ->
            gen_server:reply(From, {ok, Result}),
            {noreply, S#{pending := P2}};
        error ->
            {noreply, S}
    end;

handle_info({Port, {exit_status, N}}, #{port:=Port}=S) ->
    logger:error("jq port exited with ~p", [N]),
    {stop, {port_exited, N}, S}.
```

---

## สรุป Part 32

✅ NIF vs Port vs Port Driver trade-offs  
✅ Erlang Ports พื้นฐาน  
✅ C program protocol (packet framing)  
✅ Writing NIFs ใน C  
✅ NIF safety (< 1ms rule)  
✅ Dirty NIFs สำหรับงานหนัก  
✅ enif API reference  
✅ Port-based external programs

---

*Part 32/100 | [← ก่อนหน้า](../part31/README.md) | [ถัดไป →](../part33/README.md)*
