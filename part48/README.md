# Part 48: GraphQL API with Erlang

> **"Ask for what you need, get exactly that — nothing more, nothing less"**  
> ถามในสิ่งที่ต้องการ ได้รับในสิ่งนั้นพอดี — ไม่มากไม่น้อย

---

## สารบัญ

1. [GraphQL Concepts](#1-graphql-concepts)
2. [Schema Definition](#2-schema-definition)
3. [Resolver Implementation](#3-resolver-implementation)
4. [Query Execution Engine](#4-query-execution-engine)
5. [DataLoader Pattern](#5-dataloader-pattern)
6. [GraphQL over HTTP/WebSocket](#6-graphql-over-httpwebsocket)
7. [แบบฝึกหัด](#7-แบบฝึกหัด)

---

## 1. GraphQL Concepts

```
GraphQL vs REST:

REST:
  GET /users/1         → user
  GET /users/1/posts   → posts
  GET /posts/5/comments → comments
  (3 requests, over-fetching)

GraphQL:
  query {
    user(id: 1) {
      name
      posts {
        title
        comments { body author { name } }
      }
    }
  }
  (1 request, exactly what you need)

Key Types:
  Query:    read-only
  Mutation: write + return
  Subscription: real-time push
```

---

## 2. Schema Definition

```erlang
%% graphql_schema.erl — define GraphQL schema
-module(graphql_schema).
-export([build/0]).

build() ->
    #{
        types => #{
            user => #{
                fields => #{
                    id    => #{type => id, resolver => fun user_id/2},
                    name  => #{type => string},
                    email => #{type => string},
                    posts => #{type => {list, post}, resolver => fun user_posts/2}
                }
            },
            post => #{
                fields => #{
                    id       => #{type => id},
                    title    => #{type => string},
                    body     => #{type => string},
                    author   => #{type => user, resolver => fun post_author/2},
                    comments => #{type => {list, comment}, resolver => fun post_comments/2}
                }
            },
            comment => #{
                fields => #{
                    id     => #{type => id},
                    body   => #{type => string},
                    author => #{type => user, resolver => fun comment_author/2}
                }
            }
        },
        query => #{
            user => #{
                args => #{id => #{type => id, required => true}},
                type => user,
                resolver => fun resolve_user/2
            },
            users => #{
                args => #{limit => #{type => int, default => 10}},
                type => {list, user},
                resolver => fun resolve_users/2
            },
            posts => #{
                type => {list, post},
                resolver => fun resolve_posts/2
            }
        },
        mutation => #{
            createUser => #{
                args => #{
                    name  => #{type => string, required => true},
                    email => #{type => string, required => true}
                },
                type => user,
                resolver => fun create_user/2
            },
            updateUser => #{
                args => #{
                    id   => #{type => id, required => true},
                    name => #{type => string}
                },
                type => user,
                resolver => fun update_user/2
            }
        }
    }.

%% Resolver functions
resolve_user(#{id := Id}, _Context) ->
    user_repo:find(Id).

resolve_users(#{limit := Limit}, _Context) ->
    user_repo:list(Limit).

resolve_posts(_, _Context) ->
    post_repo:list().

user_posts(#{<<"id">> := UserId}, _Context) ->
    post_repo:by_user(UserId).

post_author(#{<<"author_id">> := AuthorId}, Context) ->
    dataloader:load(Context, users, AuthorId).

post_comments(#{<<"id">> := PostId}, _Context) ->
    comment_repo:by_post(PostId).

comment_author(#{<<"author_id">> := AuthorId}, Context) ->
    dataloader:load(Context, users, AuthorId).

create_user(#{name := Name, email := Email}, _Context) ->
    user_repo:create(#{name => Name, email => Email}).

update_user(#{id := Id, name := Name}, _Context) ->
    user_repo:update(Id, #{name => Name}).

user_id(#{<<"id">> := Id}, _) -> {ok, Id}.
```

---

## 3. Resolver Implementation

```erlang
%% graphql_resolver.erl — execute resolvers with context
-module(graphql_resolver).
-export([resolve/4]).

resolve(Type, Field, Parent, Context) ->
    Schema   = graphql_schema:build(),
    TypeDef  = maps:get(Type, maps:get(types, Schema, #{}), #{}),
    FieldDef = maps:get(Field, maps:get(fields, TypeDef, #{}), #{}),

    case maps:find(resolver, FieldDef) of
        {ok, ResolverFun} ->
            try ResolverFun(Parent, Context)
            catch
                Class:Reason:Stack ->
                    logger:error("Resolver ~p.~p failed: ~p~n~p",
                                 [Type, Field, Reason, Stack]),
                    {error, <<"Internal resolver error">>}
            end;
        error ->
            %% Default resolver: get field from parent map
            FieldBin = atom_to_binary(Field, utf8),
            case maps:find(FieldBin, Parent) of
                {ok, Value} -> {ok, Value};
                error -> {ok, null}
            end
    end.
```

---

## 4. Query Execution Engine

```erlang
%% graphql_executor.erl — parse and execute GraphQL queries
-module(graphql_executor).
-export([execute/3]).

%% Simplified execution — real impl would use a proper parser
execute(Schema, Query, Variables) ->
    case parse_query(Query) of
        {ok, AST} ->
            Context = #{variables => Variables, schema => Schema,
                        dataloader => dataloader:new()},
            execute_selection(AST, Schema, null, Context);
        {error, Reason} ->
            #{errors => [#{message => Reason}]}
    end.

execute_selection(#{type := query, selections := Sels}, Schema, _Parent, Ctx) ->
    QueryType = maps:get(query, Schema),
    Data = maps:from_list([
        {Name, execute_field(Name, Args, QueryType, null, Ctx)}
        || #{name := Name, args := Args} <- Sels
    ]),
    #{data => Data};

execute_selection(#{type := mutation, selections := Sels}, Schema, _Parent, Ctx) ->
    MutType = maps:get(mutation, Schema),
    Data = lists:foldl(fun(#{name := Name, args := Args}, Acc) ->
        Result = execute_field(Name, Args, MutType, null, Ctx),
        maps:put(Name, Result, Acc)
    end, #{}, Sels),
    #{data => Data}.

execute_field(FieldName, Args, TypeDef, Parent, Context) ->
    FieldDef = maps:get(FieldName, TypeDef, #{}),
    case maps:find(resolver, FieldDef) of
        {ok, Fun} ->
            case Fun(merge_args(Args, Parent), Context) of
                {ok, Value} -> resolve_value(Value, FieldDef, Context);
                {error, E}  -> #{errors => [#{message => E}]}
            end;
        error ->
            null
    end.

merge_args(Args, Parent) when is_map(Parent) -> maps:merge(Parent, Args);
merge_args(Args, _)                          -> Args.

resolve_value(Values, #{type := {list, SubType}}, Ctx) when is_list(Values) ->
    [resolve_object(V, SubType, Ctx) || V <- Values];
resolve_value(Value, #{type := SubType}, Ctx) when is_map(Value) ->
    resolve_object(Value, SubType, Ctx);
resolve_value(Value, _, _) -> Value.

resolve_object(Obj, _Type, _Ctx) -> Obj.   %% simplified

parse_query(Query) when is_binary(Query) ->
    %% In practice, use a proper GraphQL parser library
    {ok, #{type => query, selections => []}}.
```

---

## 5. DataLoader Pattern

```erlang
%% dataloader.erl — batch N+1 queries
-module(dataloader).
-export([new/0, load/3, run/1]).

-record(loader, {
    batches = #{}    %% Source => [Key]
}).

new() -> #loader{}.

%% Queue a key for batch loading
load(#loader{batches=B} = Loader, Source, Key) ->
    Existing = maps:get(Source, B, []),
    {deferred, Loader#loader{batches=maps:put(Source, [Key | Existing], B)}}.

%% Execute all batched loads
run(#loader{batches=Batches}) ->
    maps:map(fun(Source, Keys) ->
        UniqueKeys = lists:usort(Keys),
        Results    = fetch_batch(Source, UniqueKeys),
        maps:from_list(lists:zip(UniqueKeys, Results))
    end, Batches).

fetch_batch(users, Ids) ->
    {ok, Users} = db:query(
        "SELECT * FROM users WHERE id = ANY($1)", [Ids]),
    [maps:get(Id, maps:from_list([{maps:get(<<"id">>, U), U} || U <- Users]),
              null)
     || Id <- Ids];

fetch_batch(posts, Ids) ->
    {ok, Posts} = db:query(
        "SELECT * FROM posts WHERE id = ANY($1)", [Ids]),
    [maps:get(Id, maps:from_list([{maps:get(<<"id">>, P), P} || P <- Posts]),
              null)
     || Id <- Ids].
```

---

## 6. GraphQL over HTTP/WebSocket

```erlang
%% graphql_handler.erl — Cowboy handler for GraphQL
-module(graphql_handler).
-behaviour(cowboy_handler).
-export([init/2]).

init(Req0, State) ->
    Method = cowboy_req:method(Req0),
    handle(Method, Req0, State).

handle(<<"POST">>, Req0, State) ->
    {ok, Body, Req} = cowboy_req:read_body(Req0),
    case jsx:decode(Body, [return_maps]) of
        #{<<"query">> := Query} = Input ->
            Variables = maps:get(<<"variables">>, Input, #{}),
            Schema    = graphql_schema:build(),
            Result    = graphql_executor:execute(Schema, Query, Variables),
            Json      = jsx:encode(Result),
            Req2 = cowboy_req:reply(200,
                #{<<"content-type">> => <<"application/json">>},
                Json, Req),
            {ok, Req2, State};
        _ ->
            Req2 = cowboy_req:reply(400, #{}, <<"Invalid request">>, Req),
            {ok, Req2, State}
    end;

handle(<<"GET">>, Req0, State) ->
    %% Serve GraphiQL playground
    Html = graphiql_html(),
    Req = cowboy_req:reply(200,
        #{<<"content-type">> => <<"text/html">>},
        Html, Req0),
    {ok, Req, State}.

graphiql_html() ->
    <<"<!DOCTYPE html>
<html>
<head>
  <title>GraphiQL</title>
  <script src=\"https://cdn.jsdelivr.net/npm/graphiql/graphiql.min.js\"></script>
  <link rel=\"stylesheet\"
    href=\"https://cdn.jsdelivr.net/npm/graphiql/graphiql.min.css\" />
</head>
<body style=\"margin:0\">
  <div id=\"graphiql\" style=\"height:100vh\"></div>
  <script>
    ReactDOM.render(
      React.createElement(GraphiQL, { fetcher: GraphiQL.createFetcher({
        url: '/graphql'
      })}),
      document.getElementById('graphiql')
    );
  </script>
</body>
</html>">>.
```

---

## 7. แบบฝึกหัด

1. เพิ่ม `subscription` type สำหรับ real-time updates ผ่าน WebSocket
2. Implement input validation: check required fields และ type coercion
3. เพิ่ม pagination: `after`, `first`, `before`, `last` arguments
4. Implement field-level authorization: check permissions per field

---

## สรุป Part 48

✅ GraphQL schema definition ใน Erlang  
✅ Resolver functions  
✅ Query execution engine  
✅ DataLoader ป้องกัน N+1 queries  
✅ HTTP handler สำหรับ GraphQL  
✅ GraphiQL playground  

---

*Part 48/100 | [← ก่อนหน้า](../part47/README.md) | [ถัดไป →](../part49/README.md)*
