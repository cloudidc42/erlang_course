# Part 16: Mnesia — Distributed Database

> **"Mnesia is Erlang's built-in distributed, fault-tolerant database"**  
> Mnesia คือ distributed database ที่ built-in อยู่ใน Erlang

---

## สารบัญ

1. [Mnesia คืออะไร?](#1-mnesia-คืออะไร)
2. [Setup และ Configuration](#2-setup-และ-configuration)
3. [Schema และ Table Creation](#3-schema-และ-table-creation)
4. [CRUD Operations](#4-crud-operations)
5. [Transactions](#5-transactions)
6. [Queries และ QLC](#6-queries-และ-qlc)
7. [Dirty Operations](#7-dirty-operations)
8. [Distributed Mnesia](#8-distributed-mnesia)
9. [Backup และ Recovery](#9-backup-และ-recovery)
10. [แบบฝึกหัด](#10-แบบฝึกหัด)

---

## 1. Mnesia คืออะไร?

```
Mnesia:
├── Distributed DBMS built into Erlang/OTP
├── ใช้ ETS + DETS สำหรับ storage
├── ACID transactions
├── ทำงานได้ทั้ง in-memory และ disk
├── Replication ระหว่าง Erlang nodes
├── ข้อมูลคือ Erlang records/tuples
└── ไม่ต้องมี external DB (เหมือน SQLite แต่ distributed)

เหมาะสำหรับ:
- Configuration data
- Session data
- Small-medium datasets
- ต้องการ distribution และ fault-tolerance

ไม่เหมาะสำหรับ:
- Large datasets (>1GB per table)
- Complex SQL queries
- Write-heavy workloads
```

---

## 2. Setup และ Configuration

```erlang
%% 1. สร้าง schema ก่อน start mnesia
mnesia:create_schema([node()]).
%% สร้าง directory: Mnesia.nodename@host/

%% 2. Start mnesia
mnesia:start().

%% หรือใน sys.config
%% {mnesia, [{dir, "/var/lib/myapp/mnesia"}]}

%% 3. ใน application callback
start(_Type, _Args) ->
    ok = mnesia:create_schema([node()]),
    ok = mnesia:start(),
    create_tables(),
    myapp_sup:start_link().

create_tables() ->
    Tables = [
        {user, [{attributes, record_info(fields, user)},
                {disc_copies, [node()]},
                {type, set}]},
        {session, [{attributes, record_info(fields, session)},
                   {ram_copies, [node()]},
                   {type, set}]}
    ],
    [create_table(T, Opts) || {T, Opts} <- Tables].

create_table(Table, Opts) ->
    case mnesia:create_table(Table, Opts) of
        {atomic, ok}           -> ok;
        {aborted, {already_exists, Table}} -> ok;
        {aborted, Reason}      -> error({create_table_failed, Reason})
    end.
```

---

## 3. Schema และ Table Creation

```erlang
%% กำหนด records
-record(user, {
    id       :: pos_integer(),
    name     :: binary(),
    email    :: binary(),
    password :: binary(),
    created  :: integer()
}).

-record(session, {
    token   :: binary(),
    user_id :: pos_integer(),
    expires :: integer()
}).

%% Storage types:
%% ram_copies      — memory only, fast, lost on restart
%% disc_copies     — memory + disk (replicate writes to disk)
%% disc_only_copies — disk only, slow, persistent

%% Table options
mnesia:create_table(user, [
    {attributes, record_info(fields, user)},  %% field names
    {disc_copies, [node()]},
    {type, set},                              %% set | bag | ordered_set
    {index, [email]}                          %% secondary index
]).

%% Table info
mnesia:table_info(user, all).
mnesia:table_info(user, size).
mnesia:table_info(user, type).

%% Wait for tables (ถ้า distributed)
mnesia:wait_for_tables([user, session], 5000).

%% Add/remove columns (schema evolution)
mnesia:add_table_copy(user, OtherNode, disc_copies).
mnesia:del_table_copy(user, OtherNode).
```

---

## 4. CRUD Operations

```erlang
%% ทุก operation ต้องอยู่ใน transaction

%% Create / Update
create_user(Id, Name, Email, Password) ->
    User = #user{
        id      = Id,
        name    = Name,
        email   = Email,
        password = hash_password(Password),
        created = erlang:system_time(second)
    },
    mnesia:transaction(fun() ->
        mnesia:write(User)
    end).

%% Read
get_user(Id) ->
    mnesia:transaction(fun() ->
        case mnesia:read(user, Id) of
            [User] -> {ok, User};
            []     -> {error, not_found}
        end
    end).

%% Update
update_user_name(Id, NewName) ->
    mnesia:transaction(fun() ->
        case mnesia:read(user, Id) of
            [User] ->
                mnesia:write(User#user{name = NewName}),
                ok;
            [] ->
                {error, not_found}
        end
    end).

%% Delete
delete_user(Id) ->
    mnesia:transaction(fun() ->
        mnesia:delete({user, Id})
    end).

%% Transaction result
{atomic, Result} = mnesia:transaction(fun() -> ... end).
{aborted, Reason} = mnesia:transaction(fun() -> ... end).
```

---

## 5. Transactions

```erlang
%% ACID Transactions
mnesia:transaction(Fun) ->
    case mnesia:transaction(Fun) of
        {atomic, Result} -> {ok, Result};
        {aborted, Reason} -> {error, Reason}
    end.

%% Nested transactions
transfer(FromId, ToId, Amount) ->
    mnesia:transaction(fun() ->
        From = hd(mnesia:read(account, FromId)),
        To   = hd(mnesia:read(account, ToId)),
        
        case From#account.balance >= Amount of
            false -> mnesia:abort(insufficient_funds);
            true  ->
                mnesia:write(From#account{balance = From#account.balance - Amount}),
                mnesia:write(To#account{balance = To#account.balance + Amount}),
                ok
        end
    end).

%% Abort transaction
mnesia:transaction(fun() ->
    mnesia:write(Record),
    case validate(Record) of
        ok    -> ok;
        Error -> mnesia:abort(Error)  %% rollback
    end
end).

%% Activity types
mnesia:activity(transaction, Fun).
mnesia:activity(ets, Fun).           %% dirty, no transaction
mnesia:activity(async_dirty, Fun).   %% async dirty
mnesia:activity(sync_dirty, Fun).    %% sync dirty
```

---

## 6. Queries และ QLC

```erlang
%% qlc — Query List Comprehension สำหรับ Mnesia

%% ต้อง include
-include_lib("stdlib/include/qlc.hrl").

%% Simple query
get_all_users() ->
    mnesia:transaction(fun() ->
        qlc:eval(qlc:q([X || X <- mnesia:table(user)]))
    end).

%% Filtered query
get_users_by_name(Name) ->
    mnesia:transaction(fun() ->
        Q = qlc:q([X || X <- mnesia:table(user),
                        X#user.name =:= Name]),
        qlc:eval(Q)
    end).

%% Complex query with join
get_user_sessions(UserId) ->
    mnesia:transaction(fun() ->
        Q = qlc:q([{U, S} ||
                   U <- mnesia:table(user),
                   S <- mnesia:table(session),
                   U#user.id =:= UserId,
                   S#session.user_id =:= UserId]),
        qlc:eval(Q)
    end).

%% Sorted query
get_users_sorted() ->
    mnesia:transaction(fun() ->
        Q = qlc:sort(qlc:q([X || X <- mnesia:table(user)]),
                     [{order, ascending}]),
        qlc:eval(Q)
    end).

%% Use secondary index
get_user_by_email(Email) ->
    mnesia:transaction(fun() ->
        %% index_read: ใช้ secondary index
        mnesia:index_read(user, Email, #user.email)
    end).
```

---

## 7. Dirty Operations

```erlang
%% Dirty operations: ไม่มี transaction overhead
%% เร็วกว่า แต่ไม่ atomic, ไม่มี rollback

mnesia:dirty_write(Record).
mnesia:dirty_read(Table, Key).
mnesia:dirty_delete({Table, Key}).
mnesia:dirty_update_counter(Table, Key, Increment).
mnesia:dirty_all_keys(Table).
mnesia:dirty_index_read(Table, Value, Index).

%% เหมาะสำหรับ:
%% - Read-only lookups ที่ต้องการ performance
%% - Single-node operations ที่ไม่ต้องการ consistency
%% - Counters และ statistics

%% ตัวอย่าง
get_session_fast(Token) ->
    case mnesia:dirty_read(session, Token) of
        [Session] ->
            Now = erlang:system_time(second),
            case Session#session.expires > Now of
                true  -> {ok, Session};
                false -> {error, expired}
            end;
        [] ->
            {error, not_found}
    end.
```

---

## 8. Distributed Mnesia

```erlang
%% Setup cluster
%% Node1: mnesia:create_schema([node1@host, node2@host]).
%% ทุก node: mnesia:start().

%% Replicate table ไป node อื่น
mnesia:add_table_copy(user, 'node2@host', disc_copies).

%% Fragmented tables สำหรับ horizontal sharding
mnesia:create_table(big_table, [
    {frag_properties, [
        {node_pool, [node1@host, node2@host]},
        {n_fragments, 8},
        {n_disc_copies, 1}
    ]}
]).

%% Mnesia topology: master-master (ทุก node มีข้อมูลเท่ากัน)
%% ต่างกับ master-slave

%% Network partition: Mnesia ใช้ "majority" หรือ "first" strategy
%% ต้องเลือกว่าจะ resolve conflict อย่างไร

%% Subscribe to events
mnesia:subscribe(system).
mnesia:subscribe({table, user, simple}).
mnesia:subscribe({table, user, detailed}).

receive
    {mnesia_table_event, {write, user, NewRecord, OldRecords, ActivityId}} ->
        handle_write(NewRecord);
    {mnesia_table_event, {delete, user, {user, Key}, _, _}} ->
        handle_delete(Key)
end.
```

---

## 9. Backup และ Recovery

```erlang
%% Backup
mnesia:backup("/path/to/backup.bup").

%% Restore
mnesia:restore("/path/to/backup.bup", [{default_op, recreate_tables}]).

%% Checkpoint — consistent snapshot
{ok, Name, _} = mnesia:activate_checkpoint([
    {name, my_checkpoint},
    {max, mnesia:system_info(tables)}
]),
mnesia:backup_checkpoint(Name, "/path/to/backup.bup"),
mnesia:deactivate_checkpoint(Name).

%% Table dump to disk
mnesia:dump_tables([user, session]).

%% ดู disk files
mnesia:system_info(directory).

%% Schema evolution
mnesia:transform_table(user, fun(OldUser) ->
    %% migrate old record to new format
    OldUser
end, record_info(fields, user)).
```

---

## 10. แบบฝึกหัด

### Exercise: User Database

```erlang
-module(user_db).
-export([start/0, create/3, find/1, find_by_email/1, list/0, delete/1]).

-record(user, {
    id       :: binary(),
    name     :: binary(),
    email    :: binary(),
    created  :: integer()
}).

start() ->
    mnesia:create_schema([node()]),
    mnesia:start(),
    mnesia:create_table(user, [
        {attributes, record_info(fields, user)},
        {disc_copies, [node()]},
        {index, [email]},
        {type, set}
    ]),
    ok.

create(Name, Email, _Password) ->
    Id = uuid(),
    User = #user{
        id      = Id,
        name    = iolist_to_binary(Name),
        email   = iolist_to_binary(Email),
        created = erlang:system_time(second)
    },
    case mnesia:transaction(fun() ->
        case mnesia:index_read(user, iolist_to_binary(Email), #user.email) of
            [] ->
                mnesia:write(User),
                {ok, Id};
            [_] ->
                mnesia:abort(email_already_exists)
        end
    end) of
        {atomic, Result} -> Result;
        {aborted, Reason} -> {error, Reason}
    end.

find(Id) ->
    case mnesia:transaction(fun() -> mnesia:read(user, Id) end) of
        {atomic, [User]} -> {ok, User};
        {atomic, []}     -> {error, not_found};
        {aborted, R}     -> {error, R}
    end.

find_by_email(Email) ->
    case mnesia:transaction(fun() ->
        mnesia:index_read(user, iolist_to_binary(Email), #user.email)
    end) of
        {atomic, [User]} -> {ok, User};
        {atomic, []}     -> {error, not_found};
        {aborted, R}     -> {error, R}
    end.

list() ->
    mnesia:transaction(fun() ->
        qlc:eval(qlc:q([U || U <- mnesia:table(user)]))
    end).

delete(Id) ->
    mnesia:transaction(fun() ->
        mnesia:delete({user, Id})
    end).

uuid() ->
    binary:encode_hex(crypto:strong_rand_bytes(16)).
```

---

## สรุป Part 16

✅ Mnesia คืออะไร — distributed DBMS  
✅ Schema creation และ table setup  
✅ Storage types: ram_copies, disc_copies, disc_only_copies  
✅ CRUD ด้วย transactions  
✅ QLC — Query List Comprehension  
✅ Dirty operations สำหรับ performance  
✅ Distributed Mnesia  
✅ Backup และ recovery  
✅ Secondary indexes

---

*Part 16/100 | [← ก่อนหน้า](../part15/README.md) | [ถัดไป →](../part17/README.md)*
