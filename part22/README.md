# Part 22: REST API ครบถ้วน

> **"REST is not about HTTP methods — it's about resources and representations"**  
> REST ไม่ใช่แค่เรื่อง HTTP methods — แต่เกี่ยวกับ resources และ representations

---

## สารบัญ

1. [REST Design Principles](#1-rest-design-principles)
2. [cowboy_rest Behaviour](#2-cowboy_rest-behaviour)
3. [Content Negotiation](#3-content-negotiation)
4. [CRUD Resource Handler](#4-crud-resource-handler)
5. [Pagination](#5-pagination)
6. [Authentication Middleware](#6-authentication-middleware)
7. [JSON Encoding/Decoding](#7-json-encodingdecoding)
8. [Error Responses](#8-error-responses)
9. [API Versioning](#9-api-versioning)
10. [แบบฝึกหัด](#10-แบบฝึกหัด)

---

## 1. REST Design Principles

```
REST Principles:
1. Resource-based URLs: /users, /users/123, /users/123/posts
2. HTTP Methods:
   GET    — read
   POST   — create
   PUT    — full update (replace)
   PATCH  — partial update
   DELETE — delete
3. Stateless: ทุก request มีข้อมูลครบ
4. Representations: JSON, XML, etc.
5. HATEOAS (Hypermedia): links ใน response

HTTP Status Codes:
200 OK
201 Created
204 No Content
400 Bad Request
401 Unauthorized
403 Forbidden
404 Not Found
405 Method Not Allowed
409 Conflict
422 Unprocessable Entity
429 Too Many Requests
500 Internal Server Error
```

---

## 2. cowboy_rest Behaviour

```erlang
%% cowboy_rest: content negotiation ฟรี
-module(users_resource).
-behaviour(cowboy_rest).

-export([init/2, allowed_methods/2, content_types_provided/2,
         content_types_accepted/2, resource_exists/2,
         users_to_json/2, accept_json/2]).

init(Req, State) ->
    {cowboy_rest, Req, State}.

allowed_methods(Req, State) ->
    {[<<"GET">>, <<"POST">>, <<"HEAD">>, <<"OPTIONS">>], Req, State}.

%% Content negotiation: ตอบ client ด้วย format ที่เหมาะสม
content_types_provided(Req, State) ->
    {[
        {<<"application/json">>, users_to_json},
        {<<"text/html">>,        users_to_html}
    ], Req, State}.

content_types_accepted(Req, State) ->
    {[
        {<<"application/json">>, accept_json}
    ], Req, State}.

resource_exists(Req, State) ->
    {true, Req, State}.

users_to_json(Req, State) ->
    Users = users:list(),
    Body = jsx:encode([user_to_map(U) || U <- Users]),
    {Body, Req, State}.

users_to_html(Req, State) ->
    Users = users:list(),
    Html = render_users_html(Users),
    {Html, Req, State}.

accept_json(Req, State) ->
    {ok, Body, Req2} = cowboy_req:read_body(Req),
    Data = jsx:decode(Body, [return_maps]),
    case users:create(Data) of
        {ok, User} ->
            Req3 = cowboy_req:set_resp_body(jsx:encode(user_to_map(User)), Req2),
            {{true, <<"/api/users/", (integer_to_binary(User#user.id))/binary>>}, Req3, State};
        {error, Reason} ->
            {false, Req2, State}
    end.

user_to_map(#user{id=Id, name=Name, email=Email}) ->
    #{id => Id, name => Name, email => Email}.
```

---

## 3. Content Negotiation

```erlang
%% cowboy_rest จัดการ Accept header อัตโนมัติ
%% ถ้า client ส่ง Accept: application/json
%% cowboy จะเรียก function ที่ map กับ application/json

content_types_provided(Req, State) ->
    {[
        %% {MediaType, HandlerFunction}
        {{<<"application">>, <<"json">>, []}, to_json},
        {{<<"application">>, <<"xml">>, []}, to_xml},
        {<<"*/*">>, to_json}  %% default
    ], Req, State}.

to_json(Req, State) ->
    Data = get_data(State),
    {jsx:encode(Data), Req, State}.

to_xml(Req, State) ->
    Data = get_data(State),
    {encode_xml(Data), Req, State}.

%% ตรวจสอบ Accept header ด้วยตัวเอง
check_accept(Req) ->
    Accept = cowboy_req:header(<<"accept">>, Req, <<"*/*">>),
    case binary:match(Accept, <<"application/json">>) of
        {_, _} -> json;
        nomatch -> html
    end.
```

---

## 4. CRUD Resource Handler

```erlang
%% Resource handler ที่สมบูรณ์
-module(user_resource).
-behaviour(cowboy_rest).

-export([init/2, allowed_methods/2, content_types_provided/2,
         content_types_accepted/2, resource_exists/2,
         delete_resource/2, to_json/2, from_json/2]).

init(Req, _State) ->
    Id = cowboy_req:binding(id, Req),
    {cowboy_rest, Req, #{id => Id}}.

allowed_methods(Req, State) ->
    {[<<"GET">>, <<"PUT">>, <<"PATCH">>, <<"DELETE">>,
      <<"HEAD">>, <<"OPTIONS">>], Req, State}.

content_types_provided(Req, State) ->
    {[{<<"application/json">>, to_json}], Req, State}.

content_types_accepted(Req, State) ->
    {[{<<"application/json">>, from_json}], Req, State}.

resource_exists(Req, #{id := Id} = State) ->
    case users:find(binary_to_integer(Id)) of
        {ok, User} ->
            {true, Req, State#{user => User}};
        {error, not_found} ->
            {false, Req, State}
    end.

to_json(Req, #{user := User} = State) ->
    {jsx:encode(user_to_map(User)), Req, State}.

from_json(Req, #{id := Id} = State) ->
    {ok, Body, Req2} = cowboy_req:read_body(Req),
    Data = jsx:decode(Body, [return_maps]),
    Method = cowboy_req:method(Req),
    
    case Method of
        <<"PUT">> ->
            case users:replace(binary_to_integer(Id), Data) of
                {ok, Updated} ->
                    Req3 = cowboy_req:set_resp_body(
                        jsx:encode(user_to_map(Updated)), Req2),
                    {true, Req3, State#{user => Updated}};
                {error, Reason} ->
                    error_response(422, Reason, Req2, State)
            end;
        <<"PATCH">> ->
            case users:update(binary_to_integer(Id), Data) of
                {ok, Updated} ->
                    Req3 = cowboy_req:set_resp_body(
                        jsx:encode(user_to_map(Updated)), Req2),
                    {true, Req3, State#{user => Updated}};
                {error, Reason} ->
                    error_response(422, Reason, Req2, State)
            end
    end.

delete_resource(Req, #{id := Id} = State) ->
    case users:delete(binary_to_integer(Id)) of
        ok -> {true, Req, State};
        _  -> {false, Req, State}
    end.

error_response(Status, Reason, Req, State) ->
    Body = jsx:encode(#{error => format_error(Reason)}),
    Req2 = cowboy_req:set_resp_body(Body, Req),
    cowboy_req:reply(Status, Req2),
    {halt, Req2, State}.

user_to_map(#user{id=Id, name=N, email=E}) ->
    #{id => Id, name => N, email => E}.

format_error(A) when is_atom(A) -> atom_to_binary(A, utf8);
format_error(B) when is_binary(B) -> B;
format_error(T) -> iolist_to_binary(io_lib:format("~p", [T])).
```

---

## 5. Pagination

```erlang
%% Cursor-based pagination
-module(paginated_handler).
-export([init/2]).

init(Req, State) ->
    Params = cowboy_req:parse_qs(Req),
    Limit  = param_integer(Params, <<"limit">>, 20),
    Cursor = param_binary(Params, <<"cursor">>, undefined),
    
    {Items, NextCursor} = fetch_page(Limit, Cursor),
    
    Response = #{
        data   => Items,
        cursor => NextCursor,
        limit  => Limit
    },
    
    Req2 = cowboy_req:reply(200,
        #{<<"content-type">> => <<"application/json">>},
        jsx:encode(Response),
        Req
    ),
    {ok, Req2, State}.

fetch_page(Limit, undefined) ->
    Items = db:list_users(#{limit => Limit + 1}),
    paginate(Items, Limit);
fetch_page(Limit, Cursor) ->
    Items = db:list_users(#{limit => Limit + 1, after_cursor => Cursor}),
    paginate(Items, Limit).

paginate(Items, Limit) ->
    case length(Items) > Limit of
        true ->
            Page = lists:sublist(Items, Limit),
            Last = lists:last(Page),
            {[item_to_map(I) || I <- Page], encode_cursor(Last)};
        false ->
            {[item_to_map(I) || I <- Items], null}
    end.

encode_cursor(Item) ->
    base64:encode(term_to_binary(Item#user.id)).

param_integer(Params, Key, Default) ->
    case lists:keyfind(Key, 1, Params) of
        {Key, V} ->
            try binary_to_integer(V)
            catch _:_ -> Default
            end;
        false -> Default
    end.

param_binary(Params, Key, Default) ->
    case lists:keyfind(Key, 1, Params) of
        {Key, V} -> V;
        false    -> Default
    end.
```

---

## 6. Authentication Middleware

```erlang
%% JWT Authentication middleware
-module(auth_middleware).
-behaviour(cowboy_middleware).
-export([execute/2]).

-define(PUBLIC_PATHS, [<<"/api/login">>, <<"/api/register">>, <<"/health">>]).

execute(Req, Env) ->
    Path = cowboy_req:path(Req),
    
    case lists:member(Path, ?PUBLIC_PATHS) of
        true ->
            {ok, Req, Env};
        false ->
            authenticate(Req, Env)
    end.

authenticate(Req, Env) ->
    AuthHeader = cowboy_req:header(<<"authorization">>, Req, <<>>),
    
    case AuthHeader of
        <<"Bearer ", Token/binary>> ->
            case jwt:verify(Token) of
                {ok, Claims} ->
                    UserId = maps:get(<<"sub">>, Claims),
                    Req2 = cowboy_req:set_meta(user_id, UserId, Req),
                    {ok, Req2, Env};
                {error, _} ->
                    unauthorized(Req)
            end;
        _ ->
            unauthorized(Req)
    end.

unauthorized(Req) ->
    Req2 = cowboy_req:reply(401,
        #{<<"content-type">> => <<"application/json">>},
        jsx:encode(#{error => <<"Unauthorized">>}),
        Req
    ),
    {stop, Req2}.
```

---

## 7. JSON Encoding/Decoding

```erlang
%% ใช้ jsx library
%% rebar.config: {deps, [{jsx, "3.1.0"}]}

%% Decode
jsx:decode(<<"[1,2,3]">>).            %% [1,2,3]
jsx:decode(<<"{\"a\":1}">>, [return_maps]).  %% #{<<"a">> => 1}

%% Encode
jsx:encode([1, 2, 3]).                %% <<"[1,2,3]">>
jsx:encode(#{a => 1, b => <<"hello">>}).  %% <<"{"a":1,"b":"hello"}">>

%% Encode options
jsx:encode(Data, [pretty_print]).
jsx:encode(Data, [{indent, 2}]).

%% Custom encoding
encode_user(#user{id=Id, name=Name}) ->
    jsx:encode(#{
        id   => Id,
        name => Name
    }).

%% Handle null
jsx:encode(null).      %% <<"null">>
jsx:decode(<<"null">>). %% null

%% Nested
jsx:encode(#{users => [#{id => 1}, #{id => 2}]}).
%% <<"{"users":[{"id":1},{"id":2}]}">>

%% Safe decode
safe_decode(Body) ->
    try {ok, jsx:decode(Body, [return_maps])}
    catch _:_ -> {error, invalid_json}
    end.
```

---

## 8. Error Responses

```erlang
%% Error response module
-module(api_errors).
-export([bad_request/2, not_found/1, unauthorized/1, internal/1]).

bad_request(Reason, Req) ->
    error_reply(400, <<"Bad Request">>, Reason, Req).

not_found(Req) ->
    error_reply(404, <<"Not Found">>, <<"Resource not found">>, Req).

unauthorized(Req) ->
    error_reply(401, <<"Unauthorized">>, <<"Authentication required">>, Req).

internal(Req) ->
    error_reply(500, <<"Internal Server Error">>, <<"An error occurred">>, Req).

error_reply(Status, Title, Detail, Req) ->
    Body = jsx:encode(#{
        error => #{
            status => Status,
            title  => Title,
            detail => Detail
        }
    }),
    cowboy_req:reply(Status,
        #{<<"content-type">> => <<"application/json">>},
        Body,
        Req
    ).

%% Validation errors
validation_error(Errors, Req) ->
    Body = jsx:encode(#{
        error => #{
            status => 422,
            title  => <<"Validation Failed">>,
            errors => [#{field => F, message => M} || {F, M} <- Errors]
        }
    }),
    cowboy_req:reply(422,
        #{<<"content-type">> => <<"application/json">>},
        Body,
        Req
    ).
```

---

## 9. API Versioning

```erlang
%% URL-based versioning
routes() ->
    [{'_', [
        {"/api/v1/users",    v1_users_handler, []},
        {"/api/v2/users",    v2_users_handler, []},
        {"/api/users",       latest_users_handler, []}
    ]}].

%% Header-based versioning
versioned_init(Req, State) ->
    Version = cowboy_req:header(<<"api-version">>, Req, <<"v1">>),
    handle_version(Version, Req, State).

handle_version(<<"v1">>, Req, State) ->
    v1_handler:init(Req, State);
handle_version(<<"v2">>, Req, State) ->
    v2_handler:init(Req, State);
handle_version(_, Req, State) ->
    Req2 = cowboy_req:reply(400,
        #{<<"content-type">> => <<"application/json">>},
        jsx:encode(#{error => <<"Unsupported API version">>}),
        Req
    ),
    {ok, Req2, State}.

%% Accept header versioning
%% Accept: application/vnd.myapp.v2+json
accept_version(Req) ->
    Accept = cowboy_req:header(<<"accept">>, Req, <<"application/json">>),
    case re:run(Accept, "vnd\\.myapp\\.v(\\d+)\\+json",
                [{capture, [1], binary}]) of
        {match, [Version]} -> binary_to_integer(Version);
        nomatch -> 1  %% default v1
    end.
```

---

## 10. แบบฝึกหัด

### Exercise: Blog API

```erlang
%% สร้าง Blog API ที่สมบูรณ์
%% GET  /api/posts          — list posts (paginated)
%% GET  /api/posts/:id      — get post
%% POST /api/posts          — create post (auth required)
%% PUT  /api/posts/:id      — update post (auth required, owner only)
%% DELETE /api/posts/:id    — delete post (auth required, owner only)

-module(posts_router).
-export([routes/0]).

routes() ->
    [{'_', [
        {"/api/posts",       posts_list_handler, []},
        {"/api/posts/:id",   posts_item_handler, []},
        {"/api/login",       auth_handler, []},
        {"/health",          health_handler, []}
    ]}].

%% posts_list_handler.erl
-module(posts_list_handler).
-export([init/2]).

init(Req, State) ->
    case cowboy_req:method(Req) of
        <<"GET">>  -> list_posts(Req, State);
        <<"POST">> -> create_post(Req, State);
        _          ->
            Req2 = cowboy_req:reply(405, #{<<"allow">> => <<"GET, POST">>},
                                    <<>>, Req),
            {ok, Req2, State}
    end.

list_posts(Req, State) ->
    Qs = cowboy_req:parse_qs(Req),
    Page  = get_int_param(Qs, <<"page">>, 1),
    Limit = get_int_param(Qs, <<"limit">>, 10),
    
    Posts = blog_db:list_posts(#{page => Page, limit => Limit}),
    Total = blog_db:count_posts(),
    
    Reply = #{
        posts => [post_to_map(P) || P <- Posts],
        meta  => #{page => Page, limit => Limit, total => Total,
                   pages => ceil(Total / Limit)}
    },
    reply_json(200, Reply, Req, State).

create_post(Req, State) ->
    UserId = cowboy_req:meta(user_id, Req),
    {ok, Body, Req2} = cowboy_req:read_body(Req),
    case safe_decode(Body) of
        {ok, #{<<"title">> := Title, <<"body">> := PostBody}} ->
            case blog_db:create_post(#{user_id => UserId,
                                       title => Title,
                                       body => PostBody}) of
                {ok, Post} -> reply_json(201, post_to_map(Post), Req2, State);
                {error, R} -> reply_json(422, #{error => R}, Req2, State)
            end;
        _ ->
            reply_json(400, #{error => <<"Invalid request body">>}, Req2, State)
    end.

reply_json(Status, Data, Req, State) ->
    Req2 = cowboy_req:reply(Status,
        #{<<"content-type">> => <<"application/json">>},
        jsx:encode(Data), Req),
    {ok, Req2, State}.

post_to_map(#{id := Id, title := T, body := B, user_id := U}) ->
    #{id => Id, title => T, body => B, author_id => U}.

safe_decode(Body) ->
    try {ok, jsx:decode(Body, [return_maps])}
    catch _:_ -> {error, invalid_json}
    end.

get_int_param(Qs, Key, Default) ->
    case lists:keyfind(Key, 1, Qs) of
        {Key, V} -> try binary_to_integer(V) catch _:_ -> Default end;
        false    -> Default
    end.
```

---

## สรุป Part 22

✅ REST design principles  
✅ cowboy_rest behaviour  
✅ Content negotiation  
✅ CRUD resource handlers  
✅ Cursor-based pagination  
✅ JWT authentication middleware  
✅ JSX JSON library  
✅ Error response patterns  
✅ API versioning strategies

---

*Part 22/100 | [← ก่อนหน้า](../part21/README.md) | [ถัดไป →](../part23/README.md)*
