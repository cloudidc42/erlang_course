# Part 96: Production War Stories — Real Incidents and Postmortems

> **"Every production incident is a paid lesson — make sure you learn it"**  
> ทุก incident ในระบบ production คือบทเรียนที่จ่ายเงินแล้ว — ตรวจสอบให้แน่ใจว่าคุณได้เรียนรู้

---

## สารบัญ

1. [Incident #1: The Atom Table Explosion](#1-incident-1-the-atom-table-explosion)
2. [Incident #2: Binary Leak in Long-Running Processes](#2-incident-2-binary-leak-in-long-running-processes)
3. [Incident #3: Cascading Timeout Failure](#3-incident-3-cascading-timeout-failure)
4. [Incident #4: ETS Concurrency Surprise](#4-incident-4-ets-concurrency-surprise)
5. [Incident #5: Hot Code Upgrade Gone Wrong](#5-incident-5-hot-code-upgrade-gone-wrong)
6. [Writing a Postmortem](#6-writing-a-postmortem)
7. [แบบฝึกหัด](#7-แบบฝึกหัด)

---

## 1. Incident #1: The Atom Table Explosion

```
INCIDENT REPORT: Node crash after 6 hours of load
═══════════════════════════════════════════════════

Timeline:
  10:00 — Deployed new release with user metadata feature
  14:30 — Memory alerts: node using 8GB (normal: 2GB)
  16:00 — Node became unresponsive, OOM killer triggered
  16:05 — Node restarted, symptom returned within 2 hours

Root Cause Analysis:
  New code converted user_id strings to atoms for ETS key lookup:
  UserId = list_to_atom(binary_to_list(UserIdBin)),  %% DANGER!
  Atoms are never garbage collected.
  1M users × ~30 bytes/atom = 30MB per hour → OOM after 6 hours

Detection:
  erlang:system_info(atom_count) was growing unboundedly
  Normal: ~40,000 atoms. At crash: ~800,000 atoms.

Fix:
  %% WRONG:
  Key = list_to_atom(binary_to_list(Id)),

  %% RIGHT — atoms only for known-finite sets:
  Key = Id,  %% binaries are fine as ETS keys

Prevention:
  - Never convert external input to atoms
  - Add atom_count metric to monitoring
  - Limit: erlang:system_info(atom_limit) = 1,048,576 (1M default)
  - If you must: use existing_atom flag to fail safely:
    try binary_to_existing_atom(Bin, utf8)
    catch error:badarg -> {error, unknown_atom}
    end.
```

---

## 2. Incident #2: Binary Leak in Long-Running Processes

```erlang
%% INCIDENT: Payment processor grew from 50MB to 4GB over 72 hours
%% Root cause: large binaries kept alive by old heap references

%% The symptom appeared in:
-module(payment_processor).
-behaviour(gen_server).

handle_cast({process_batch, LargeBinaryPayload}, State) ->
    %% Parse the 10MB payload
    #{transactions := Txns} = json:decode(LargeBinaryPayload),
    %% Process each transaction
    Results = [process_transaction(T) || T <- Txns],
    %% BUG: LargeBinaryPayload is now referenced in State via Results
    %% even though we only need the processed results
    {noreply, State#{last_results => Results,
                     last_raw => LargeBinaryPayload}}.  %% Don't keep this!

%% THE FIX: discard the large binary immediately
handle_cast({process_batch, LargeBinaryPayload}, State) ->
    #{transactions := Txns} = json:decode(LargeBinaryPayload),
    Results = [process_transaction(T) || T <- Txns],
    %% Explicitly let go of the large binary
    erlang:garbage_collect(),  %% optional: force GC of this process
    {noreply, State#{last_results => Results}}.

%% DETECTION: monitor binary memory per process
detect_binary_leak() ->
    [{Pid, binary_memory_mb(Pid)}
     || Pid <- processes(),
        binary_memory_mb(Pid) > 10].

binary_memory_mb(Pid) ->
    case process_info(Pid, binary) of
        {binary, Bins} ->
            TotalBytes = lists:sum([Size || {_, Size, _} <- Bins]),
            TotalBytes div (1024 * 1024);
        _ -> 0
    end.

%% LESSON: Sub-binaries (slices) hold reference to ENTIRE original binary
%% This is safe:    binary:copy(binary:part(Big, 0, 100))
%% This retains:    binary:part(Big, 0, 100)  (keeps Big alive)
process_transaction(T) -> T.
```

---

## 3. Incident #3: Cascading Timeout Failure

```erlang
%% INCIDENT: 15-minute outage — one slow database query took down the API
%%
%% Architecture (before fix):
%%   HTTP Request
%%     → api_handler (gen_server:call with 5s timeout)
%%       → order_service (gen_server:call with 4s timeout)
%%         → db_pool:query (4s timeout)
%%           → PostgreSQL (running slow vacuum)
%%
%% What happened:
%%   PostgreSQL vacuum locked a table → all queries queued
%%   db_pool workers all waiting → pool exhausted
%%   order_service calls piled up (calls are serialized in gen_server!)
%%   api_handler calls timed out → HTTP 500s
%%   Meanwhile: all processes stuck, mailboxes filling up
%%   Node eventually ran out of memory from queued messages

%% ROOT CAUSE: synchronous call chains under load

%% LESSON 1: gen_server:call serializes — one slow call blocks all callers
check_mailbox_health() ->
    [{Pid, Len}
     || Pid <- processes(),
        {message_queue_len, Len} <- [process_info(Pid, message_queue_len)],
        Len > 1000].  %% Alert if any process queue > 1000 messages

%% LESSON 2: Use async where possible, set aggressive timeouts
safe_db_query(Sql, Params) ->
    case db_pool:query(Sql, Params) of
        {ok, _} = Result -> Result;
        {error, timeout} ->
            telemetry:execute([db, timeout], #{}, #{sql => Sql}),
            {error, service_unavailable}
    end.

%% LESSON 3: Circuit breaker prevents cascading
circuit_breaker_call(Service, Request) ->
    case circuit_breaker:state(Service) of
        open ->
            {error, {circuit_open, Service}};
        _ ->
            T1 = erlang:monotonic_time(millisecond),
            Result = gen_server:call(Service, Request, 2000),
            Dur = erlang:monotonic_time(millisecond) - T1,
            case Result of
                {error, _} -> circuit_breaker:record_failure(Service);
                _          -> circuit_breaker:record_success(Service)
            end,
            telemetry:execute([service, call], #{duration_ms => Dur},
                             #{service => Service}),
            Result
    end.
```

---

## 4. Incident #4: ETS Concurrency Surprise

```erlang
%% INCIDENT: Balance went negative despite check-before-deduct logic
%%
%% The BUG (classic TOCTOU race condition):
-module(wallet_bad).

deduct(UserId, Amount) ->
    case ets:lookup(balances, UserId) of
        [{UserId, Balance}] when Balance >= Amount ->
            %% RACE CONDITION: another process can deduct between lookup and update!
            NewBalance = Balance - Amount,
            ets:insert(balances, {UserId, NewBalance}),
            ok;
        _ ->
            {error, insufficient_funds}
    end.

%% THE FIX: use atomic ets:update_counter/4 with threshold guard
-module(wallet_good).

deduct(UserId, Amount) ->
    try
        %% Atomic: subtract Amount, minimum 0 (returns new value)
        %% If it would go below 0, throws badarg
        NewBalance = ets:update_counter(balances, UserId,
            {2, -Amount, 0, 0}),  %% {Pos, Incr, Threshold, SetValue}
        {ok, NewBalance}
    catch
        error:badarg ->
            {error, insufficient_funds}
    end.

%% DETECTION: enable concurrent_flags for your tables:
init_table() ->
    ets:new(balances, [
        named_table,
        public,
        {write_concurrency, true},   %% multiple writers allowed
        {read_concurrency, true}     %% multiple readers allowed (default now)
    ]).
```

---

## 5. Incident #5: Hot Code Upgrade Gone Wrong

```erlang
%% INCIDENT: gen_server crashed immediately after hot upgrade
%% Error: {function_clause, [{my_server, handle_call, [{v1_request, ...}, ...]}]}
%%
%% CAUSE: new code got old message format from messages queued before upgrade
%%
%% OLD CODE (v1):
handle_call({get_user, UserId}, _From, State) ->
    {reply, maps:get(UserId, State), State}.

%% NEW CODE (v2) — changed message format:
handle_call({get_user, UserId, Options}, _From, State) ->
    Opts = maps:merge(#{include_deleted => false}, Options),
    {reply, get_user_with_opts(State, UserId, Opts), State}.

%% FIX: handle both formats during transition period
handle_call({get_user, UserId}, From, State) ->
    %% Handle old format for messages queued before upgrade
    handle_call({get_user, UserId, #{}}, From, State);

handle_call({get_user, UserId, Options}, _From, State) ->
    Opts = maps:merge(#{include_deleted => false}, Options),
    {reply, get_user_with_opts(State, UserId, Opts), State}.

%% LESSON: During hot upgrades, the process mailbox may contain
%% messages sent with the OLD format. Always handle both until
%% the transition period ends (all clients updated).

%% DETECTION: Test hot upgrades in staging with message queuing:
test_hot_upgrade() ->
    {ok, Pid} = old_server:start_link(),
    %% Queue a message before upgrade
    Pid ! {old_format_message, data},
    %% Perform upgrade
    sys:suspend(Pid),
    code_change:upgrade(my_server, "1.0", "2.0"),
    sys:resume(Pid),
    %% Verify old message handled correctly
    ok.

get_user_with_opts(_State, _UserId, _Opts) -> #{}.
```

---

## 6. Writing a Postmortem

```
POSTMORTEM TEMPLATE
════════════════════════════════════════════════════

# Incident: [Brief Title]

**Date:** 2026-03-15
**Duration:** 47 minutes (15:23 — 16:10 UTC)
**Severity:** P1 (full outage)
**Status:** Resolved

## Summary
One paragraph describing what happened, how it impacted users,
and how it was resolved.

## Impact
- User-facing: 100% of payment attempts failed
- Revenue impact: ~$12,000 in failed transactions
- SLA breach: 99.9% monthly SLA requires <43.8 min downtime

## Timeline (UTC)
15:23 — Alert fired: payment_success_rate < 95%
15:26 — On-call engineer paged
15:34 — Root cause identified: OOM on payment-node-2
15:40 — Decision: restart node rather than debug live
15:47 — Node restarted, traffic shifted to node-1 and node-3
15:55 — Full traffic restored, all metrics nominal
16:10 — Incident declared resolved, postmortem scheduled

## Root Cause
[Detailed technical explanation, no blame]

## Contributing Factors
- No memory limit set on ETS table
- Binary metric not in our dashboards
- Only one payment node in rotation at time of incident

## Detection
How we found out (alert, user report, internal tool).

## Resolution
What we did to stop the bleeding.

## Action Items
| Action | Owner | Due |
|--------|-------|-----|
| Add ETS memory metric to Prometheus | @alice | 2026-03-22 |
| Set max_heap_size on payment processes | @bob | 2026-03-22 |
| Add second payment node for redundancy | @carol | 2026-04-01 |
| Write runbook for memory incidents | @dave | 2026-03-29 |

## Lessons Learned
What worked, what didn't, what we're changing.
```

---

## 7. แบบฝึกหัด

1. เพิ่ม atom count monitoring ใน telemetry pipeline ที่สร้างใน Part 71
2. สร้าง binary leak detector ที่ alert เมื่อ process มี binary memory > 100MB
3. เขียน runbook สำหรับ "node ไม่ respond" scenario step by step
4. Simulate cascading failure ใน dev environment ด้วย chaos tools จาก Part 84

---

## สรุป Part 96

✅ Atom explosion: list_to_atom from user input, detection, binary_to_existing_atom  
✅ Binary leak: sub-binary reference, binary inspection per process  
✅ Cascading timeouts: serialized gen_server calls, circuit breaker pattern  
✅ ETS race condition: TOCTOU with lookup+insert, fix with update_counter  
✅ Hot upgrade message format: handle old format during transition  
✅ Postmortem template: timeline, root cause, action items, lessons  

---

*Part 96/100 | [← ก่อนหน้า](../part95/README.md) | [ถัดไป →](../part97/README.md)*
