# Part 24: Database Integration — PostgreSQL

> **"Databases are external services — always handle connection failures"**  
> Database คือ external service — จัดการ connection failure เสมอ

---

## สารบัญ

1. [Database Options ใน Erlang](#1-database-options-ใน-erlang)
2. [epgsql — PostgreSQL Driver](#2-epgsql--postgresql-driver)
3. [Connection Pool ด้วย poolboy](#3-connection-pool-ด้วย-poolboy)
4. [Query Execution](#4-query-execution)
5. [Transactions](#5-transactions)
6. [Prepared Statements](#6-prepared-statements)
7. [Migration ด้วย epgsql](#7-migration-ด้วย-epgsql)
8. [Repository Pattern](#8-repository-pattern)
9. [แบบฝึกหัด](#9-แบบฝึกหัด)

---

## 1. Database Options ใน Erlang

```
PostgreSQL:
- epgsql: Native Erlang driver (แนะนำ)
- pgsql: older driver

MySQL:
- mysql-otp: เพิ่ม dependency

SQLite:
- esqlite: NIF-based

MongoDB:
- mongodb-erlang

Redis:
- eredis

Connection Pool:
- poolboy: generic process pool (แนะนำ)
- eredis_pool: Redis specific
```

---

## 2. epgsql — PostgreSQL Driver

```erlang
%% rebar.config
{deps, [
    {epgsql, "4.7.0"},
    {poolboy, "1.5.2"}
]}.

%% Direct connection (ไม่ใช้ pool)
{ok, Conn} = epgsql:connect(#{
    host     => "localhost",
    port     => 5432,
    username => "myuser",
    password => "mypass",
    database => "mydb",
    timeout  => 5000,
    ssl      => false
}).

%% Simple query
{ok, Cols, Rows} = epgsql:squery(Conn, "SELECT id, name FROM users").

%% Parameterized query (ป้องกัน SQL injection)
{ok, Cols, Rows} = epgsql:equery(Conn, 
    "SELECT id, name FROM users WHERE id = $1",
    [42]
).

%% Close connection
epgsql:close(Conn).
```

---

## 3. Connection Pool ด้วย poolboy

```erlang
%% src/db_pool.erl
-module(db_pool).
-export([start_link/1, with_connection/1]).

-define(POOL_NAME, db_pool).
-define(POOL_SIZE, 10).

start_link(DbConfig) ->
    PoolArgs = [{name, {local, ?POOL_NAME}},
                {worker_module, db_worker},
                {size, ?POOL_SIZE},
                {max_overflow, 5}],
    poolboy:start_link(PoolArgs, DbConfig).

with_connection(Fun) ->
    poolboy:transaction(?POOL_NAME, Fun).

%% src/db_worker.erl — worker ที่ hold connection
-module(db_worker).
-behaviour(gen_server).
-behaviour(poolboy_worker).

-export([start_link/1]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2, terminate/2]).

start_link(DbConfig) ->
    gen_server:start_link(?MODULE, DbConfig, []).

init(DbConfig) ->
    process_flag(trap_exit, true),
    {ok, Conn} = connect(DbConfig),
    {ok, #{conn => Conn, config => DbConfig}}.

handle_call({query, SQL, Params}, _From, #{conn := Conn} = State) ->
    Result = epgsql:equery(Conn, SQL, Params),
    {reply, Result, State};

handle_call({squery, SQL}, _From, #{conn := Conn} = State) ->
    Result = epgsql:squery(Conn, SQL),
    {reply, Result, State}.

handle_cast(_Msg, State) ->
    {noreply, State}.

handle_info({'EXIT', _Conn, Reason}, #{config := Config} = State) ->
    logger:warning("DB connection lost: ~p, reconnecting...", [Reason]),
    case connect(Config) of
        {ok, NewConn} ->
            {noreply, State#{conn => NewConn}};
        {error, _} ->
            {stop, connection_failed, State}
    end.

terminate(_Reason, #{conn := Conn}) ->
    epgsql:close(Conn),
    ok.

connect(#{host := H, port := P, username := U,
          password := Pass, database := DB}) ->
    epgsql:connect(#{host => H, port => P, username => U,
                     password => Pass, database => DB,
                     timeout => 5000}).
```

---

## 4. Query Execution

```erlang
%% Query helper module
-module(db).
-export([query/2, query/3, execute/2]).

query(SQL, Params) ->
    db_pool:with_connection(fun(Worker) ->
        case gen_server:call(Worker, {query, SQL, Params}) of
            {ok, Cols, Rows} ->
                {ok, rows_to_maps(Cols, Rows)};
            {error, Reason} ->
                {error, format_error(Reason)}
        end
    end).

query(SQL, Params, Opts) ->
    Timeout = maps:get(timeout, Opts, 5000),
    db_pool:with_connection(fun(Worker) ->
        case gen_server:call(Worker, {query, SQL, Params}, Timeout) of
            {ok, Cols, Rows} ->
                {ok, rows_to_maps(Cols, Rows)};
            {error, _} = Error ->
                Error
        end
    end).

execute(SQL, Params) ->
    db_pool:with_connection(fun(Worker) ->
        case gen_server:call(Worker, {query, SQL, Params}) of
            {ok, _Count} -> ok;
            {error, Reason} -> {error, format_error(Reason)}
        end
    end).

rows_to_maps(Cols, Rows) ->
    ColNames = [element(2, Col) || Col <- Cols],
    [maps:from_list(lists:zip(ColNames, tuple_to_list(Row))) || Row <- Rows].

format_error({error, #error{code = Code, message = Msg}}) ->
    #{code => Code, message => Msg};
format_error(Other) ->
    Other.
```

---

## 5. Transactions

```erlang
%% Transaction helper
transaction(Fun) ->
    db_pool:with_connection(fun(Worker) ->
        gen_server:call(Worker, {squery, "BEGIN"}),
        try
            Result = Fun(Worker),
            gen_server:call(Worker, {squery, "COMMIT"}),
            {ok, Result}
        catch
            Class:Reason ->
                gen_server:call(Worker, {squery, "ROLLBACK"}),
                {error, {Class, Reason}}
        end
    end).

%% ตัวอย่าง transfer
transfer_money(FromId, ToId, Amount) ->
    transaction(fun(Worker) ->
        {ok, [#{<<"balance">> := FromBal}]} =
            gen_server:call(Worker, {query,
                "SELECT balance FROM accounts WHERE id = $1 FOR UPDATE",
                [FromId]}),
        
        case FromBal >= Amount of
            false ->
                throw(insufficient_funds);
            true ->
                gen_server:call(Worker, {query,
                    "UPDATE accounts SET balance = balance - $1 WHERE id = $2",
                    [Amount, FromId]}),
                gen_server:call(Worker, {query,
                    "UPDATE accounts SET balance = balance + $1 WHERE id = $2",
                    [Amount, ToId]}),
                ok
        end
    end).
```

---

## 6. Prepared Statements

```erlang
%% Prepare statement ครั้งเดียว ใช้หลายครั้ง
prepare_statements(Conn) ->
    Statements = [
        {find_user,    "SELECT id, name, email FROM users WHERE id = $1"},
        {find_by_email,"SELECT id, name, email FROM users WHERE email = $1"},
        {create_user,  "INSERT INTO users (name, email) VALUES ($1, $2) RETURNING id"},
        {update_user,  "UPDATE users SET name=$1, email=$2 WHERE id=$3"},
        {delete_user,  "DELETE FROM users WHERE id=$1"}
    ],
    [begin
        {ok, _} = epgsql:parse(Conn, atom_to_list(Name), SQL, []),
        ok
     end || {Name, SQL} <- Statements].

%% Execute prepared statement
execute_prepared(Conn, find_user, [Id]) ->
    epgsql:prepared_query(Conn, "find_user", [Id]).

execute_prepared(Conn, create_user, [Name, Email]) ->
    epgsql:prepared_query(Conn, "create_user", [Name, Email]).
```

---

## 7. Migration ด้วย epgsql

```erlang
%% Simple migration system
-module(db_migration).
-export([run/1]).

run(Conn) ->
    ok = ensure_migrations_table(Conn),
    Migrations = load_migrations(),
    Applied = get_applied(Conn),
    Pending = [M || M <- Migrations, not lists:member(M, Applied)],
    
    lists:foreach(fun(Migration) ->
        apply_migration(Conn, Migration)
    end, Pending).

ensure_migrations_table(Conn) ->
    SQL = "CREATE TABLE IF NOT EXISTS schema_migrations (
             version VARCHAR(255) PRIMARY KEY,
             applied_at TIMESTAMP DEFAULT NOW()
           )",
    case epgsql:squery(Conn, SQL) of
        {ok, _, _} -> ok;
        {ok, _} -> ok
    end.

load_migrations() ->
    MigrationDir = filename:join(code:priv_dir(myapp), "migrations"),
    {ok, Files} = file:list_dir(MigrationDir),
    SortedFiles = lists:sort(Files),
    [F || F <- SortedFiles, filename:extension(F) =:= ".sql"].

get_applied(Conn) ->
    {ok, _, Rows} = epgsql:squery(Conn,
        "SELECT version FROM schema_migrations ORDER BY version"),
    [binary_to_list(V) || {V} <- Rows].

apply_migration(Conn, Filename) ->
    Version = filename:rootname(Filename),
    MigrationDir = filename:join(code:priv_dir(myapp), "migrations"),
    {ok, SQL} = file:read_file(filename:join(MigrationDir, Filename)),
    
    epgsql:squery(Conn, "BEGIN"),
    try
        case epgsql:squery(Conn, binary_to_list(SQL)) of
            {ok, _, _} -> ok;
            {ok, _} -> ok;
            {error, Reason} -> throw({migration_failed, Reason})
        end,
        epgsql:equery(Conn,
            "INSERT INTO schema_migrations (version) VALUES ($1)",
            [Version]),
        epgsql:squery(Conn, "COMMIT"),
        logger:info("Applied migration: ~s", [Version])
    catch
        throw:Error ->
            epgsql:squery(Conn, "ROLLBACK"),
            throw(Error)
    end.
```

---

## 8. Repository Pattern

```erlang
%% user_repo.erl — ทุก DB query สำหรับ users
-module(user_repo).
-export([find/1, find_by_email/1, create/1, update/2, delete/1, list/1]).

-define(TABLE, "users").

find(Id) when is_integer(Id) ->
    SQL = "SELECT id, name, email, created_at FROM users WHERE id = $1",
    case db:query(SQL, [Id]) of
        {ok, [Row]} -> {ok, row_to_user(Row)};
        {ok, []}    -> {error, not_found};
        Error       -> Error
    end.

find_by_email(Email) ->
    SQL = "SELECT id, name, email, created_at FROM users WHERE email = $1",
    case db:query(SQL, [Email]) of
        {ok, [Row]} -> {ok, row_to_user(Row)};
        {ok, []}    -> {error, not_found};
        Error       -> Error
    end.

create(#{name := Name, email := Email}) ->
    SQL = "INSERT INTO users (name, email, created_at) VALUES ($1, $2, NOW()) RETURNING id, name, email, created_at",
    case db:query(SQL, [Name, Email]) of
        {ok, [Row]} -> {ok, row_to_user(Row)};
        {error, #{code := <<"23505">>}} -> {error, email_taken};  %% unique violation
        Error -> Error
    end.

update(Id, Changes) ->
    {Sets, Params} = build_update_sets(Changes),
    SQL = io_lib:format(
        "UPDATE users SET ~s WHERE id = $~p RETURNING id, name, email, created_at",
        [Sets, length(Params) + 1]
    ),
    case db:query(iolist_to_binary(SQL), Params ++ [Id]) of
        {ok, [Row]} -> {ok, row_to_user(Row)};
        {ok, []}    -> {error, not_found};
        Error       -> Error
    end.

delete(Id) ->
    SQL = "DELETE FROM users WHERE id = $1",
    case db:execute(SQL, [Id]) of
        ok    -> ok;
        Error -> Error
    end.

list(#{page := Page, limit := Limit}) ->
    Offset = (Page - 1) * Limit,
    SQL = "SELECT id, name, email, created_at FROM users ORDER BY id LIMIT $1 OFFSET $2",
    case db:query(SQL, [Limit, Offset]) of
        {ok, Rows} -> {ok, [row_to_user(R) || R <- Rows]};
        Error      -> Error
    end.

row_to_user(#{<<"id">> := Id, <<"name">> := Name,
              <<"email">> := Email, <<"created_at">> := CreatedAt}) ->
    #{id => Id, name => Name, email => Email, created_at => CreatedAt}.

build_update_sets(Changes) ->
    Fields = maps:to_list(Changes),
    {Sets, Params, _} = lists:foldl(
        fun({Field, Value}, {S, P, I}) ->
            SetClause = io_lib:format("~s = $~p", [Field, I]),
            {[SetClause | S], [Value | P], I + 1}
        end,
        {[], [], 1},
        Fields
    ),
    {string:join(lists:reverse(Sets), ", "), lists:reverse(Params)}.
```

---

## 9. แบบฝึกหัด

### Exercise: Blog Database Layer

```erlang
%% สร้าง repository สำหรับ blog
%% Tables: posts, comments, tags, post_tags

%% Migration file: priv/migrations/001_create_blog_tables.sql
%% CREATE TABLE posts (
%%   id SERIAL PRIMARY KEY,
%%   title VARCHAR(255) NOT NULL,
%%   body TEXT NOT NULL,
%%   author_id INTEGER NOT NULL,
%%   published BOOLEAN DEFAULT false,
%%   created_at TIMESTAMP DEFAULT NOW()
%% );
%%
%% CREATE TABLE comments (
%%   id SERIAL PRIMARY KEY,
%%   post_id INTEGER REFERENCES posts(id) ON DELETE CASCADE,
%%   author_id INTEGER NOT NULL,
%%   body TEXT NOT NULL,
%%   created_at TIMESTAMP DEFAULT NOW()
%% );

-module(post_repo).
-export([create/1, find/1, list_published/1, publish/1, add_comment/2]).

create(#{title := Title, body := Body, author_id := AuthorId}) ->
    SQL = "INSERT INTO posts (title, body, author_id) VALUES ($1, $2, $3) RETURNING *",
    case db:query(SQL, [Title, Body, AuthorId]) of
        {ok, [Row]} -> {ok, row_to_post(Row)};
        Error       -> Error
    end.

find(Id) ->
    SQL = "SELECT p.*, COUNT(c.id) as comment_count
           FROM posts p
           LEFT JOIN comments c ON c.post_id = p.id
           WHERE p.id = $1
           GROUP BY p.id",
    case db:query(SQL, [Id]) of
        {ok, [Row]} -> {ok, row_to_post(Row)};
        {ok, []}    -> {error, not_found};
        Error       -> Error
    end.

list_published(#{limit := L, offset := O}) ->
    SQL = "SELECT * FROM posts WHERE published = true ORDER BY created_at DESC LIMIT $1 OFFSET $2",
    case db:query(SQL, [L, O]) of
        {ok, Rows} -> {ok, [row_to_post(R) || R <- Rows]};
        Error      -> Error
    end.

publish(Id) ->
    SQL = "UPDATE posts SET published = true WHERE id = $1",
    db:execute(SQL, [Id]).

add_comment(PostId, #{author_id := AuthorId, body := Body}) ->
    SQL = "INSERT INTO comments (post_id, author_id, body) VALUES ($1, $2, $3) RETURNING *",
    case db:query(SQL, [PostId, AuthorId, Body]) of
        {ok, [Row]} -> {ok, row_to_comment(Row)};
        Error       -> Error
    end.

row_to_post(Row) ->
    #{
        id         => maps:get(<<"id">>, Row),
        title      => maps:get(<<"title">>, Row),
        body       => maps:get(<<"body">>, Row),
        author_id  => maps:get(<<"author_id">>, Row),
        published  => maps:get(<<"published">>, Row),
        created_at => maps:get(<<"created_at">>, Row)
    }.

row_to_comment(Row) ->
    #{
        id         => maps:get(<<"id">>, Row),
        post_id    => maps:get(<<"post_id">>, Row),
        author_id  => maps:get(<<"author_id">>, Row),
        body       => maps:get(<<"body">>, Row),
        created_at => maps:get(<<"created_at">>, Row)
    }.
```

---

## สรุป Part 24

✅ epgsql PostgreSQL driver  
✅ Connection pool ด้วย poolboy  
✅ Query helpers: query/2,3, execute/2  
✅ Row to map conversion  
✅ Transactions  
✅ Prepared statements  
✅ Simple migration system  
✅ Repository pattern

---

*Part 24/100 | [← ก่อนหน้า](../part23/README.md) | [ถัดไป →](../part25/README.md)*
