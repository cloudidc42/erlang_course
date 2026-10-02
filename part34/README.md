# Part 34: Release Management และ Deployment

> **"A release is a self-contained, deployable artifact"**  
> Release คือ artifact ที่พร้อม deploy โดยไม่ต้องพึ่ง Erlang installation

---

## สารบัญ

1. [Release คืออะไร?](#1-release-คืออะไร)
2. [rebar3 release configuration](#2-rebar3-release-configuration)
3. [relx configuration](#3-relx-configuration)
4. [vm.args และ sys.config](#4-vmargs-และ-sysconfig)
5. [Building Releases](#5-building-releases)
6. [Release Commands](#6-release-commands)
7. [Docker Deployment](#7-docker-deployment)
8. [Deployment Strategies](#8-deployment-strategies)
9. [แบบฝึกหัด](#9-แบบฝึกหัด)

---

## 1. Release คืออะไร?

```
Release structure:
myapp-1.0.0/
├── bin/
│   ├── myapp           ← start script
│   └── myapp-1.0.0     ← version-specific script
├── erts-14.0/          ← bundled Erlang runtime
│   └── bin/
│       ├── erl
│       └── epmd
├── lib/
│   ├── myapp-1.0.0/    ← your application
│   │   ├── ebin/
│   │   └── priv/
│   ├── cowboy-2.10.0/  ← dependencies
│   └── kernel-9.0/     ← OTP apps
└── releases/
    └── 1.0.0/
        ├── myapp.rel
        ├── sys.config
        └── vm.args

ข้อดี:
- ไม่ต้องติดตั้ง Erlang บน production server
- Self-contained bundle
- ง่ายต่อ deploy
- Support hot upgrades (relup)
```

---

## 2. rebar3 release configuration

```erlang
%% rebar.config
{erl_opts, [
    debug_info,
    {parse_transform, lager_transform}
]}.

{deps, [
    {cowboy, "2.10.0"},
    {jsx, "3.1.0"},
    {poolboy, "1.5.2"}
]}.

{relx, [
    {release, {myapp, "1.0.0"}, [
        myapp,           %% your app
        sasl,            %% OTP system app logging
        crypto           %% if needed
    ]},

    {mode, prod},        %% prod | dev | minimal

    %% Include ERTS (Erlang runtime)
    {include_erts, true},

    %% Strip debug info for smaller release
    {debug_info, strip},

    %% Extended start script
    {extended_start_script, true},

    %% Overlay: copy config files into release
    {overlay, [
        {mkdir,  "log"},
        {copy,   "config/sys.config", "releases/{{release_version}}/sys.config"},
        {copy,   "config/vm.args",    "releases/{{release_version}}/vm.args"},
        {template, "config/vars.config", "releases/{{release_version}}/vars.config"}
    ]},

    %% Template variables
    {overlay_vars, "config/vars.config"}
]}.

%% Separate profiles for dev/prod
{profiles, [
    {prod, [
        {relx, [{mode, prod}, {include_erts, true}]}
    ]},
    {dev, [
        {relx, [{mode, dev}, {include_erts, false}]}
    ]}
]}.
```

---

## 3. relx configuration

```erlang
%% config/relx.config (alternative to rebar.config)
{release, {myapp, "1.0.0"}, [
    myapp,
    sasl
]}.

{extended_start_script, true}.
{include_erts, true}.
{sys_config, "./config/sys.config"}.
{vm_args, "./config/vm.args"}.

%% Multiple releases
{release, {myapp, "2.0.0"}, [myapp, sasl]}.
{release, {myworker, "1.0.0"}, [myworker, sasl]}.

%% Exclude certain apps
{excl_apps, [wx, observer, debugger]}.

%% Force include specific modules
{incl_cond, include}.

%% Boot script: which app to start first
{boot_rel, myapp}.
```

---

## 4. vm.args และ sys.config

```bash
# config/vm.args
## Node name
-name myapp@127.0.0.1
## Short name (for single-machine)
## -sname myapp

## Cookie for distributed Erlang
-setcookie my_secret_cookie_change_this

## Max ports
+Q 65536

## Max atoms
+t 1048576

## Schedulers
## +S 4:4      # 4 online, 4 online for dirty

## SMP
-smp enable

## Kernel poll
+K true

## Async threads
+A 128

## Time warp mode (recommended: multi_time_warp)
+C multi_time_warp

## Heap strategy
+h 99          # initial heap size (words)
+hms 233       # min binary vheap size

## Crash dumps
-env ERL_CRASH_DUMP /var/log/myapp/erl_crash.dump
-env ERL_CRASH_DUMP_BYTES 104857600   # 100MB

## Logging
-env ERL_MAX_ETS_TABLES 8192
```

```erlang
%% config/sys.config
[
    {kernel, [
        {logger_level, info},
        {logger, [
            {handler, default, logger_std_h, #{
                config => #{
                    file => "/var/log/myapp/app.log",
                    max_no_bytes => 10485760,  %% 10MB
                    max_no_files => 5
                },
                formatter => {logger_formatter, #{
                    template => [time, " ", level, " ", msg, "\n"]
                }}
            }}
        ]}
    ]},

    {myapp, [
        {port, 8080},
        {db_host, "localhost"},
        {db_port, 5432},
        {db_name, "myapp_prod"},
        {db_pool_size, 10},
        {redis_url, "redis://localhost:6379"}
    ]},

    {cowboy, [
        {num_acceptors, 100},
        {max_connections, 10000}
    ]}
].
```

---

## 5. Building Releases

```bash
# Build development release
rebar3 release

# Build production release
rebar3 as prod release

# Create tarball for deployment
rebar3 as prod tar
# Creates: _build/prod/rel/myapp/myapp-1.0.0.tar.gz

# Generate upgrade release
rebar3 as prod relup

# Clean
rebar3 clean
rebar3 as prod clean

# Check release
rebar3 as prod release && ./_build/prod/rel/myapp/bin/myapp console

# Release directory structure after build
ls _build/prod/rel/myapp/
# bin/ erts-14.0/ lib/ releases/
```

---

## 6. Release Commands

```bash
# Start in foreground (console mode)
./bin/myapp console

# Start in background
./bin/myapp start

# Stop
./bin/myapp stop

# Restart
./bin/myapp restart

# Attach to running node
./bin/myapp attach
# Detach: Ctrl+D (not Ctrl+C!)

# Remote shell
./bin/myapp remote_console

# Check status
./bin/myapp ping

# Hot upgrade
./bin/myapp upgrade "2.0.0"

# Show running version
./bin/myapp versions

# Evaluate expression on running node
./bin/myapp eval 'io:format("~p~n", [erlang:memory()]).'

# RPC call
./bin/myapp rpc mymodule myfunction arg1 arg2

# Generate crash dump
./bin/myapp escript scripts/debug.escript
```

---

## 7. Docker Deployment

```dockerfile
# Dockerfile (multi-stage build)

# Stage 1: Build
FROM erlang:26-alpine AS builder
WORKDIR /app

# Install build dependencies
RUN apk add --no-cache git make gcc libc-dev

# Copy dependency files first (cache layer)
COPY rebar.config rebar.lock ./
RUN rebar3 get-deps

# Copy source
COPY . .

# Build release
RUN rebar3 as prod release && \
    rebar3 as prod tar

# Stage 2: Runtime (minimal image)
FROM alpine:3.18

# Install runtime deps only
RUN apk add --no-cache libstdc++ libgcc ncurses-libs openssl-libs-static

WORKDIR /app

# Copy release tarball
COPY --from=builder /app/_build/prod/rel/myapp/myapp-*.tar.gz .

# Extract
RUN tar xzf myapp-*.tar.gz && rm myapp-*.tar.gz

# Create log dir
RUN mkdir -p /var/log/myapp

# Non-root user
RUN addgroup -S erlang && adduser -S erlang -G erlang
USER erlang

EXPOSE 8080

CMD ["/app/bin/myapp", "foreground"]
```

```yaml
# docker-compose.yml
version: '3.8'
services:
  myapp:
    build: .
    ports:
      - "8080:8080"
    environment:
      - DATABASE_URL=postgres://user:pass@db:5432/myapp
      - SECRET_KEY=change_me
    depends_on:
      - db
      - redis
    restart: unless-stopped

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
    volumes:
      - pgdata:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine

volumes:
  pgdata:
```

---

## 8. Deployment Strategies

```bash
# Strategy 1: Blue-Green Deployment
# Run 2 environments (blue/green)
# Switch load balancer from blue to green
# Zero downtime

# Step 1: Deploy to green (inactive)
ssh green-server "cd /app && ./bin/myapp stop; \
    tar xzf myapp-2.0.0.tar.gz && ./bin/myapp start"

# Step 2: Health check green
curl http://green:8080/health

# Step 3: Switch load balancer
nginx -c /etc/nginx/blue-green.conf -s reload

# Step 4: Keep blue as rollback
# If issues: switch back to blue immediately

# Strategy 2: Rolling Update
# Update servers one by one
for server in server1 server2 server3; do
    ssh $server "
        ./bin/myapp stop
        tar xzf myapp-2.0.0.tar.gz
        ./bin/myapp start
    "
    # Wait and check
    sleep 10
    curl http://$server:8080/health || exit 1
done

# Strategy 3: Hot Upgrade (Erlang-specific)
# Deploy WITHOUT downtime
scp myapp-2.0.0.tar.gz prod-server:/app/releases/
ssh prod-server "./bin/myapp upgrade 2.0.0"
```

---

## 9. แบบฝึกหัด

### Exercise: Production Deployment Checklist

```bash
# สร้าง deployment script สมบูรณ์

#!/bin/bash
set -e

APP_NAME="myapp"
VERSION="$1"
DEPLOY_DIR="/app"
BACKUP_DIR="/app/backups"

if [ -z "$VERSION" ]; then
    echo "Usage: $0 <version>"
    exit 1
fi

echo "Deploying $APP_NAME $VERSION..."

# 1. Backup current version
echo "Backing up current version..."
mkdir -p $BACKUP_DIR
cp -r $DEPLOY_DIR $BACKUP_DIR/$(date +%Y%m%d_%H%M%S)

# 2. Download release
echo "Downloading release..."
aws s3 cp s3://releases/$APP_NAME-$VERSION.tar.gz /tmp/

# 3. Extract
echo "Extracting..."
cd $DEPLOY_DIR
tar xzf /tmp/$APP_NAME-$VERSION.tar.gz

# 4. Run pre-deploy scripts
echo "Running migrations..."
./bin/$APP_NAME eval "myapp_migrations:run()."

# 5. Hot upgrade if running, cold start if not
if ./bin/$APP_NAME ping &>/dev/null; then
    echo "Performing hot upgrade to $VERSION..."
    ./bin/$APP_NAME upgrade $VERSION
else
    echo "Starting $APP_NAME $VERSION..."
    ./bin/$APP_NAME start
fi

# 6. Health check
echo "Health check..."
sleep 5
if curl -sf http://localhost:8080/health; then
    echo "Deployment successful!"
else
    echo "Health check failed! Rolling back..."
    ./bin/$APP_NAME stop
    # restore backup
    exit 1
fi
```

---

## สรุป Part 34

✅ Release structure และ components  
✅ rebar3 release configuration  
✅ relx settings  
✅ vm.args: node name, cookie, schedulers  
✅ sys.config: application configuration  
✅ Building releases  
✅ Release commands: start/stop/console/upgrade  
✅ Docker multi-stage deployment  
✅ Blue-green, rolling, hot upgrade strategies

---

*Part 34/100 | [← ก่อนหน้า](../part33/README.md) | [ถัดไป →](../part35/README.md)*
