# Part 25: Hot Code Upgrades

> **"Hot code upgrades are Erlang's superpower — update without downtime"**  
> Hot code upgrades คือ superpower ของ Erlang — อัปเดตโดยไม่ต้อง downtime

---

## สารบัญ

1. [Hot Code Upgrade คืออะไร?](#1-hot-code-upgrade-คืออะไร)
2. [Code Loading Mechanism](#2-code-loading-mechanism)
3. [Module Versions](#3-module-versions)
4. [code_change Callback](#4-code_change-callback)
5. [Release Upgrades ด้วย relup](#5-release-upgrades-ด้วย-relup)
6. [Appup Files](#6-appup-files)
7. [Upgrading GenServer](#7-upgrading-genserver)
8. [Best Practices](#8-best-practices)
9. [แบบฝึกหัด](#9-แบบฝึกหัด)

---

## 1. Hot Code Upgrade คืออะไร?

```
Hot Code Upgrade:
├── เปลี่ยน code ขณะ system กำลัง run
├── ไม่ต้อง restart process
├── ไม่ต้อง stop service
├── State migration ได้
└── Rollback ถ้าล้มเหลว

Erlang สามารถทำได้เพราะ:
1. BEAM load modules แบบ dynamic
2. Processes มี isolated state
3. OTP behaviours มี code_change callback
4. Release tools สร้าง upgrade scripts

ใช้ใน:
- Telecom systems (ต้องการ 99.999% uptime)
- RabbitMQ: upgrade ขณะ handling messages
- WhatsApp: billion+ users, zero downtime
```

---

## 2. Code Loading Mechanism

```erlang
%% BEAM สามารถ hold 2 versions ของ module พร้อมกัน
%% Current: version ที่กำลังใช้งาน
%% Old: version ก่อนหน้า

%% Load module ใหม่
code:load_file(mymod).
code:load_binary(mymod, "mymod.beam", Beam).

%% Purge old version (ถ้า process ยังใช้ old version → crash)
code:soft_purge(mymod).  %% return false ถ้ายังมี process ใช้
code:purge(mymod).       %% force kill processes ที่ใช้ old version

%% ตรวจสอบ version ปัจจุบัน
code:which(mymod).       %% path ของ .beam file

%% ตรวจสอบ loaded modules
code:all_loaded().

%% Module info
mymod:module_info().
mymod:module_info(attributes).

%% เรียก old version explicitly
mymod:old_version_function()  %% ไม่ได้, ทำแบบนี้ไม่ได้โดยตรง

%% External call (ใช้ current version เสมอ)
?MODULE:my_function().  %% ขณะ running, เรียก current version

%% Internal call (ใช้ version เดิมที่ compile)
my_function().          %% local call, ใช้ version ที่ function อยู่
```

---

## 3. Module Versions

```erlang
%% กำหนด version ใน module
-module(my_server).
-vsn("2.0.0").

%% ดู version
my_server:module_info(attributes).
%% [{vsn, ["2.0.0"]}, ...]

%% OTP system ใช้ vsn สำหรับ upgrade detection
```

---

## 4. code_change Callback

```erlang
%% gen_server code_change
%% เรียกเมื่อ module ถูก upgrade ขณะ process กำลัง run

-module(my_server).
-behaviour(gen_server).

%% Old state format (version 1.0)
-record(state_v1, {
    count :: integer(),
    name  :: binary()
}).

%% New state format (version 2.0)
-record(state_v2, {
    count    :: integer(),
    name     :: binary(),
    metadata :: map()      %% new field
}).

%% code_change(OldVsn, OldState, Extra)
code_change("1.0", #state_v1{count=C, name=N}, _Extra) ->
    %% Migrate from v1 to v2
    NewState = #state_v2{count=C, name=N, metadata=#{}},
    {ok, NewState};

code_change({down, "1.0"}, #state_v2{count=C, name=N}, _Extra) ->
    %% Downgrade from v2 to v1
    OldState = #state_v1{count=C, name=N},
    {ok, OldState};

code_change(_OldVsn, State, _Extra) ->
    %% No migration needed
    {ok, State}.
```

---

## 5. Release Upgrades ด้วย relup

```bash
# 1. Build version 1.0
rebar3 release

# 2. แก้ code และเพิ่ม version ใน .app.src
#    {vsn, "2.0.0"}

# 3. Build version 2.0
rebar3 release

# 4. สร้าง appup file
# src/myapp.appup
# {"2.0.0",
#   [{"1.0.0", [{update, my_server, {advanced, []}}]}],
#   [{"1.0.0", [{update, my_server, {advanced, []}}]}]
# }.

# 5. สร้าง relup
rebar3 relup --name myapp --vsn 2.0.0 --old_vsn 1.0.0

# 6. สร้าง upgrade tarball
rebar3 as prod tar

# 7. Deploy (บน running system)
release_handler:unpack_release("myapp-2.0.0").
release_handler:install_release("2.0.0").
release_handler:make_permanent("2.0.0").
```

---

## 6. Appup Files

```erlang
%% src/myapp.appup
{"2.0.0",
 %% Upgrade instructions (1.0.0 → 2.0.0)
 [{"1.0.0",
   [
     %% update module code only
     {load_module, my_utils},

     %% update GenServer (calls code_change)
     {update, my_server, {advanced, []}},

     %% add new module
     {add_module, new_feature},

     %% remove old module
     {delete_module, old_feature},

     %% supervisor changes
     {update, my_sup, supervisor},

     %% restart application
     {restart_application, myapp}
   ]}
 ],
 %% Downgrade instructions (2.0.0 → 1.0.0)
 [{"1.0.0",
   [
     {load_module, my_utils},
     {update, my_server, {advanced, []}},
     {delete_module, new_feature},
     {add_module, old_feature},
     {update, my_sup, supervisor}
   ]}
 ]
}.

%% Appup Instructions:
%% {load_module, Mod}                  — load new code
%% {load_module, Mod, DepMods}         — with dependencies
%% {update, Mod, {advanced, Extra}}    — call code_change
%% {update, Mod, supervisor}           — update supervisor
%% {add_module, Mod}                   — new module
%% {delete_module, Mod}                — remove module
%% {add_application, App}              — new application
%% {remove_application, App}           — remove application
%% {restart_application, App}          — restart app
```

---

## 7. Upgrading GenServer

```erlang
%% ตัวอย่าง: upgrade counter server จาก v1 ไป v2

%% Version 1: เก็บ count เป็น integer
-module(counter).
-vsn("1.0").
-behaviour(gen_server).

init([]) -> {ok, 0}.

handle_call(get, _From, Count) -> {reply, Count, Count};
handle_call(inc, _From, Count) -> {reply, Count+1, Count+1}.

code_change(_OldVsn, State, _Extra) -> {ok, State}.

%% ========================
%% Version 2: เก็บ count + metadata
-module(counter).
-vsn("2.0").
-behaviour(gen_server).

-record(state, {count :: integer(), updates :: integer()}).

init([]) -> {ok, #state{count=0, updates=0}}.

handle_call(get, _From, #state{count=C}=S) ->
    {reply, C, S};
handle_call(inc, _From, #state{count=C, updates=U}=S) ->
    New = S#state{count=C+1, updates=U+1},
    {reply, C+1, New}.

%% Migrate state: integer → record
code_change("1.0", Count, _Extra) when is_integer(Count) ->
    {ok, #state{count=Count, updates=0}};
code_change(_OldVsn, State, _Extra) ->
    {ok, State}.
```

---

## 8. Best Practices

```erlang
%% 1. ทดสอบ code_change ก่อน deploy production
test_code_change() ->
    OldState = 42,       %% state จาก v1
    {ok, NewState} = counter:code_change("1.0", OldState, []),
    io:format("New state: ~p~n", [NewState]).

%% 2. Keep old state format compat ไว้
%% อย่าลบ field จาก record ทันที
%% deprecated fields → เก็บไว้ข้าม 1 version ก่อน

%% 3. ทำ upgrade ทีละ version
%% v1.0 → v1.1 → v2.0 (ไม่ใช่ v1.0 → v2.0 ข้าม)

%% 4. Test upgrade path
test_upgrade() ->
    %% Start v1
    {ok, Pid} = counter_v1:start_link(),
    counter_v1:inc(Pid),
    counter_v1:inc(Pid),
    V1State = sys:get_state(Pid),
    io:format("V1 state: ~p~n", [V1State]),

    %% Upgrade
    sys:suspend(Pid),
    code:load_file(counter),  %% load new version
    sys:change_code(Pid, counter, "1.0", []),
    sys:resume(Pid),

    %% Verify
    V2State = sys:get_state(Pid),
    io:format("V2 state after upgrade: ~p~n", [V2State]).

%% 5. sys:change_code — manual code upgrade
sys:change_code(Pid, Module, OldVsn, Extra).
%% เรียก code_change ใน process Pid
```

---

## 9. แบบฝึกหัด

### Exercise: Upgrade Cache Server

```erlang
%% Version 1: Simple cache
-module(cache_v1).
-behaviour(gen_server).

init([]) -> {ok, #{}}.

handle_call({get, K}, _, S) -> {reply, maps:get(K, S, miss), S};
handle_call({put, K, V}, _, S) -> {reply, ok, S#{K => V}}.

code_change(_, S, _) -> {ok, S}.

%% Version 2: Cache with TTL
-module(cache_v2).
-behaviour(gen_server).

%% New format: #{key => {value, expires_at}}
init([]) -> {ok, #{}}.

handle_call({get, K}, _, Cache) ->
    Now = erlang:system_time(second),
    case maps:find(K, Cache) of
        {ok, {V, Exp}} when Exp > Now -> {reply, {ok, V}, Cache};
        {ok, {_, _}} ->
            {reply, miss, maps:remove(K, Cache)};
        error ->
            {reply, miss, Cache}
    end;
handle_call({put, K, V, TTL}, _, Cache) ->
    Exp = erlang:system_time(second) + TTL,
    {reply, ok, Cache#{K => {V, Exp}}}.

%% Migrate v1 state → v2
code_change("1.0", OldCache, _) when is_map(OldCache) ->
    DefaultTTL = 3600,
    Now = erlang:system_time(second),
    NewCache = maps:map(fun(_, V) -> {V, Now + DefaultTTL} end, OldCache),
    {ok, NewCache};
code_change(_, State, _) ->
    {ok, State}.
```

---

## สรุป Part 25

✅ Hot code upgrade concept  
✅ BEAM code loading mechanism (current/old versions)  
✅ Module versions ด้วย -vsn  
✅ code_change callback  
✅ Release upgrades ด้วย relup  
✅ Appup files และ instructions  
✅ Upgrading GenServer state  
✅ Best practices

---

*Part 25/100 | [← ก่อนหน้า](../part24/README.md) | [ถัดไป →](../part26/README.md)*
