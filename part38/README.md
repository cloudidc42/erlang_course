# Part 38: Security Best Practices

> **"Security is a process, not a product"**  
> Security คือกระบวนการ ไม่ใช่ผลิตภัณฑ์

---

## สารบัญ

1. [OWASP Top 10 ใน Erlang](#1-owasp-top-10-ใน-erlang)
2. [Input Validation](#2-input-validation)
3. [SQL Injection Prevention](#3-sql-injection-prevention)
4. [XSS Prevention](#4-xss-prevention)
5. [Secrets Management](#5-secrets-management)
6. [TLS/SSL Configuration](#6-tlsssl-configuration)
7. [Secure Headers](#7-secure-headers)
8. [Audit Logging](#8-audit-logging)
9. [แบบฝึกหัด](#9-แบบฝึกหัด)

---

## 1. OWASP Top 10 ใน Erlang

```
OWASP Top 10 และวิธีแก้ใน Erlang:

A01 Broken Access Control:
  ✓ RBAC middleware บน every route
  ✓ Check ownership ก่อน return data

A02 Cryptographic Failures:
  ✓ ใช้ crypto module ของ OTP
  ✓ AES-256-GCM สำหรับ encryption
  ✓ bcrypt/argon2 สำหรับ passwords

A03 Injection:
  ✓ Parameterized queries ด้วย epgsql
  ✓ Input validation แบบ whitelist
  ✓ ไม่ใช้ string interpolation ใน queries

A05 Security Misconfiguration:
  ✓ vm.args ที่ปลอดภัย (ไม่เปิด cookie=)
  ✓ Firewall ports ที่ไม่จำเป็น
  ✓ TLS สำหรับทุก connection

A07 Identification and Authentication Failures:
  ✓ JWT short-lived tokens
  ✓ Rate limiting on login endpoint
  ✓ bcrypt password hashing

A09 Security Logging and Monitoring Failures:
  ✓ Audit log ทุก sensitive action
  ✓ Alert on suspicious patterns
```

---

## 2. Input Validation

```erlang
%% validator.erl — Input validation module
-module(validator).
-export([validate/2, required/2, min_length/3, max_length/3,
         email/2, integer_range/4, enum/3]).

validate(Data, Rules) ->
    Errors = lists:filtermap(fun({Field, Rule}) ->
        Value = maps:get(Field, Data, undefined),
        case apply_rule(Field, Value, Rule) of
            ok          -> false;
            {error, Msg} -> {true, {Field, Msg}}
        end
    end, Rules),
    case Errors of
        []     -> {ok, Data};
        Errors -> {error, maps:from_list(Errors)}
    end.

%% Rules
required(Field, Value) when Value =:= undefined; Value =:= <<>> ->
    {error, iolist_to_binary([atom_to_binary(Field), " is required"])};
required(_, _) -> ok.

min_length(Field, Value, Min) when is_binary(Value) ->
    case byte_size(Value) >= Min of
        true  -> ok;
        false -> {error, iolist_to_binary([atom_to_binary(Field),
                           " must be at least ", integer_to_binary(Min), " characters"])}
    end;
min_length(_, _, _) -> ok.

max_length(Field, Value, Max) when is_binary(Value) ->
    case byte_size(Value) =< Max of
        true  -> ok;
        false -> {error, iolist_to_binary([atom_to_binary(Field),
                           " must be at most ", integer_to_binary(Max), " characters"])}
    end;
max_length(_, _, _) -> ok.

email(Field, Value) when is_binary(Value) ->
    case re:run(Value, <<"^[^@\\s]+@[^@\\s]+\\.[^@\\s]+$">>) of
        {match, _} -> ok;
        nomatch    -> {error, iolist_to_binary([atom_to_binary(Field),
                                                " is not a valid email"])}
    end.

integer_range(Field, Value, Min, Max) when is_integer(Value) ->
    case Value >= Min andalso Value =< Max of
        true  -> ok;
        false -> {error, iolist_to_binary([atom_to_binary(Field),
                           " must be between ", integer_to_binary(Min),
                           " and ", integer_to_binary(Max)])}
    end.

enum(Field, Value, Allowed) ->
    case lists:member(Value, Allowed) of
        true  -> ok;
        false -> {error, iolist_to_binary([atom_to_binary(Field),
                           " must be one of: ",
                           lists:join(<<", ">>, [atom_to_binary(A, utf8) || A <- Allowed])])}
    end.

apply_rule(Field, Value, {required})           -> required(Field, Value);
apply_rule(Field, Value, {min_length, Min})    -> min_length(Field, Value, Min);
apply_rule(Field, Value, {max_length, Max})    -> max_length(Field, Value, Max);
apply_rule(Field, Value, {email})              -> email(Field, Value);
apply_rule(Field, Value, {range, Min, Max})    -> integer_range(Field, Value, Min, Max);
apply_rule(Field, Value, {enum, Allowed})      -> enum(Field, Value, Allowed).

%% Usage example
validate_user(Data) ->
    validator:validate(Data, [
        {username, {required}},
        {username, {min_length, 3}},
        {username, {max_length, 50}},
        {email,    {required}},
        {email,    {email}},
        {password, {required}},
        {password, {min_length, 8}},
        {age,      {range, 18, 120}}
    ]).
```

---

## 3. SQL Injection Prevention

```erlang
%% NEVER do this:
bad_query(Username) ->
    Sql = "SELECT * FROM users WHERE username = '" ++ Username ++ "'",
    db:raw_query(Sql).

%% ALWAYS use parameterized queries:
good_query(Username) ->
    db:query("SELECT * FROM users WHERE username = $1", [Username]).

%% epgsql parameterized queries are safe by design:
{ok, Rows} = epgsql:equery(Conn,
    "SELECT id, email FROM users WHERE username = $1",
    [Username]).

%% Safe integer conversion
safe_get_id(BinId) ->
    try
        Id = binary_to_integer(BinId),
        if Id > 0 -> {ok, Id};
           true   -> {error, invalid_id}
        end
    catch
        _:_ -> {error, invalid_id}
    end.

%% Safe use in handler
get_user_handler(Req, State) ->
    IdBin = cowboy_req:binding(id, Req),
    case safe_get_id(IdBin) of
        {ok, Id} ->
            case user_repo:find(Id) of
                {ok, User} -> reply_json(200, User, Req, State);
                _ -> reply_json(404, #{error => <<"not found">>}, Req, State)
            end;
        {error, invalid_id} ->
            reply_json(400, #{error => <<"invalid id">>}, Req, State)
    end.
```

---

## 4. XSS Prevention

```erlang
%% html_escape.erl — Escape HTML special characters
-module(html_escape).
-export([escape/1]).

escape(Bin) when is_binary(Bin) ->
    escape_chars(Bin, <<>>);
escape(Str) when is_list(Str) ->
    escape(list_to_binary(Str)).

escape_chars(<<>>, Acc) -> Acc;
escape_chars(<<$&, Rest/binary>>, Acc) -> escape_chars(Rest, <<Acc/binary, "&amp;">>);
escape_chars(<<$<, Rest/binary>>, Acc) -> escape_chars(Rest, <<Acc/binary, "&lt;">>);
escape_chars(<<$>, Rest/binary>>, Acc) -> escape_chars(Rest, <<Acc/binary, "&gt;">>);
escape_chars(<<$", Rest/binary>>, Acc) -> escape_chars(Rest, <<Acc/binary, "&quot;">>);
escape_chars(<<$', Rest/binary>>, Acc) -> escape_chars(Rest, <<Acc/binary, "&#x27;">>);
escape_chars(<<C,  Rest/binary>>, Acc) -> escape_chars(Rest, <<Acc/binary, C>>).

%% ใช้ใน template rendering
render_user_profile(User) ->
    Name  = html_escape:escape(maps:get(name, User, <<>>)),
    Bio   = html_escape:escape(maps:get(bio, User, <<>>)),
    <<"<div class='profile'>"
      "<h1>", Name/binary, "</h1>"
      "<p>",  Bio/binary,  "</p>"
      "</div>">>.

%% สำหรับ JSON API: เนื้อหาไม่ถูก rendered เป็น HTML โดยตรง
%% แต่ต้องมี Content-Type: application/json เสมอ
%% Browser จะไม่ execute JSON เป็น HTML
```

---

## 5. Secrets Management

```erlang
%% secrets.erl — Secrets management
-module(secrets).
-export([get/1, get/2, rotate/2]).

%% อ่าน secret จาก environment variables เท่านั้น
%% NEVER hardcode secrets in source code

get(Key) ->
    get(Key, undefined).

get(Key, Default) ->
    EnvKey = key_to_env(Key),
    case os:getenv(EnvKey) of
        false ->
            %% Fallback to sys.config (for dev only)
            application:get_env(myapp, Key, Default);
        Value ->
            list_to_binary(Value)
    end.

key_to_env(db_password)   -> "DB_PASSWORD";
key_to_env(jwt_secret)    -> "JWT_SECRET";
key_to_env(api_key)       -> "API_KEY";
key_to_env(redis_password) -> "REDIS_PASSWORD";
key_to_env(Key) ->
    string:uppercase(atom_to_list(Key)).

%% Rotation: update without restart
rotate(Key, NewValue) ->
    EnvKey = key_to_env(Key),
    os:putenv(EnvKey, binary_to_list(NewValue)),
    logger:info("Secret rotated: ~p", [Key]).

%% ตัวอย่าง: ไม่ทำ
bad_config() ->
    <<"hardcoded_secret_do_not_do_this">>.  %% NEVER!

%% ทำแบบนี้แทน
good_config() ->
    secrets:get(jwt_secret).

%% Validate secrets exist at startup
validate_secrets() ->
    Required = [jwt_secret, db_password],
    Missing  = [K || K <- Required, secrets:get(K) =:= undefined],
    case Missing of
        []     -> ok;
        Keys   ->
            logger:critical("Missing required secrets: ~p", [Keys]),
            {error, missing_secrets}
    end.
```

---

## 6. TLS/SSL Configuration

```erlang
%% Secure TLS configuration

start_https() ->
    Port    = 443,
    CertDir = code:priv_dir(myapp),

    cowboy:start_tls(https,
        [
            {port,     Port},
            {certfile, filename:join(CertDir, "ssl/server.crt")},
            {keyfile,  filename:join(CertDir, "ssl/server.key")},

            %% Modern TLS configuration
            {versions, ['tlsv1.3', 'tlsv1.2']},  %% disable TLS 1.0, 1.1

            %% Strong cipher suites (TLS 1.3 uses its own)
            {ciphers, [
                "TLS_AES_256_GCM_SHA384",
                "TLS_CHACHA20_POLY1305_SHA256",
                "TLS_AES_128_GCM_SHA256",
                %% TLS 1.2 ciphers:
                "ECDHE-ECDSA-AES256-GCM-SHA384",
                "ECDHE-RSA-AES256-GCM-SHA384"
            ]},

            %% Disable weak options
            {secure_renegotiate, true},
            {reuse_sessions,     false},   %% for forward secrecy

            %% HSTS header (also set in response headers)
            {honor_cipher_order, server}
        ],
        #{env => #{dispatch => Dispatch}}
    ).

%% HTTP → HTTPS redirect
start_http_redirect() ->
    cowboy:start_clear(http, [{port, 80}], #{
        env => #{dispatch => cowboy_router:compile([
            {'_', [{"/[...]", redirect_handler, []}]}
        ])}
    }).

%% redirect_handler.erl
redirect_handler_init(Req, State) ->
    Host = cowboy_req:host(Req),
    Path = cowboy_req:path(Req),
    Qs   = cowboy_req:qs(Req),
    Url  = case Qs of
        <<>> -> <<"https://", Host/binary, Path/binary>>;
        _    -> <<"https://", Host/binary, Path/binary, "?", Qs/binary>>
    end,
    Req2 = cowboy_req:reply(301, #{<<"location">> => Url}, <<>>, Req),
    {ok, Req2, State}.
```

---

## 7. Secure Headers

```erlang
%% security_headers_middleware.erl
-module(security_headers_middleware).
-behaviour(cowboy_middleware).
-export([execute/2]).

execute(Req, Env) ->
    Req2 = add_security_headers(Req),
    {ok, Req2, Env}.

add_security_headers(Req) ->
    Headers = #{
        %% Prevent clickjacking
        <<"x-frame-options">> => <<"DENY">>,

        %% Prevent MIME type sniffing
        <<"x-content-type-options">> => <<"nosniff">>,

        %% XSS protection (legacy browsers)
        <<"x-xss-protection">> => <<"1; mode=block">>,

        %% HSTS: force HTTPS for 1 year
        <<"strict-transport-security">> =>
            <<"max-age=31536000; includeSubDomains; preload">>,

        %% Content Security Policy
        <<"content-security-policy">> =>
            <<"default-src 'self'; "
              "script-src 'self'; "
              "style-src 'self' 'unsafe-inline'; "
              "img-src 'self' data: https:; "
              "connect-src 'self'; "
              "font-src 'self'; "
              "object-src 'none'; "
              "frame-ancestors 'none'">>,

        %% Referrer Policy
        <<"referrer-policy">> => <<"strict-origin-when-cross-origin">>,

        %% Permissions Policy
        <<"permissions-policy">> =>
            <<"geolocation=(), microphone=(), camera=()">>
    },
    maps:fold(fun(K, V, R) ->
        cowboy_req:set_resp_header(K, V, R)
    end, Req, Headers).
```

---

## 8. Audit Logging

```erlang
%% audit_log.erl — Tamper-evident audit trail
-module(audit_log).
-export([log/3, log/4, recent/2]).

log(Action, Actor, Resource) ->
    log(Action, Actor, Resource, #{}).

log(Action, Actor, Resource, Extra) ->
    Entry = #{
        id        => make_id(),
        action    => Action,
        actor_id  => Actor,
        resource  => Resource,
        ip        => get_ip(),
        timestamp => erlang:system_time(second),
        extra     => Extra
    },
    %% Write to immutable audit log table
    db:query(
        "INSERT INTO audit_log (id, action, actor_id, resource, "
        "ip_address, created_at, extra) "
        "VALUES ($1, $2, $3, $4, $5, to_timestamp($6), $7)",
        [maps:get(id, Entry), Action, Actor, Resource,
         maps:get(ip, Entry), maps:get(timestamp, Entry),
         jsx:encode(Extra)]
    ),
    %% Also log to structured logger
    logger:info("AUDIT action=~s actor=~p resource=~s",
                [Action, Actor, Resource]).

recent(ActorId, Limit) ->
    {ok, Rows} = db:query(
        "SELECT * FROM audit_log WHERE actor_id = $1 "
        "ORDER BY created_at DESC LIMIT $2",
        [ActorId, Limit]),
    db:rows_to_maps(Rows).

get_ip() ->
    case get(request_ip) of
        undefined -> <<"unknown">>;
        Ip        -> Ip
    end.

make_id() -> binary:encode_hex(crypto:strong_rand_bytes(8)).

%% Typical usage
handle_delete_user(UserId, AdminId, Req) ->
    {Ip, _} = cowboy_req:peer(Req),
    put(request_ip, list_to_binary(inet:ntoa(Ip))),

    case user_repo:delete(UserId) of
        ok ->
            audit_log:log(<<"user.deleted">>, AdminId,
                          <<"user:", (integer_to_binary(UserId))/binary>>,
                          #{ip => inet:ntoa(Ip)}),
            reply_json(204, #{}, Req, #{});
        Error ->
            Error
    end.
```

---

## 9. แบบฝึกหัด

### Exercise: Security Audit Checklist

```erlang
%% สร้าง security check module
-module(security_checker).
-export([run_checks/0]).

run_checks() ->
    Checks = [
        {<<"TLS enabled">>,       check_tls()},
        {<<"Secrets from env">>,  check_secrets()},
        {<<"Auth middleware">>,   check_auth()},
        {<<"Rate limiting">>,     check_rate_limit()},
        {<<"Audit logging">>,     check_audit_log()},
        {<<"Input validation">>,  check_validation()}
    ],
    Failed = [{Name, R} || {Name, R} <- Checks, R =/= ok],
    case Failed of
        [] ->
            io:format("All security checks passed!~n");
        _ ->
            io:format("Failed checks:~n"),
            lists:foreach(fun({Name, R}) ->
                io:format("  FAIL: ~s — ~p~n", [Name, R])
            end, Failed)
    end.

check_tls() ->
    case cowboy:get_listener_info(https) of
        {ok, _} -> ok;
        _       -> {fail, <<"HTTPS not started">>}
    end.

check_secrets() ->
    case os:getenv("JWT_SECRET") of
        false -> {fail, <<"JWT_SECRET not set">>};
        _     -> ok
    end.

check_auth() ->
    %% Verify auth middleware is in Cowboy middleware stack
    ok.  %% implement based on your config

check_rate_limit() ->
    case whereis(token_bucket) of
        undefined -> {fail, <<"Rate limiter not running">>};
        _         -> ok
    end.

check_audit_log() ->
    case db:query("SELECT COUNT(*) FROM audit_log LIMIT 1", []) of
        {ok, _} -> ok;
        _       -> {fail, <<"Audit log table not accessible">>}
    end.

check_validation() -> ok.
```

---

## สรุป Part 38

✅ OWASP Top 10 ใน Erlang context  
✅ Input validation module  
✅ SQL injection prevention  
✅ XSS prevention (HTML escaping)  
✅ Secrets management (env vars)  
✅ TLS/SSL configuration  
✅ Secure HTTP headers  
✅ Audit logging

---

*Part 38/100 | [← ก่อนหน้า](../part37/README.md) | [ถัดไป →](../part39/README.md)*
