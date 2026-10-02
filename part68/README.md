# Part 68: Healthcare Platform

> **"In healthcare, reliability is measured in lives — Erlang's fault tolerance is perfect for this"**  
> ในการแพทย์ ความน่าเชื่อถือวัดด้วยชีวิต — fault tolerance ของ Erlang เหมาะสมอย่างยิ่ง

---

## สารบัญ

1. [Patient Record System](#1-patient-record-system)
2. [Medical Device Integration](#2-medical-device-integration)
3. [FHIR API Implementation](#3-fhir-api-implementation)
4. [Clinical Alert System](#4-clinical-alert-system)
5. [HIPAA Audit Logging](#5-hipaa-audit-logging)
6. [Appointment Scheduling](#6-appointment-scheduling)
7. [แบบฝึกหัด](#7-แบบฝึกหัด)

---

## 1. Patient Record System

```erlang
%% patient_records.erl — Electronic Health Record (EHR) system
-module(patient_records).
-export([create_patient/2, get_patient/1, update_patient/2,
         add_encounter/2, get_encounters/2, search_patients/1]).

-record(patient, {
    id           :: binary(),
    first_name   :: binary(),
    last_name    :: binary(),
    dob          :: {calendar:year(), calendar:month(), calendar:day()},
    gender       :: male | female | other | unknown,
    mrn          :: binary(),  %% Medical Record Number
    created_at   :: integer(),
    updated_at   :: integer()
}).

-record(encounter, {
    id           :: binary(),
    patient_id   :: binary(),
    type         :: inpatient | outpatient | emergency | telemedicine,
    provider_id  :: binary(),
    start_time   :: integer(),
    end_time     :: integer() | undefined,
    chief_complaint :: binary(),
    diagnoses    :: [icd10_code()],
    status       :: active | finished | cancelled
}).

-type icd10_code() :: binary().  %% e.g. <<"A00.0">>

create_patient(PatientData, CreatedBy) ->
    PatientId = generate_patient_id(),
    MRN       = generate_mrn(),
    Now       = os:system_time(millisecond),
    Patient   = #patient{
        id         = PatientId,
        first_name = maps:get(first_name, PatientData),
        last_name  = maps:get(last_name, PatientData),
        dob        = maps:get(dob, PatientData),
        gender     = maps:get(gender, PatientData, unknown),
        mrn        = MRN,
        created_at = Now,
        updated_at = Now
    },
    case db:execute(
        "INSERT INTO patients (id,first_name,last_name,dob,gender,mrn,created_at,updated_at)
         VALUES ($1,$2,$3,$4,$5,$6,$7,$8)",
        [PatientId, Patient#patient.first_name, Patient#patient.last_name,
         format_date(Patient#patient.dob), atom_to_binary(Patient#patient.gender),
         MRN, Now, Now]) of
        {ok, _} ->
            %% HIPAA audit log
            audit_log:record(#{
                action    => patient_created,
                entity    => patient,
                entity_id => PatientId,
                actor     => CreatedBy,
                timestamp => Now
            }),
            {ok, Patient};
        {error, Reason} ->
            {error, Reason}
    end.

get_patient(PatientId) ->
    case db:query(
        "SELECT id,first_name,last_name,dob,gender,mrn FROM patients WHERE id=$1 AND deleted_at IS NULL",
        [PatientId]) of
        {ok, [Row]}  -> {ok, row_to_patient(Row)};
        {ok, []}     -> {error, not_found};
        {error, Err} -> {error, Err}
    end.

add_encounter(PatientId, EncounterData) ->
    EncId = generate_encounter_id(),
    Now   = os:system_time(millisecond),
    Enc   = #encounter{
        id              = EncId,
        patient_id      = PatientId,
        type            = maps:get(type, EncounterData, outpatient),
        provider_id     = maps:get(provider_id, EncounterData),
        start_time      = Now,
        end_time        = undefined,
        chief_complaint = maps:get(chief_complaint, EncounterData, <<>>),
        diagnoses       = maps:get(diagnoses, EncounterData, []),
        status          = active
    },
    {ok, _} = db:execute(
        "INSERT INTO encounters (id,patient_id,type,provider_id,start_time,chief_complaint,status)
         VALUES ($1,$2,$3,$4,$5,$6,$7)",
        [EncId, PatientId, atom_to_binary(Enc#encounter.type),
         Enc#encounter.provider_id, Now,
         Enc#encounter.chief_complaint, <<"active">>]),
    {ok, Enc}.

get_encounters(PatientId, Opts) ->
    Limit  = maps:get(limit, Opts, 20),
    Offset = maps:get(offset, Opts, 0),
    {ok, Rows} = db:query(
        "SELECT id,type,provider_id,start_time,end_time,chief_complaint,status
         FROM encounters WHERE patient_id=$1
         ORDER BY start_time DESC LIMIT $2 OFFSET $3",
        [PatientId, Limit, Offset]),
    {ok, [row_to_encounter(R) || R <- Rows]}.

search_patients(Query) ->
    SearchTerm = <<"%", Query/binary, "%">>,
    {ok, Rows} = db:query(
        "SELECT id,first_name,last_name,dob,mrn FROM patients
         WHERE (first_name ILIKE $1 OR last_name ILIKE $1 OR mrn ILIKE $1)
           AND deleted_at IS NULL
         ORDER BY last_name, first_name LIMIT 20",
        [SearchTerm]),
    {ok, [row_to_patient(R) || R <- Rows]}.

row_to_patient(Row) ->
    #patient{
        id         = maps:get(<<"id">>, Row),
        first_name = maps:get(<<"first_name">>, Row),
        last_name  = maps:get(<<"last_name">>, Row),
        mrn        = maps:get(<<"mrn">>, Row)
    }.

row_to_encounter(Row) ->
    #encounter{
        id              = maps:get(<<"id">>, Row),
        type            = binary_to_atom(maps:get(<<"type">>, Row)),
        chief_complaint = maps:get(<<"chief_complaint">>, Row),
        status          = binary_to_atom(maps:get(<<"status">>, Row))
    }.

generate_patient_id() ->
    base64:encode(crypto:strong_rand_bytes(12)).

generate_encounter_id() ->
    base64:encode(crypto:strong_rand_bytes(12)).

generate_mrn() ->
    N = erlang:unique_integer([positive]),
    iolist_to_binary(io_lib:format("MRN~8..0B", [N rem 100000000])).

format_date({Y, M, D}) ->
    iolist_to_binary(io_lib:format("~4..0B-~2..0B-~2..0B", [Y, M, D])).

update_patient(_PatientId, _Updates) -> ok.
```

---

## 2. Medical Device Integration

```erlang
%% device_monitor.erl — medical device data collection
-module(device_monitor).
-behaviour(gen_server).
-export([start_link/2, get_vitals/1, subscribe_alerts/2]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

-record(vital_signs, {
    timestamp      :: integer(),
    heart_rate     :: float() | undefined,
    blood_pressure :: {systolic(), diastolic()} | undefined,
    spo2           :: float() | undefined,  %% oxygen saturation
    temperature    :: float() | undefined,
    respiratory_rate :: float() | undefined
}).

-type systolic()  :: float().
-type diastolic() :: float().

-record(state, {
    device_id    :: binary(),
    patient_id   :: binary(),
    connection   :: pid() | undefined,
    latest       :: #vital_signs{} | undefined,
    alerts       :: [pid()]
}).

-define(ALERT_THRESHOLDS, #{
    heart_rate     => {40, 130},   %% {min, max} bpm
    spo2           => {90, 100},   %% % oxygen
    systolic_bp    => {80, 180},   %% mmHg
    temperature    => {35.0, 39.5} %% Celsius
}).

start_link(DeviceId, PatientId) ->
    gen_server:start_link(?MODULE, {DeviceId, PatientId}, []).

get_vitals(Pid) ->
    gen_server:call(Pid, get_vitals).

subscribe_alerts(Pid, AlarmPid) ->
    gen_server:cast(Pid, {subscribe, AlarmPid}).

init({DeviceId, PatientId}) ->
    %% Connect to medical device via HL7 or proprietary protocol
    self() ! connect,
    {ok, #state{device_id=DeviceId, patient_id=PatientId,
                alerts=[], latest=undefined}}.

handle_call(get_vitals, _From, #state{latest=Latest} = State) ->
    {reply, Latest, State}.

handle_cast({subscribe, Pid}, #state{alerts=Alerts} = State) ->
    {noreply, State#state{alerts=[Pid | Alerts]}}.

handle_info(connect, #state{device_id=DevId} = State) ->
    case device_protocol:connect(DevId) of
        {ok, ConnPid} ->
            {noreply, State#state{connection=ConnPid}};
        {error, Reason} ->
            logger:warning("Device ~p connection failed: ~p, retrying...", [DevId, Reason]),
            erlang:send_after(5000, self(), connect),
            {noreply, State}
    end;

handle_info({vitals, RawData}, State) ->
    Vitals = parse_vitals(RawData),
    %% Check for alerts
    Alerts = check_thresholds(Vitals),
    [send_alert(Pid, Alerts, State) || Pid <- State#state.alerts, Alerts =/= []],
    %% Store in time-series DB
    store_vitals(State#state.patient_id, Vitals),
    {noreply, State#state{latest=Vitals}};

handle_info({'EXIT', ConnPid, Reason},
            #state{connection=ConnPid} = State) ->
    logger:warning("Device connection lost: ~p, reconnecting", [Reason]),
    erlang:send_after(2000, self(), connect),
    {noreply, State#state{connection=undefined}}.

parse_vitals(#{<<"hr">> := HR, <<"spo2">> := SpO2,
               <<"bp_sys">> := Sys, <<"bp_dia">> := Dia,
               <<"temp">> := Temp} = Data) ->
    #vital_signs{
        timestamp      = os:system_time(millisecond),
        heart_rate     = binary_to_float_safe(HR),
        blood_pressure = {binary_to_float_safe(Sys), binary_to_float_safe(Dia)},
        spo2           = binary_to_float_safe(SpO2),
        temperature    = binary_to_float_safe(Temp),
        respiratory_rate = maps:get(<<"rr">>, Data, undefined)
    }.

check_thresholds(#vital_signs{heart_rate=HR, spo2=SpO2,
                               blood_pressure={Sys, _}, temperature=Temp}) ->
    Checks = [
        {heart_rate, HR, maps:get(heart_rate, ?ALERT_THRESHOLDS)},
        {spo2, SpO2, maps:get(spo2, ?ALERT_THRESHOLDS)},
        {systolic_bp, Sys, maps:get(systolic_bp, ?ALERT_THRESHOLDS)},
        {temperature, Temp, maps:get(temperature, ?ALERT_THRESHOLDS)}
    ],
    [#{vital => V, value => Val, threshold => T}
     || {V, Val, {Min, Max} = T} <- Checks,
        Val =/= undefined,
        (Val < Min orelse Val > Max)].

send_alert(Pid, Alerts, #state{patient_id=PId, device_id=DId}) ->
    Pid ! {clinical_alert, #{
        patient_id => PId,
        device_id  => DId,
        alerts     => Alerts,
        timestamp  => os:system_time(millisecond)
    }}.

store_vitals(PatientId, Vitals) ->
    timeseries:insert(vitals, PatientId, Vitals).

binary_to_float_safe(B) when is_binary(B) ->
    try binary_to_float(B)
    catch _:_ ->
        try float(binary_to_integer(B))
        catch _:_ -> undefined
        end
    end;
binary_to_float_safe(N) when is_number(N) -> float(N);
binary_to_float_safe(_) -> undefined.
```

---

## 3. FHIR API Implementation

```erlang
%% fhir_handler.erl — HL7 FHIR R4 REST API
-module(fhir_handler).
-behaviour(cowboy_handler).
-export([init/2]).

-define(FHIR_VERSION, <<"4.0.1">>).
-define(MIME_FHIR, <<"application/fhir+json">>).

init(Req0, State) ->
    case cowboy_req:method(Req0) of
        <<"GET">>    -> handle_get(Req0, State);
        <<"POST">>   -> handle_post(Req0, State);
        <<"PUT">>    -> handle_put(Req0, State);
        <<"DELETE">> -> handle_delete(Req0, State);
        _            -> reply_fhir(405, error_outcome(<<"method_not_allowed">>), Req0, State)
    end.

handle_get(Req, State) ->
    ResourceType = cowboy_req:binding(resource_type, Req),
    Id           = cowboy_req:binding(id, Req, undefined),
    case {ResourceType, Id} of
        {<<"Patient">>, undefined} ->
            %% Search patients
            Params = cowboy_req:parse_qs(Req),
            case fhir_patient:search(Params) of
                {ok, Bundle} -> reply_fhir(200, Bundle, Req, State);
                {error, E}   -> reply_fhir(400, error_outcome(E), Req, State)
            end;
        {<<"Patient">>, PatientId} ->
            case patient_records:get_patient(PatientId) of
                {ok, Patient}    -> reply_fhir(200, patient_to_fhir(Patient), Req, State);
                {error, not_found} -> reply_fhir(404, error_outcome(<<"not_found">>), Req, State)
            end;
        {<<"Observation">>, undefined} ->
            Params  = cowboy_req:parse_qs(Req),
            PatId   = proplists:get_value(<<"patient">>, Params),
            case observations:search(PatId, Params) of
                {ok, Bundle} -> reply_fhir(200, Bundle, Req, State);
                {error, E}   -> reply_fhir(400, error_outcome(E), Req, State)
            end;
        _ ->
            reply_fhir(404, error_outcome(<<"resource_not_found">>), Req, State)
    end.

handle_post(Req, State) ->
    ResourceType = cowboy_req:binding(resource_type, Req),
    {ok, Body, Req1} = cowboy_req:read_body(Req),
    case jsx:decode(Body, [return_maps]) of
        Resource when is_map(Resource) ->
            case ResourceType of
                <<"Patient">>  ->
                    PatientData = fhir_to_patient_data(Resource),
                    case patient_records:create_patient(PatientData, get_user(Req1)) of
                        {ok, Patient} ->
                            FhirPat = patient_to_fhir(Patient),
                            Req2 = cowboy_req:set_resp_header(
                                <<"location">>,
                                <<"/Patient/", (patient_id(Patient))/binary>>,
                                Req1),
                            reply_fhir(201, FhirPat, Req2, State);
                        {error, E} ->
                            reply_fhir(400, error_outcome(E), Req1, State)
                    end;
                _ ->
                    reply_fhir(400, error_outcome(<<"unsupported_resource">>), Req1, State)
            end;
        _ ->
            reply_fhir(400, error_outcome(<<"invalid_json">>), Req1, State)
    end.

handle_put(Req, State) ->
    reply_fhir(501, error_outcome(<<"not_implemented">>), Req, State).

handle_delete(Req, State) ->
    reply_fhir(405, error_outcome(<<"delete_not_supported">>), Req, State).

patient_to_fhir(#patient{id=Id, first_name=FN, last_name=LN, dob=DOB, mrn=MRN}) ->
    #{
        <<"resourceType">> => <<"Patient">>,
        <<"id">>           => Id,
        <<"identifier">>   => [#{
            <<"system">> => <<"urn:oid:2.16.840.1.113883.4.1">>,
            <<"value">>  => MRN
        }],
        <<"name">> => [#{
            <<"family">> => LN,
            <<"given">>  => [FN]
        }],
        <<"birthDate">> => format_fhir_date(DOB),
        <<"meta">> => #{
            <<"versionId">>   => <<"1">>,
            <<"fhirVersion">> => ?FHIR_VERSION
        }
    }.

fhir_to_patient_data(Resource) ->
    [Name | _] = maps:get(<<"name">>, Resource, [#{}]),
    #{
        last_name  => maps:get(<<"family">>, Name, <<>>),
        first_name => hd(maps:get(<<"given">>, Name, [<<>>]))
    }.

error_outcome(Reason) ->
    #{
        <<"resourceType">> => <<"OperationOutcome">>,
        <<"issue">> => [#{
            <<"severity">>   => <<"error">>,
            <<"code">>       => <<"processing">>,
            <<"diagnostics">> => format_reason(Reason)
        }]
    }.

reply_fhir(Status, Body, Req, State) ->
    Json = jsx:encode(Body),
    Req1 = cowboy_req:reply(Status,
               #{<<"content-type">> => ?MIME_FHIR,
                 <<"x-fhir-version">> => ?FHIR_VERSION},
               Json, Req),
    {ok, Req1, State}.

format_reason(R) when is_binary(R) -> R;
format_reason(R) -> iolist_to_binary(io_lib:format("~p", [R])).

format_fhir_date({Y, M, D}) ->
    iolist_to_binary(io_lib:format("~4..0B-~2..0B-~2..0B", [Y, M, D]));
format_fhir_date(undefined) -> null.

get_user(Req) ->
    cowboy_req:header(<<"x-user-id">>, Req, <<"unknown">>).

patient_id(#patient{id=Id}) -> Id.
```

---

## 4. Clinical Alert System

```erlang
%% clinical_alerts.erl — patient safety alert management
-module(clinical_alerts).
-behaviour(gen_server).
-export([start_link/0, send_alert/2, acknowledge/2, get_active/1]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

-define(ESCALATION_TIMER, 300000).  %% 5 minutes before escalation

-record(alert, {
    id         :: binary(),
    patient_id :: binary(),
    type       :: critical | warning | info,
    message    :: binary(),
    source     :: atom(),
    created_at :: integer(),
    acked_by   :: binary() | undefined,
    acked_at   :: integer() | undefined,
    escalated  :: boolean()
}).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

send_alert(PatientId, AlertData) ->
    gen_server:cast(?MODULE, {alert, PatientId, AlertData}).

acknowledge(AlertId, UserId) ->
    gen_server:call(?MODULE, {ack, AlertId, UserId}).

get_active(PatientId) ->
    gen_server:call(?MODULE, {get_active, PatientId}).

init([]) ->
    Alerts = ets:new(clinical_alerts, [set, named_table,
                                        {keypos, #alert.id}]),
    {ok, #{alerts => Alerts}}.

handle_cast({alert, PatientId, #{type := Type, message := Msg} = Data},
            #{alerts := Alerts} = State) ->
    AlertId = generate_alert_id(),
    Now     = os:system_time(millisecond),
    Alert   = #alert{
        id         = AlertId,
        patient_id = PatientId,
        type       = Type,
        message    = Msg,
        source     = maps:get(source, Data, unknown),
        created_at = Now,
        escalated  = false
    },
    ets:insert(Alerts, Alert),
    %% Notify care team immediately
    notify_care_team(PatientId, Alert),
    %% Set escalation timer for unacknowledged critical alerts
    case Type of
        critical ->
            erlang:send_after(?ESCALATION_TIMER, self(),
                              {escalate, AlertId});
        _ -> ok
    end,
    telemetry:execute([clinical_alert, sent], #{count => 1},
                      #{type => Type, patient => PatientId}),
    {noreply, State};

handle_call({ack, AlertId, UserId}, _From, #{alerts := Alerts} = State) ->
    case ets:lookup(Alerts, AlertId) of
        [Alert] ->
            Now     = os:system_time(millisecond),
            Updated = Alert#alert{acked_by=UserId, acked_at=Now},
            ets:insert(Alerts, Updated),
            audit_log:record(#{action => alert_acknowledged,
                                alert_id => AlertId, user => UserId}),
            {reply, ok, State};
        [] ->
            {reply, {error, not_found}, State}
    end;

handle_call({get_active, PatientId}, _From, #{alerts := Alerts} = State) ->
    Active = ets:match_object(Alerts,
                              #alert{patient_id=PatientId,
                                     acked_by=undefined, _='_'}),
    {reply, {ok, Active}, State}.

handle_info({escalate, AlertId}, #{alerts := Alerts} = State) ->
    case ets:lookup(Alerts, AlertId) of
        [#alert{acked_by=undefined} = Alert] ->
            %% Not acknowledged: escalate to supervisor
            escalate_alert(Alert),
            ets:insert(Alerts, Alert#alert{escalated=true});
        _ -> ok
    end,
    {noreply, State}.

notify_care_team(PatientId, Alert) ->
    %% Look up assigned nurses/doctors
    case care_team:get_on_duty(PatientId) of
        {ok, Team} ->
            [notify_provider(P, Alert) || P <- Team];
        _ -> ok
    end.

notify_provider(#{channel := pager, id := Id}, Alert) ->
    pager_gateway:send(Id, format_alert_message(Alert));
notify_provider(#{channel := ws, pid := Pid}, Alert) ->
    Pid ! {clinical_alert, Alert};
notify_provider(_, _) -> ok.

escalate_alert(Alert) ->
    logger:critical("UNACKNOWLEDGED CRITICAL ALERT after 5 min: ~p", [Alert]),
    on_call_supervisor:page(Alert).

format_alert_message(#alert{type=T, message=Msg, patient_id=PId}) ->
    iolist_to_binary(io_lib:format("[~s] Patient ~s: ~s",
                                   [string:uppercase(atom_to_list(T)), PId, Msg])).

generate_alert_id() ->
    base64:encode(crypto:strong_rand_bytes(9)).
```

---

## 5. HIPAA Audit Logging

```erlang
%% audit_log.erl — HIPAA-compliant immutable audit trail
-module(audit_log).
-export([record/1, search/2, export_range/2]).

-define(AUDIT_TABLE, hipaa_audit_log).

record(#{action := Action, entity := Entity, entity_id := EntityId,
         actor := Actor, timestamp := Ts} = Event) ->
    %% Create tamper-evident entry with hash chain
    PrevHash = get_last_hash(),
    Entry = #{
        id          => generate_audit_id(),
        action      => Action,
        entity      => Entity,
        entity_id   => EntityId,
        actor       => Actor,
        timestamp   => Ts,
        ip_address  => maps:get(ip, Event, undefined),
        user_agent  => maps:get(user_agent, Event, undefined),
        details     => maps:get(details, Event, #{}),
        prev_hash   => PrevHash
    },
    EntryHash = compute_hash(Entry),
    FinalEntry = Entry#{hash => EntryHash},
    %% Insert into DB — immutable (no UPDATE/DELETE allowed)
    {ok, _} = db:execute(
        "INSERT INTO audit_log (id,action,entity,entity_id,actor,
                                timestamp,details,prev_hash,entry_hash)
         VALUES ($1,$2,$3,$4,$5,$6,$7::jsonb,$8,$9)",
        [maps:get(id, FinalEntry),
         atom_to_binary(Action, utf8),
         atom_to_binary(Entity, utf8),
         EntityId, Actor, Ts,
         jsx:encode(maps:get(details, FinalEntry, #{})),
         PrevHash, EntryHash]),
    %% Also log to syslog for offsite copy
    syslog:info("AUDIT ~p ~p ~p by ~p", [Action, Entity, EntityId, Actor]),
    ok.

search(Filters, Opts) ->
    {Conditions, Params} = build_conditions(Filters),
    Limit  = maps:get(limit, Opts, 100),
    Offset = maps:get(offset, Opts, 0),
    SQL    = iolist_to_binary([
        "SELECT id,action,entity,entity_id,actor,timestamp,details ",
        "FROM audit_log WHERE ",
        string:join(Conditions, " AND "),
        " ORDER BY timestamp DESC LIMIT $",
        integer_to_binary(length(Params) + 1),
        " OFFSET $",
        integer_to_binary(length(Params) + 2)
    ]),
    db:query(SQL, Params ++ [Limit, Offset]).

export_range(StartTs, EndTs) ->
    {ok, Rows} = db:query(
        "SELECT * FROM audit_log WHERE timestamp BETWEEN $1 AND $2 ORDER BY timestamp",
        [StartTs, EndTs]),
    %% Verify integrity of exported records
    verify_hash_chain(Rows),
    Rows.

verify_hash_chain([]) -> ok;
verify_hash_chain([_First | Rest]) ->
    verify_hash_chain_pairs(Rest).

verify_hash_chain_pairs([]) -> ok;
verify_hash_chain_pairs([_Entry | Rest]) ->
    %% In a real implementation, verify each entry's prev_hash
    %% matches the hash of the previous entry
    verify_hash_chain_pairs(Rest).

build_conditions(Filters) ->
    maps:fold(fun(actor, V, {Conds, Ps}) ->
        {["actor = $" ++ integer_to_list(length(Ps) + 1) | Conds], [V | Ps]};
    (entity_id, V, {Conds, Ps}) ->
        {["entity_id = $" ++ integer_to_list(length(Ps) + 1) | Conds], [V | Ps]};
    (action, V, {Conds, Ps}) ->
        {["action = $" ++ integer_to_list(length(Ps) + 1) | Conds],
         [atom_to_binary(V) | Ps]};
    (_, _, Acc) -> Acc
    end, {["1=1"], []}, Filters).

compute_hash(Entry) ->
    Data = jsx:encode(maps:without([hash, prev_hash], Entry)),
    crypto:hash(sha256, Data).

get_last_hash() ->
    case db:query("SELECT entry_hash FROM audit_log ORDER BY timestamp DESC LIMIT 1", []) of
        {ok, [#{<<"entry_hash">> := Hash}]} -> Hash;
        _ -> <<0:256>>
    end.

generate_audit_id() ->
    base64:encode(crypto:strong_rand_bytes(16)).
```

---

## 6. Appointment Scheduling

```erlang
%% scheduler.erl — medical appointment scheduling
-module(scheduler).
-export([get_availability/3, book_appointment/4, cancel_appointment/2,
         get_appointments/2]).

get_availability(ProviderId, Date, DurationMins) ->
    %% Get provider's working hours
    {StartHour, EndHour} = get_working_hours(ProviderId, Date),
    %% Get existing appointments
    {ok, Booked} = get_booked_slots(ProviderId, Date),
    %% Calculate available slots
    AllSlots = generate_slots(Date, StartHour, EndHour, DurationMins),
    Available = subtract_booked(AllSlots, Booked, DurationMins),
    {ok, Available}.

book_appointment(PatientId, ProviderId, SlotTime, DurationMins) ->
    %% Pessimistic locking: SELECT FOR UPDATE
    case db:transaction(fun(Conn) ->
        %% Check slot is still available
        {ok, Conflicts} = db:query_in_txn(Conn,
            "SELECT id FROM appointments
             WHERE provider_id = $1
               AND start_time < $2 + ($3 * interval '1 minute')
               AND start_time + (duration_mins * interval '1 minute') > $2
               AND status != 'cancelled'
             FOR UPDATE",
            [ProviderId, format_datetime(SlotTime),
             integer_to_binary(DurationMins)]),
        case Conflicts of
            [] ->
                AppId = generate_appointment_id(),
                db:execute_in_txn(Conn,
                    "INSERT INTO appointments
                     (id,patient_id,provider_id,start_time,duration_mins,status)
                     VALUES ($1,$2,$3,$4,$5,'scheduled')",
                    [AppId, PatientId, ProviderId,
                     format_datetime(SlotTime), DurationMins]),
                {ok, AppId};
            _ ->
                {error, slot_not_available}
        end
    end) of
        {ok, {ok, AppId}} ->
            %% Send reminder
            schedule_reminder(AppId, PatientId, SlotTime),
            {ok, AppId};
        {ok, {error, Reason}} ->
            {error, Reason};
        {error, Reason} ->
            {error, Reason}
    end.

cancel_appointment(AppointmentId, CancelledBy) ->
    Now = os:system_time(millisecond),
    case db:execute(
        "UPDATE appointments SET status='cancelled', cancelled_at=$2, cancelled_by=$3
         WHERE id=$1 AND status='scheduled'",
        [AppointmentId, Now, CancelledBy]) of
        {ok, #{affected_rows := 1}} ->
            audit_log:record(#{action => appointment_cancelled,
                                entity => appointment,
                                entity_id => AppointmentId,
                                actor => CancelledBy,
                                timestamp => Now}),
            ok;
        {ok, #{affected_rows := 0}} ->
            {error, not_found_or_already_cancelled}
    end.

get_appointments(PatientId, Opts) ->
    From  = maps:get(from, Opts, os:system_time(second)),
    Limit = maps:get(limit, Opts, 20),
    db:query(
        "SELECT id,provider_id,start_time,duration_mins,status
         FROM appointments
         WHERE patient_id=$1 AND start_time>=$2 AND status!='cancelled'
         ORDER BY start_time LIMIT $3",
        [PatientId, From, Limit]).

generate_slots(_Date, StartHour, EndHour, DurationMins) ->
    NumSlots = (EndHour - StartHour) * 60 div DurationMins,
    [StartHour * 60 + N * DurationMins
     || N <- lists:seq(0, NumSlots - 1)].

subtract_booked(AllSlots, Booked, DurationMins) ->
    BookedMinutes = [slot_to_minutes(S) || S <- Booked],
    [S || S <- AllSlots,
          not overlaps(S, S + DurationMins, BookedMinutes)].

overlaps(Start, End, BookedList) ->
    lists:any(fun({BS, BE}) -> Start < BE andalso End > BS end, BookedList).

slot_to_minutes(#{start := Start, duration := D}) -> {Start, Start + D}.

get_booked_slots(ProviderId, Date) ->
    db:query("SELECT start_time, duration_mins FROM appointments
              WHERE provider_id=$1 AND start_time::date=$2 AND status!='cancelled'",
             [ProviderId, Date]).

get_working_hours(_ProviderId, _Date) ->
    {8, 17}.  %% 8am to 5pm default

generate_appointment_id() ->
    base64:encode(crypto:strong_rand_bytes(12)).

format_datetime(Ms) ->
    Secs = Ms div 1000,
    Dt   = calendar:gregorian_seconds_to_datetime(
               calendar:datetime_to_gregorian_seconds({{1970,1,1},{0,0,0}}) + Secs),
    {{Y,Mo,D},{H,Mi,S}} = Dt,
    iolist_to_binary(io_lib:format("~4..0B-~2..0B-~2..0B ~2..0B:~2..0B:~2..0B",
                                   [Y,Mo,D,H,Mi,S])).

schedule_reminder(_AppId, _PatientId, _SlotTime) -> ok.
```

---

## 7. แบบฝึกหัด

1. Implement HL7 v2.x parser: parse PID segment จาก HL7 message
2. เพิ่ม medication reconciliation: ตรวจสอบ drug interactions
3. สร้าง telemedicine WebRTC signaling server
4. Implement consent management: track patient consent ตาม HIPAA requirements

---

## สรุป Part 68

✅ Patient Record System (EHR) ด้วย PostgreSQL  
✅ Medical device integration ด้วย vital signs monitoring  
✅ FHIR R4 REST API implementation  
✅ Clinical alert system พร้อม escalation  
✅ HIPAA audit logging ด้วย hash chain  
✅ Appointment scheduling ด้วย optimistic locking  

---

*Part 68/100 | [← ก่อนหน้า](../part67/README.md) | [ถัดไป →](../part69/README.md)*
