# Part 55: Erlang Build System Mastery

> **"A reliable build is the first test your code must pass"**  
> Build ที่เชื่อถือได้คือ test แรกที่ code ของคุณต้องผ่าน

---

## สารบัญ

1. [rebar3 Advanced Usage](#1-rebar3-advanced-usage)
2. [Umbrella Projects](#2-umbrella-projects)
3. [Custom rebar3 Plugins](#3-custom-rebar3-plugins)
4. [Dependency Management](#4-dependency-management)
5. [CI/CD Pipeline](#5-cicd-pipeline)
6. [Docker Multi-Stage Build](#6-docker-multi-stage-build)
7. [แบบฝึกหัด](#7-แบบฝึกหัด)

---

## 1. rebar3 Advanced Usage

```erlang
%% rebar.config — production-grade configuration
{erl_opts, [
    debug_info,
    {parse_transform, lager_transform},  %% if using lager
    {i, "include"},
    warn_unused_vars,
    warn_shadow_vars,
    warn_unused_import,
    warnings_as_errors
]}.

{deps, [
    {cowboy,    "2.10.0"},
    {jsx,       "3.1.0"},
    {epgsql,    "4.7.1"},
    {poolboy,   "1.5.2"},
    {hackney,   "1.20.1"},
    {telemetry, "1.2.1"},
    {bcrypt,    "1.2.0"},
    {meck,      "0.9.2"},
    {proper,    "1.4.0"}
]}.

%% Profiles: different deps/opts per environment
{profiles, [
    {test, [
        {deps, [
            {meck,   "0.9.2"},
            {proper, "1.4.0"}
        ]},
        {erl_opts, [debug_info, {d, 'TEST'}]},
        {cover_enabled, true}
    ]},
    {prod, [
        {erl_opts, [
            no_debug_info,
            {d, 'PROD'},
            inline,
            {hipe, [o3]}   %% HiPE native compilation (OTP < 24)
        ]},
        {relx, [
            {release, {myapp, "1.0.0"}, [myapp, sasl]},
            {mode, prod},
            {include_erts, true},
            {extended_start_script, true}
        ]}
    ]}
]}.

%% Aliases: shortcuts for common workflows
{alias, [
    {check, [compile, {eunit, "--cover"}, {ct, ""}, cover, dialyzer]},
    {ci,    [compile, {eunit, "--cover"}, {ct, ""},
             cover, {cover, "--min_coverage 80"}]}
]}.

%% Dialyzer
{dialyzer, [
    {warnings, [underspecs, no_return]},
    {plt_apps, top_level_deps},
    {plt_extra_apps, [kernel, stdlib, sasl, crypto]}
]}.

%% Shell config
{shell, [
    {apps, [myapp]},
    {config, "config/shell.config"}
]}.
```

---

## 2. Umbrella Projects

```
myplatform/
  ├── rebar.config           ← root config
  ├── apps/
  │   ├── myapp_core/       ← core business logic
  │   │   ├── src/
  │   │   └── rebar.config
  │   ├── myapp_api/        ← HTTP API layer
  │   │   ├── src/
  │   │   └── rebar.config
  │   ├── myapp_worker/     ← background jobs
  │   │   ├── src/
  │   │   └── rebar.config
  │   └── myapp_release/    ← release definition
  │       └── rebar.config
  └── _build/
```

```erlang
%% Root rebar.config for umbrella
{project_plugins, [
    rebar3_auto,   %% auto-recompile on save
    rebar3_hex     %% publish to hex.pm
]}.

{deps, []}.  %% shared deps go here

%% Build all apps
{erl_opts, [debug_info]}.

%% per-app config in apps/myapp_*/rebar.config
```

```erlang
%% apps/myapp_api/rebar.config
{deps, [
    {myapp_core, {path, "../myapp_core"}},
    {cowboy, "2.10.0"}
]}.
```

---

## 3. Custom rebar3 Plugins

```erlang
%% rebar3_check_migrations plugin
-module(rebar3_check_migrations).
-behaviour(provider).
-export([init/1, do/1, format_error/1]).

-define(PROVIDER, check_migrations).
-define(DEPS, [compile]).

init(State) ->
    Provider = providers:create([
        {name,       ?PROVIDER},
        {module,     ?MODULE},
        {bare,       true},
        {deps,       ?DEPS},
        {example,    "rebar3 check_migrations"},
        {short_desc, "Check that all migrations have been applied"},
        {desc,       "Compares migrations in src/migrations/ with DB"}
    ]),
    {ok, rebar_state:add_provider(State, Provider)}.

do(State) ->
    rebar_api:info("Checking migrations...", []),
    MigrationDir = "src/migrations",
    case filelib:is_dir(MigrationDir) of
        false ->
            rebar_api:info("No migrations directory found", []),
            {ok, State};
        true ->
            Files = filelib:wildcard(MigrationDir ++ "/*.sql"),
            rebar_api:info("Found ~p migration files", [length(Files)]),
            {ok, State}
    end.

format_error(Reason) ->
    io_lib:format("~p", [Reason]).
```

---

## 4. Dependency Management

```erlang
%% rebar.lock — lock file for reproducible builds (auto-generated)
%% Always commit this file!

%% Override a transitive dependency version
{overrides, [
    {override, cowlib, [{erl_opts, []}]}
]}.

%% Use a fork of a dependency
{deps, [
    {cowboy, {git, "https://github.com/myfork/cowboy.git",
              {branch, "fix-websocket-issue"}}}
]}.

%% Conditional deps
{deps, [
    {lager, {git, "https://github.com/erlang-lager/lager.git", {tag, "3.9.2"}}}
]}.

%% Check for outdated deps
%% rebar3 update
%% rebar3 deps  (list resolved versions)
```

```bash
# Common rebar3 commands
rebar3 compile          # compile
rebar3 shell            # Erlang shell with app loaded
rebar3 eunit            # unit tests
rebar3 ct               # common tests
rebar3 cover            # coverage report
rebar3 dialyzer         # type analysis
rebar3 xref             # cross-reference analysis
rebar3 as prod release  # build release
rebar3 as prod tar      # tarball for deployment
rebar3 upgrade dep_name # upgrade one dependency
rebar3 lock             # verify lock file
rebar3 tree             # show dependency tree
```

---

## 5. CI/CD Pipeline

```yaml
# .github/workflows/erlang.yml
name: Erlang CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        otp: ['26', '27']

    steps:
    - uses: actions/checkout@v4

    - name: Setup Erlang/OTP
      uses: erlef/setup-beam@v1
      with:
        otp-version: ${{ matrix.otp }}
        rebar3-version: '3.23'

    - name: Cache deps
      uses: actions/cache@v3
      with:
        path: _build/default/lib
        key: ${{ runner.os }}-build-${{ hashFiles('rebar.lock') }}

    - name: Compile
      run: rebar3 compile

    - name: Run tests
      run: rebar3 as test eunit --cover && rebar3 ct

    - name: Coverage check
      run: rebar3 cover --min_coverage 75

    - name: Dialyzer
      run: rebar3 dialyzer

  build-release:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    steps:
    - uses: actions/checkout@v4
    - uses: erlef/setup-beam@v1
      with:
        otp-version: '26'
        rebar3-version: '3.23'

    - name: Build release
      run: rebar3 as prod release

    - name: Build Docker image
      run: docker build -t myapp:${{ github.sha }} .

    - name: Push to registry
      run: |
        docker push myapp:${{ github.sha }}
        docker tag myapp:${{ github.sha }} myapp:latest
        docker push myapp:latest
```

---

## 6. Docker Multi-Stage Build

```dockerfile
# Dockerfile — production multi-stage build
FROM erlang:26-alpine AS builder

RUN apk add --no-cache git build-base

WORKDIR /app
COPY rebar.config rebar.lock ./
RUN rebar3 deps

COPY . .
RUN rebar3 as prod release

# ─── Runtime stage ────────────────────────────────
FROM alpine:3.20 AS runtime

RUN apk add --no-cache libstdc++ openssl ncurses-libs

WORKDIR /app

# Copy only the release
COPY --from=builder /app/_build/prod/rel/myapp ./

# Non-root user
RUN addgroup -S erlang && adduser -S erlang -G erlang
USER erlang

EXPOSE 8080

ENV RELEASE_COOKIE=mysecretcookie \
    NODE_NAME=myapp@127.0.0.1

HEALTHCHECK --interval=10s --timeout=5s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1

ENTRYPOINT ["bin/myapp"]
CMD ["foreground"]
```

```yaml
# docker-compose.yml — local development
version: '3.8'

services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - DB_HOST=postgres
      - DB_PORT=5432
      - DB_NAME=myapp_dev
      - DB_USER=postgres
      - DB_PASS=postgres
    depends_on:
      postgres:
        condition: service_healthy

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: myapp_dev
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 5
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

---

## 7. แบบฝึกหัด

1. สร้าง umbrella project ด้วย 2 apps: `core` และ `api`
2. เขียน rebar3 plugin ที่ verify ว่าทุก module มี `-vsn` attribute
3. เพิ่ม GitHub Actions step สำหรับ security scanning ด้วย `rebar3 xref`
4. Optimize Docker image size: ใช้ `strip` บน ERTS binaries

---

## สรุป Part 55

✅ rebar3 production configuration  
✅ Umbrella project structure  
✅ Custom rebar3 plugins  
✅ Dependency management และ lock files  
✅ GitHub Actions CI/CD pipeline  
✅ Docker multi-stage production build  

---

*Part 55/100 | [← ก่อนหน้า](../part54/README.md) | [ถัดไป →](../part56/README.md)*
