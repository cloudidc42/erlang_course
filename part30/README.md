# Part 30: Authentication และ Authorization

> **"Security is not a feature — it's a foundation"**  
> Security ไม่ใช่ feature — มันคือ foundation

---

## สารบัญ

1. [Authentication vs Authorization](#1-authentication-vs-authorization)
2. [Password Hashing](#2-password-hashing)
3. [JWT Tokens](#3-jwt-tokens)
4. [Session-Based Auth](#4-session-based-auth)
5. [OAuth2 Flow](#5-oauth2-flow)
6. [Role-Based Access Control (RBAC)](#6-role-based-access-control-rbac)
7. [API Key Authentication](#7-api-key-authentication)
8. [Cowboy Auth Middleware](#8-cowboy-auth-middleware)
9. [แบบฝึกหัด](#9-แบบฝึกหัด)

---

## 1. Authentication vs Authorization

```
Authentication (ฉันคือใคร?):
  - Username/Password
  - Token (JWT, API Key)
  - OAuth (Google, GitHub)
  - Certificate

Authorization (ฉันทำอะไรได้?):
  - Role-based (Admin, User, Guest)
  - Permission-based (read:users, write:posts)
  - Resource-based (owner of post X)
  - Policy-based

Flow:
  Request → [Auth Middleware] → verify identity
                               ↓
                          [Authz Check] → check permissions
                               ↓
                          [Handler] → process request
```

---

## 2. Password Hashing

```erlang
%% ใช้ bcrypt หรือ argon2 สำหรับ password hashing
%% อย่าใช้ MD5, SHA1, SHA256 สำหรับ password!

%% rebar.config
%% {deps, [{bcrypt, "1.2.3"}]}

%% hash password
hash_password(Password) ->
    {ok, Hash} = bcrypt:hashpw(Password, bcrypt:gen_salt(12)),
    list_to_binary(Hash).

%% verify password
verify_password(Password, Hash) ->
    HashStr = binary_to_list(Hash),
    case bcrypt:hashpw(Password, HashStr) of
        {ok, HashStr} -> true;
        _             -> false
    end.

%% ถ้าไม่มี bcrypt: ใช้ crypto + salt (less secure แต่ใช้ได้)
hash_password_simple(Password) ->
    Salt = crypto:strong_rand_bytes(16),
    Iterations = 100000,
    KeyLen = 32,
    Hash = crypto:pbkdf2_hmac(sha256, Password, Salt, Iterations, KeyLen),
    %% เก็บ salt + hash ด้วยกัน
    <<Salt:16/binary, Hash:32/binary>>.

verify_password_simple(Password, <<Salt:16/binary, StoredHash:32/binary>>) ->
    Hash = crypto:pbkdf2_hmac(sha256, Password, Salt, 100000, 32),
    crypto:hash_equals(Hash, StoredHash).

%% ใน user_repo:
register_user(Username, Password) ->
    Hash = hash_password(Password),
    db:insert(users, #{username => Username, password_hash => Hash}).

login(Username, Password) ->
    case db:find_by(users, username, Username) of
        {ok, #{password_hash := Hash} = User} ->
            case verify_password(Password, Hash) of
                true  -> {ok, maps:remove(password_hash, User)};
                false -> {error, invalid_credentials}
            end;
        {error, not_found} ->
            %% Constant time to prevent timing attacks
            verify_password(<<"dummy">>, hash_password(<<"dummy">>)),
            {error, invalid_credentials}
    end.
```

---

## 3. JWT Tokens

```erlang
%% jwt_auth.erl — JWT สร้างและตรวจสอบ
-module(jwt_auth).
-export([create_token/2, verify_token/1, refresh_token/1]).

-define(SECRET, <<"my_very_secret_key_change_in_production">>).
-define(ACCESS_TTL,  900).    %% 15 minutes
-define(REFRESH_TTL, 604800). %% 7 days

%% สร้าง JWT token
create_token(UserId, Extra) ->
    Now = erlang:system_time(second),
    Claims = #{
        sub => UserId,
        iat => Now,
        exp => Now + ?ACCESS_TTL,
        jti => make_jti()
    },
    AllClaims = maps:merge(Extra, Claims),
    sign_jwt(AllClaims).

%% ตรวจสอบ JWT
verify_token(Token) ->
    case decode_jwt(Token) of
        {ok, Claims} ->
            Now = erlang:system_time(second),
            case maps:get(exp, Claims, 0) of
                Exp when Exp > Now ->
                    {ok, Claims};
                _ ->
                    {error, token_expired}
            end;
        Error ->
            Error
    end.

%% Refresh token
refresh_token(RefreshToken) ->
    case verify_refresh_token(RefreshToken) of
        {ok, #{sub := UserId}} ->
            {ok, AccessToken}  = create_token(UserId, #{}),
            {ok, RefreshToken2} = create_refresh_token(UserId),
            {ok, #{
                access_token  => AccessToken,
                refresh_token => RefreshToken2
            }};
        Error ->
            Error
    end.

%% JWT เป็น base64url(header).base64url(payload).base64url(signature)
sign_jwt(Claims) ->
    Header  = jsx:encode(#{alg => <<"HS256">>, typ => <<"JWT">>}),
    Payload = jsx:encode(Claims),
    H = base64url_encode(Header),
    P = base64url_encode(Payload),
    Msg = <<H/binary, ".", P/binary>>,
    Sig = base64url_encode(crypto:mac(hmac, sha256, ?SECRET, Msg)),
    <<Msg/binary, ".", Sig/binary>>.

decode_jwt(Token) ->
    case binary:split(Token, <<".">>, [global]) of
        [H, P, S] ->
            Msg = <<H/binary, ".", P/binary>>,
            ExpSig = base64url_encode(
                crypto:mac(hmac, sha256, ?SECRET, Msg)),
            case crypto:hash_equals(S, ExpSig) of
                true ->
                    Claims = jsx:decode(base64url_decode(P), [return_maps]),
                    {ok, Claims};
                false ->
                    {error, invalid_signature}
            end;
        _ ->
            {error, invalid_token_format}
    end.

base64url_encode(Data) ->
    Base = base64:encode(Data),
    %% Convert standard base64 to url-safe
    B1 = binary:replace(Base, <<"+">>, <<"-">>, [global]),
    B2 = binary:replace(B1,  <<"/">>, <<"_">>, [global]),
    binary:replace(B2, <<"=">>, <<>>, [global]).

base64url_decode(Data) ->
    Padded = pad_base64(Data),
    B1 = binary:replace(Padded, <<"-">>, <<"+">>, [global]),
    B2 = binary:replace(B1,    <<"_">>, <<"/">>, [global]),
    base64:decode(B2).

pad_base64(Data) ->
    case byte_size(Data) rem 4 of
        0 -> Data;
        2 -> <<Data/binary, "==">>;
        3 -> <<Data/binary, "=">>;
        _ -> Data
    end.

make_jti() -> base64url_encode(crypto:strong_rand_bytes(16)).

create_refresh_token(UserId) ->
    Now = erlang:system_time(second),
    Claims = #{sub => UserId, iat => Now, exp => Now + ?REFRESH_TTL,
               type => <<"refresh">>},
    {ok, sign_jwt(Claims)}.

verify_refresh_token(Token) ->
    case decode_jwt(Token) of
        {ok, #{type := <<"refresh">>} = Claims} ->
            Now = erlang:system_time(second),
            case maps:get(exp, Claims, 0) > Now of
                true  -> {ok, Claims};
                false -> {error, token_expired}
            end;
        _ ->
            {error, invalid_refresh_token}
    end.
```

---

## 4. Session-Based Auth

```erlang
%% session_store.erl — Server-side sessions
-module(session_store).
-behaviour(gen_server).

-export([start_link/0, create/1, get/1, delete/1, refresh/1]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

-define(SESSION_TTL, 1800).  %% 30 minutes

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

create(Data) ->
    SessionId = make_session_id(),
    gen_server:call(?MODULE, {create, SessionId, Data}),
    {ok, SessionId}.

get(SessionId) ->
    gen_server:call(?MODULE, {get, SessionId}).

delete(SessionId) ->
    gen_server:cast(?MODULE, {delete, SessionId}).

refresh(SessionId) ->
    gen_server:call(?MODULE, {refresh, SessionId}).

init([]) ->
    ets:new(sessions, [named_table, set, public,
                       {read_concurrency, true}]),
    erlang:send_after(60000, self(), cleanup),
    {ok, #{}}.

handle_call({create, Id, Data}, _From, S) ->
    Exp = erlang:system_time(second) + ?SESSION_TTL,
    ets:insert(sessions, {Id, Data, Exp}),
    {reply, ok, S};

handle_call({get, Id}, _From, S) ->
    Now = erlang:system_time(second),
    case ets:lookup(sessions, Id) of
        [{Id, Data, Exp}] when Exp > Now ->
            {reply, {ok, Data}, S};
        [{Id, _, _}] ->
            ets:delete(sessions, Id),
            {reply, {error, session_expired}, S};
        [] ->
            {reply, {error, not_found}, S}
    end;

handle_call({refresh, Id}, _From, S) ->
    Now = erlang:system_time(second),
    case ets:lookup(sessions, Id) of
        [{Id, Data, Exp}] when Exp > Now ->
            NewExp = Now + ?SESSION_TTL,
            ets:insert(sessions, {Id, Data, NewExp}),
            {reply, ok, S};
        _ ->
            {reply, {error, not_found}, S}
    end;

handle_cast({delete, Id}, S) ->
    ets:delete(sessions, Id),
    {noreply, S};

handle_info(cleanup, S) ->
    Now = erlang:system_time(second),
    ets:select_delete(sessions,
        [{{'_','_','$1'}, [{'<','$1',Now}], [true]}]),
    erlang:send_after(60000, self(), cleanup),
    {noreply, S};
handle_info(_, S) -> {noreply, S}.

make_session_id() ->
    base64:encode(crypto:strong_rand_bytes(24)).
```

---

## 5. OAuth2 Flow

```erlang
%% oauth2.erl — OAuth2 authorization code flow
-module(oauth2).
-export([authorization_url/2, exchange_code/2, get_user_info/1]).

%% Provider config
github_config() ->
    #{
        client_id     => <<"YOUR_CLIENT_ID">>,
        client_secret => <<"YOUR_CLIENT_SECRET">>,
        auth_url      => <<"https://github.com/login/oauth/authorize">>,
        token_url     => <<"https://github.com/login/oauth/access_token">>,
        user_url      => <<"https://api.github.com/user">>,
        scope         => <<"read:user,user:email">>
    }.

%% Step 1: Redirect user to provider
authorization_url(Provider, State) ->
    Config = get_config(Provider),
    Params = uri_string:compose_query([
        {<<"client_id">>,    maps:get(client_id, Config)},
        {<<"state">>,        State},
        {<<"scope">>,        maps:get(scope, Config)},
        {<<"redirect_uri">>, <<"http://localhost:8080/auth/callback">>}
    ]),
    AuthUrl = maps:get(auth_url, Config),
    <<AuthUrl/binary, "?", Params/binary>>.

%% Step 2: Exchange code for token
exchange_code(Provider, Code) ->
    Config = get_config(Provider),
    Body = uri_string:compose_query([
        {<<"client_id">>,     maps:get(client_id, Config)},
        {<<"client_secret">>, maps:get(client_secret, Config)},
        {<<"code">>,          Code},
        {<<"redirect_uri">>,  <<"http://localhost:8080/auth/callback">>}
    ]),
    Headers = [
        {<<"content-type">>,  <<"application/x-www-form-urlencoded">>},
        {<<"accept">>,        <<"application/json">>}
    ],
    TokenUrl = maps:get(token_url, Config),
    case hackney:request(post, TokenUrl, Headers, Body, [with_body]) of
        {ok, 200, _, RespBody} ->
            {ok, jsx:decode(RespBody, [return_maps])};
        {ok, Status, _, RespBody} ->
            {error, {Status, RespBody}};
        Error -> Error
    end.

%% Step 3: Get user info
get_user_info(Provider, AccessToken) ->
    Config = get_config(Provider),
    Headers = [{<<"authorization">>, <<"Bearer ", AccessToken/binary>>}],
    UserUrl = maps:get(user_url, Config),
    case hackney:request(get, UserUrl, Headers, <<>>, [with_body]) of
        {ok, 200, _, Body} ->
            {ok, jsx:decode(Body, [return_maps])};
        Error -> Error
    end.

get_config(github) -> github_config().
```

---

## 6. Role-Based Access Control (RBAC)

```erlang
%% rbac.erl — Role-Based Access Control
-module(rbac).
-export([has_permission/3, check/3, roles_for/1]).

%% Define permissions per role
-define(ROLE_PERMISSIONS, #{
    admin => [
        read_users, write_users, delete_users,
        read_posts, write_posts, delete_posts,
        manage_settings
    ],
    moderator => [
        read_users,
        read_posts, write_posts, delete_posts
    ],
    user => [
        read_posts, write_posts
    ],
    guest => [
        read_posts
    ]
}).

has_permission(Role, Permission, _Resource) ->
    Perms = maps:get(Role, ?ROLE_PERMISSIONS, []),
    lists:member(Permission, Perms).

%% check/3: return ok | {error, forbidden}
check(User, Permission, Resource) ->
    Roles = roles_for(User),
    Allowed = lists:any(fun(Role) ->
        has_permission(Role, Permission, Resource)
    end, Roles),
    if
        Allowed -> ok;
        true    -> {error, forbidden}
    end.

%% Get user roles (from DB, token claims, etc)
roles_for(#{roles := Roles}) when is_list(Roles) ->
    [binary_to_atom(R, utf8) || R <- Roles];
roles_for(#{role := Role}) ->
    [binary_to_atom(Role, utf8)];
roles_for(_) ->
    [guest].

%% Resource-based check: can user access specific resource?
can_access(User, post, PostId) ->
    case check(User, read_posts, PostId) of
        ok -> ok;
        _ ->
            %% Check if user is the owner
            UserId = maps:get(id, User),
            case post_repo:find(PostId) of
                {ok, #{author_id := UserId}} -> ok;
                _ -> {error, forbidden}
            end
    end.
```

---

## 7. API Key Authentication

```erlang
%% api_key_auth.erl
-module(api_key_auth).
-export([create_key/1, verify_key/1, revoke_key/1]).

create_key(UserId) ->
    Key = generate_key(),
    %% Store hashed key in DB (never store plaintext)
    Hash = hash_key(Key),
    db:insert(api_keys, #{
        key_hash => Hash,
        user_id  => UserId,
        created  => erlang:system_time(second),
        last_used => null,
        active   => true
    }),
    %% Return plaintext key to user ONCE
    {ok, Key}.

verify_key(Key) ->
    Hash = hash_key(Key),
    case db:find_by(api_keys, key_hash, Hash) of
        {ok, #{user_id:=UserId, active:=true} = KeyRecord} ->
            %% Update last_used
            db:update(api_keys, maps:get(id, KeyRecord),
                      #{last_used => erlang:system_time(second)}),
            {ok, UserId};
        {ok, #{active := false}} ->
            {error, key_revoked};
        {error, not_found} ->
            {error, invalid_key}
    end.

revoke_key(KeyId) ->
    db:update(api_keys, KeyId, #{active => false}).

generate_key() ->
    %% Format: prefix_randombytes
    Rand = binary:encode_hex(crypto:strong_rand_bytes(24)),
    <<"sk_", Rand/binary>>.

hash_key(Key) ->
    crypto:hash(sha256, Key).
```

---

## 8. Cowboy Auth Middleware

```erlang
%% auth_middleware.erl
-module(auth_middleware).
-behaviour(cowboy_middleware).
-export([execute/2]).

%% Public paths ที่ไม่ต้อง auth
-define(PUBLIC_PATHS, [
    <<"/api/auth/login">>,
    <<"/api/auth/register">>,
    <<"/api/auth/refresh">>,
    <<"/health">>
]).

execute(Req, Env) ->
    Path = cowboy_req:path(Req),
    case lists:member(Path, ?PUBLIC_PATHS) of
        true ->
            {ok, Req, Env};
        false ->
            case extract_token(Req) of
                {ok, Token} ->
                    case jwt_auth:verify_token(Token) of
                        {ok, Claims} ->
                            Req2 = cowboy_req:set_resp_header(
                                <<"x-user-id">>,
                                integer_to_binary(maps:get(sub, Claims)),
                                Req
                            ),
                            %% Store claims in request for handlers
                            Env2 = Env#{user_claims => Claims},
                            {ok, Req2, Env2};
                        {error, token_expired} ->
                            reply_401(<<"Token expired">>, Req);
                        {error, _} ->
                            reply_401(<<"Invalid token">>, Req)
                    end;
                error ->
                    reply_401(<<"Missing authorization">>, Req)
            end
    end.

extract_token(Req) ->
    %% Support: Bearer token in Authorization header
    case cowboy_req:header(<<"authorization">>, Req) of
        <<"Bearer ", Token/binary>> -> {ok, Token};
        _ ->
            %% Fallback: token in query string (for WebSocket)
            Qs = cowboy_req:parse_qs(Req),
            case lists:keyfind(<<"token">>, 1, Qs) of
                {_, Token} -> {ok, Token};
                false      -> error
            end
    end.

reply_401(Message, Req) ->
    Req2 = cowboy_req:reply(401,
        #{<<"content-type">> => <<"application/json">>,
          <<"www-authenticate">> => <<"Bearer">>},
        jsx:encode(#{error => Message}),
        Req
    ),
    {stop, Req2}.
```

---

## 9. แบบฝึกหัด

### Exercise: Complete Auth System

```erlang
%% ใช้ร่วมกัน: สร้าง complete auth system
%% 1. Register: hash password → save user
%% 2. Login: verify password → create JWT + refresh token
%% 3. Protected routes: verify JWT
%% 4. Refresh: exchange refresh token for new access token

%% auth_handler.erl
-module(auth_handler).
-export([init/2]).

init(Req, State) ->
    Method = cowboy_req:method(Req),
    Path   = cowboy_req:path(Req),
    handle(Method, Path, Req, State).

handle(<<"POST">>, <<"/api/auth/register">>, Req, State) ->
    {ok, Body, Req2} = cowboy_req:read_body(Req),
    #{<<"username">> := U, <<"password">> := P} = jsx:decode(Body, [return_maps]),
    case user_repo:register(U, P) of
        {ok, User} ->
            reply_json(201, #{user => User}, Req2, State);
        {error, already_exists} ->
            reply_json(409, #{error => <<"Username taken">>}, Req2, State)
    end;

handle(<<"POST">>, <<"/api/auth/login">>, Req, State) ->
    {ok, Body, Req2} = cowboy_req:read_body(Req),
    #{<<"username">> := U, <<"password">> := P} = jsx:decode(Body, [return_maps]),
    case user_repo:authenticate(U, P) of
        {ok, #{id := UserId} = User} ->
            {ok, Access}  = jwt_auth:create_token(UserId, #{}),
            {ok, Refresh} = jwt_auth:create_refresh_token(UserId),
            reply_json(200, #{
                access_token  => Access,
                refresh_token => Refresh,
                user => maps:remove(password_hash, User)
            }, Req2, State);
        {error, invalid_credentials} ->
            reply_json(401, #{error => <<"Invalid credentials">>}, Req2, State)
    end;

handle(<<"POST">>, <<"/api/auth/refresh">>, Req, State) ->
    {ok, Body, Req2} = cowboy_req:read_body(Req),
    #{<<"refresh_token">> := Token} = jsx:decode(Body, [return_maps]),
    case jwt_auth:refresh_token(Token) of
        {ok, Tokens}  -> reply_json(200, Tokens, Req2, State);
        {error, Err}  -> reply_json(401, #{error => Err}, Req2, State)
    end.

reply_json(Status, Data, Req, State) ->
    Req2 = cowboy_req:reply(Status,
        #{<<"content-type">> => <<"application/json">>},
        jsx:encode(Data), Req),
    {ok, Req2, State}.
```

---

## สรุป Part 30

✅ Authentication vs Authorization  
✅ Password hashing ด้วย bcrypt/PBKDF2  
✅ JWT tokens: sign, verify, refresh  
✅ Session-based authentication  
✅ OAuth2 authorization code flow  
✅ Role-Based Access Control (RBAC)  
✅ API Key authentication  
✅ Cowboy auth middleware

---

*Part 30/100 | [← ก่อนหน้า](../part29/README.md) | [ถัดไป →](../part31/README.md)*
