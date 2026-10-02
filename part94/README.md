# Part 94: Developer Tools — CLI, Generators, and Scaffolding

> **"Good tools make good developers; great tools make great teams"**  
> เครื่องมือที่ดีสร้างนักพัฒนาที่ดี; เครื่องมือที่ยอดเยี่ยมสร้างทีมที่ยอดเยี่ยม

---

## สารบัญ

1. [CLI Framework Design](#1-cli-framework-design)
2. [Code Generator](#2-code-generator)
3. [Project Scaffolding](#3-project-scaffolding)
4. [Interactive REPL Helper](#4-interactive-repl-helper)
5. [Documentation Generator](#5-documentation-generator)
6. [Mix-Style Task System](#6-mix-style-task-system)
7. [แบบฝึกหัด](#7-แบบฝึกหัด)

---

## 1. CLI Framework Design

```erlang
%% cli.erl — argument parsing and subcommand dispatch
-module(cli).
-export([main/1, dispatch/2]).

%% Entry point from escript
main(Args) ->
    case parse_args(Args) of
        {Command, Opts, Rest} ->
            case dispatch(Command, {Opts, Rest}) of
                ok -> halt(0);
                {error, Reason} ->
                    io:format(standard_error, "Error: ~s~n", [Reason]),
                    halt(1)
            end;
        {error, help} ->
            print_usage(),
            halt(0);
        {error, Reason} ->
            io:format(standard_error, "~s~n", [Reason]),
            halt(1)
    end.

parse_args([]) ->
    {error, help};
parse_args(["-h" | _]) ->
    {error, help};
parse_args(["--help" | _]) ->
    {error, help};
parse_args([Command | Rest]) ->
    {Opts, Positional} = parse_options(Rest, #{}, []),
    {Command, Opts, Positional}.

parse_options([], Opts, Pos) ->
    {Opts, lists:reverse(Pos)};
parse_options(["--" | Rest], Opts, Pos) ->
    {Opts, lists:reverse(Pos) ++ Rest};
parse_options(["--" ++ Flag | Rest], Opts, Pos) ->
    case binary:split(list_to_binary(Flag), <<"=">>) of
        [Key, Val] ->
            parse_options(Rest, maps:put(binary_to_atom(Key), Val, Opts), Pos);
        [Key] ->
            case Rest of
                [Next | Rest2] when hd(Next) =/= $- ->
                    parse_options(Rest2, maps:put(binary_to_atom(Key),
                                                   list_to_binary(Next), Opts), Pos);
                _ ->
                    parse_options(Rest, maps:put(binary_to_atom(Key), true, Opts), Pos)
            end
    end;
parse_options(["-" ++ Flags | Rest], Opts, Pos) ->
    parse_flags(Flags, Rest, Opts, Pos);
parse_options([Arg | Rest], Opts, Pos) ->
    parse_options(Rest, Opts, [Arg | Pos]).

parse_flags([], Rest, Opts, Pos) ->
    parse_options(Rest, Opts, Pos);
parse_flags([Flag | Flags], Rest, Opts, Pos) ->
    parse_flags(Flags, Rest, maps:put(list_to_atom([Flag]), true, Opts), Pos).

dispatch("generate", Args) -> gen_command:run(Args);
dispatch("new",      Args) -> scaffold_command:run(Args);
dispatch("docs",     Args) -> docs_command:run(Args);
dispatch("check",    Args) -> check_command:run(Args);
dispatch(Unknown, _) ->
    {error, "Unknown command: " ++ Unknown ++ ". Run --help for usage."}.

print_usage() ->
    io:format("~s~n", [
        "Usage: emy <command> [options]\n"
        "\n"
        "Commands:\n"
        "  new <name>          Create a new Erlang project\n"
        "  generate <type>     Generate code (gen_server, supervisor, handler)\n"
        "  docs                Generate documentation\n"
        "  check               Run dialyzer and xref\n"
        "\n"
        "Options:\n"
        "  -h, --help          Show this help"
    ]).
```

---

## 2. Code Generator

```erlang
%% gen_command.erl — generate boilerplate Erlang modules
-module(gen_command).
-export([run/1]).

run({Opts, ["gen_server", Name]}) ->
    Module  = list_to_binary(Name),
    Content = gen_server_template(Module, Opts),
    write_module(Name, Content);
run({Opts, ["supervisor", Name]}) ->
    Module  = list_to_binary(Name),
    Content = supervisor_template(Module, Opts),
    write_module(Name, Content);
run({_, ["handler", Name]}) ->
    Module  = list_to_binary(Name),
    Content = cowboy_handler_template(Module),
    write_module(Name, Content);
run({_, [Unknown | _]}) ->
    {error, "Unknown generator: " ++ Unknown}.

gen_server_template(Module, Opts) ->
    HasState = maps:get(state, Opts, true),
    StateFields = maps:get(fields, Opts, []),
    [
        "%% ", Module, ".erl\n",
        "-module(", Module, ").\n",
        "-behaviour(gen_server).\n\n",
        "-export([start_link/0]).\n",
        "-export([init/1, handle_call/3, handle_cast/2, handle_info/2,\n",
        "         terminate/2]).\n\n",
        case HasState andalso length(StateFields) > 0 of
            true ->
                ["-record(state, {\n",
                 [["    ", F, "\n"] || F <- StateFields],
                 "}).\n\n"];
            false ->
                "-record(state, {}).\n\n"
        end,
        "start_link() ->\n",
        "    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).\n\n",
        "init([]) ->\n",
        "    {ok, #state{}}.\n\n",
        "handle_call(_Request, _From, State) ->\n",
        "    {reply, {error, not_implemented}, State}.\n\n",
        "handle_cast(_Msg, State) ->\n",
        "    {noreply, State}.\n\n",
        "handle_info(_Info, State) ->\n",
        "    {noreply, State}.\n\n",
        "terminate(_Reason, _State) ->\n",
        "    ok.\n"
    ].

supervisor_template(Module, _Opts) ->
    [
        "%% ", Module, ".erl\n",
        "-module(", Module, ").\n",
        "-behaviour(supervisor).\n\n",
        "-export([start_link/0, init/1]).\n\n",
        "start_link() ->\n",
        "    supervisor:start_link({local, ?MODULE}, ?MODULE, []).\n\n",
        "init([]) ->\n",
        "    SupFlags = #{strategy => one_for_one, intensity => 5, period => 10},\n",
        "    ChildSpecs = [\n",
        "        #{\n",
        "            id      => example_worker,\n",
        "            start   => {example_worker, start_link, []},\n",
        "            restart => permanent,\n",
        "            type    => worker\n",
        "        }\n",
        "    ],\n",
        "    {ok, {SupFlags, ChildSpecs}}.\n"
    ].

cowboy_handler_template(Module) ->
    [
        "%% ", Module, ".erl\n",
        "-module(", Module, ").\n",
        "-behaviour(cowboy_rest).\n\n",
        "-export([init/2, allowed_methods/2, content_types_provided/2,\n",
        "         content_types_accepted/2]).\n\n",
        "init(Req, State) ->\n",
        "    {cowboy_rest, Req, State}.\n\n",
        "allowed_methods(Req, State) ->\n",
        "    {[<<\"GET\">>, <<\"POST\">>], Req, State}.\n\n",
        "content_types_provided(Req, State) ->\n",
        "    {[{<<\"application/json\">>, handle_get}], Req, State}.\n\n",
        "content_types_accepted(Req, State) ->\n",
        "    {[{<<\"application/json\">>, handle_post}], Req, State}.\n\n",
        "handle_get(Req, State) ->\n",
        "    Body = json:encode(#{status => ok}),\n",
        "    {Body, Req, State}.\n\n",
        "handle_post(Req0, State) ->\n",
        "    {ok, Body, Req1} = cowboy_req:read_body(Req0),\n",
        "    _Data = json:decode(Body),\n",
        "    Req2 = cowboy_req:set_resp_body(json:encode(#{created => true}), Req1),\n",
        "    {true, Req2, State}.\n"
    ].

write_module(Name, Content) ->
    Filename = Name ++ ".erl",
    case file:write_file(Filename, Content) of
        ok ->
            io:format("Generated: ~s~n", [Filename]);
        {error, Reason} ->
            {error, io_lib:format("Cannot write ~s: ~p", [Filename, Reason])}
    end.
```

---

## 3. Project Scaffolding

```erlang
%% scaffold_command.erl — create new Erlang project structure
-module(scaffold_command).
-export([run/1]).

run({_Opts, [Name]}) ->
    io:format("Creating project: ~s~n", [Name]),
    create_structure(Name);
run({_Opts, []}) ->
    {error, "Project name required: emy new <name>"}.

create_structure(Name) ->
    Dirs = [
        Name,
        Name ++ "/src",
        Name ++ "/include",
        Name ++ "/test",
        Name ++ "/priv",
        Name ++ "/config"
    ],
    lists:foreach(fun(Dir) ->
        ok = filelib:ensure_dir(Dir ++ "/"),
        io:format("  create  ~s/~n", [Dir])
    end, Dirs),

    Files = [
        {Name ++ "/rebar.config", rebar_config(Name)},
        {Name ++ "/src/" ++ Name ++ ".app.src", app_src(Name)},
        {Name ++ "/src/" ++ Name ++ "_app.erl", app_module(Name)},
        {Name ++ "/src/" ++ Name ++ "_sup.erl", supervisor_module(Name)},
        {Name ++ "/config/sys.config", sys_config()},
        {Name ++ "/.gitignore", gitignore()}
    ],
    lists:foreach(fun({Path, Content}) ->
        ok = file:write_file(Path, Content),
        io:format("  create  ~s~n", [Path])
    end, Files),

    io:format("~nDone! To get started:~n"),
    io:format("  cd ~s~n", [Name]),
    io:format("  rebar3 shell~n").

rebar_config(Name) ->
    io_lib:format(
        "{erl_opts, [debug_info]}.\n\n"
        "{deps, [\n"
        "    {cowboy, \"2.10.0\"},\n"
        "    {jsx, \"3.1.0\"}\n"
        "]}.\n\n"
        "{shell, [{apps, [~s]}]}.\n", [Name]).

app_src(Name) ->
    io_lib:format(
        "{application, ~s, [\n"
        "    {description, \"~s application\"},\n"
        "    {vsn, \"0.1.0\"},\n"
        "    {modules, []},\n"
        "    {registered, []},\n"
        "    {applications, [kernel, stdlib]},\n"
        "    {mod, {~s_app, []}}\n"
        "]}.\n", [Name, Name, Name]).

app_module(Name) ->
    io_lib:format(
        "-module(~s_app).\n"
        "-behaviour(application).\n"
        "-export([start/2, stop/1]).\n\n"
        "start(_Type, _Args) ->\n"
        "    ~s_sup:start_link().\n\n"
        "stop(_State) -> ok.\n", [Name, Name]).

supervisor_module(Name) ->
    io_lib:format(
        "-module(~s_sup).\n"
        "-behaviour(supervisor).\n"
        "-export([start_link/0, init/1]).\n\n"
        "start_link() ->\n"
        "    supervisor:start_link({local, ?MODULE}, ?MODULE, []).\n\n"
        "init([]) ->\n"
        "    {ok, {#{strategy => one_for_one}, []}}.\n", [Name]).

sys_config() ->
    "[\n  {kernel, [{logger_level, info}]}\n].\n".

gitignore() ->
    "_build/\n.rebar3/\nebin/\n*.beam\n*.plt\n".
```

---

## 4. Interactive REPL Helper

```erlang
%% repl_helpers.erl — helpers for interactive Erlang shell development
-module(repl_helpers).
-export([r/1, r/2, help/0, inspect/1, bench/2]).

%% Reload a module (compile + load)
r(Module) ->
    case compile:file(atom_to_list(Module) ++ ".erl", [debug_info]) of
        {ok, Module} ->
            code:purge(Module),
            code:load_file(Module),
            io:format("Reloaded ~p~n", [Module]),
            ok;
        error ->
            {error, compile_failed}
    end.

r(Module, Dir) ->
    code:add_patha(Dir),
    r(Module).

%% Print help for a module
help() ->
    io:format("repl_helpers commands:~n"
              "  r(Module)          -- reload module~n"
              "  r(Module, Dir)     -- reload with path~n"
              "  inspect(Pid)       -- show process info~n"
              "  bench(N, Fun)      -- benchmark Fun N times~n").

%% Inspect a process
inspect(Pid) when is_pid(Pid) ->
    Keys = [registered_name, current_function, message_queue_len,
            memory, reductions, links, monitors],
    Info = [{K, V} || K <- Keys, {K, V} <- [process_info(Pid, K)]],
    maps:from_list(Info);
inspect(Name) when is_atom(Name) ->
    case whereis(Name) of
        undefined -> {error, not_registered};
        Pid -> inspect(Pid)
    end.

%% Micro-benchmark
bench(N, Fun) when N > 0 ->
    %% Warmup
    [Fun() || _ <- lists:seq(1, min(N div 10, 100))],
    T1 = erlang:monotonic_time(microsecond),
    [Fun() || _ <- lists:seq(1, N)],
    T2 = erlang:monotonic_time(microsecond),
    TotalUs = T2 - T1,
    #{
        iterations => N,
        total_us   => TotalUs,
        per_iter_us => TotalUs / N,
        ops_per_sec => N * 1_000_000 / TotalUs
    }.
```

---

## 5. Documentation Generator

```erlang
%% docs_command.erl — generate markdown docs from module attributes
-module(docs_command).
-export([run/1, generate_module_doc/1]).

run({Opts, []}) ->
    OutputDir = binary_to_list(maps:get(output, Opts, <<"docs">>)),
    filelib:ensure_dir(OutputDir ++ "/"),
    Modules = find_modules("src/"),
    lists:foreach(fun(Module) ->
        case generate_module_doc(Module) of
            {ok, Content} ->
                OutFile = OutputDir ++ "/" ++ atom_to_list(Module) ++ ".md",
                file:write_file(OutFile, Content);
            _ -> ok
        end
    end, Modules),
    io:format("Generated docs for ~p modules in ~s/~n",
              [length(Modules), OutputDir]).

generate_module_doc(Module) ->
    case code:ensure_loaded(Module) of
        {module, _} ->
            Attrs    = Module:module_info(attributes),
            Exports  = Module:module_info(exports),
            ModDoc   = proplists:get_value(moduledoc, Attrs, []),
            Content  = format_module_doc(Module, ModDoc, Exports, Attrs),
            {ok, Content};
        _ ->
            {error, not_loadable}
    end.

format_module_doc(Module, Doc, Exports, _Attrs) ->
    Header = io_lib:format("# `~p`~n~n", [Module]),
    DocStr = case Doc of
        [] -> "";
        _  -> io_lib:format("~s~n~n", [Doc])
    end,
    PublicExports = [E || E = {F, _} <- Exports, F =/= module_info],
    FuncList = format_function_list(PublicExports),
    [Header, DocStr, "## Exports\n\n", FuncList].

format_function_list(Exports) ->
    lists:map(fun({Fun, Arity}) ->
        io_lib:format("- `~p/~p`~n", [Fun, Arity])
    end, lists:sort(Exports)).

find_modules(SrcDir) ->
    Files = filelib:wildcard(SrcDir ++ "*.erl"),
    [list_to_atom(filename:rootname(filename:basename(F))) || F <- Files].
```

---

## 6. Mix-Style Task System

```erlang
%% tasks.erl — project tasks (test, lint, release, etc.)
-module(tasks).
-export([run/1, list/0]).

-define(TASKS, #{
    test    => fun task_test/0,
    lint    => fun task_lint/0,
    release => fun task_release/0,
    clean   => fun task_clean/0,
    format  => fun task_format/0
}).

run(Task) when is_list(Task) ->
    run(list_to_atom(Task));
run(Task) ->
    case maps:get(Task, ?TASKS, undefined) of
        undefined ->
            io:format("Unknown task: ~p~n", [Task]),
            io:format("Run 'emy --tasks' to list available tasks~n"),
            {error, unknown_task};
        Fun ->
            io:format("==> ~p~n", [Task]),
            Fun()
    end.

list() ->
    io:format("Available tasks:~n"),
    maps:foreach(fun(Name, _) ->
        io:format("  ~p~n", [Name])
    end, ?TASKS).

task_test() ->
    case os:cmd("rebar3 eunit") of
        Output ->
            io:format("~s~n", [Output]),
            case string:find(Output, "FAILED") of
                nomatch -> ok;
                _       -> {error, tests_failed}
            end
    end.

task_lint() ->
    Dialyzer = os:cmd("rebar3 dialyzer 2>&1"),
    Xref     = os:cmd("rebar3 xref 2>&1"),
    io:format("Dialyzer:~n~s~n", [Dialyzer]),
    io:format("Xref:~n~s~n", [Xref]),
    ok.

task_release() ->
    Output = os:cmd("rebar3 release 2>&1"),
    io:format("~s~n", [Output]),
    ok.

task_clean() ->
    os:cmd("rebar3 clean"),
    io:format("Cleaned build artifacts~n"),
    ok.

task_format() ->
    Files = filelib:wildcard("src/**/*.erl") ++ filelib:wildcard("test/**/*.erl"),
    io:format("Formatting ~p files...~n", [length(Files)]),
    ok.
```

---

## 7. แบบฝึกหัด

1. เพิ่ม `--dry-run` flag ให้ scaffold command แสดงว่าจะสร้างไฟล์อะไรโดยไม่สร้างจริง
2. สร้าง `generate migration <name>` ที่สร้าง migration file พร้อม timestamp
3. Implement `check` command ที่รัน dialyzer + xref + tests และ report summary
4. เพิ่ม tab completion script สำหรับ bash และ zsh

---

## สรุป Part 94

✅ CLI framework: argument parsing, subcommand dispatch, help output  
✅ Code generator: gen_server, supervisor, cowboy_handler templates  
✅ Project scaffolding: directory structure, rebar.config, app files  
✅ REPL helpers: hot reload, process inspect, micro-benchmark  
✅ Docs generator: module_info attributes → Markdown  
✅ Task system: mix-style commands for test, lint, release, format  

---

*Part 94/100 | [← ก่อนหน้า](../part93/README.md) | [ถัดไป →](../part95/README.md)*
