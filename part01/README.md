# Part 01: บทนำ Erlang และการติดตั้ง

> **"Erlang was designed for fault-tolerance, not for making programmers happy"**  
> — Joe Armstrong, ผู้สร้าง Erlang

---

## สารบัญ

1. [Erlang คืออะไร?](#1-erlang-คืออะไร)
2. [ประวัติและที่มา](#2-ประวัติและที่มา)
3. [ทำไมต้องใช้ Erlang?](#3-ทำไมต้องใช้-erlang)
4. [ระบบที่ใช้ Erlang ในโลกจริง](#4-ระบบที่ใช้-erlang-ในโลกจริง)
5. [การติดตั้ง Erlang](#5-การติดตั้ง-erlang)
6. [การติดตั้ง rebar3](#6-การติดตั้ง-rebar3)
7. [Erlang Shell (erl)](#7-erlang-shell-erl)
8. [Hello World แรก](#8-hello-world-แรก)
9. [โครงสร้าง Project](#9-โครงสร้าง-project)
10. [การ Compile และ Run](#10-การ-compile-และ-run)
11. [เครื่องมือสำคัญ](#11-เครื่องมือสำคัญ)
12. [แบบฝึกหัด](#12-แบบฝึกหัด)

---

## 1. Erlang คืออะไร?

Erlang เป็นภาษาโปรแกรมมิ่งที่ออกแบบมาเฉพาะสำหรับ:

- **Concurrent programming** — รองรับ millions of processes พร้อมกัน
- **Distributed systems** — หลาย node ทำงานร่วมกันได้ทันที  
- **Fault-tolerant systems** — ระบบที่ทนต่อความผิดพลาด
- **Soft real-time systems** — ตอบสนองภายในเวลาที่กำหนด

Erlang ทำงานบน **BEAM Virtual Machine** (Bogdan/Björn's Erlang Abstract Machine) ซึ่งมีความสามารถพิเศษที่ไม่มีในภาษาอื่น

### คุณสมบัติหลักของ Erlang

```
┌─────────────────────────────────────────────────┐
│              ERLANG FEATURES                     │
├─────────────────────────────────────────────────┤
│  ✓ Functional Programming                        │
│  ✓ Pattern Matching                              │
│  ✓ Immutable Data                               │
│  ✓ Lightweight Processes (Actors)               │
│  ✓ Message Passing (no shared memory)           │
│  ✓ Hot Code Swapping                            │
│  ✓ Built-in Distribution                        │
│  ✓ Garbage Collection per Process               │
│  ✓ "Let it Crash" Philosophy                    │
│  ✓ Supervisor Trees                             │
└─────────────────────────────────────────────────┘
```

---

## 2. ประวัติและที่มา

```
ไทม์ไลน์ Erlang
════════════════

1982 ── Ericsson เริ่มโปรเจ็กต์วิจัยภาษาใหม่
        เป้าหมาย: ภาษาสำหรับระบบโทรศัพท์ที่ต้องทำงาน 24/7

1986 ── Joe Armstrong, Robert Virding, Mike Williams
        เริ่มพัฒนา Erlang ที่ Ericsson Computer Science Lab

1987 ── Erlang เวอร์ชันแรกเสร็จสมบูรณ์
        ชื่อมาจาก "Ericsson Language" หรือ "Agner Krarup Erlang"
        (นักคณิตศาสตร์ผู้บุกเบิกทฤษฎี telephone network)

1998 ── Ericsson ปล่อย Erlang เป็น Open Source
        หลังจากใช้ภายในสำเร็จในระบบ AXD301 Switch
        (uptime 99.9999999% = ~31 milliseconds downtime/year!)

2006 ── Erlang/OTP R11B - เพิ่ม SMP support
        รองรับ multi-core processors

2012 ── Erlang R15B - เพิ่ม dirty schedulers
        
2014 ── Erlang 17.0 - Maps type เพิ่มเข้ามา

2019 ── Erlang OTP 22 - JIT compiler เริ่มพัฒนา

2021 ── Erlang OTP 24 - JIT compiler เปิดใช้งาน
        Performance เพิ่มขึ้น 25-40%

2023 ── Erlang OTP 26 - เวอร์ชันปัจจุบัน
```

---

## 3. ทำไมต้องใช้ Erlang?

### เปรียบเทียบกับภาษาอื่น

| คุณสมบัติ | Erlang | Go | Java | Python | Node.js |
|-----------|--------|-----|------|--------|---------|
| Concurrency Model | Actor/Message | Goroutines | Threads | GIL limited | Event Loop |
| Fault Tolerance | ★★★★★ | ★★★ | ★★★ | ★★ | ★★ |
| Distributed | ★★★★★ | ★★★ | ★★★ | ★★ | ★★★ |
| Hot Reload | ★★★★★ | ✗ | ✗ | ✗ | ✗ |
| Latency | Consistent | Low | Variable | High | Variable |
| Learning Curve | Steep | Moderate | Moderate | Easy | Easy |

### BEAM vs JVM

```
BEAM Virtual Machine
┌────────────────────────────────────────────┐
│  Process 1  │  Process 2  │  Process 3     │
│  GC: Local  │  GC: Local  │  GC: Local     │
│  Stack: Own │  Stack: Own │  Stack: Own    │
├─────────────────────────────────────────────┤
│         Scheduler 1 │ Scheduler 2           │
│         (CPU Core 1)│ (CPU Core 2)          │
├─────────────────────────────────────────────┤
│              BEAM VM                        │
│  Message Queues │ ETS │ I/O │ Timers        │
└─────────────────────────────────────────────┘

JVM (Java Virtual Machine)
┌────────────────────────────────────────────┐
│    Thread 1  │  Thread 2  │  Thread 3      │
│    Stack     │  Stack     │  Stack         │
├─────────────────────────────────────────────┤
│         Shared Heap Memory                  │
│         (Global GC = Stop the World)        │
├─────────────────────────────────────────────┤
│              JVM                            │
└─────────────────────────────────────────────┘
```

**ข้อได้เปรียบของ BEAM:**
- Process isolation → GC ทีละ process ไม่ stop ทั้งระบบ
- Message passing → ไม่มี race conditions
- Process ใน BEAM ใช้ memory เพียง ~300 bytes ต่อ process
- สามารถ spawn 100,000+ processes บนเครื่องธรรมดา

---

## 4. ระบบที่ใช้ Erlang ในโลกจริง

### บริษัทและระบบสำคัญ

```
WhatsApp (Meta)
├── ก่อนขาย: 2 million connections ต่อ server
├── 450 million users, 50 engineers
└── ใช้ Erlang สำหรับ messaging backend

RabbitMQ
├── Message broker ที่ใช้งานมากที่สุดในโลก
└── เขียนด้วย Erlang ทั้งหมด

CouchDB (Apache)
├── Document database
└── Replication protocol เขียนด้วย Erlang

Riak (Basho)
├── Distributed key-value database
└── Built on Erlang/OTP

Discord
├── ใช้ Elixir (runs on BEAM) 
└── 5 million concurrent users

Nintendo
├── Pokemon servers
└── Online gaming infrastructure

Ericsson
├── AXD301 ATM Switch
└── 99.9999999% availability = 31ms downtime/year
```

### ตัวเลขที่น่าประทับใจ

```
WhatsApp Statistics (2014, before acquisition)
══════════════════════════════════════════════
• 450 million users
• 50 billion messages/day
• 1 million new registrations/day
• Only 50 ENGINEERS total!
• Cost: ~$1 per million users

Comparison:
• Twitter: 1000+ engineers for 200M users
• Facebook: 10,000+ engineers for 1B users
```

---

## 5. การติดตั้ง Erlang

### Ubuntu/Debian

```bash
# วิธีที่ 1: ติดตั้งจาก apt (เวอร์ชันอาจไม่ใหม่ที่สุด)
sudo apt-get update
sudo apt-get install -y erlang

# ตรวจสอบเวอร์ชัน
erl -version
```

```bash
# วิธีที่ 2: ติดตั้งจาก Erlang Solutions (แนะนำ - ได้เวอร์ชันล่าสุด)
wget https://packages.erlang-solutions.com/erlang-solutions_2.0_all.deb
sudo dpkg -i erlang-solutions_2.0_all.deb
sudo apt-get update
sudo apt-get install -y esl-erlang

# ตรวจสอบเวอร์ชัน
erl -version
# Erlang (SMP,ASYNC_THREADS) (BEAM) emulator version 14.x
```

```bash
# วิธีที่ 3: ใช้ asdf (Version Manager - แนะนำมากที่สุดสำหรับนักพัฒนา)
# ติดตั้ง asdf ก่อน
git clone https://github.com/asdf-vm/asdf.git ~/.asdf --branch v0.14.0
echo '. "$HOME/.asdf/asdf.sh"' >> ~/.bashrc
source ~/.bashrc

# เพิ่ม erlang plugin
asdf plugin add erlang

# ติดตั้ง Erlang เวอร์ชันที่ต้องการ
asdf install erlang 26.2.1
asdf global erlang 26.2.1

# ตรวจสอบ
erl -version
```

### macOS

```bash
# วิธีที่ 1: Homebrew
brew install erlang

# วิธีที่ 2: asdf (แนะนำ)
brew install asdf
asdf plugin add erlang
asdf install erlang 26.2.1
asdf global erlang 26.2.1
```

### Windows

```powershell
# วิธีที่ 1: Installer จาก erlang.org
# ดาวน์โหลด OTP installer จาก https://www.erlang.org/downloads

# วิธีที่ 2: Chocolatey
choco install erlang

# วิธีที่ 3: WSL2 (แนะนำ)
# ใช้ Ubuntu ใน WSL2 และทำตามขั้นตอน Ubuntu
```

### Docker (ทางเลือกที่ง่ายที่สุด)

```bash
# Pull Erlang image
docker pull erlang:26

# รัน Erlang shell ใน container
docker run -it --rm erlang:26 erl

# รัน project ใน Docker
docker run -it --rm -v $(pwd):/app -w /app erlang:26 bash
```

### ตรวจสอบการติดตั้ง

```bash
# ตรวจสอบ Erlang version
$ erl -version
Erlang (SMP,ASYNC_THREADS) (BEAM) emulator version 14.2.1

# เข้า Erlang interactive shell
$ erl
Erlang/OTP 26 [erts-14.2.1] [source] [64-bit] [smp:8:8] [ds:8:8:10] [async-threads:1] [jit:ns]

Eshell V14.2.1 (press Ctrl+G to abort, type help(). for help)
1> 
```

---

## 6. การติดตั้ง rebar3

rebar3 เป็น build tool มาตรฐานสำหรับ Erlang

```bash
# ดาวน์โหลด rebar3
curl -fsSL https://s3.amazonaws.com/rebar3/rebar3 > rebar3

# ให้ permission
chmod +x rebar3

# ย้ายไป PATH
sudo mv rebar3 /usr/local/bin/

# ตรวจสอบ
rebar3 --version
# rebar 3.22.1 on Erlang/OTP 26 Erts 14.2.1
```

### สร้าง Project แรกด้วย rebar3

```bash
# สร้าง project ใหม่
rebar3 new app hello_world
cd hello_world

# โครงสร้าง project ที่ได้
hello_world/
├── _build/              # Build artifacts (อย่า commit)
├── apps/
│   └── hello_world/
│       └── src/
│           ├── hello_world_app.erl
│           ├── hello_world_sup.erl
│           └── hello_world.app.src
├── config/
│   └── sys.config
├── rebar.config         # Project configuration
└── .gitignore
```

---

## 7. Erlang Shell (erl)

Erlang Interactive Shell เป็นเครื่องมือที่ทรงพลังมาก

### เริ่มต้นใช้งาน Shell

```erlang
% เปิด shell
$ erl
Erlang/OTP 26 [erts-14.2.1] [64-bit] [smp:8:8]

Eshell V14.2.1
1>
```

### คำสั่งพื้นฐานใน Shell

```erlang
% การคำนวณง่ายๆ
1> 2 + 3.
5
2> 10 * 4.
40
3> 100 / 3.
33.333333333333336
4> 100 div 3.    % Integer division
33
5> 100 rem 3.    % Remainder
1

% ตัวแปร (ขึ้นต้นด้วยตัวพิมพ์ใหญ่)
6> X = 42.
42
7> X.
42

% ไม่สามารถ reassign ได้ใน Erlang!
8> X = 100.
** exception error: no match of right hand side value 100

% String (เป็น list ของ integers)
9> Name = "Hello".
"Hello"

% Atom (ขึ้นต้นด้วยตัวพิมพ์เล็ก)
10> ok.
ok
11> erlang.
erlang
```

### คำสั่งควบคุม Shell

```erlang
% ดู help
1> help().

** shell internal commands **
b()        -- display all variable bindings
e(N)       -- repeat the expression in query <N>
f()        -- forget all variable bindings
f(X)       -- forget the binding of variable X
h()        -- history
history(N) -- set how many previous commands to keep
results(N) -- set how many previous command results to keep
catch_exception(B) -- how exceptions are handled
v(N)       -- use the value of query <N>
rd(R,D)    -- define a record
rf()       -- remove all record information
rf(R)      -- remove record information about R
rl()       -- display all record information
rp(Term)   -- display Term using the shell's record information
rr(File)   -- read record information from File (wildcards allowed)
rr(F,R)    -- read selected record information from file(s)
rr(F,R,O)  -- read selected record information with options

% ดูตัวแปรทั้งหมด
2> b().
Name = "Hello"
X = 42
ok

% ลบตัวแปรทั้งหมด
3> f().
ok

% ออกจาก shell
4> q().
% หรือกด Ctrl+C แล้วเลือก q
```

### การใช้ Shell อย่างมีประสิทธิภาพ

```erlang
% เรียก built-in functions
1> lists:sort([3,1,4,1,5,9,2,6]).
[1,1,2,3,4,5,6,9]

2> string:upper("hello world").
"HELLO WORLD"

3> erlang:now().
{1706,123456,789012}

% ดู documentation ใน shell
4> h(lists, sort).
%% หรือ
4> h(lists).

% ดู module info
5> lists:module_info().
[{module,lists},
 {exports,[...]},
 ...]
```

---

## 8. Hello World แรก

### วิธีที่ 1: ใน Shell โดยตรง

```erlang
$ erl
1> io:format("Hello, World!~n").
Hello, World!
ok
```

### วิธีที่ 2: สร้างไฟล์ .erl

สร้างไฟล์ `hello.erl`:

```erlang
%% File: hello.erl
%% Module สำหรับ Hello World

-module(hello).           %% ชื่อ module ต้องตรงกับชื่อไฟล์
-export([world/0]).       %% export ฟังก์ชันที่จะใช้จากภายนอก

%% ฟังก์ชัน world/0 (ชื่อ world, 0 arguments)
world() ->
    io:format("Hello, World!~n").
```

Compile และ Run ใน shell:

```erlang
$ erl
1> c(hello).              %% compile ไฟล์ hello.erl
{ok,hello}
2> hello:world().
Hello, World!
ok
```

### วิธีที่ 3: สร้าง Script ที่ run ได้เลย

สร้างไฟล์ `hello_script.escript`:

```erlang
#!/usr/bin/env escript
%% -*- erlang -*-

main(_Args) ->
    io:format("Hello, World!~n"),
    io:format("This is an Erlang script!~n").
```

```bash
# ให้ permission และ run
chmod +x hello_script.escript
./hello_script.escript
# Hello, World!
# This is an Erlang script!
```

### วิธีที่ 4: สร้าง rebar3 Project

```bash
# สร้าง project
rebar3 new escript hello_escript
cd hello_escript

# แก้ไข src/hello_escript.erl
```

```erlang
%% File: src/hello_escript.erl
-module(hello_escript).
-export([main/1]).

main(_Args) ->
    io:format("Hello, World from rebar3!~n"),
    io:format("Erlang version: ~s~n", [erlang:system_info(otp_release)]).
```

```bash
# Build และ Run
rebar3 escriptize
./_build/default/bin/hello_escript
# Hello, World from rebar3!
# Erlang version: 26
```

---

## 9. โครงสร้าง Project

### โครงสร้างมาตรฐาน Erlang/OTP

```
my_app/
├── apps/                    # OTP Applications (multi-app project)
│   └── my_app/
│       ├── include/         # Header files (.hrl)
│       │   └── my_app.hrl
│       ├── priv/            # Static files, NIF libraries
│       │   └── assets/
│       ├── src/             # Source code (.erl)
│       │   ├── my_app.erl          # API module
│       │   ├── my_app_app.erl      # Application callback
│       │   ├── my_app_sup.erl      # Root supervisor
│       │   └── my_app_worker.erl   # Worker process
│       └── test/            # Test files
│           └── my_app_SUITE.erl
├── config/
│   ├── sys.config           # System configuration
│   └── vm.args              # VM arguments
├── _build/                  # Build artifacts (gitignore)
├── rebar.config             # Build configuration
├── rebar.lock               # Dependency lock file
└── .gitignore
```

### ไฟล์สำคัญ: rebar.config

```erlang
%% File: rebar.config
{erl_opts, [
    debug_info,          %% เก็บ debug info
    {parse_transform, lager_transform}
]}.

{deps, [
    {cowboy, "2.10.0"},  %% Web server
    {jsx, "3.1.0"}       %% JSON library
]}.

{shell, [
    {config, "config/sys.config"}
]}.

{profiles, [
    {test, [
        {deps, [{meck, "0.9.2"}]}  %% Mock library สำหรับ test
    ]}
]}.
```

### ไฟล์สำคัญ: .app.src

```erlang
%% File: src/my_app.app.src
{application, my_app, [
    {description, "My Erlang Application"},
    {vsn, "1.0.0"},
    {registered, []},
    {mod, {my_app_app, []}},
    {applications, [
        kernel,
        stdlib
    ]},
    {env, [
        {port, 8080},
        {debug, false}
    ]},
    {licenses, ["Apache-2.0"]},
    {links, []}
]}.
```

---

## 10. การ Compile และ Run

### Compile ด้วย erlc

```bash
# Compile ไฟล์เดียว
erlc hello.erl
# ได้ไฟล์ hello.beam

# Compile พร้อม options
erlc +debug_info hello.erl

# Compile หลายไฟล์
erlc src/*.erl -o ebin/

# Run
erl -noshell -s hello world -s init stop
# หรือ
erl -pa ebin/ -noshell -eval "hello:world()" -s init stop
```

### Compile ด้วย rebar3

```bash
# Compile project
rebar3 compile

# Run shell พร้อม project
rebar3 shell

# Run tests
rebar3 eunit

# สร้าง release
rebar3 release

# สร้าง escript
rebar3 escriptize

# Clean build
rebar3 clean
```

### การ Run แบบต่างๆ

```bash
# 1. Interactive shell
$ erl
1> hello:world().

# 2. Run แล้วออก
$ erl -noshell -eval "hello:world()" -s init stop

# 3. Run script
$ escript my_script.escript

# 4. Run release
$ _build/default/rel/my_app/bin/my_app start
$ _build/default/rel/my_app/bin/my_app console
$ _build/default/rel/my_app/bin/my_app stop
```

---

## 11. เครื่องมือสำคัญ

### Observer - GUI สำหรับ Monitor ระบบ

```erlang
% เปิดใน shell
1> observer:start().
```

Observer ช่วยดู:
- Process list และสถานะ
- Memory usage
- Message queues
- ETS tables
- Application supervision trees

### Dialyzer - Static Type Checker

```bash
# Build PLT (Persistent Lookup Table) ครั้งแรก
dialyzer --build_plt --apps erts kernel stdlib

# ตรวจสอบ code
dialyzer src/

# ด้วย rebar3
rebar3 dialyzer
```

### Mnesia - Built-in Database

```erlang
% เริ่มใช้ Mnesia
1> mnesia:start().
ok
2> mnesia:info().
```

### Debugger

```erlang
% เปิด debugger
1> debugger:start().
% หรือ
1> int:m(hello).    % interpret module hello
2> int:break(hello, 5).  % set breakpoint at line 5
```

### การ Profile Performance

```erlang
% ใช้ fprof
1> fprof:apply(fun() -> my_module:heavy_function() end).
2> fprof:profile().
3> fprof:analyse([totals, {sort, own}]).
```

### Editor/IDE Setup

**VS Code:**
```bash
# ติดตั้ง extension
code --install-extension erlang-ls.erlang-ls

# สร้าง .erlang_ls.config
echo 'apps_dirs:
  - "apps/**"
include_dirs:
  - "include"' > .erlang_ls.config
```

**IntelliJ IDEA:**
- ติดตั้ง plugin: Erlang

**Emacs:**
```elisp
;; เพิ่มใน .emacs
(use-package erlang
  :ensure t
  :mode (("\\.erl\\'" . erlang-mode)
         ("\\.hrl\\'" . erlang-mode)))
```

---

## 12. แบบฝึกหัด

### Exercise 1: Hello World หลายรูปแบบ

สร้างไฟล์ `exercises/ex01.erl` ที่มีฟังก์ชัน:
1. `hello/0` — แสดง "Hello, World!"
2. `hello/1` — รับชื่อและแสดง "Hello, [Name]!"
3. `hello/2` — รับชื่อและภาษา แสดงตามภาษาที่กำหนด

```erlang
%% ตัวอย่างที่ต้องการ:
ex01:hello().
%% Hello, World!

ex01:hello("Alice").
%% Hello, Alice!

ex01:hello("Bob", thai).
%% สวัสดี, Bob!

ex01:hello("Charlie", english).
%% Hello, Charlie!
```

### Exercise 2: Calculator

สร้าง `exercises/calculator.erl`:
- `add(X, Y)` — บวก
- `subtract(X, Y)` — ลบ  
- `multiply(X, Y)` — คูณ
- `divide(X, Y)` — หาร (จัดการ division by zero)
- `power(Base, Exp)` — ยกกำลัง

### Exercise 3: ทำความคุ้นเคยกับ Shell

ทดลองคำสั่งต่อไปนี้ใน erl shell:

```erlang
% 1. คำนวณ
1> (100 + 50) * 2 / 3.

% 2. ดูข้อมูล process
2> erlang:system_info(process_count).

% 3. spawn process แรก
3> spawn(fun() -> io:format("I'm a process!~n") end).

% 4. ดู registered processes
4> registered().

% 5. ดู memory info
5> erlang:memory().
```

### Solutions

```erlang
%% File: exercises/ex01.erl
-module(ex01).
-export([hello/0, hello/1, hello/2]).

hello() ->
    io:format("Hello, World!~n").

hello(Name) ->
    io:format("Hello, ~s!~n", [Name]).

hello(Name, thai) ->
    io:format("สวัสดี, ~s!~n", [Name]);
hello(Name, japanese) ->
    io:format("こんにちは, ~s!~n", [Name]);
hello(Name, _) ->
    io:format("Hello, ~s!~n", [Name]).
```

```erlang
%% File: exercises/calculator.erl
-module(calculator).
-export([add/2, subtract/2, multiply/2, divide/2, power/2]).

add(X, Y) -> X + Y.
subtract(X, Y) -> X - Y.
multiply(X, Y) -> X * Y.

divide(_, 0) ->
    {error, division_by_zero};
divide(X, Y) ->
    {ok, X / Y}.

power(_, 0) -> 1;
power(Base, Exp) when Exp > 0 ->
    Base * power(Base, Exp - 1).
```

---

## สรุป Part 01

ใน Part นี้คุณได้เรียนรู้:

✅ Erlang คืออะไรและทำไมถึงสำคัญ  
✅ ประวัติและที่มาของ Erlang  
✅ การติดตั้ง Erlang และ rebar3  
✅ การใช้ Erlang Shell เบื้องต้น  
✅ การเขียน Hello World หลายรูปแบบ  
✅ โครงสร้าง Project มาตรฐาน  
✅ การ Compile และ Run โปรแกรม  
✅ เครื่องมือที่สำคัญ  

---

## ต่อไป: [Part 02 — ประเภทข้อมูลพื้นฐานและ Syntax](../part02/README.md)

ใน Part ถัดไปเราจะเรียนรู้:
- ประเภทข้อมูลทั้งหมดใน Erlang (Integer, Float, Atom, Boolean, etc.)
- Syntax พื้นฐาน
- การประกาศตัวแปร
- Arithmetic, Comparison, Logical operators
- การทำงานกับ String และ Atom

---

*Part 01/100 | [ถัดไป →](../part02/README.md)*
