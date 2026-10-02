# Part 81: Security Engineering

> **"Security is not a feature, it's a property of the system"**  
> ความปลอดภัยไม่ใช่ feature — มันคือคุณสมบัติของระบบทั้งหมด

---

## สารบัญ

1. [Security Fundamentals](#1-security-fundamentals)
2. [JWT Authentication](#2-jwt-authentication)
3. [OAuth2 / OIDC Integration](#3-oauth2--oidc-integration)
4. [Role-Based Access Control (RBAC)](#4-role-based-access-control-rbac)
5. [Input Validation and Sanitization](#5-input-validation-and-sanitization)
6. [Security Headers and CORS](#6-security-headers-and-cors)
7. [แบบฝึกหัด](#7-แบบฝึกหัด)

---

## 1. Security Fundamentals

```
The OWASP Top 10 — Erlang Edition
════════════════════════════════════════════════════════

A01 Broken Access Control
  Fix: RBAC middleware, resource ownership checks

A02 Cryptographic Failures
  Fix: Use crypto module correctly, never roll your own

A03 Injection (SQL, Command)
  Fix: Parameterized queries, never interpolate user input

A04 Insecure Design
  Fix: Threat modeling, principle of least privilege

A05 Security Misconfiguration
  Fix: Production config checklist, no debug endpoints in prod

A06 Vulnerable Components
  Fix: rebar3 audit, keep deps updated

A07 Authentication Failures
  Fix: JWT validation, session management

A08 Software and Data Integrity
  Fix: Signed releases, verify dependencies

A09 Security Logging Failures
  Fix: Immutable audit log, log all auth events

A10 Server-Side Request Forgery (SSRF)
  Fix: Whitelist allowed outbound URLs, validate URLs

ERLANG-SPECIFIC RISKS:
  • atom() exhaustion via binary_to_atom/1 with user input
  • RPC exposure: never expose erlang:node() to users
  • Distribution port: erlang cookie must not be guessable
  • Port overflow: validate external program output sizes
```

---

## 2. JWT Authentication

```erlang
%% jwt_auth.erl — JWT creation and validation
-module(jwt_auth).
-export([create_token/2, verify_token/1, refresh_token/1]).

-define(ALGORITHM, <<"HS256">>).
-define(TOKEN_TTL,  3600).       % 1 hour
-define(REFRESH_TTL, 604800).    % 7 days

create_token(UserId, Claims) ->
    Secret = get_secret(),
    Now    = erlang:system_time(second),
    Payload = maps:merge(Claims, #{
        sub => UserId,
        iat => Now,
        exp => Now + ?TOKEN_TTL,
        jti => generate_jti()
    }),
    sign_token(Payload, Secret).

verify_token(Token) ->
    Secret = get_secret(),
    case decode_and_verify(Token, Secret) of
        {ok, Payload} ->
            Now = erlang:system_time(second),
            case maps:get(<<"exp">>, Payload, 0) > Now of
                true  ->
                    %% Check if token is revoked
                    Jti = maps:get(<<"jti">>, Payload),
                    case token_blacklist:is_revoked(Jti) of
                        false -> {ok, Payload};
                        true  -> {error, token_revoked}
                    end;
                false ->
                    {error, token_expired}
            end;
        {error, _} = Err -> Err
    end.

refresh_token(RefreshToken) ->
    case verify_token(RefreshToken) of
        {ok, #{<<"sub">> := UserId, <<"type">> := <<"refresh">>} = Payload} ->
            %% Revoke old refresh token
            token_blacklist:revoke(maps:get(<<"jti">>, Payload)),
            %% Issue new access + refresh tokens
            AccessToken  = create_token(UserId, #{type => access}),
            NewRefresh   = sign_token(
                #{sub => UserId, type => refresh,
                  iat => erlang:system_time(second),
                  exp => erlang:system_time(second) + ?REFRESH_TTL,
                  jti => generate_jti()},
                get_secret()),
            {ok, #{access_token => AccessToken,
                   refresh_token => NewRefresh}};
        {ok, _} ->
            {error, not_a_refresh_token};
        Err -> Err
    end.

sign_token(Payload, Secret) ->
    Header  = base64url(json:encode(#{alg => ?ALGORITHM, typ => <<"JWT">>})),
    Body    = base64url(json:encode(Payload)),
    Signing = <<Header/binary, ".", Body/binary>>,
    Sig     = base64url(crypto:mac(hmac, sha256, Secret, Signing)),
    <<Signing/binary, ".", Sig/binary>>.

decode_and_verify(Token, Secret) ->
    case binary:split(Token, <<".">>, [global]) of
        [HeaderB64, BodyB64, SigB64] ->
            Signing  = <<HeaderB64/binary, ".", BodyB64/binary>>,
            Expected = base64url(crypto:mac(hmac, sha256, Secret, Signing)),
            case crypto:hash_equals(Expected, SigB64) of
                true ->
                    Payload = json:decode(base64url_decode(BodyB64)),
                    {ok, Payload};
                false ->
                    {error, invalid_signature}
            end;
        _ ->
            {error, malformed_token}
    end.

base64url(Data) ->
    B64 = base64:encode(Data),
    %% Convert to URL-safe format
    << <<case C of $+ -> $-; $/ -> $_; $= -> <<>>; _ -> C end>>
       || <<C>> <= B64 >>.

base64url_decode(Data) ->
    Padded = pad_base64(Data),
    Standard = << <<case C of $- -> $+; $_ -> $/; _ -> C end>>
                   || <<C>> <= Padded >>,
    base64:decode(Standard).

pad_base64(B) ->
    Rem = byte_size(B) rem 4,
    case Rem of
        0 -> B;
        2 -> <<B/binary, "==">>;
        3 -> <<B/binary, "=">>
    end.

get_secret() ->
    case application:get_env(myapp, jwt_secret) of
        {ok, Secret} when byte_size(Secret) >= 32 -> Secret;
        _ -> error(jwt_secret_not_configured_or_too_short)
    end.

generate_jti() ->
    base64:encode(crypto:strong_rand_bytes(16)).
```

---

## 3. OAuth2 / OIDC Integration

```erlang
%% oauth2_client.erl — OAuth2 authorization code flow
-module(oauth2_client).
-export([build_auth_url/2, exchange_code/2, get_userinfo/1]).

-define(GOOGLE_AUTH_URL, <<"https://accounts.google.com/o/oauth2/v2/auth">>).
-define(GOOGLE_TOKEN_URL, <<"https://oauth2.googleapis.com/token">>).
-define(GOOGLE_USERINFO_URL, <<"https://www.googleapis.com/oauth2/v3/userinfo">>).

build_auth_url(State, Nonce) ->
    Params = uri_string:compose_query([
        {"client_id",     get_client_id()},
        {"redirect_uri",  get_redirect_uri()},
        {"response_type", "code"},
        {"scope",         "openid email profile"},
        {"state",         State},
        {"nonce",         Nonce}
    ]),
    <<?GOOGLE_AUTH_URL/binary, "?", Params/binary>>.

exchange_code(Code, State) ->
    %% Verify state to prevent CSRF
    case session_store:verify_oauth_state(State) of
        false -> {error, invalid_state};
        true  ->
            Body = uri_string:compose_query([
                {"code",          binary_to_list(Code)},
                {"client_id",     get_client_id()},
                {"client_secret", get_client_secret()},
                {"redirect_uri",  get_redirect_uri()},
                {"grant_type",    "authorization_code"}
            ]),
            case httpc:request(post,
                               {?GOOGLE_TOKEN_URL, [],
                                "application/x-www-form-urlencoded",
                                Body},
                               [], []) of
                {ok, {{_, 200, _}, _, RespBody}} ->
                    TokenData = json:decode(list_to_binary(RespBody)),
                    {ok, TokenData};
                {ok, {{_, Code, _}, _, ErrBody}} ->
                    {error, {http_error, Code, ErrBody}};
                {error, _} = Err -> Err
            end
    end.

get_userinfo(AccessToken) ->
    Headers = [{"Authorization",
                "Bearer " ++ binary_to_list(AccessToken)}],
    case httpc:request(get, {?GOOGLE_USERINFO_URL, Headers}, [], []) of
        {ok, {{_, 200, _}, _, Body}} ->
            {ok, json:decode(list_to_binary(Body))};
        {ok, {{_, Code, _}, _, _}} ->
            {error, {http_error, Code}};
        {error, _} = Err -> Err
    end.

get_client_id()     -> application:get_env(myapp, google_client_id,     <<>>).
get_client_secret() -> application:get_env(myapp, google_client_secret, <<>>).
get_redirect_uri()  -> application:get_env(myapp, google_redirect_uri,  <<>>).
```

---

## 4. Role-Based Access Control (RBAC)

```erlang
%% rbac.erl — role-based access control
-module(rbac).
-export([check/3, has_permission/3, get_permissions/1]).

%% Permission: {resource, action}
%% e.g., {posts, read}, {users, delete}, {admin, all}

-define(ROLE_PERMISSIONS, #{
    admin => [
        {users, all}, {posts, all}, {comments, all},
        {settings, all}, {reports, read}
    ],
    moderator => [
        {posts, read}, {posts, delete},
        {comments, read}, {comments, delete},
        {users, read}
    ],
    user => [
        {posts, read}, {posts, create},
        {own_posts, all},
        {comments, read}, {comments, create},
        {own_comments, all},
        {profile, read}
    ],
    guest => [
        {posts, read},
        {comments, read}
    ]
}).

%% Check if user has permission for resource+action
check(UserId, Resource, Action) ->
    case get_user_roles(UserId) of
        {ok, Roles} ->
            has_permission_in_roles(Roles, Resource, Action);
        {error, _} ->
            false
    end.

has_permission(Roles, Resource, Action) when is_list(Roles) ->
    has_permission_in_roles(Roles, Resource, Action).

has_permission_in_roles([], _Resource, _Action) -> false;
has_permission_in_roles([Role | Rest], Resource, Action) ->
    Perms = maps:get(Role, ?ROLE_PERMISSIONS, []),
    case permission_matches(Perms, Resource, Action) of
        true  -> true;
        false -> has_permission_in_roles(Rest, Resource, Action)
    end.

permission_matches([], _Resource, _Action) -> false;
permission_matches([{Resource, all} | _], Resource, _Action) -> true;
permission_matches([{all, all} | _], _Resource, _Action)     -> true;
permission_matches([{Resource, Action} | _], Resource, Action) -> true;
permission_matches([_ | Rest], Resource, Action) ->
    permission_matches(Rest, Resource, Action).

get_permissions(Role) ->
    maps:get(Role, ?ROLE_PERMISSIONS, []).

get_user_roles(UserId) ->
    %% Cache roles in ETS
    case ets:lookup(user_roles_cache, UserId) of
        [{_, Roles, Exp}] when Exp > erlang:system_time(second) ->
            {ok, Roles};
        _ ->
            case db:query("SELECT role FROM user_roles WHERE user_id = $1",
                          [UserId]) of
                {ok, Rows} ->
                    Roles = [binary_to_atom(R) || {R} <- Rows],
                    ets:insert(user_roles_cache,
                               {UserId, Roles,
                                erlang:system_time(second) + 300}),
                    {ok, Roles};
                {error, _} = Err -> Err
            end
    end.
```

```erlang
%% auth_middleware.erl — Cowboy middleware combining JWT + RBAC
-module(auth_middleware).
-behaviour(cowboy_middleware).
-export([execute/2]).

%% Paths that don't require authentication
-define(PUBLIC_PATHS, [
    <<"/health">>,
    <<"/ready">>,
    <<"/auth/login">>,
    <<"/auth/register">>,
    <<"/auth/callback">>
]).

execute(Req, Env) ->
    Path = cowboy_req:path(Req),
    case lists:member(Path, ?PUBLIC_PATHS) of
        true ->
            {ok, Req, Env};
        false ->
            case extract_and_verify_token(Req) of
                {ok, Claims} ->
                    %% Add claims to request meta for handlers
                    Req1 = cowboy_req:set_meta(auth_claims, Claims, Req),
                    Req2 = cowboy_req:set_meta(
                        user_id, maps:get(<<"sub">>, Claims), Req1),
                    {ok, Req2, Env};
                {error, Reason} ->
                    Response = cowboy_req:reply(401,
                        #{<<"content-type">> => <<"application/json">>},
                        json:encode(#{error => unauthorized, reason => Reason}),
                        Req),
                    {stop, Response}
            end
    end.

extract_and_verify_token(Req) ->
    case cowboy_req:header(<<"authorization">>, Req) of
        <<"Bearer ", Token/binary>> ->
            jwt_auth:verify_token(Token);
        undefined ->
            {error, missing_token};
        _ ->
            {error, invalid_auth_header}
    end.
```

---

## 5. Input Validation and Sanitization

```erlang
%% validator.erl — input validation library
-module(validator).
-export([validate/2, sanitize_html/1, safe_atom/1]).

%% Validate a map against a schema
validate(Input, Schema) ->
    Errors = maps:fold(fun(Field, Rules, Acc) ->
        Value = maps:get(Field, Input, undefined),
        FieldErrors = validate_field(Field, Value, Rules),
        Acc ++ FieldErrors
    end, [], Schema),
    case Errors of
        [] -> {ok, coerce_types(Input, Schema)};
        _  -> {error, Errors}
    end.

validate_field(Field, undefined, Rules) ->
    case lists:member(required, Rules) of
        true  -> [{Field, required}];
        false -> []
    end;
validate_field(Field, Value, Rules) ->
    lists:foldl(fun(Rule, Errs) ->
        case check_rule(Value, Rule) of
            ok            -> Errs;
            {error, Msg}  -> [{Field, Msg} | Errs]
        end
    end, [], Rules).

check_rule(V, required) when V =/= undefined -> ok;
check_rule(undefined, required) -> {error, is_required};

check_rule(V, {min_length, N}) when is_binary(V), byte_size(V) >= N -> ok;
check_rule(_, {min_length, N}) -> {error, {too_short, N}};

check_rule(V, {max_length, N}) when is_binary(V), byte_size(V) =< N -> ok;
check_rule(_, {max_length, N}) -> {error, {too_long, N}};

check_rule(V, email) ->
    case re:run(V, <<"^[^@]+@[^@]+\\.[^@]+$">>) of
        {match, _} -> ok;
        nomatch    -> {error, invalid_email}
    end;

check_rule(V, {regex, Pattern}) ->
    case re:run(V, Pattern) of
        {match, _} -> ok;
        nomatch    -> {error, {invalid_format, Pattern}}
    end;

check_rule(V, {min, N}) when is_number(V), V >= N -> ok;
check_rule(_, {min, N}) -> {error, {below_minimum, N}};

check_rule(V, {max, N}) when is_number(V), V =< N -> ok;
check_rule(_, {max, N}) -> {error, {above_maximum, N}};

check_rule(V, {one_of, Options}) ->
    case lists:member(V, Options) of
        true  -> ok;
        false -> {error, {not_in_options, Options}}
    end;

check_rule(_, _) -> ok.

%% Basic HTML sanitization (strip tags for display)
sanitize_html(Input) when is_binary(Input) ->
    %% Remove HTML tags
    re:replace(Input, <<"<[^>]*>">>, <<"">>, [global, {return, binary}]).

%% Safe atom creation — only allows known atoms
safe_atom(Binary) when is_binary(Binary) ->
    try binary_to_existing_atom(Binary, utf8)
    catch error:badarg -> {error, unknown_atom}
    end.

coerce_types(Input, _Schema) -> Input.  % simplified
```

---

## 6. Security Headers and CORS

```erlang
%% security_middleware.erl — add security headers to all responses
-module(security_middleware).
-behaviour(cowboy_middleware).
-export([execute/2]).

execute(Req, Env) ->
    %% Add security headers to ALL responses
    Req1 = add_security_headers(Req),
    {ok, Req1, Env}.

add_security_headers(Req) ->
    Headers = #{
        %% Prevent MIME type sniffing
        <<"x-content-type-options">> => <<"nosniff">>,
        %% Prevent clickjacking
        <<"x-frame-options">> => <<"DENY">>,
        %% XSS protection (legacy browsers)
        <<"x-xss-protection">> => <<"1; mode=block">>,
        %% HSTS: force HTTPS for 1 year
        <<"strict-transport-security">> =>
            <<"max-age=31536000; includeSubDomains">>,
        %% Content Security Policy
        <<"content-security-policy">> =>
            <<"default-src 'self'; "
              "script-src 'self' 'nonce-{nonce}'; "
              "style-src 'self' 'unsafe-inline'; "
              "img-src 'self' data: https:; "
              "connect-src 'self'">>,
        %% Referrer policy
        <<"referrer-policy">> => <<"strict-origin-when-cross-origin">>,
        %% Permissions policy
        <<"permissions-policy">> =>
            <<"camera=(), microphone=(), geolocation=()">>
    },
    maps:fold(fun(Name, Value, R) ->
        cowboy_req:set_resp_header(Name, Value, R)
    end, Req, Headers).
```

```erlang
%% cors_middleware.erl — CORS handling
-module(cors_middleware).
-behaviour(cowboy_middleware).
-export([execute/2]).

-define(ALLOWED_ORIGINS, [
    <<"https://app.mycompany.com">>,
    <<"https://admin.mycompany.com">>
]).

execute(Req, Env) ->
    Origin = cowboy_req:header(<<"origin">>, Req, undefined),
    Method = cowboy_req:method(Req),

    case {Origin, Method} of
        {undefined, _} ->
            %% No origin header — not a CORS request
            {ok, Req, Env};
        {_, <<"OPTIONS">>} ->
            %% Preflight request
            Req1 = handle_preflight(Origin, Req),
            {stop, Req1};
        _ ->
            %% Simple or actual CORS request
            Req1 = add_cors_headers(Origin, Req),
            {ok, Req1, Env}
    end.

handle_preflight(Origin, Req) ->
    Req1 = add_cors_headers(Origin, Req),
    cowboy_req:reply(204,
        #{<<"access-control-allow-methods">>  =>
              <<"GET, POST, PUT, PATCH, DELETE, OPTIONS">>,
          <<"access-control-allow-headers">>  =>
              <<"content-type, authorization, x-request-id">>,
          <<"access-control-max-age">> => <<"86400">>},
        <<>>, Req1).

add_cors_headers(Origin, Req) ->
    case lists:member(Origin, ?ALLOWED_ORIGINS) of
        true ->
            cowboy_req:set_resp_headers(#{
                <<"access-control-allow-origin">>      => Origin,
                <<"access-control-allow-credentials">> => <<"true">>,
                <<"vary">> => <<"Origin">>
            }, Req);
        false ->
            Req
    end.
```

---

## 7. แบบฝึกหัด

1. เพิ่ม token blacklist ที่ใช้ Redis แทน ETS เพื่อรองรับหลาย nodes
2. Implement PKCE (Proof Key for Code Exchange) สำหรับ OAuth2 mobile apps
3. สร้าง audit log ที่บันทึกทุก authentication event พร้อม IP และ user agent
4. Implement permission inheritance: admin inherits all moderator permissions

---

## สรุป Part 81

✅ OWASP Top 10 mitigation strategies for Erlang  
✅ JWT: signing with HMAC-SHA256, expiry, revocation blacklist  
✅ OAuth2/OIDC: authorization code flow, CSRF state verification  
✅ RBAC: role → permissions mapping, middleware enforcement  
✅ Input validation: rule-based schema validation  
✅ Security headers: CSP, HSTS, X-Frame-Options, CORS  

---

*Part 81/100 | [← ก่อนหน้า](../part80/README.md) | [ถัดไป →](../part82/README.md)*
