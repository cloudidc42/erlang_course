# Part 14: OTP Application Structure

> **"An OTP application is the unit of deployment in Erlang"**  
> OTP Application คือหน่วยของการ deploy ใน Erlang

---

## สารบัญ

1. [OTP Application คืออะไร?](#1-otp-application-คืออะไร)
2. [Application Resource File (.app)](#2-application-resource-file-app)
3. [Application Callback Module](#3-application-callback-module)
4. [rebar3 Project Structure](#4-rebar3-project-structure)
5. [Application Configuration](#5-application-configuration)
6. [Application Environment](#6-application-environment)
7. [Dependent Applications](#7-dependent-applications)
8. [Application Lifecycle](#8-application-lifecycle)
9. [Release Management](#9-release-management)
10. [แบบฝึกหัด](#10-แบบฝึกหัด)

---

## 1. OTP Application คืออะไร?

```
OTP Application:
├── Code (modules)
├── Configuration (.app.src)
├── Supervision Tree
├── Dependencies
└── Version info

ประโยชน์:
- Unit of deployment
- Dependency management
- Configuration management
- Hot code upgrade support
- Standard lifecycle (start/stop)
```

---

## 2. Application Resource File (.app)

```erlang
%% src/myapp.app.src (template — rebar3 แปลงเป็น ebin/myapp.app)

{application, myapp, [
    {description, "My Erlang Application"},
    {vsn, "1.0.0"},
    {registered, [myapp_server, myapp_cache]},  %% registered process names
    {applications, [
        kernel,
        stdlib,
        crypto,
        cowboy  %% dependencies
    ]},
    {mod, {myapp_app, []}},  %% Application callback module
    {env, [
        {port, 8080},
        {max_connections, 1000},
        {debug, false}
    ]},
    {modules, []},  %% รายชื่อ modules (rebar3 generate อัตโนมัติ)
    {licenses, ["Apache-2.0"]},
    {links, [{"GitHub", "https://github.com/user/myapp"}]}
]}.
```

---

## 3. Application Callback Module

```erlang
%% src/myapp_app.erl

-module(myapp_app).
-behaviour(application).

-export([start/2, stop/1]).

start(normal, _StartArgs) ->
    %% เริ่ม supervision tree
    case myapp_sup:start_link() of
        {ok, Pid} ->
            {ok, Pid};
        {error, Reason} ->
            {error, Reason}
    end;
start(takeover, _StartArgs) ->
    %% distributed: takeover จาก node อื่น
    myapp_sup:start_link();
start(failover, _StartArgs) ->
    %% distributed: failover
    myapp_sup:start_link().

stop(_State) ->
    %% cleanup (ส่วนใหญ่ supervisor จัดการให้)
    ok.

%% Application start types:
%% normal    — ปกติ
%% {takeover, Node} — takeover distributed app
%% {failover, Node} — failover distributed app
```

---

## 4. rebar3 Project Structure

```
myapp/
├── rebar.config          %% build configuration
├── src/
│   ├── myapp.app.src     %% application resource file
│   ├── myapp_app.erl     %% application callback
│   ├── myapp_sup.erl     %% root supervisor
│   ├── myapp_server.erl  %% main server (GenServer)
│   └── myapp_utils.erl   %% utility functions
├── include/
│   └── myapp.hrl         %% header files
├── priv/
│   └── www/              %% static files, migrations, etc.
├── test/
│   ├── myapp_SUITE.erl   %% common test suite
│   └── myapp_server_tests.erl  %% eunit tests
├── config/
│   ├── sys.config        %% runtime configuration
│   └── vm.args           %% VM arguments
└── _build/               %% build output (gitignore)
```

### rebar.config

```erlang
%% rebar.config

{erl_opts, [
    debug_info,
    {parse_transform, lager_transform}  %% ถ้าใช้ lager
]}.

{deps, [
    {cowboy, "2.10.0"},
    {jsx, "3.1.0"},
    {poolboy, "1.5.2"}
]}.

{shell, [
    {config, "config/sys.config"},
    {apps, [myapp]}
]}.

{relx, [
    {release, {myapp, "1.0.0"}, [myapp]},
    {mode, dev},
    {sys_config, "config/sys.config"},
    {vm_args, "config/vm.args"}
]}.

{profiles, [
    {prod, [
        {relx, [{mode, prod}, {include_erts, true}]}
    ]},
    {test, [
        {deps, [{meck, "0.9.2"}]},
        {erl_opts, [nowarn_export_all]}
    ]}
]}.

{ct_opts, [
    {dir, "test"},
    {logdir, "_build/test/logs"}
]}.
```

---

## 5. Application Configuration

```erlang
%% config/sys.config

[
    {myapp, [
        {port, 8080},
        {db_host, "localhost"},
        {db_port, 5432},
        {max_connections, 100}
    ]},
    {kernel, [
        {logger, [
            {handler, default, logger_std_h, #{
                formatter => {logger_formatter, #{
                    template => [time, " [", level, "] ", msg, "\n"]
                }}
            }}
        ]},
        {logger_level, info}
    ]}
].

%% config/vm.args
%% -name myapp@localhost
%% -setcookie secret_cookie
%% +K true
%% +A 64
%% -env ERL_MAX_PORTS 65536
```

---

## 6. Application Environment

```erlang
%% อ่าน config ใน code
application:get_env(myapp, port).
%% {ok, 8080} | undefined

application:get_env(myapp, port, 8080).
%% 8080 (with default)

application:get_all_env(myapp).
%% [{port, 8080}, {db_host, "localhost"}, ...]

%% Set config at runtime
application:set_env(myapp, port, 9090).

%% Helper
-define(APP, myapp).

get_config(Key, Default) ->
    application:get_env(?APP, Key, Default).

get_required_config(Key) ->
    case application:get_env(?APP, Key) of
        {ok, Value} -> Value;
        undefined   -> error({missing_config, Key})
    end.

%% Config module pattern
-module(myapp_config).
-export([port/0, db_host/0, db_port/0, max_connections/0]).

port()            -> application:get_env(?APP, port, 8080).
db_host()         -> application:get_env(?APP, db_host, "localhost").
db_port()         -> application:get_env(?APP, db_port, 5432).
max_connections() -> application:get_env(?APP, max_connections, 100).
```

---

## 7. Dependent Applications

```erlang
%% OTP applications ที่ต้องเริ่มก่อน myapp
{applications, [
    kernel,   %% always required
    stdlib,   %% always required
    crypto,
    ssl,
    ranch,    %% cowboy depends on this
    cowboy
]}.

%% เริ่ม application และ dependencies
application:ensure_all_started(myapp).
%% {ok, [crypto, ranch, cowboy, myapp]}

%% เริ่ม application เดียว
application:start(myapp).
%% {ok, {already_started, myapp}} ถ้า started แล้ว

%% หยุด
application:stop(myapp).

%% ดู running applications
application:which_applications().
%% [{myapp, "My Application", "1.0.0"},
%%  {cowboy, "Small, fast, modern HTTP server", "2.10.0"},
%%  ...]

%% ดู loaded (not necessarily started)
application:loaded_applications().
```

---

## 8. Application Lifecycle

```erlang
%% Lifecycle:
%% 1. load     — โหลด .app file และ modules
%% 2. start    — เรียก application:start/2 callback
%% 3. running  — application กำลังทำงาน
%% 4. stop     — เรียก application:stop/1 callback
%% 5. unload   — ถอน modules ออกจาก memory

%% รับ notification เมื่อ application status เปลี่ยน
-behaviour(application).

prep_stop(State) ->
    %% เรียกก่อน supervisor terminate
    %% State = state จาก start/2
    State.

stop(State) ->
    %% cleanup
    ok.

%% Application master: process ที่ manage lifecycle
%% ปกติไม่ต้อง interact โดยตรง

%% Graceful shutdown
init:stop().         %% หยุดทั้ง node
init:stop(Reason).   %% หยุดด้วย reason
application:stop(myapp).  %% หยุดเฉพาะ app
```

---

## 9. Release Management

```erlang
%% Release: package ทั้งหมดที่จำเป็นในการ deploy
%% รวม: Erlang runtime + applications + config

%% สร้าง release
%% rebar3 release
%% rebar3 as prod release

%% Start release
%% _build/prod/rel/myapp/bin/myapp start
%% _build/prod/rel/myapp/bin/myapp console (interactive)
%% _build/prod/rel/myapp/bin/myapp daemon  (background)

%% Hot upgrade
%% 1. เพิ่ม version ใน .app.src
%% 2. สร้าง relup file
%% 3. rebar3 relup
%% 4. deploy และ upgrade ขณะ running

%% vm.args template
-module(vm_args_example).
%% -name myapp@127.0.0.1
%% -setcookie mycookie
%% +K true        (kernel poll)
%% +A 64          (async threads)
%% +P 1000000     (max processes)
%% -env ERL_MAX_ETS_TABLES 8192
%% +S 8:8         (schedulers:online)

%% sys.config for prod
prod_config() ->
    [
        {myapp, [
            {port, 80},
            {db_host, {system, "DB_HOST", "localhost"}},
            %% read from env var
            {log_level, warning}
        ]},
        {kernel, [
            {logger_level, warning}
        ]}
    ].
```

---

## 10. แบบฝึกหัด

### Exercise: สร้าง OTP Application ครบ

```erlang
%% สร้าง todo_app ที่สมบูรณ์

%% src/todo_app.app.src
{application, todo_app, [
    {description, "Todo List Application"},
    {vsn, "1.0.0"},
    {registered, [todo_sup, todo_server]},
    {applications, [kernel, stdlib]},
    {mod, {todo_app, []}},
    {env, [
        {max_todos, 1000},
        {persistence_file, "todos.dat"}
    ]}
]}.

%% src/todo_app.erl
-module(todo_app).
-behaviour(application).
-export([start/2, stop/1]).

start(_Type, _Args) ->
    todo_sup:start_link().

stop(_State) ->
    ok.

%% src/todo_sup.erl
-module(todo_sup).
-behaviour(supervisor).
-export([start_link/0, init/1]).

start_link() ->
    supervisor:start_link({local, ?MODULE}, ?MODULE, []).

init([]) ->
    {ok, {
        #{strategy => one_for_one, intensity => 3, period => 30},
        [#{id => todo_server,
           start => {todo_server, start_link, []},
           restart => permanent,
           shutdown => 5000,
           type => worker,
           modules => [todo_server]}]
    }}.

%% src/todo_server.erl
-module(todo_server).
-behaviour(gen_server).
-export([start_link/0, add/1, list/0, done/1]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2, terminate/2]).

-record(state, {todos = #{}, next_id = 1}).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

add(Title)  -> gen_server:call(?MODULE, {add, Title}).
list()      -> gen_server:call(?MODULE, list).
done(Id)    -> gen_server:call(?MODULE, {done, Id}).

init([]) ->
    PersistFile = application:get_env(todo_app, persistence_file, "todos.dat"),
    State = load_state(PersistFile),
    {ok, State}.

handle_call({add, Title}, _From, #state{todos=T, next_id=Id}=S) ->
    Todo = #{id => Id, title => iolist_to_binary(Title), done => false},
    {reply, {ok, Id}, S#state{todos=T#{Id=>Todo}, next_id=Id+1}};

handle_call(list, _From, #state{todos=T}=S) ->
    {reply, {ok, maps:values(T)}, S};

handle_call({done, Id}, _From, #state{todos=T}=S) ->
    case maps:find(Id, T) of
        {ok, Todo} ->
            {reply, ok, S#state{todos=T#{Id=>Todo#{done=>true}}}};
        error ->
            {reply, {error, not_found}, S}
    end.

handle_cast(_Msg, State) -> {noreply, State}.
handle_info(_Info, State) -> {noreply, State}.

terminate(_Reason, State) ->
    PersistFile = application:get_env(todo_app, persistence_file, "todos.dat"),
    save_state(State, PersistFile),
    ok.

load_state(File) ->
    case file:read_file(File) of
        {ok, Bin} ->
            try binary_to_term(Bin)
            catch _:_ -> #state{}
            end;
        _ -> #state{}
    end.

save_state(State, File) ->
    file:write_file(File, term_to_binary(State)).
```

---

## สรุป Part 14

✅ OTP Application structure  
✅ .app.src resource file  
✅ Application callback (start/2, stop/1)  
✅ rebar3 project layout  
✅ sys.config และ vm.args  
✅ application:get_env/2,3  
✅ Dependency management  
✅ Application lifecycle  
✅ Release basics

---

*Part 14/100 | [← ก่อนหน้า](../part13/README.md) | [ถัดไป →](../part15/README.md)*
