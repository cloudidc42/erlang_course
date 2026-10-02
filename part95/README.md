# Part 95: Publishing an Open Source Erlang Library

> **"Code shared is code multiplied; code documented is code understood"**  
> Code ที่แบ่งปันคือ code ที่ทวีคูณ; code ที่มี doc คือ code ที่เข้าใจได้

---

## สารบัญ

1. [Library Design Principles](#1-library-design-principles)
2. [Package Structure](#2-package-structure)
3. [Public API Design](#3-public-api-design)
4. [Versioning and Changelog](#4-versioning-and-changelog)
5. [Testing for Library Authors](#5-testing-for-library-authors)
6. [Publishing to hex.pm](#6-publishing-to-hexpm)
7. [แบบฝึกหัด](#7-แบบฝึกหัด)

---

## 1. Library Design Principles

```
Open Source Library Design Checklist
═════════════════════════════════════════════════════

STABILITY PROMISE:
  □ Documented public API (exported functions)
  □ Semantic versioning (MAJOR.MINOR.PATCH)
  □ Changelog maintained (CHANGELOG.md)
  □ Deprecation notices before breaking changes

USABILITY:
  □ Clear README with quick start (< 2 minutes to first result)
  □ Type specs on all public functions
  □ Meaningful error messages (not {error, 1})
  □ No global state — library is a guest in the user's app
  □ No forced supervision tree — user controls process topology

CORRECTNESS:
  □ Property-based tests for core functions
  □ Integration tests with common Erlang versions
  □ CI via GitHub Actions
  □ Dialyzer clean on release

PERFORMANCE:
  □ Benchmarks checked in for regression detection
  □ Memory profile for large inputs
  □ No hidden O(n²) operations

COMPATIBILITY:
  □ Works with OTP 25, 26, 27
  □ No undocumented dependencies on build tools
  □ Optional dependencies via conditional compilation
```

---

## 2. Package Structure

```
mylib/
├── src/
│   ├── mylib.app.src           -- application metadata
│   ├── mylib.erl               -- main API module
│   ├── mylib_core.erl          -- implementation (internal)
│   └── mylib_utils.erl         -- internal helpers
│
├── include/
│   └── mylib.hrl               -- public type definitions only
│
├── test/
│   ├── mylib_SUITE.erl         -- Common Test integration tests
│   ├── mylib_prop_SUITE.erl    -- PropEr property tests
│   └── mylib_bench.erl         -- micro-benchmarks
│
├── examples/
│   └── basic_usage.erl         -- example that actually compiles
│
├── rebar.config                -- build + test deps
├── CHANGELOG.md                -- what changed in each version
├── LICENSE                     -- MIT / Apache 2.0
└── README.md                   -- quick start + API reference
```

---

## 3. Public API Design

```erlang
%% mylib.erl — clean public API with stable contract
-module(mylib).

%% All public functions are exported here — nowhere else
-export([new/0, new/1, put/3, get/2, get/3, delete/2, size/1,
         to_list/1, from_list/1, fold/3]).

%% Only type specs that users need — implementation types are -opaque
-type store() :: mylib_core:store().
-type key()   :: term().
-type value() :: term().

-export_type([store/0, key/0, value/0]).

%% Create a new empty store
-spec new() -> store().
new() -> mylib_core:new(#{}).

%% Create with options
-spec new(Options :: map()) -> store().
new(Opts) -> mylib_core:new(Opts).

%% Put a value — returns updated store
-spec put(store(), key(), value()) -> store().
put(Store, Key, Value) -> mylib_core:put(Store, Key, Value).

%% Get a value
-spec get(store(), key()) -> {ok, value()} | {error, not_found}.
get(Store, Key) -> mylib_core:get(Store, Key).

%% Get with default
-spec get(store(), key(), value()) -> value().
get(Store, Key, Default) ->
    case mylib_core:get(Store, Key) of
        {ok, V} -> V;
        {error, not_found} -> Default
    end.

%% Delete a key — idempotent, no error if missing
-spec delete(store(), key()) -> store().
delete(Store, Key) -> mylib_core:delete(Store, Key).

%% Number of entries
-spec size(store()) -> non_neg_integer().
size(Store) -> mylib_core:size(Store).

%% Convert to/from list
-spec to_list(store()) -> [{key(), value()}].
to_list(Store) -> mylib_core:to_list(Store).

-spec from_list([{key(), value()}]) -> store().
from_list(List) ->
    lists:foldl(fun({K, V}, S) -> put(S, K, V) end, new(), List).

%% Fold over all entries
-spec fold(fun((key(), value(), Acc) -> Acc), Acc, store()) -> Acc.
fold(Fun, Init, Store) -> mylib_core:fold(Fun, Init, Store).
```

---

## 4. Versioning and Changelog

```
CHANGELOG.md — Keep a Changelog format
═══════════════════════════════════════════════════

# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [2.1.0] - 2026-03-15
### Added
- `mylib:fold/3` for accumulating over all entries
- Optional `{max_size, N}` option to cap store capacity

### Changed
- `mylib:get/2` now returns `{error, not_found}` instead of `undefined`
  (old behaviour still available via `mylib:get/3` with default)

### Deprecated
- `mylib:lookup/2` — use `mylib:get/2` instead; removed in 3.0.0

## [2.0.0] - 2026-01-10
### Breaking Changes
- Renamed `mylib:insert/3` to `mylib:put/3` for consistency
- Minimum OTP version raised from 24 to 25

### Fixed
- Memory leak when deleting keys with binary values (#42)

## [1.0.0] - 2025-06-01
### Added
- Initial release
```

```erlang
%% Version in app.src — single source of truth
%% src/mylib.app.src
{application, mylib, [
    {description, "A fast key-value store library"},
    {vsn, "2.1.0"},                %% <-- this is the version
    {modules, []},
    {registered, []},
    {applications, [kernel, stdlib]},
    {env, []},
    {licenses, ["Apache-2.0"]},
    {links, [{"GitHub", "https://github.com/yourname/mylib"}]}
]}.

%% rebar.config — version from app.src, no duplication
{hex, [{doc, ex_doc}]}.
```

---

## 5. Testing for Library Authors

```erlang
%% mylib_prop_SUITE.erl — property tests proving API invariants
-module(mylib_prop_SUITE).
-include_lib("proper/include/proper.hrl").
-include_lib("common_test/include/ct.hrl").
-export([all/0, prop_put_get/1, prop_size/1, prop_round_trip/1]).

all() -> [prop_put_get, prop_size, prop_round_trip].

prop_put_get(_Config) ->
    proper:quickcheck(?FORALL({Key, Value}, {term(), term()},
        begin
            S = mylib:new(),
            S2 = mylib:put(S, Key, Value),
            mylib:get(S2, Key) =:= {ok, Value}
        end), [{numtests, 1000}]).

prop_size(_Config) ->
    proper:quickcheck(?FORALL(Pairs, list({term(), term()}),
        begin
            S = mylib:from_list(Pairs),
            %% Size equals number of distinct keys
            UniqueKeys = length(lists:usort([K || {K, _} <- Pairs])),
            mylib:size(S) =:= UniqueKeys
        end), [{numtests, 500}]).

prop_round_trip(_Config) ->
    proper:quickcheck(?FORALL(Pairs, list({term(), term()}),
        begin
            Original = lists:ukeysort(1, Pairs),  %% deduplicate
            S = mylib:from_list(Original),
            Recovered = lists:keysort(1, mylib:to_list(S)),
            Recovered =:= Original
        end), [{numtests, 500}]).
```

---

## 6. Publishing to hex.pm

```
Publishing to hex.pm — step by step
════════════════════════════════════════════════════

1. Register account
   $ rebar3 hex user register

2. Authenticate
   $ rebar3 hex user auth

3. Verify package before publishing
   $ rebar3 hex build    -- builds the .tar for inspection
   $ rebar3 hex publish --dry-run   -- shows what would be published

4. Publish
   $ rebar3 hex publish

5. Verify
   - Package visible at: https://hex.pm/packages/mylib
   - Docs at: https://hexdocs.pm/mylib
```

```erlang
%% rebar.config — complete configuration for a published library
{erl_opts, [
    debug_info,
    warn_export_vars,
    warn_shadow_vars,
    warn_obsolete_guard
]}.

{deps, []}.  %% Minimize dependencies for library users

{profiles, [
    {test, [
        {deps, [
            {proper, "1.4.0"},
            {meck, "0.9.2"}
        ]}
    ]}
]}.

{hex, [
    {doc, ex_doc}  %% Use ex_doc for nice HTML docs
]}.

{ex_doc, [
    {extras, [
        {"README.md", #{title => "Overview"}},
        {"CHANGELOG.md", #{title => "Changelog"}},
        {"LICENSE", #{title => "License"}}
    ]},
    {main, "README.md"},
    {homepage_url, "https://github.com/yourname/mylib"},
    {source_url, "https://github.com/yourname/mylib"}
]}.

{dialyzer, [
    {warnings, [
        unmatched_returns,
        error_handling,
        underspecs
    ]}
]}.

{xref_checks, [
    undefined_function_calls,
    undefined_functions,
    locals_not_used,
    exports_not_used
]}.
```

---

## 7. แบบฝึกหัด

1. สร้าง GitHub Actions workflow ที่รัน tests บน OTP 25, 26, 27 ทั้งหมด
2. Write CONTRIBUTING.md: how to open issues, PR conventions, code style
3. สร้าง `examples/` ที่ compile และ run ได้จริง
4. Benchmark library กับ alternatives (maps, ets) เพื่อรู้ trade-off

---

## สรุป Part 95

✅ Design principles: stability promise, usability checklist, compat matrix  
✅ Package structure: src/, include/, test/, examples/ layout  
✅ Public API: opaque types, specs, consistent error format, no global state  
✅ Changelog: Keep a Changelog format, semantic versioning  
✅ Library testing: PropEr properties proving API invariants  
✅ hex.pm publishing: register, build, dry-run, publish, ex_doc  

---

*Part 95/100 | [← ก่อนหน้า](../part94/README.md) | [ถัดไป →](../part96/README.md)*
