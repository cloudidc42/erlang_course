# Part 67: Telecom Systems — OTP Origins

> **"Erlang was born in telecom — understanding its roots makes you a better engineer"**  
> Erlang ถือกำเนิดในโทรคมนาคม — เข้าใจรากเหง้าทำให้คุณเป็น engineer ที่ดีกว่า

---

## สารบัญ

1. [Call Control System](#1-call-control-system)
2. [Session Border Controller](#2-session-border-controller)
3. [Billing and CDR](#3-billing-and-cdr)
4. [Fault Tolerance in Telecom](#4-fault-tolerance-in-telecom)
5. [Protocol State Machines](#5-protocol-state-machines)
6. [Load Balancing Calls](#6-load-balancing-calls)
7. [แบบฝึกหัด](#7-แบบฝึกหัด)

---

## 1. Call Control System

```erlang
%% call_controller.erl — telephone call lifecycle management
-module(call_controller).
-behaviour(gen_statem).
-export([start_link/2, answer/1, hangup/1, hold/1, transfer/2]).
-export([callback_mode/0, init/1, idle/3, ringing/3,
         active/3, on_hold/3, transferring/3]).

-record(call_data, {
    call_id    :: binary(),
    from       :: binary(),   %% caller number
    to         :: binary(),   %% called number
    started    :: integer(),  %% unix ms
    answered   :: integer() | undefined,
    codec      :: atom(),     %% g711, g729, opus
    media_pid  :: pid() | undefined,
    billing_pid :: pid() | undefined
}).

callback_mode() -> state_functions.

start_link(CallId, Opts) ->
    gen_statem:start_link(?MODULE, {CallId, Opts}, []).

answer(Pid)             -> gen_statem:call(Pid, answer).
hangup(Pid)             -> gen_statem:call(Pid, hangup).
hold(Pid)               -> gen_statem:call(Pid, hold).
transfer(Pid, Target)   -> gen_statem:call(Pid, {transfer, Target}).

init({CallId, #{from := From, to := To, codec := Codec}}) ->
    process_flag(trap_exit, true),
    logger:info("Call ~p started: ~p -> ~p", [CallId, From, To]),
    telemetry:execute([call, initiated], #{count => 1},
                      #{from => From, to => To}),
    Data = #call_data{
        call_id  = CallId,
        from     = From,
        to       = To,
        started  = os:system_time(millisecond),
        codec    = Codec
    },
    %% Notify called party
    {ok, ringing, Data, [{timeout, 30000, no_answer}]}.

%% IDLE state (not really used, kept for completeness)
idle({call, From}, {originate, To}, Data) ->
    {next_state, ringing, Data#call_data{to=To},
     [{reply, From, ok}, {timeout, 30000, no_answer}]}.

%% RINGING: waiting for answer
ringing({call, From}, answer, Data) ->
    Now = os:system_time(millisecond),
    {ok, MediaPid}   = media_server:start_session(Data#call_data.codec),
    {ok, BillingPid} = billing:start_session(Data#call_data.call_id,
                                              Data#call_data.from),
    Data1 = Data#call_data{answered=Now, media_pid=MediaPid,
                            billing_pid=BillingPid},
    telemetry:execute([call, answered], #{duration_until_answer => Now - Data#call_data.started}, #{}),
    {next_state, active, Data1, [{reply, From, ok}]};

ringing({call, From}, hangup, Data) ->
    telemetry:execute([call, abandoned], #{count => 1}, #{}),
    {stop_and_reply, normal, [{reply, From, ok}], Data};

ringing(timeout, no_answer, Data) ->
    telemetry:execute([call, no_answer], #{count => 1}, #{}),
    {stop, normal, Data}.

%% ACTIVE: call in progress
active({call, From}, hangup, Data) ->
    cleanup_call(Data),
    {stop_and_reply, normal, [{reply, From, ok}], Data};

active({call, From}, hold, #call_data{media_pid=MediaPid} = Data) ->
    media_server:hold(MediaPid),
    {next_state, on_hold, Data, [{reply, From, ok}]};

active({call, From}, {transfer, Target}, Data) ->
    {next_state, transferring, Data#call_data{to=Target},
     [{reply, From, ok}]};

active(info, {'EXIT', MediaPid, Reason}, #call_data{media_pid=MediaPid} = Data) ->
    logger:error("Media session died: ~p", [Reason]),
    cleanup_call(Data),
    {stop, {media_failure, Reason}, Data}.

%% ON_HOLD: media paused
on_hold({call, From}, answer, #call_data{media_pid=MediaPid} = Data) ->
    media_server:resume(MediaPid),
    {next_state, active, Data, [{reply, From, ok}]};

on_hold({call, From}, hangup, Data) ->
    cleanup_call(Data),
    {stop_and_reply, normal, [{reply, From, ok}], Data}.

%% TRANSFERRING: call being transferred
transferring({call, From}, {transfer_complete, NewPid}, Data) ->
    {next_state, active, Data#call_data{media_pid=NewPid},
     [{reply, From, ok}]};
transferring({call, From}, hangup, Data) ->
    cleanup_call(Data),
    {stop_and_reply, normal, [{reply, From, ok}], Data}.

cleanup_call(#call_data{call_id=Id, billing_pid=BP, media_pid=MP,
                        answered=Answered, started=Started}) ->
    Duration = case Answered of
        undefined -> 0;
        _         -> os:system_time(millisecond) - Answered
    end,
    telemetry:execute([call, completed], #{duration_ms => Duration}, #{id => Id}),
    catch billing:end_session(BP, Duration),
    catch media_server:stop_session(MP),
    logger:info("Call ~p ended, duration=~pms", [Id, Duration]).
```

---

## 2. Session Border Controller

```erlang
%% sbc.erl — simplified Session Border Controller
-module(sbc).
-behaviour(gen_server).
-export([start_link/0, register_endpoint/3, route_call/2,
         deregister/1, get_endpoints/0]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

-record(endpoint, {
    id          :: binary(),
    address     :: {inet:ip_address(), inet:port_number()},
    codec_list  :: [atom()],
    registered  :: integer(),
    expires     :: integer(),
    call_count  :: non_neg_integer()
}).

-record(state, {
    endpoints :: ets:tab(),
    routes    :: ets:tab()
}).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

register_endpoint(Id, Address, Opts) ->
    gen_server:call(?MODULE, {register, Id, Address, Opts}).

route_call(From, To) ->
    gen_server:call(?MODULE, {route, From, To}).

deregister(Id) ->
    gen_server:cast(?MODULE, {deregister, Id}).

get_endpoints() ->
    gen_server:call(?MODULE, get_endpoints).

init([]) ->
    Endpoints = ets:new(sbc_endpoints, [set, {keypos, 2}]),
    Routes    = ets:new(sbc_routes, [set]),
    %% Periodic cleanup of expired registrations
    erlang:send_after(60000, self(), cleanup_expired),
    {ok, #state{endpoints=Endpoints, routes=Routes}}.

handle_call({register, Id, Address, Opts}, _From,
            #state{endpoints=Eps} = State) ->
    Ttl = maps:get(ttl, Opts, 3600),
    Now = os:system_time(second),
    EP  = #endpoint{
        id         = Id,
        address    = Address,
        codec_list = maps:get(codecs, Opts, [g711, g729]),
        registered = Now,
        expires    = Now + Ttl,
        call_count = 0
    },
    ets:insert(Eps, EP),
    logger:info("Endpoint registered: ~p at ~p", [Id, Address]),
    {reply, ok, State};

handle_call({route, From, To}, _From, #state{endpoints=Eps} = State) ->
    case find_endpoint(To, Eps) of
        {ok, EP} ->
            %% Negotiate codec (find common)
            FromCodecs = get_codecs(From, Eps),
            CommonCodecs = lists:filter(fun(C) ->
                lists:member(C, EP#endpoint.codec_list)
            end, FromCodecs),
            case CommonCodecs of
                []         -> {reply, {error, no_common_codec}, State};
                [Codec|_]  ->
                    ets:update_element(Eps, To,
                        {#endpoint.call_count, EP#endpoint.call_count + 1}),
                    {reply, {ok, #{target  => EP#endpoint.address,
                                   codec   => Codec}}, State}
            end;
        error ->
            {reply, {error, endpoint_not_found}, State}
    end;

handle_call(get_endpoints, _From, #state{endpoints=Eps} = State) ->
    All = ets:tab2list(Eps),
    Now = os:system_time(second),
    Active = [EP || EP <- All, EP#endpoint.expires > Now],
    {reply, {ok, Active}, State}.

handle_cast({deregister, Id}, #state{endpoints=Eps} = State) ->
    ets:delete(Eps, Id),
    {noreply, State}.

handle_info(cleanup_expired, #state{endpoints=Eps} = State) ->
    Now = os:system_time(second),
    Expired = ets:select(Eps,
        [{#endpoint{id='$1', expires='$2', _='_'},
          [{'<', '$2', Now}],
          ['$1']}]),
    [ets:delete(Eps, Id) || Id <- Expired],
    logger:info("Cleaned ~p expired registrations", [length(Expired)]),
    erlang:send_after(60000, self(), cleanup_expired),
    {noreply, State}.

find_endpoint(Id, Eps) ->
    case ets:lookup(Eps, Id) of
        [EP] -> {ok, EP};
        []   -> error
    end.

get_codecs(Id, Eps) ->
    case find_endpoint(Id, Eps) of
        {ok, EP} -> EP#endpoint.codec_list;
        error    -> [g711]
    end.
```

---

## 3. Billing and CDR

```erlang
%% cdr.erl — Call Detail Records generation
-module(cdr).
-export([create/4, finalize/3, export_batch/1]).

-record(cdr, {
    call_id       :: binary(),
    from          :: binary(),
    to            :: binary(),
    start_time    :: calendar:datetime(),
    answer_time   :: calendar:datetime() | undefined,
    end_time      :: calendar:datetime(),
    duration_secs :: non_neg_integer(),
    codec         :: atom(),
    cause         :: normal | busy | no_answer | failed,
    bytes_in      :: non_neg_integer(),
    bytes_out     :: non_neg_integer(),
    cost_units    :: float()
}).

create(CallId, From, To, StartMs) ->
    #cdr{
        call_id    = CallId,
        from       = From,
        to         = To,
        start_time = ms_to_datetime(StartMs),
        duration_secs = 0,
        bytes_in   = 0,
        bytes_out  = 0,
        cost_units = 0.0,
        cause      = normal
    }.

finalize(CDR, EndMs, #{answer_ms := AnswerMs,
                        bytes_in  := BytesIn,
                        bytes_out := BytesOut,
                        cause     := Cause}) ->
    StartMs = datetime_to_ms(CDR#cdr.start_time),
    Duration = max(0, (EndMs - case AnswerMs of
        undefined -> EndMs;
        _         -> AnswerMs
    end) div 1000),
    Cost = calculate_cost(CDR#cdr.from, CDR#cdr.to, Duration),
    CDR#cdr{
        answer_time   = case AnswerMs of
            undefined -> undefined;
            _         -> ms_to_datetime(AnswerMs)
        end,
        end_time      = ms_to_datetime(EndMs),
        duration_secs = Duration,
        bytes_in      = BytesIn,
        bytes_out     = BytesOut,
        cost_units    = Cost,
        cause         = Cause
    }.

export_batch(CDRs) ->
    %% Write to PostgreSQL
    {ok, Conn} = db:checkout(),
    try
        Values = [cdr_to_row(C) || C <- CDRs],
        db:execute(Conn,
            "INSERT INTO cdrs (call_id,from_number,to_number,start_time,
                               end_time,duration_secs,cause,cost_units)
             VALUES " ++ placeholders(length(CDRs), 8),
            lists:flatten(Values))
    after
        db:checkin(Conn)
    end.

cdr_to_row(#cdr{call_id=Id, from=F, to=T, start_time=S,
                end_time=E, duration_secs=D, cause=C, cost_units=Cost}) ->
    [Id, F, T, format_datetime(S), format_datetime(E), D, atom_to_binary(C), Cost].

calculate_cost(From, To, DurationSecs) ->
    Rate = lookup_rate(From, To),
    Rate * DurationSecs / 60.

lookup_rate(_From, <<"00">>) -> 0.15;  %% International: 15 cents/min
lookup_rate(_From, _To)      -> 0.02.  %% Domestic: 2 cents/min

placeholders(N, Cols) ->
    Rows = [
        "(" ++ string:join(["$" ++ integer_to_list(I)
                             || I <- lists:seq((R-1)*Cols+1, R*Cols)], ",") ++ ")"
        || R <- lists:seq(1, N)],
    string:join(Rows, ",").

ms_to_datetime(Ms) ->
    Seconds = Ms div 1000,
    calendar:gregorian_seconds_to_datetime(
        calendar:datetime_to_gregorian_seconds({{1970,1,1},{0,0,0}}) + Seconds).

datetime_to_ms(DT) ->
    Epoch = calendar:datetime_to_gregorian_seconds({{1970,1,1},{0,0,0}}),
    (calendar:datetime_to_gregorian_seconds(DT) - Epoch) * 1000.

format_datetime({{Y,Mo,D},{H,Mi,S}}) ->
    iolist_to_binary(io_lib:format("~4..0B-~2..0B-~2..0B ~2..0B:~2..0B:~2..0B",
                                   [Y, Mo, D, H, Mi, S])).
```

---

## 4. Fault Tolerance in Telecom

```erlang
%% telecom_sup.erl — 9-nines fault tolerance pattern
-module(telecom_sup).
-behaviour(supervisor).
-export([start_link/0, init/1]).

%% Telecom requirement: 99.9999999% uptime = 31ms downtime/year
%% Key principles:
%% 1. Redundant supervisors
%% 2. Fast restart (< 1 second)
%% 3. Zero-downtime hot upgrades
%% 4. Geographic distribution

start_link() ->
    supervisor:start_link({local, ?MODULE}, ?MODULE, []).

init([]) ->
    %% one_for_one: each child independent
    %% High intensity/period: telecom must restart very quickly
    {ok, {
        #{strategy  => one_for_one,
          intensity => 100,   %% allow 100 restarts...
          period    => 5},    %% ...in 5 seconds
        [
            %% Call routing: must always be available
            #{id       => call_router,
              start    => {call_router, start_link, []},
              restart  => permanent,
              shutdown => 5000},

            %% Registration: endpoints can come and go
            #{id       => registration_server,
              start    => {registration_server, start_link, []},
              restart  => permanent,
              shutdown => 5000},

            %% CDR: temporary — don't restart, just log and move on
            #{id       => cdr_writer,
              start    => {cdr_writer, start_link, []},
              restart  => transient,   %% only restart on abnormal exit
              shutdown => 10000}
        ]
    }}.

%% Alarm management
-module(alarm_manager).
-behaviour(gen_event).
-export([init/1, handle_event/2, handle_call/2]).

init([]) -> {ok, []}.

handle_event({alarm, call_failure_rate, Rate}, State) when Rate > 0.01 ->
    %% More than 1% call failure rate: alert NOC
    notify_noc(#{alarm => call_failure_rate, value => Rate,
                  severity => major}),
    {ok, State};
handle_event({alarm, registration_timeout, Count}, State) when Count > 100 ->
    notify_noc(#{alarm => mass_deregistration, count => Count,
                  severity => critical}),
    {ok, State};
handle_event(_Event, State) ->
    {ok, State}.

handle_call(_, State) -> {ok, ok, State}.

notify_noc(Alarm) ->
    %% In real system: SNMP trap, email, SMS
    logger:critical("NOC ALARM: ~p", [Alarm]).
```

---

## 5. Protocol State Machines

```erlang
%% sip_fsm.erl — simplified SIP dialog state machine
-module(sip_fsm).
-behaviour(gen_statem).
-export([start_link/1, send_invite/2, send_ack/1, send_bye/1,
         receive_response/2]).
-export([callback_mode/0, init/1, calling/3, proceeding/3,
         completed/3, terminated/3]).

callback_mode() -> state_functions.

start_link(DialogId) ->
    gen_statem:start_link(?MODULE, DialogId, []).

send_invite(Pid, Target)      -> gen_statem:call(Pid, {send_invite, Target}).
send_ack(Pid)                 -> gen_statem:call(Pid, send_ack).
send_bye(Pid)                 -> gen_statem:call(Pid, send_bye).
receive_response(Pid, Resp)   -> gen_statem:cast(Pid, {response, Resp}).

init(DialogId) ->
    {ok, calling, #{dialog_id => DialogId, retransmit_count => 0}}.

%% CALLING: INVITE sent, waiting for response
calling({call, From}, {send_invite, Target}, Data) ->
    sip_transport:send(#{method => invite, target => Target}),
    TimerRef = erlang:send_after(500, self(), retransmit),
    {keep_state, Data#{target => Target, timer => TimerRef},
     [{reply, From, ok}]};

calling(cast, {response, #{status := Status}}, Data)
        when Status >= 100, Status < 200 ->
    %% Provisional response: move to proceeding
    cancel_timer(Data),
    {next_state, proceeding, Data};

calling(cast, {response, #{status := 200}}, Data) ->
    cancel_timer(Data),
    {next_state, completed, Data};

calling(cast, {response, #{status := Status}}, Data)
        when Status >= 300 ->
    cancel_timer(Data),
    {next_state, terminated, Data#{reason => Status}};

calling(info, retransmit, #{retransmit_count := N, target := Target} = Data)
        when N < 7 ->
    sip_transport:send(#{method => invite, target => Target}),
    Delay = round(500 * math:pow(2, N)),
    TimerRef = erlang:send_after(Delay, self(), retransmit),
    {keep_state, Data#{retransmit_count => N + 1, timer => TimerRef}};

calling(info, retransmit, Data) ->
    {next_state, terminated, Data#{reason => timeout}}.

%% PROCEEDING: provisional response received
proceeding(cast, {response, #{status := 200}}, Data) ->
    {next_state, completed, Data};
proceeding(cast, {response, #{status := S}}, Data) when S >= 300 ->
    {next_state, terminated, Data#{reason => S}}.

%% COMPLETED: final response, waiting for ACK
completed({call, From}, send_ack, Data) ->
    sip_transport:send(#{method => ack}),
    {next_state, terminated, Data#{reason => normal},
     [{reply, From, ok}]}.

terminated({call, From}, send_bye, Data) ->
    sip_transport:send(#{method => bye}),
    {keep_state, Data, [{reply, From, ok}]}.

cancel_timer(#{timer := Ref}) -> erlang:cancel_timer(Ref);
cancel_timer(_) -> ok.
```

---

## 6. Load Balancing Calls

```erlang
%% call_lb.erl — call load balancer with health checks
-module(call_lb).
-behaviour(gen_server).
-export([start_link/1, get_server/0, report_success/1,
         report_failure/1, server_health/0]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

-record(server, {
    id           :: binary(),
    address      :: {inet:ip_address(), pos_integer()},
    weight       :: pos_integer(),
    current_calls :: non_neg_integer(),
    failures      :: non_neg_integer(),
    healthy       :: boolean()
}).

start_link(Servers) ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, Servers, []).

get_server() ->
    gen_server:call(?MODULE, get_server).

report_success(ServerId) ->
    gen_server:cast(?MODULE, {success, ServerId}).

report_failure(ServerId) ->
    gen_server:cast(?MODULE, {failure, ServerId}).

server_health() ->
    gen_server:call(?MODULE, health).

init(ServerConfigs) ->
    Servers = [#server{
        id           = maps:get(id, S),
        address      = maps:get(address, S),
        weight       = maps:get(weight, S, 1),
        current_calls = 0,
        failures     = 0,
        healthy      = true
    } || S <- ServerConfigs],
    %% Periodic health check
    erlang:send_after(5000, self(), health_check),
    {ok, Servers}.

handle_call(get_server, _From, Servers) ->
    Healthy = [S || S <- Servers, S#server.healthy],
    case Healthy of
        [] -> {reply, {error, no_healthy_servers}, Servers};
        _  ->
            %% Least connections with weight
            Scored = [{S#server.current_calls / S#server.weight, S}
                      || S <- Healthy],
            {_, Best} = hd(lists:sort(Scored)),
            Updated = update_server(Best#server.id,
                                    fun(S) ->
                                        S#server{current_calls = S#server.current_calls + 1}
                                    end,
                                    Servers),
            {reply, {ok, Best#server.id, Best#server.address}, Updated}
    end;

handle_call(health, _From, Servers) ->
    {reply, [{S#server.id, S#server.healthy, S#server.current_calls}
             || S <- Servers], Servers}.

handle_cast({success, ServerId}, Servers) ->
    Updated = update_server(ServerId,
        fun(S) -> S#server{current_calls = max(0, S#server.current_calls - 1),
                           failures = 0} end,
        Servers),
    {noreply, Updated};

handle_cast({failure, ServerId}, Servers) ->
    Updated = update_server(ServerId,
        fun(S) ->
            NewFails = S#server.failures + 1,
            Healthy  = NewFails < 3,  %% unhealthy after 3 failures
            S#server{current_calls = max(0, S#server.current_calls - 1),
                     failures = NewFails, healthy = Healthy}
        end,
        Servers),
    {noreply, Updated};

handle_info(health_check, Servers) ->
    Updated = [do_health_check(S) || S <- Servers],
    erlang:send_after(5000, self(), health_check),
    {noreply, Updated}.

do_health_check(#server{address={Host, Port}} = S) ->
    case gen_tcp:connect(Host, Port, [], 2000) of
        {ok, Sock} ->
            gen_tcp:close(Sock),
            S#server{healthy=true, failures=0};
        {error, _} ->
            S#server{healthy=false}
    end.

update_server(Id, Fun, Servers) ->
    [case S#server.id =:= Id of
         true  -> Fun(S);
         false -> S
     end || S <- Servers].
```

---

## 7. แบบฝึกหัด

1. Implement DTMF digit detection: รับ RTP stream และ detect 0-9, *, #
2. สร้าง call recording system: save RTP packets ลง file พร้อม CDR
3. เพิ่ม conference call support: สร้าง audio mixing สำหรับ N participants
4. Implement call queuing: hold calls จนกว่า agent จะว่าง

---

## สรุป Part 67

✅ Call control system ด้วย gen_statem lifecycle  
✅ Session Border Controller: registration, routing, codec negotiation  
✅ Billing และ CDR generation + bulk export  
✅ 9-nines fault tolerance patterns  
✅ SIP dialog FSM  
✅ Load balancing calls ด้วย least connections + health checks  

---

*Part 67/100 | [← ก่อนหน้า](../part66/README.md) | [ถัดไป →](../part68/README.md)*
