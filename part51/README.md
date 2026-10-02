# Part 51: Erlang in Fintech

> **"Financial systems demand correctness above all — Erlang delivers"**  
> ระบบการเงินต้องการความถูกต้องเหนือสิ่งอื่นใด — Erlang ทำได้

---

## สารบัญ

1. [Fintech Architecture](#1-fintech-architecture)
2. [Double-Entry Bookkeeping](#2-double-entry-bookkeeping)
3. [Transaction Idempotency](#3-transaction-idempotency)
4. [Fraud Detection](#4-fraud-detection)
5. [Reconciliation](#5-reconciliation)
6. [Regulatory Compliance Logging](#6-regulatory-compliance-logging)
7. [แบบฝึกหัด](#7-แบบฝึกหัด)

---

## 1. Fintech Architecture

```
Fintech System:

  [Mobile/Web Client]
       |
  [API Gateway] — rate limit, auth
       |
  [Payment Service] — idempotency, saga
       |
  [Ledger Service] ← double-entry
       |
  [Postgres] + [Event Store]
       |
  [Fraud Engine] — rules + ML scoring
       |
  [Compliance Logger] — immutable audit trail
```

---

## 2. Double-Entry Bookkeeping

```erlang
%% ledger.erl — double-entry accounting
-module(ledger).
-export([debit/3, credit/3, transfer/4, balance/1]).

%% Every transaction: debit one account, credit another
%% Debit:  Assets/Expenses increase | Liabilities/Equity/Revenue decrease
%% Credit: Liabilities/Equity/Revenue increase | Assets/Expenses decrease
%% Rule: Sum of all debits = Sum of all credits (always)

debit(AccountId, Amount, TxnId) when Amount > 0 ->
    post_entry(TxnId, AccountId, debit, Amount).

credit(AccountId, Amount, TxnId) when Amount > 0 ->
    post_entry(TxnId, AccountId, credit, Amount).

transfer(FromId, ToId, Amount, TxnId) ->
    db:transaction(fun() ->
        %% Check idempotency
        case get_txn(TxnId) of
            {ok, _} -> {error, duplicate_transaction};
            not_found ->
                {ok, _} = debit(FromId, Amount, TxnId),
                {ok, _} = credit(ToId, Amount, TxnId),
                record_txn(TxnId, FromId, ToId, Amount)
        end
    end).

balance(AccountId) ->
    {ok, Rows} = db:query(
        "SELECT "
        "  COALESCE(SUM(CASE WHEN type='debit'  THEN amount ELSE 0 END), 0) debits,"
        "  COALESCE(SUM(CASE WHEN type='credit' THEN amount ELSE 0 END), 0) credits "
        "FROM ledger_entries WHERE account_id = $1",
        [AccountId]
    ),
    [#{<<"debits">> := D, <<"credits">> := C}] = Rows,
    D - C.   %% For asset accounts: debit balance = positive

post_entry(TxnId, AccountId, Type, Amount) ->
    db:execute(
        "INSERT INTO ledger_entries "
        "(transaction_id, account_id, type, amount, created_at) "
        "VALUES ($1, $2, $3, $4, NOW())",
        [TxnId, AccountId, atom_to_binary(Type, utf8), Amount]
    ).

get_txn(TxnId) ->
    case db:query("SELECT id FROM transactions WHERE id = $1", [TxnId]) of
        {ok, [Row | _]} -> {ok, Row};
        {ok, []}        -> not_found
    end.

record_txn(TxnId, From, To, Amount) ->
    db:execute(
        "INSERT INTO transactions (id, from_account, to_account, amount) "
        "VALUES ($1, $2, $3, $4)",
        [TxnId, From, To, Amount]
    ).
```

---

## 3. Transaction Idempotency

```erlang
%% idempotency.erl — ensure transactions execute exactly once
-module(idempotency).
-export([with_idempotency/3]).

%% Idempotency key: client-generated UUID
%% Same key = same response, no re-execution

with_idempotency(IdempotencyKey, Fun, TTL) ->
    case check_cache(IdempotencyKey) of
        {ok, CachedResult} ->
            {cached, CachedResult};
        miss ->
            %% Acquire distributed lock for this key
            case lock:acquire(IdempotencyKey, 30_000) of
                {ok, LockRef} ->
                    %% Double-check after lock
                    case check_cache(IdempotencyKey) of
                        {ok, CachedResult} ->
                            lock:release(LockRef),
                            {cached, CachedResult};
                        miss ->
                            Result = try Fun()
                                     catch C:R -> {error, {C, R}}
                                     end,
                            store_result(IdempotencyKey, Result, TTL),
                            lock:release(LockRef),
                            {fresh, Result}
                    end;
                {error, timeout} ->
                    {error, concurrent_request}
            end
    end.

check_cache(Key) ->
    case ets:lookup(idempotency_cache, Key) of
        [{Key, Result, ExpiresAt}] ->
            case os:system_time(second) < ExpiresAt of
                true  -> {ok, Result};
                false -> miss
            end;
        [] -> miss
    end.

store_result(Key, Result, TTL) ->
    ExpiresAt = os:system_time(second) + TTL,
    ets:insert(idempotency_cache, {Key, Result, ExpiresAt}).
```

---

## 4. Fraud Detection

```erlang
%% fraud_detector.erl — rule-based fraud detection
-module(fraud_detector).
-export([check/2]).

-define(VELOCITY_WINDOW, 3600).   %% 1 hour
-define(MAX_HOURLY, 10_000).      %% USD cents
-define(MAX_TXN_COUNT, 10).
-define(SUSPICIOUS_COUNTRIES, [<<"XX">>, <<"YY">>]).

check(UserId, Txn = #{amount := Amount, country := Country}) ->
    Rules = [
        fun() -> check_velocity(UserId, Amount) end,
        fun() -> check_country(Country) end,
        fun() -> check_unusual_amount(UserId, Amount) end,
        fun() -> check_card_pattern(UserId) end
    ],
    Scores = [score_rule(R()) || R <- Rules],
    TotalScore = lists:sum(Scores),
    Decision = case TotalScore of
        S when S >= 80 -> block;
        S when S >= 50 -> review;
        _ -> allow
    end,
    log_check(UserId, Txn, TotalScore, Decision),
    {Decision, TotalScore}.

check_velocity(UserId, Amount) ->
    Window   = os:system_time(second) - ?VELOCITY_WINDOW,
    {ok, Rows} = db:query(
        "SELECT COALESCE(SUM(amount), 0) total, COUNT(*) count "
        "FROM transactions WHERE user_id = $1 AND created_at > to_timestamp($2)",
        [UserId, Window]
    ),
    [#{<<"total">> := Total, <<"count">> := Count}] = Rows,
    if Total + Amount > ?MAX_HOURLY -> {risk, 60};
       Count >= ?MAX_TXN_COUNT      -> {risk, 40};
       true                         -> {ok, 0}
    end.

check_country(Country) ->
    case lists:member(Country, ?SUSPICIOUS_COUNTRIES) of
        true  -> {risk, 70};
        false -> {ok, 0}
    end.

check_unusual_amount(UserId, Amount) ->
    {ok, Rows} = db:query(
        "SELECT COALESCE(AVG(amount), 0) avg_amount "
        "FROM transactions WHERE user_id = $1",
        [UserId]
    ),
    [#{<<"avg_amount">> := Avg}] = Rows,
    case Avg > 0 andalso Amount > Avg * 5 of
        true  -> {risk, 30};
        false -> {ok, 0}
    end.

check_card_pattern(_UserId) -> {ok, 0}.  %% TODO: implement

score_rule({risk, Score}) -> Score;
score_rule({ok, _})       -> 0.

log_check(UserId, Txn, Score, Decision) ->
    logger:info("fraud_check user=~p score=~p decision=~p txn=~p",
                [UserId, Score, Decision, Txn]).
```

---

## 5. Reconciliation

```erlang
%% reconciliation.erl — daily reconciliation job
-module(reconciliation).
-export([run/1]).

run(Date) ->
    logger:info("Starting reconciliation for ~s", [Date]),
    Steps = [
        fun() -> reconcile_ledger(Date) end,
        fun() -> reconcile_external(Date) end,
        fun() -> generate_report(Date) end
    ],
    lists:foldl(fun(Step, ok) ->
        case Step() of
            ok           -> ok;
            {error, Err} -> throw({reconciliation_failed, Err})
        end
    end, ok, Steps).

reconcile_ledger(Date) ->
    {ok, Rows} = db:query(
        "SELECT "
        "  SUM(CASE WHEN type='debit' THEN amount ELSE 0 END) total_debits, "
        "  SUM(CASE WHEN type='credit' THEN amount ELSE 0 END) total_credits "
        "FROM ledger_entries WHERE DATE(created_at) = $1",
        [Date]
    ),
    [#{<<"total_debits">> := D, <<"total_credits">> := C}] = Rows,
    case D =:= C of
        true  ->
            logger:info("Ledger balanced: ~p", [D]),
            ok;
        false ->
            Diff = abs(D - C),
            logger:error("Ledger UNBALANCED! Diff=~p", [Diff]),
            {error, {unbalanced, Diff}}
    end.

reconcile_external(_Date) -> ok.  %% Compare with payment processor

generate_report(Date) ->
    {ok, Rows} = db:query(
        "SELECT account_id, "
        "  SUM(CASE WHEN type='debit' THEN amount ELSE -amount END) balance "
        "FROM ledger_entries WHERE DATE(created_at) = $1 "
        "GROUP BY account_id ORDER BY account_id",
        [Date]
    ),
    logger:info("Daily report: ~p accounts, date=~s", [length(Rows), Date]),
    ok.
```

---

## 6. Regulatory Compliance Logging

```erlang
%% compliance_log.erl — immutable audit trail
-module(compliance_log).
-export([log_transaction/2, log_access/3, export/2]).

log_transaction(TxnId, Details) ->
    db:execute(
        "INSERT INTO compliance_log "
        "(event_type, entity_id, details, hash, created_at) "
        "VALUES ('transaction', $1, $2::jsonb, $3, NOW())",
        [TxnId, jsx:encode(Details), compute_hash(TxnId, Details)]
    ).

log_access(UserId, Resource, Action) ->
    db:execute(
        "INSERT INTO compliance_log "
        "(event_type, entity_id, details, created_at) "
        "VALUES ('access', $1, $2::jsonb, NOW())",
        [UserId, jsx:encode(#{resource => Resource, action => Action})]
    ).

%% Export for regulatory reporting
export(FromDate, ToDate) ->
    {ok, Rows} = db:query(
        "SELECT * FROM compliance_log "
        "WHERE created_at BETWEEN $1 AND $2 "
        "ORDER BY created_at",
        [FromDate, ToDate]
    ),
    %% Write to encrypted file or send to regulator API
    {ok, Rows}.

compute_hash(TxnId, Details) ->
    Data = <<TxnId/binary, (jsx:encode(Details))/binary>>,
    Hash = crypto:hash(sha256, Data),
    base64:encode(Hash).
```

---

## 7. แบบฝึกหัด

1. เพิ่ม `ledger:freeze_account/1` ที่ block transactions แต่ยังอ่านได้
2. สร้าง `settlement.erl`: batch หลาย transactions เป็น single settlement
3. เพิ่ม velocity check: limit ตาม risk tier (silver/gold/platinum)
4. สร้าง integrity check: verify ว่า hash chain ใน compliance_log ไม่ถูก tamper

---

## สรุป Part 51

✅ Double-entry bookkeeping ใน Erlang  
✅ Transaction idempotency  
✅ Rule-based fraud detection  
✅ Daily reconciliation  
✅ Immutable compliance audit trail  

---

*Part 51/100 | [← ก่อนหน้า](../part50/README.md) | [ถัดไป →](../part52/README.md)*
