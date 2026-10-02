# Part 98: Team Processes — Code Review, RFC, and ADR

> **"A team that learns together ships together"**  
> ทีมที่เรียนรู้ด้วยกันจะ ship ด้วยกัน

---

## สารบัญ

1. [Code Review Culture](#1-code-review-culture)
2. [Erlang-Specific Review Checklist](#2-erlang-specific-review-checklist)
3. [RFC Process](#3-rfc-process)
4. [On-Call and Incident Management](#4-on-call-and-incident-management)
5. [Knowledge Sharing Practices](#5-knowledge-sharing-practices)
6. [Team Health Metrics](#6-team-health-metrics)
7. [แบบฝึกหัด](#7-แบบฝึกหัด)

---

## 1. Code Review Culture

```
Code Review Principles
═══════════════════════════════════════════════════════

GOALS (in priority order):
  1. Correctness: does the code do what it's supposed to?
  2. Clarity: will a new team member understand it in 6 months?
  3. Consistency: does it follow project conventions?
  4. Performance: are there obvious bottlenecks?
  Style is lowest priority — use a formatter (erlfmt) for that.

REVIEWER MINDSET:
  - You are a collaborator, not a gatekeeper
  - "Have you considered..." > "You should..."
  - Distinguish blocking issues from suggestions
  - Approve when it's good enough, not perfect
  - Leave at most 10 comments per PR; prioritize

AUTHOR MINDSET:
  - Small PRs are reviewed faster (< 400 lines ideal)
  - Write a good PR description — what, why, testing done
  - Respond to every comment (even "acknowledged")
  - Don't take comments personally

TURNAROUND:
  - Reviewers respond within 1 business day
  - Authors address feedback within 1 business day
  - After 2 rounds without merge → sync call to unblock
```

---

## 2. Erlang-Specific Review Checklist

```erlang
%% CHECKLIST for Erlang code reviews

%% ✅ Process Design
%%  □ Does each process have a single clear responsibility?
%%  □ Is there a supervisor for each long-lived process?
%%  □ Is the restart strategy correct for this process type?
%%  □ Are processes communicating asynchronously where possible?

%% ✅ Error Handling
%%  □ Are errors returned as {error, Reason}, not thrown?
%%  □ Is the catch clause specific (not catch _:_)?
%%  □ Does the supervisor strategy match the failure semantics?

%% ✅ Type Safety
%%  □ Are all public functions spec-annotated?
%%  □ Do types match the documentation?
%%  □ Are dialyzer warnings resolved?

%% ✅ Performance
%%  □ No list operations in hot paths? (use maps/ets instead)
%%  □ No atom creation from external input?
%%  □ ETS tables use {read_concurrency, true} where appropriate?

%% ✅ Security
%%  □ No user input used as atom, module name, or file path?
%%  □ Sensitive data not logged?
%%  □ External calls have timeouts?

%% EXAMPLE: catch clause too broad
%% BAD:
handle_request(Req) ->
    try
        do_work(Req)
    catch
        _:_ -> {error, internal}  %% swallows programmer errors!
    end.

%% GOOD: only catch expected errors
handle_request(Req) ->
    try
        do_work(Req)
    catch
        throw:{validation_error, Msg} ->
            {error, {validation, Msg}};
        error:{db_error, Reason} ->
            logger:warning("DB error: ~p", [Reason]),
            {error, db_unavailable}
    end.

%% EXAMPLE: missing supervisor
%% BAD: spawning long-lived process without supervision
start_worker() ->
    spawn(fun worker_loop/0).  %% dies silently, nobody notices

%% GOOD: start under supervision
start_worker() ->
    supervisor:start_child(my_sup, #{
        id    => worker,
        start => {my_worker, start_link, []},
        restart => permanent,
        type    => worker
    }).

worker_loop() -> ok.
do_work(_Req) -> ok.
```

---

## 3. RFC Process

```markdown
# RFC Template: Request for Comments

**RFC Number:** RFC-0042
**Title:** Introduce Circuit Breakers for External API Calls
**Author:** @alice
**Status:** Draft → Discussion → Accepted/Rejected
**Created:** 2026-03-15
**Discussion Deadline:** 2026-03-29

## Summary (2-3 sentences)

Add circuit breaker wrappers around all external API calls to prevent
cascading failures when downstream services degrade. This addresses
the P1 incident in March where slow Stripe API responses caused our
entire payment service to degrade.

## Motivation

Current situation: a single slow downstream call blocks our process
for up to 30 seconds, exhausting the caller's message queue.
After the March incident (see postmortem PM-023), we need a mechanism
to fail fast when a dependency is unavailable.

## Proposed Solution

Implement the circuit breaker pattern (Part 85) for all external calls:

```erlang
-define(CIRCUIT_CONFIG, #{
    threshold => 5,       %% failures before opening
    timeout   => 60000,   %% 60s before half-open
    window    => 10000    %% 10s rolling window
}).
```

Every call to Stripe, Twilio, and warehouse APIs goes through
`circuit_breaker:call/3` with a defined fallback.

## Alternatives Considered

**A: Aggressive timeouts only**
  Rejected: Timeouts don't prevent cascading — they just delay it.

**B: Per-call retry with exponential backoff**
  Partially accepted: Combine with circuit breakers; retry on CLOSED.

## Open Questions

1. Should circuit state be shared across nodes, or per-node?
2. Who owns the circuit breaker config (env var vs DB)?
3. Do we need a manual override (force-open, force-close)?

## Implementation Plan

Week 1: Core circuit_breaker module + unit tests
Week 2: Integration into stripe_client and twilio_client
Week 3: Metrics dashboard, runbook

## Review Notes

[Leave comments here or in PR discussion]
```

---

## 4. On-Call and Incident Management

```
On-Call Handbook Excerpts
═════════════════════════════════════════════════════

ALERT RESPONSE LEVELS:

P1 — Complete outage (revenue impact)
  Response: Immediate (< 5 min)
  Escalation: page backup if not acknowledged in 10 min
  Communication: update status page every 15 min

P2 — Degraded (partial outage, elevated errors)
  Response: < 30 min
  Escalation: if not resolved in 1 hour
  Communication: update status page once, then on resolution

P3 — Minor issue (no user impact)
  Response: during business hours
  Ticket created for next sprint

RUNBOOK TEMPLATE:
  Alert Name: payment_success_rate_low
  What it means: < 95% of payment attempts succeeding in last 5 min
  Likely causes:
    1. Stripe API degradation → check status.stripe.com
    2. DB connection pool exhausted → check db_pool metrics
    3. Authentication service down → check auth service health
  Steps to diagnose:
    $ beam_inspect:top_processes()  -- look for stuck processes
    $ db_pool:stats()               -- check pool utilization
    $ circuit_breaker:status(stripe_client)  -- check circuit state
  Resolution:
    - Stripe issue: wait, no action (circuit will re-close when they recover)
    - DB pool: increase pool size or restart slow queries
  Escalation: page backend lead if unresolved after 30 min
```

---

## 5. Knowledge Sharing Practices

```
Erlang Team Knowledge Practices
═════════════════════════════════════════════════════

WEEKLY TECH TALK (30 min):
  Format: one engineer presents something they learned
  Topics: production incident analysis, new OTP feature,
          Erlang pattern deep-dive, performance finding
  Recorded: yes (async for remote team members)

PAIR PROGRAMMING:
  When: onboarding (first month), complex new features, debugging production
  Format: driver/navigator, swap every 30 min
  Documentation: write down decisions made during session

ARCHITECTURE REVIEW:
  When: any new system component, major refactor, new external dependency
  Format: author writes RFC, team reviews async, 30-min sync if contested
  Artifacts: ADR merged to repo regardless of decision (even rejections)

CODE TOUR FOR NEW JOINERS:
  Week 1: pair with buddy through setup and first feature
  Week 2: supervised first PR (reviewer explains every comment)
  Week 3: solo PR with review
  Month 2: lead a feature, ask for review

DOCUMENTATION CULTURE:
  □ ADRs in /docs/adr/ for every significant decision
  □ Runbooks in /docs/runbooks/ for every alert
  □ Postmortems in /docs/postmortems/ for every P1
  □ System diagrams updated within a week of architectural changes
```

---

## 6. Team Health Metrics

```erlang
%% team_metrics.erl — track developer experience metrics
-module(team_metrics).
-export([weekly_report/0]).

%% DORA Metrics: indicators of engineering team health
%%
%% Elite performers (top quartile):
%%   Deployment frequency:      Multiple times per day
%%   Lead time for changes:     < 1 hour (commit to production)
%%   Change failure rate:       < 5%
%%   MTTR (mean time to restore): < 1 hour

%% Calculated from git + deployment logs
weekly_report() ->
    #{
        deploys_per_week  => count_deploys(),
        lead_time_median  => median_lead_time(),
        failure_rate      => calculate_failure_rate(),
        mttr_minutes      => average_mttr(),

        %% Team wellbeing indicators
        on_call_pages_per_week  => count_pages_this_week(),
        pr_review_time_hours    => avg_pr_review_time(),
        tech_debt_items         => count_tech_debt_tickets()
    }.

%% When to worry:
%%   lead_time > 1 week: deployment pipeline too complex
%%   failure_rate > 15%: insufficient testing or review
%%   mttr > 4 hours: poor observability or runbooks
%%   on_call_pages > 10/week: system reliability issues
%%   pr_review_time > 48 hours: team bandwidth problem

count_deploys() -> 0.
median_lead_time() -> 0.
calculate_failure_rate() -> 0.0.
average_mttr() -> 0.
count_pages_this_week() -> 0.
avg_pr_review_time() -> 0.
count_tech_debt_tickets() -> 0.
```

---

## 7. แบบฝึกหัด

1. Audit code review process: measure P50/P99 review turnaround time ใน 3 เดือนที่ผ่านมา
2. เขียน RFC สำหรับ improvement ที่อยากทำใน codebase ของทีม
3. สร้าง runbook สำหรับ top 3 alerts ที่ fire บ่อยที่สุด
4. วัด DORA metrics ของทีม และเปรียบเทียบกับ industry benchmarks

---

## สรุป Part 98

✅ Code review culture: goals, reviewer/author mindset, turnaround SLA  
✅ Erlang checklist: process design, error handling, type safety, security  
✅ RFC process: template with summary, motivation, alternatives, open questions  
✅ On-call handbook: P1/P2/P3 response levels, runbook template  
✅ Knowledge sharing: tech talks, pair programming, code tours, documentation  
✅ DORA metrics: deployment frequency, lead time, failure rate, MTTR  

---

*Part 98/100 | [← ก่อนหน้า](../part97/README.md) | [ถัดไป →](../part99/README.md)*
