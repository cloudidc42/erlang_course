# Part 93: Financial Systems — Double-Entry and Reconciliation

> **"Every debit has a credit — and every credit has an audit trail"**  
> ทุก debit มี credit — และทุก credit มี audit trail

---

## สารบัญ

1. [Double-Entry Bookkeeping](#1-double-entry-bookkeeping)
2. [Ledger and Account Management](#2-ledger-and-account-management)
3. [Transaction Processing](#3-transaction-processing)
4. [Reconciliation Engine](#4-reconciliation-engine)
5. [Fraud Detection Rules Engine](#5-fraud-detection-rules-engine)
6. [Financial Reports](#6-financial-reports)
7. [แบบฝึกหัด](#7-แบบฝึกหัด)

---

## 1. Double-Entry Bookkeeping

```
Double-Entry Bookkeeping Fundamentals
═══════════════════════════════════════════════════════

RULE: Every transaction has two sides that must balance
  Total debits = Total credits  (always)

Account types and normal balance:
  Asset:      Debit increases, Credit decreases
  Liability:  Credit increases, Debit decreases
  Equity:     Credit increases, Debit decreases
  Revenue:    Credit increases, Debit decreases
  Expense:    Debit increases, Credit decreases

Example: Customer pays $100 for product

  Debit:  Cash (Asset)           +$100
  Credit: Revenue (Revenue)      +$100

Example: Pay $50 supplier invoice

  Debit:  Accounts Payable (Liability)  -$50  (reduce debt)
  Credit: Cash (Asset)                  -$50  (reduce cash)

Why Erlang for financial systems?
  - Immutable message passing: no race conditions on account balances
  - BEAM process isolation: payment failure doesn't take down reporting
  - Pattern matching: express financial rules declaratively
  - Let-it-crash: supervisor restarts without data loss (DB is source of truth)
```

---

## 2. Ledger and Account Management

```erlang
%% ledger.erl — chart of accounts and ledger operations
-module(ledger).
-export([create_account/3, get_balance/1, get_account/1,
         credit/3, debit/3]).

-define(ACCOUNT_TYPES, [asset, liability, equity, revenue, expense]).

%% Account structure in PostgreSQL:
%% CREATE TABLE accounts (
%%   id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
%%   code VARCHAR(20) UNIQUE NOT NULL,  -- e.g. '1001' for Cash
%%   name VARCHAR(200) NOT NULL,
%%   type VARCHAR(20) NOT NULL,         -- asset|liability|equity|revenue|expense
%%   parent_code VARCHAR(20),           -- for sub-accounts
%%   currency CHAR(3) NOT NULL DEFAULT 'USD',
%%   created_at TIMESTAMPTZ DEFAULT NOW()
%% );
%%
%% CREATE TABLE journal_entries (
%%   id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
%%   transaction_id UUID NOT NULL REFERENCES transactions(id),
%%   account_code VARCHAR(20) NOT NULL REFERENCES accounts(code),
%%   entry_type VARCHAR(6) NOT NULL,    -- 'debit' or 'credit'
%%   amount_cents BIGINT NOT NULL CHECK (amount_cents > 0),
%%   description TEXT,
%%   created_at TIMESTAMPTZ DEFAULT NOW()
%% );

create_account(Code, Name, Type) when lists:member(Type, ?ACCOUNT_TYPES) ->
    db_pool:execute(
        "INSERT INTO accounts (code, name, type) VALUES ($1, $2, $3)",
        [Code, Name, atom_to_binary(Type)]).

get_balance(AccountCode) ->
    case db_pool:query(
        "SELECT
           SUM(CASE WHEN entry_type = 'debit'  THEN amount_cents ELSE 0 END) as debits,
           SUM(CASE WHEN entry_type = 'credit' THEN amount_cents ELSE 0 END) as credits
         FROM journal_entries
         WHERE account_code = $1",
        [AccountCode]) of
        {ok, [{Debits, Credits}]} ->
            D = Debits  orelse 0,
            C = Credits orelse 0,
            {ok, #{debits => D, credits => C, net => D - C}};
        Err -> Err
    end.

get_account(AccountCode) ->
    db_pool:query(
        "SELECT code, name, type, currency FROM accounts WHERE code = $1",
        [AccountCode]).

credit(AccountCode, AmountCents, Description) when AmountCents > 0 ->
    add_journal_entry(AccountCode, credit, AmountCents, Description).

debit(AccountCode, AmountCents, Description) when AmountCents > 0 ->
    add_journal_entry(AccountCode, debit, AmountCents, Description).

add_journal_entry(AccountCode, Type, AmountCents, Description) ->
    db_pool:execute(
        "INSERT INTO journal_entries (account_code, entry_type, amount_cents, description)
         VALUES ($1, $2, $3, $4)",
        [AccountCode, atom_to_binary(Type), AmountCents, Description]).
```

---

## 3. Transaction Processing

```erlang
%% transactions.erl — atomic double-entry transactions
-module(transactions).
-export([record/3, record_payment/4, void/2]).

%% Record a transaction with balanced journal entries
%% Entries: [{AccountCode, debit|credit, AmountCents, Description}]
record(Description, Reference, Entries) ->
    %% Validate balance before inserting
    case validate_balanced(Entries) of
        ok ->
            db_pool:transaction(fun(Conn) ->
                record_transaction(Conn, Description, Reference, Entries)
            end);
        {error, _} = Err -> Err
    end.

validate_balanced(Entries) ->
    TotalDebits  = lists:sum([A || {_, debit,  A, _} <- Entries]),
    TotalCredits = lists:sum([A || {_, credit, A, _} <- Entries]),
    case TotalDebits =:= TotalCredits of
        true  -> ok;
        false -> {error, {unbalanced, TotalDebits, TotalCredits}}
    end.

record_transaction(Conn, Description, Reference, Entries) ->
    {ok, [{TxId}]} = epgsql:equery(Conn,
        "INSERT INTO transactions (description, reference, created_at)
         VALUES ($1, $2, NOW())
         RETURNING id",
        [Description, Reference]),
    lists:foreach(fun({AccountCode, Type, AmountCents, Desc}) ->
        {ok, _} = epgsql:equery(Conn,
            "INSERT INTO journal_entries
               (transaction_id, account_code, entry_type, amount_cents, description)
             VALUES ($1, $2, $3, $4, $5)",
            [TxId, AccountCode, atom_to_binary(Type), AmountCents, Desc])
    end, Entries),
    {ok, TxId}.

%% High-level: record a payment from customer
record_payment(CustomerId, AmountCents, Method, Reference) ->
    CashAccount    = account_code_for_method(Method),
    RevenueAccount = <<"4000">>,  %% Revenue
    record(
        <<"Customer payment">>,
        Reference,
        [
            {CashAccount,    debit,  AmountCents, <<"Received payment from ", CustomerId/binary>>},
            {RevenueAccount, credit, AmountCents, <<"Revenue for payment ", Reference/binary>>}
        ]).

%% Void: reverse a transaction by creating equal-and-opposite entries
void(TransactionId, VoidReason) ->
    case db_pool:query(
        "SELECT account_code, entry_type, amount_cents, description
         FROM journal_entries WHERE transaction_id = $1",
        [TransactionId]) of
        {ok, Rows} ->
            VoidEntries = [{Code, flip_type(Type), Amount,
                            <<"VOID: ", Desc/binary, " — ", VoidReason/binary>>}
                           || {Code, Type, Amount, Desc} <- Rows],
            record(<<"VOID: ", VoidReason/binary>>, TransactionId, VoidEntries);
        Err -> Err
    end.

flip_type(<<"debit">>)  -> credit;
flip_type(<<"credit">>) -> debit.

account_code_for_method(card)         -> <<"1010">>;  %% Bank account
account_code_for_method(bank_transfer)-> <<"1010">>;
account_code_for_method(cash)         -> <<"1001">>;  %% Cash
account_code_for_method(_)            -> <<"1010">>.
```

---

## 4. Reconciliation Engine

```erlang
%% reconciliation.erl — match internal records with bank statements
-module(reconciliation).
-export([import_bank_statement/2, reconcile/2, get_discrepancies/1]).

import_bank_statement(AccountCode, Entries) ->
    db_pool:transaction(fun(_Conn) ->
        lists:foreach(fun(#{date := Date, amount := Amount,
                             reference := Ref, description := Desc}) ->
            db_pool:execute(
                "INSERT INTO bank_statement_entries
                   (account_code, date, amount_cents, reference, description, reconciled)
                 VALUES ($1, $2, $3, $4, $5, false)
                 ON CONFLICT (account_code, reference) DO NOTHING",
                [AccountCode, Date, Amount, Ref, Desc])
        end, Entries)
    end).

reconcile(AccountCode, StatementDate) ->
    %% Find unreconciled bank entries
    {ok, BankEntries} = db_pool:query(
        "SELECT id, amount_cents, reference
         FROM bank_statement_entries
         WHERE account_code = $1 AND date = $2 AND reconciled = false",
        [AccountCode, StatementDate]),

    %% Find matching journal entries
    lists:foreach(fun({BankId, Amount, Reference}) ->
        case find_matching_journal_entry(AccountCode, Amount, Reference) of
            {ok, JournalId} ->
                mark_reconciled(BankId, JournalId);
            not_found ->
                ok  %% Will appear in discrepancies
        end
    end, BankEntries),

    get_discrepancies(AccountCode).

find_matching_journal_entry(AccountCode, Amount, Reference) ->
    case db_pool:query(
        "SELECT je.id FROM journal_entries je
         JOIN transactions t ON t.id = je.transaction_id
         WHERE je.account_code = $1
           AND je.amount_cents = $2
           AND t.reference = $3
           AND je.reconciled_at IS NULL
         LIMIT 1",
        [AccountCode, Amount, Reference]) of
        {ok, [{JournalId}]} -> {ok, JournalId};
        {ok, []}            -> not_found
    end.

mark_reconciled(BankId, JournalId) ->
    Now = erlang:system_time(second),
    db_pool:execute(
        "UPDATE bank_statement_entries SET reconciled = true,
         reconciled_at = $1, matched_journal_id = $2 WHERE id = $3",
        [Now, JournalId, BankId]),
    db_pool:execute(
        "UPDATE journal_entries SET reconciled_at = $1 WHERE id = $2",
        [Now, JournalId]).

get_discrepancies(AccountCode) ->
    db_pool:query(
        "SELECT 'bank_only' as type, date, amount_cents, reference, description
         FROM bank_statement_entries
         WHERE account_code = $1 AND reconciled = false
         UNION ALL
         SELECT 'journal_only', je.created_at::date,
                je.amount_cents, t.reference, je.description
         FROM journal_entries je
         JOIN transactions t ON t.id = je.transaction_id
         WHERE je.account_code = $1
           AND je.reconciled_at IS NULL
           AND je.created_at < NOW() - INTERVAL '3 days'",
        [AccountCode]).
```

---

## 5. Fraud Detection Rules Engine

```erlang
%% fraud_rules.erl — declarative fraud detection rules
-module(fraud_rules).
-export([check/2, add_rule/2, remove_rule/1]).

-define(RULES_TABLE, fraud_rules).

-record(rule, {
    id,
    name,
    condition,   %% fun(Transaction) -> boolean()
    action,      %% block | flag | review
    priority = 100
}).

%% Check a transaction against all rules
check(Transaction, Context) ->
    Rules = get_sorted_rules(),
    check_rules(Transaction, Context, Rules, []).

check_rules(_Tx, _Ctx, [], Findings) ->
    {ok, Findings};
check_rules(Tx, Ctx, [#rule{condition = Cond, action = Action, name = Name} | Rest],
            Findings) ->
    case safe_eval(Cond, Tx, Ctx) of
        true ->
            Finding = #{rule => Name, action => Action},
            case Action of
                block ->
                    {blocked, [Finding | Findings]};
                _ ->
                    check_rules(Tx, Ctx, Rest, [Finding | Findings])
            end;
        false ->
            check_rules(Tx, Ctx, Rest, Findings)
    end.

%% Built-in rules
default_rules() ->
    [
        #rule{id = velocity_1h,
              name = <<"velocity_1h">>,
              priority = 10,
              action = block,
              condition = fun(Tx, Ctx) ->
                Count = maps:get(tx_count_1h, Ctx, 0),
                Count > 20 orelse maps:get(amount_cents, Tx) > 10_000_00  %% $10k
              end},

        #rule{id = new_device,
              name = <<"new_device">>,
              priority = 50,
              action = flag,
              condition = fun(_Tx, Ctx) ->
                maps:get(is_new_device, Ctx, false)
              end},

        #rule{id = high_value,
              name = <<"high_value">>,
              priority = 30,
              action = review,
              condition = fun(Tx, _Ctx) ->
                maps:get(amount_cents, Tx, 0) > 50_000_00  %% $50k
              end}
    ].

safe_eval(Cond, Tx, Ctx) ->
    try Cond(Tx, Ctx)
    catch _:_ -> false
    end.

add_rule(Id, Rule) ->
    ets:insert(?RULES_TABLE, Rule#rule{id = Id}).

remove_rule(Id) ->
    ets:delete(?RULES_TABLE, Id).

get_sorted_rules() ->
    Rules = ets:tab2list(?RULES_TABLE) ++ default_rules(),
    lists:sort(fun(A, B) -> A#rule.priority =< B#rule.priority end, Rules).
```

---

## 6. Financial Reports

```erlang
%% financial_reports.erl — balance sheet, P&L, cash flow
-module(financial_reports).
-export([balance_sheet/1, profit_and_loss/2, trial_balance/1]).

balance_sheet(AsOfDate) ->
    {ok, Assets}      = account_balances(AsOfDate, asset),
    {ok, Liabilities} = account_balances(AsOfDate, liability),
    {ok, Equity}      = account_balances(AsOfDate, equity),
    TotalAssets = sum_balances(Assets),
    TotalLiab   = sum_balances(Liabilities),
    TotalEquity = sum_balances(Equity),
    #{
        as_of           => AsOfDate,
        assets          => #{accounts => Assets, total => TotalAssets},
        liabilities     => #{accounts => Liabilities, total => TotalLiab},
        equity          => #{accounts => Equity, total => TotalEquity},
        balanced        => TotalAssets =:= TotalLiab + TotalEquity
    }.

profit_and_loss(FromDate, ToDate) ->
    {ok, Revenue}  = period_balances(FromDate, ToDate, revenue),
    {ok, Expenses} = period_balances(FromDate, ToDate, expense),
    TotalRevenue  = sum_balances(Revenue),
    TotalExpenses = sum_balances(Expenses),
    NetIncome     = TotalRevenue - TotalExpenses,
    #{
        period     => #{from => FromDate, to => ToDate},
        revenue    => #{accounts => Revenue, total => TotalRevenue},
        expenses   => #{accounts => Expenses, total => TotalExpenses},
        net_income => NetIncome
    }.

trial_balance(AsOfDate) ->
    db_pool:query(
        "SELECT a.code, a.name, a.type,
                SUM(CASE WHEN je.entry_type = 'debit'  THEN je.amount_cents ELSE 0 END) as debits,
                SUM(CASE WHEN je.entry_type = 'credit' THEN je.amount_cents ELSE 0 END) as credits
         FROM accounts a
         LEFT JOIN journal_entries je ON je.account_code = a.code
         LEFT JOIN transactions t ON t.id = je.transaction_id
         WHERE t.created_at::date <= $1 OR t.id IS NULL
         GROUP BY a.code, a.name, a.type
         ORDER BY a.code",
        [AsOfDate]).

account_balances(AsOfDate, Type) ->
    db_pool:query(
        "SELECT a.code, a.name,
                SUM(CASE WHEN je.entry_type = 'debit'  THEN je.amount_cents ELSE 0 END)
                - SUM(CASE WHEN je.entry_type = 'credit' THEN je.amount_cents ELSE 0 END)
                as balance
         FROM accounts a
         LEFT JOIN journal_entries je ON je.account_code = a.code
         LEFT JOIN transactions t ON t.id = je.transaction_id
         WHERE a.type = $1 AND (t.created_at::date <= $2 OR t.id IS NULL)
         GROUP BY a.code, a.name",
        [atom_to_binary(Type), AsOfDate]).

period_balances(From, To, Type) ->
    db_pool:query(
        "SELECT a.code, a.name,
                SUM(je.amount_cents) as total
         FROM accounts a
         JOIN journal_entries je ON je.account_code = a.code
         JOIN transactions t ON t.id = je.transaction_id
         WHERE a.type = $1
           AND t.created_at::date BETWEEN $2 AND $3
         GROUP BY a.code, a.name",
        [atom_to_binary(Type), From, To]).

sum_balances(Rows) ->
    lists:sum([B || {_, _, B} <- Rows, is_integer(B)]).
```

---

## 7. แบบฝึกหัด

1. Implement currency conversion: store all amounts in base currency, convert on display
2. สร้าง accounts payable workflow: invoice → approve → pay → reconcile
3. เพิ่ม audit log: ทุก change ต้องบันทึก who changed what and when
4. Implement month-end close: lock prior periods เพื่อป้องกันการแก้ไขย้อนหลัง

---

## สรุป Part 93

✅ Double-entry fundamentals: debit/credit balance, account types  
✅ Ledger: chart of accounts, balance query aggregating journal entries  
✅ Transactions: balanced validation, atomic insertion, void by reversal  
✅ Reconciliation: import bank statement, match to journal, discrepancy report  
✅ Fraud detection: rules engine with priority, block/flag/review actions  
✅ Financial reports: balance sheet, P&L, trial balance queries  

---

*Part 93/100 | [← ก่อนหน้า](../part92/README.md) | [ถัดไป →](../part94/README.md)*
