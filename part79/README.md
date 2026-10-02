# Part 79: Ad Tech Platform

> **"Every ad impression is a real-time auction won in milliseconds"**  
> ทุก ad impression คือการประมูลแบบ real-time ที่ชนะในเวลาไม่กี่มิลลิวินาที

---

## สารบัญ

1. [Ad Tech Architecture](#1-ad-tech-architecture)
2. [Real-Time Bidding Engine](#2-real-time-bidding-engine)
3. [Ad Targeting Engine](#3-ad-targeting-engine)
4. [Impression and Click Tracking](#4-impression-and-click-tracking)
5. [Budget Management](#5-budget-management)
6. [Fraud Detection](#6-fraud-detection)
7. [แบบฝึกหัด](#7-แบบฝึกหัด)

---

## 1. Ad Tech Architecture

```
Real-Time Bidding (RTB) Flow
════════════════════════════════════════════════════════

User visits page
      │ (1ms)
      ▼
Publisher Ad Server → sends BID REQUEST to SSP
      │
      │ (2ms)
      ▼
Supply-Side Platform (SSP) → broadcasts to DSPs
      │
      │ (5ms budget total)
      ▼
Demand-Side Platform (DSP) ← OUR ERLANG SYSTEM
  ├── Targeting match (user segments)
  ├── Budget check (campaign has funds?)
  ├── Bid calculation (CPM, frequency cap)
  └── BID RESPONSE back to SSP
      │
      │ (auction)
      ▼
Winning bid → Ad displayed
  ├── Impression tracked
  ├── Click tracked (if user clicks)
  └── Conversion tracked (if user converts)

ERLANG FIT:
  ✓ Sub-millisecond ETS lookups for user profiles
  ✓ Atomic budget deduction
  ✓ Concurrent bid evaluation
  ✓ High-throughput impression ingestion
  ✓ Process-per-campaign budget management
```

---

## 2. Real-Time Bidding Engine

```erlang
%% rtb_bidder.erl — core RTB bid evaluation
-module(rtb_bidder).
-behaviour(gen_server).

-export([start_link/0, evaluate_bid_request/1]).
-export([init/1, handle_call/3, handle_cast/2]).

-define(BID_TIMEOUT_MS, 80).    % must respond within 80ms
-define(MAX_BID_CPM, 50.0).     % $50 CPM ceiling

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

evaluate_bid_request(BidRequest) ->
    gen_server:call(?MODULE, {bid, BidRequest}, ?BID_TIMEOUT_MS).

init([]) ->
    {ok, #{}}.

handle_call({bid, BidRequest}, _From, State) ->
    StartMs = erlang:monotonic_time(millisecond),
    Result  = do_evaluate(BidRequest),
    Duration = erlang:monotonic_time(millisecond) - StartMs,

    telemetry:execute([rtb, bid, evaluated],
                      #{duration_ms => Duration},
                      #{result => element(1, Result)}),
    {reply, Result, State}.

handle_cast(_Msg, State) -> {noreply, State}.

do_evaluate(#{user_id := UserId, publisher_id := PubId,
              floor_price := FloorCpm} = Request) ->
    %% Step 1: Get user profile (ETS lookup, sub-ms)
    UserProfile = targeting:get_user_profile(UserId),

    %% Step 2: Find matching campaigns
    Candidates = targeting:find_matching_campaigns(UserProfile, PubId),

    %% Step 3: For each candidate, calculate bid
    Bids = lists:filtermap(fun(Campaign) ->
        case calculate_bid(Campaign, UserProfile, Request) of
            {ok, BidCpm} when BidCpm >= FloorCpm ->
                {true, {BidCpm, Campaign}};
            _ ->
                false
        end
    end, Candidates),

    %% Step 4: Select highest bidder
    case lists:sort(fun({A,_}, {B,_}) -> A > B end, Bids) of
        [] ->
            {no_bid, <<>>};
        [{WinBidCpm, WinCampaign} | _] ->
            {bid, #{cpm       => WinBidCpm,
                    campaign  => maps:get(id, WinCampaign),
                    ad_id     => select_ad(WinCampaign, UserProfile),
                    currency  => <<"USD">>}}
    end.

calculate_bid(Campaign, UserProfile, Request) ->
    %% Check budget
    case budget_manager:check_budget(maps:get(id, Campaign)) of
        {ok, available} -> ok;
        _               -> {error, no_budget}
    end,

    %% Check frequency cap
    case frequency_cap:check(maps:get(id, Campaign),
                              maps:get(user_id, Request)) of
        ok -> ok;
        over_cap -> {error, frequency_cap}
    end,

    %% Calculate bid price
    BaseCpm    = maps:get(base_cpm, Campaign, 1.0),
    UserScore  = targeting:score_user(UserProfile, Campaign),
    AdjustedCpm = BaseCpm * UserScore,
    MaxCpm     = min(AdjustedCpm, ?MAX_BID_CPM),

    case MaxCpm > 0 of
        true  -> {ok, MaxCpm};
        false -> {error, zero_bid}
    end.

select_ad(Campaign, _UserProfile) ->
    %% Simple: pick first creative, could be smarter (A/B test creatives)
    Ads = maps:get(ad_ids, Campaign, []),
    case Ads of
        [] -> undefined;
        [Ad | _] -> Ad
    end.
```

---

## 3. Ad Targeting Engine

```erlang
%% targeting.erl — user profile matching and scoring
-module(targeting).
-export([get_user_profile/1, find_matching_campaigns/2, score_user/2]).

-define(PROFILE_CACHE, user_profile_cache).
-define(CAMPAIGN_INDEX, campaign_targeting_index).

get_user_profile(UserId) ->
    case ets:lookup(?PROFILE_CACHE, UserId) of
        [{_, Profile, Exp}] when Exp > erlang:system_time(second) ->
            Profile;
        _ ->
            Profile = load_profile(UserId),
            %% Cache for 5 minutes
            ets:insert(?PROFILE_CACHE,
                       {UserId, Profile, erlang:system_time(second) + 300}),
            Profile
    end.

load_profile(UserId) ->
    %% Aggregate user signals
    #{
        user_id    => UserId,
        segments   => get_segments(UserId),
        interests  => get_interests(UserId),
        demographics => get_demographics(UserId),
        device     => get_device_info(UserId),
        location   => get_location(UserId)
    }.

find_matching_campaigns(UserProfile, PublisherId) ->
    %% Index campaigns by targeting criteria for fast lookup
    Segments   = maps:get(segments, UserProfile, []),
    Interests  = maps:get(interests, UserProfile, []),
    Demographics = maps:get(demographics, UserProfile, #{}),

    %% Get candidate campaigns from ETS index
    SegmentCampaigns  = lookup_by_segments(Segments),
    InterestCampaigns = lookup_by_interests(Interests),

    %% Union all candidates, then filter
    AllCandidates = lists:usort(SegmentCampaigns ++ InterestCampaigns),

    %% Filter: must match all targeting criteria
    lists:filter(fun(Campaign) ->
        matches_targeting(Campaign, UserProfile, PublisherId)
    end, AllCandidates).

matches_targeting(Campaign, UserProfile, PublisherId) ->
    Targeting = maps:get(targeting, Campaign, #{}),

    %% Check each targeting criterion
    check_geo(Targeting, UserProfile) andalso
    check_device(Targeting, UserProfile) andalso
    check_publisher_whitelist(Targeting, PublisherId) andalso
    check_dayparting(Targeting).

check_geo(Targeting, UserProfile) ->
    case maps:get(geo, Targeting, undefined) of
        undefined -> true;
        AllowedCountries ->
            Country = maps:get(country, maps:get(location, UserProfile, #{}), <<>>),
            lists:member(Country, AllowedCountries)
    end.

check_device(Targeting, UserProfile) ->
    case maps:get(device_types, Targeting, undefined) of
        undefined -> true;
        AllowedDevices ->
            Device = maps:get(type, maps:get(device, UserProfile, #{}), unknown),
            lists:member(Device, AllowedDevices)
    end.

check_publisher_whitelist(Targeting, PubId) ->
    case maps:get(publisher_whitelist, Targeting, undefined) of
        undefined -> true;
        Whitelist -> lists:member(PubId, Whitelist)
    end.

check_dayparting(Targeting) ->
    case maps:get(hours, Targeting, undefined) of
        undefined -> true;
        AllowedHours ->
            {H, _, _} = time(),
            lists:member(H, AllowedHours)
    end.

score_user(UserProfile, Campaign) ->
    Segments  = maps:get(segments, UserProfile, []),
    Interests = maps:get(interests, UserProfile, []),
    TargetSegs = maps:get(target_segments, Campaign, []),
    TargetInts = maps:get(target_interests, Campaign, []),

    %% Score = overlap between user and campaign targeting
    SegOverlap = length([S || S <- Segments, lists:member(S, TargetSegs)]),
    IntOverlap = length([I || I <- Interests, lists:member(I, TargetInts)]),

    %% Normalize to 0.5-2.0 multiplier
    BaseScore = 1.0,
    Boost     = (SegOverlap + IntOverlap) * 0.1,
    min(2.0, BaseScore + Boost).

lookup_by_segments(_Segments) -> [].
lookup_by_interests(_Interests) -> [].
get_segments(_) -> [].
get_interests(_) -> [].
get_demographics(_) -> #{}.
get_device_info(_) -> #{type => mobile}.
get_location(_) -> #{country => <<"US">>}.
```

---

## 4. Impression and Click Tracking

```erlang
%% impression_tracker.erl — high-throughput impression pipeline
-module(impression_tracker).
-behaviour(gen_server).

-export([start_link/0, track_impression/1, track_click/1,
         track_conversion/2]).
-export([init/1, handle_cast/2, handle_info/2, handle_call/3]).

-define(FLUSH_INTERVAL, 1000).  % flush every second
-define(BATCH_SIZE, 10000).

-record(state, {
    impressions = [] :: list(),
    clicks      = [] :: list(),
    conversions = [] :: list()
}).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

track_impression(Event) ->
    gen_server:cast(?MODULE, {impression, Event}).

track_click(Event) ->
    gen_server:cast(?MODULE, {click, Event}).

track_conversion(ImpressionId, Value) ->
    gen_server:cast(?MODULE, {conversion, ImpressionId, Value}).

init([]) ->
    schedule_flush(),
    {ok, #state{}}.

handle_cast({impression, Event}, State) ->
    NewEvent = Event#{timestamp => timestamp(), type => impression},
    Impressions = [NewEvent | State#state.impressions],
    case length(Impressions) >= ?BATCH_SIZE of
        true  ->
            flush_impressions(Impressions),
            {noreply, State#state{impressions = []}};
        false ->
            {noreply, State#state{impressions = Impressions}}
    end;

handle_cast({click, Event}, State) ->
    NewEvent = Event#{timestamp => timestamp(), type => click},
    {noreply, State#state{clicks = [NewEvent | State#state.clicks]}};

handle_cast({conversion, ImpressionId, Value}, State) ->
    Event = #{impression_id => ImpressionId, value => Value,
              timestamp => timestamp()},
    {noreply, State#state{conversions = [Event | State#state.conversions]}}.

handle_info(flush, State) ->
    flush_impressions(State#state.impressions),
    flush_clicks(State#state.clicks),
    flush_conversions(State#state.conversions),
    schedule_flush(),
    {noreply, #state{}}.

handle_call(_Req, _From, State) -> {reply, ok, State}.

flush_impressions([]) -> ok;
flush_impressions(Impressions) ->
    %% Write to ClickHouse for analytics
    kafka_producer:produce_batch(<<"impressions">>,
        [json:encode(I) || I <- Impressions]).

flush_clicks([]) -> ok;
flush_clicks(Clicks) ->
    kafka_producer:produce_batch(<<"clicks">>,
        [json:encode(C) || C <- Clicks]).

flush_conversions([]) -> ok;
flush_conversions(Conversions) ->
    kafka_producer:produce_batch(<<"conversions">>,
        [json:encode(C) || C <- Conversions]).

schedule_flush() ->
    erlang:send_after(?FLUSH_INTERVAL, self(), flush).

timestamp() -> erlang:system_time(millisecond).
```

---

## 5. Budget Management

```erlang
%% budget_manager.erl — real-time campaign budget with atomic deduction
-module(budget_manager).
-behaviour(gen_server).

-export([start_link/0, check_budget/1, deduct/2, add_budget/2,
         get_spend/1]).
-export([init/1, handle_call/3, handle_cast/2]).

%% ETS-based budget: atomic operations, no DB roundtrip per bid
-define(BUDGET_TABLE, campaign_budgets).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

check_budget(CampaignId) ->
    case ets:lookup(?BUDGET_TABLE, CampaignId) of
        [{_, Remaining, _DailyLimit, _Spent}] when Remaining > 0 ->
            {ok, available};
        [{_, 0, _, _}] ->
            {error, budget_exhausted};
        [] ->
            {error, campaign_not_found}
    end.

deduct(CampaignId, CostCents) ->
    %% Atomic decrement
    case ets:update_counter(?BUDGET_TABLE, CampaignId,
                            [{2, -CostCents, 0, 0},  % remaining: min floor 0
                             {4, CostCents}],         % spent: increment
                            undefined) of
        undefined -> {error, campaign_not_found};
        [0, _]    -> {error, budget_exhausted};
        [Rem, _]  -> {ok, Rem}
    end.

add_budget(CampaignId, AmountCents) ->
    gen_server:call(?MODULE, {add_budget, CampaignId, AmountCents}).

get_spend(CampaignId) ->
    case ets:lookup(?BUDGET_TABLE, CampaignId) of
        [{_, Remaining, Daily, Spent}] ->
            {ok, #{remaining => Remaining, daily_limit => Daily,
                   spent => Spent}};
        [] ->
            {error, not_found}
    end.

init([]) ->
    ets:new(?BUDGET_TABLE, [named_table, public,
                             {write_concurrency, true}]),
    %% Load active campaigns from DB
    load_campaigns(),
    %% Reset daily budgets at midnight
    schedule_daily_reset(),
    {ok, #{}}.

handle_call({add_budget, CampaignId, Amount}, _From, State) ->
    ets:update_counter(?BUDGET_TABLE, CampaignId, [{2, Amount}]),
    {reply, ok, State}.

handle_cast(_Msg, State) -> {noreply, State}.

load_campaigns() ->
    {ok, Rows} = db:query(
        "SELECT id, daily_budget_cents FROM campaigns WHERE status = 'active'",
        []),
    lists:foreach(fun({Id, DailyBudget}) ->
        ets:insert(?BUDGET_TABLE, {Id, DailyBudget, DailyBudget, 0})
    end, Rows).

schedule_daily_reset() ->
    %% Calculate ms until midnight UTC
    {_Date, {H, M, S}} = calendar:universal_time(),
    SecondsUntilMidnight = (24 * 3600) - (H * 3600 + M * 60 + S),
    erlang:send_after(SecondsUntilMidnight * 1000, self(), reset_daily).
```

---

## 6. Fraud Detection

```erlang
%% fraud_detector.erl — detect invalid clicks and impressions
-module(fraud_detector).
-export([check_impression/1, check_click/1]).

-define(CLICK_RATE_WINDOW, 3600).    % 1 hour
-define(MAX_CLICKS_PER_IP, 10).
-define(MAX_CLICKS_PER_USER, 5).

check_impression(#{ip := IP, user_agent := UA} = Event) ->
    Signals = [
        check_bot_useragent(UA),
        check_datacenter_ip(IP),
        check_impression_flood(IP)
    ],
    case lists:any(fun(S) -> S =/= ok end, Signals) of
        true  -> {invalid, collect_reasons(Signals)};
        false -> {valid, Event}
    end.

check_click(#{ip := IP, user_id := UserId,
              impression_id := ImpressionId} = Event) ->
    Signals = [
        check_click_rate_ip(IP),
        check_click_rate_user(UserId),
        check_impression_exists(ImpressionId),
        check_click_time(Event)
    ],
    case lists:any(fun(S) -> S =/= ok end, Signals) of
        true  -> {invalid, collect_reasons(Signals)};
        false -> {valid, Event}
    end.

check_bot_useragent(UA) ->
    BotPatterns = [<<"bot">>, <<"crawler">>, <<"spider">>,
                   <<"slurp">>, <<"wget">>, <<"curl">>],
    UALower = string:lowercase(UA),
    case lists:any(fun(P) -> binary:match(UALower, P) =/= nomatch end,
                   BotPatterns) of
        true  -> {suspicious, bot_useragent};
        false -> ok
    end.

check_datacenter_ip(IP) ->
    %% Check against known datacenter IP ranges
    case ip_lookup:is_datacenter(IP) of
        true  -> {suspicious, datacenter_ip};
        false -> ok
    end.

check_impression_flood(IP) ->
    Key = {impression_count, IP},
    Count = ets:update_counter(fraud_counters, Key, {2, 1}, {Key, 0}),
    case Count > 1000 of  % More than 1000 impressions/period from same IP
        true  -> {suspicious, impression_flood};
        false -> ok
    end.

check_click_rate_ip(IP) ->
    Key = {click_rate_ip, IP},
    Count = ets:update_counter(fraud_counters, Key, {2, 1}, {Key, 0}),
    case Count > ?MAX_CLICKS_PER_IP of
        true  -> {invalid, ip_click_rate_exceeded};
        false -> ok
    end.

check_click_rate_user(UserId) ->
    Key = {click_rate_user, UserId},
    Count = ets:update_counter(fraud_counters, Key, {2, 1}, {Key, 0}),
    case Count > ?MAX_CLICKS_PER_USER of
        true  -> {invalid, user_click_rate_exceeded};
        false -> ok
    end.

check_impression_exists(ImpressionId) ->
    case impression_store:exists(ImpressionId) of
        true  -> ok;
        false -> {invalid, impression_not_found}
    end.

check_click_time(#{impression_timestamp := ImprTs, timestamp := ClickTs}) ->
    DeltaMs = ClickTs - ImprTs,
    case DeltaMs < 200 of
        true  -> {suspicious, click_too_fast};
        false -> ok
    end;
check_click_time(_) -> ok.

collect_reasons(Signals) ->
    [R || {_, R} <- Signals, is_atom(R)].
```

---

## 7. แบบฝึกหัด

1. Implement second-price auction (Vickrey): winner pays runner-up bid + $0.01
2. เพิ่ม frequency cap ที่ track ต่อ campaign ต่อ user ต่อ day
3. สร้าง pacing algorithm ที่ spread budget ตลอดวันแทนที่จะ spend ทันที
4. Implement viewability tracking: บันทึกว่า ad ถูกมองเห็นจริงหรือไม่

---

## สรุป Part 79

✅ RTB flow: bid request → targeting → bid calculation → response  
✅ Targeting engine: ETS user profiles, campaign index, multi-criteria matching  
✅ Impression tracking: high-throughput buffer, Kafka batch flush  
✅ Budget management: atomic ETS deduction, no DB roundtrip per bid  
✅ Fraud detection: bot UA, IP rate limiting, click timing analysis  

---

*Part 79/100 | [← ก่อนหน้า](../part78/README.md) | [ถัดไป →](../part80/README.md)*
