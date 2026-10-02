# Part 77: Logistics and Supply Chain Systems

> **"A package not delivered is a promise not kept"**  
> พัสดุที่ยังไม่ได้ส่งคือสัญญาที่ยังไม่ได้รักษา

---

## สารบัญ

1. [Logistics Domain Model](#1-logistics-domain-model)
2. [Package Tracking System](#2-package-tracking-system)
3. [Route Optimization](#3-route-optimization)
4. [Warehouse Management](#4-warehouse-management)
5. [Delivery Assignment](#5-delivery-assignment)
6. [Real-Time Driver Tracking](#6-real-time-driver-tracking)
7. [แบบฝึกหัด](#7-แบบฝึกหัด)

---

## 1. Logistics Domain Model

```erlang
%% logistics_types.erl — core domain types
-module(logistics_types).

%% Package states
-type package_status() ::
    created | picked_up | in_transit | at_hub |
    out_for_delivery | delivered | failed_delivery |
    returned | lost.

%% Location with timestamp
-type waypoint() :: #{
    lat       := float(),
    lng       := float(),
    address   := binary(),
    timestamp := integer()
}.

%% Package record
-type package() :: #{
    id           := binary(),
    tracking_no  := binary(),
    sender_id    := binary(),
    recipient    :: #{name := binary(), address := binary(),
                      phone := binary()},
    weight_kg    := float(),
    dimensions   :: #{length := float(), width := float(), height := float()},
    service_type :: standard | express | overnight,
    status       := package_status(),
    waypoints    := [waypoint()],
    estimated_delivery :: integer() | undefined,
    actual_delivery    :: integer() | undefined
}.

%% Delivery route
-type route() :: #{
    id         := binary(),
    driver_id  := binary(),
    vehicle_id := binary(),
    stops      := [stop()],
    status     := planned | in_progress | completed,
    date       := calendar:date()
}.

-type stop() :: #{
    package_id := binary(),
    address    := binary(),
    lat        := float(),
    lng        := float(),
    time_window :: {integer(), integer()} | undefined,
    completed  := boolean()
}.
```

---

## 2. Package Tracking System

```erlang
%% package_tracker.erl — track packages through the delivery pipeline
-module(package_tracker).
-behaviour(gen_server).

-export([start_link/0, create_package/1, update_status/3,
         get_package/1, get_history/1]).
-export([init/1, handle_call/3, handle_cast/2]).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

create_package(Attrs) ->
    gen_server:call(?MODULE, {create, Attrs}).

update_status(PackageId, Status, Location) ->
    gen_server:call(?MODULE, {update_status, PackageId, Status, Location}).

get_package(PackageId) ->
    gen_server:call(?MODULE, {get, PackageId}).

get_history(PackageId) ->
    gen_server:call(?MODULE, {history, PackageId}).

init([]) ->
    {ok, #{}}.

handle_call({create, Attrs}, _From, State) ->
    PackageId  = generate_id(),
    TrackingNo = generate_tracking_no(),
    Package = Attrs#{
        id           => PackageId,
        tracking_no  => TrackingNo,
        status       => created,
        waypoints    => [],
        created_at   => timestamp()
    },
    ok = db_packages:insert(Package),
    %% Publish event
    event_bus:publish(package_events, #{
        type       => package_created,
        package_id => PackageId,
        tracking   => TrackingNo
    }),
    {reply, {ok, Package}, State};

handle_call({update_status, PackageId, Status, Location}, _From, State) ->
    case db_packages:get(PackageId) of
        {ok, Package} ->
            Waypoint = Location#{timestamp => timestamp()},
            Updated  = Package#{
                status    => Status,
                waypoints => maps:get(waypoints, Package) ++ [Waypoint]
            },
            ok = db_packages:update(PackageId, Updated),
            notify_recipient(Package, Status),
            event_bus:publish(package_events, #{
                type       => status_updated,
                package_id => PackageId,
                status     => Status,
                location   => Location
            }),
            {reply, ok, State};
        {error, _} = Err ->
            {reply, Err, State}
    end;

handle_call({get, PackageId}, _From, State) ->
    Result = db_packages:get(PackageId),
    {reply, Result, State};

handle_call({history, PackageId}, _From, State) ->
    case db_packages:get(PackageId) of
        {ok, #{waypoints := Waypoints}} ->
            {reply, {ok, Waypoints}, State};
        Err -> {reply, Err, State}
    end.

notify_recipient(Package, delivered) ->
    #{recipient := #{phone := Phone}} = Package,
    sms_service:send(Phone, <<"Your package has been delivered!">>);
notify_recipient(Package, failed_delivery) ->
    #{recipient := #{phone := Phone}} = Package,
    sms_service:send(Phone, <<"Delivery failed. We will retry tomorrow.">>);
notify_recipient(_, _) -> ok.

generate_id() -> base64:encode(crypto:strong_rand_bytes(12)).
generate_tracking_no() ->
    N = integer_to_binary(rand:uniform(999999999)),
    <<"TK", N/binary>>.
timestamp() -> erlang:system_time(millisecond).
```

---

## 3. Route Optimization

```erlang
%% route_optimizer.erl — nearest-neighbor TSP approximation
-module(route_optimizer).
-export([optimize_route/2, estimate_duration/1]).

-define(SPEED_KMH, 30).   % average urban delivery speed

%% Optimize order of stops using nearest-neighbor heuristic
optimize_route(DepotLocation, Stops) ->
    case Stops of
        [] -> [];
        _  ->
            {Route, _} = nearest_neighbor(DepotLocation, Stops, []),
            add_estimated_times(Route, DepotLocation)
    end.

nearest_neighbor(_Current, [], Visited) ->
    {lists:reverse(Visited), 0};
nearest_neighbor(Current, Remaining, Visited) ->
    %% Find nearest unvisited stop
    {Nearest, Rest} = find_nearest(Current, Remaining),
    nearest_neighbor(
        #{lat => maps:get(lat, Nearest), lng => maps:get(lng, Nearest)},
        Rest,
        [Nearest | Visited]
    ).

find_nearest(Current, [First | Rest]) ->
    InitDist = haversine(Current, First),
    lists:foldl(fun(Stop, {BestStop, BestRest}) ->
        D = haversine(Current, Stop),
        case D < haversine(Current, BestStop) of
            true  ->
                NewRest = [BestStop | lists:delete(Stop, BestRest)],
                {Stop, NewRest};
            false ->
                {BestStop, BestRest}
        end
    end, {First, Rest}, Rest).

%% Haversine distance formula (km)
haversine(A, B) ->
    Lat1 = deg_to_rad(maps:get(lat, A)),
    Lat2 = deg_to_rad(maps:get(lat, B)),
    DLat = deg_to_rad(maps:get(lat, B) - maps:get(lat, A)),
    DLng = deg_to_rad(maps:get(lng, B) - maps:get(lng, A)),
    H = math:sin(DLat/2) * math:sin(DLat/2) +
        math:cos(Lat1) * math:cos(Lat2) *
        math:sin(DLng/2) * math:sin(DLng/2),
    6371.0 * 2 * math:atan2(math:sqrt(H), math:sqrt(1 - H)).

deg_to_rad(Deg) -> Deg * math:pi() / 180.

add_estimated_times(Stops, Depot) ->
    {WithTimes, _} = lists:foldl(fun(Stop, {Acc, PrevLoc}) ->
        DistKm  = haversine(PrevLoc, Stop),
        DriveMins = round(DistKm / ?SPEED_KMH * 60),
        PrevTime = case Acc of
            [] -> erlang:system_time(second);
            [Prev | _] -> maps:get(eta, Prev, erlang:system_time(second))
        end,
        Eta = PrevTime + DriveMins * 60 + 300,  % +5 min delivery time
        {[Stop#{eta => Eta, drive_mins => DriveMins} | Acc],
         #{lat => maps:get(lat, Stop), lng => maps:get(lng, Stop)}}
    end, {[], Depot}, Stops),
    lists:reverse(WithTimes).

estimate_duration(Stops) ->
    lists:sum([maps:get(drive_mins, S, 0) || S <- Stops]).
```

---

## 4. Warehouse Management

```erlang
%% warehouse.erl — inventory location management
-module(warehouse).
-behaviour(gen_server).

-export([start_link/1, receive_package/2, pick_package/1,
         get_location/1, scan_barcode/2]).
-export([init/1, handle_call/3, handle_cast/2]).

%% Warehouse grid: aisle-bay-level (e.g., A-12-3)
-type location() :: {Aisle :: char(), Bay :: integer(), Level :: integer()}.

-record(state, {
    warehouse_id :: binary(),
    inventory    :: ets:tid()   % package_id → location
}).

start_link(WarehouseId) ->
    gen_server:start_link({local, wh_name(WarehouseId)}, ?MODULE,
                          [WarehouseId], []).

receive_package(WarehouseId, PackageId) ->
    gen_server:call(wh_name(WarehouseId), {receive, PackageId}).

pick_package(PackageId) ->
    %% Find which warehouse has this package
    gen_server:call(inventory_lookup, {pick, PackageId}).

get_location(PackageId) ->
    gen_server:call(inventory_lookup, {location, PackageId}).

scan_barcode(WarehouseId, Barcode) ->
    gen_server:call(wh_name(WarehouseId), {scan, Barcode}).

init([WarehouseId]) ->
    Tid = ets:new(inventory, [private]),
    {ok, #state{warehouse_id = WarehouseId, inventory = Tid}}.

handle_call({receive, PackageId}, _From, State) ->
    Location = assign_location(State),
    ets:insert(State#state.inventory, {PackageId, Location}),
    package_tracker:update_status(PackageId, at_hub, #{
        warehouse_id => State#state.warehouse_id,
        location     => format_location(Location)
    }),
    {reply, {ok, Location}, State};

handle_call({scan, Barcode}, _From, State) ->
    %% Handle barcode scan — could be pickup or drop-off
    case ets:lookup(State#state.inventory, Barcode) of
        [{_, Location}] ->
            {reply, {found, Barcode, Location}, State};
        [] ->
            {reply, {not_found, Barcode}, State}
    end.

handle_cast(_Msg, State) -> {noreply, State}.

assign_location(#state{inventory = Inv}) ->
    %% Simple: find an empty location
    Used = [L || {_, L} <- ets:tab2list(Inv)],
    find_empty_location(Used).

find_empty_location(Used) ->
    Aisles = lists:seq($A, $H),
    Bays   = lists:seq(1, 20),
    Levels = lists:seq(1, 5),
    Candidates = [{A, B, L} || A <- Aisles, B <- Bays, L <- Levels],
    Available  = Candidates -- Used,
    hd(Available).

format_location({Aisle, Bay, Level}) ->
    iolist_to_binary([Aisle, integer_to_binary(Bay), "-",
                      integer_to_binary(Level)]).

wh_name(Id) -> binary_to_atom(<<"warehouse_", Id/binary>>).
```

---

## 5. Delivery Assignment

```erlang
%% delivery_assigner.erl — assign packages to drivers/routes
-module(delivery_assigner).
-export([assign_daily_routes/1, assign_package/2]).

assign_daily_routes(Date) ->
    %% Get all packages ready for delivery today
    {ok, Packages} = db_packages:get_ready_for_delivery(Date),

    %% Group by delivery zone
    Zones = group_by_zone(Packages),

    %% For each zone, assign to an available driver and optimize route
    Routes = maps:fold(fun(Zone, ZonePackages, Acc) ->
        case find_available_driver(Zone) of
            {ok, Driver} ->
                Stops   = packages_to_stops(ZonePackages),
                Optimized = route_optimizer:optimize_route(Driver, Stops),
                Route = #{
                    id        => generate_route_id(),
                    driver_id => maps:get(id, Driver),
                    zone      => Zone,
                    stops     => Optimized,
                    date      => Date,
                    status    => planned
                },
                ok = db_routes:insert(Route),
                notify_driver(Driver, Route),
                [Route | Acc];
            {error, no_driver} ->
                logger:warning("No driver for zone ~p", [Zone]),
                Acc
        end
    end, [], Zones),
    {ok, Routes}.

group_by_zone(Packages) ->
    lists:foldl(fun(Pkg, Zones) ->
        Zone = get_delivery_zone(maps:get(recipient, Pkg)),
        maps:update_with(Zone, fun(Pkgs) -> [Pkg | Pkgs] end, [Pkg], Zones)
    end, #{}, Packages).

packages_to_stops(Packages) ->
    [#{
        package_id => maps:get(id, P),
        address    => maps:get(address, maps:get(recipient, P)),
        lat        => maps:get(lat, P, 0.0),
        lng        => maps:get(lng, P, 0.0),
        completed  => false
    } || P <- Packages].

assign_package(PackageId, DriverId) ->
    %% Add to existing route for driver
    {ok, Route} = db_routes:get_active_route(DriverId),
    {ok, Package} = db_packages:get(PackageId),
    Stop = #{package_id => PackageId,
             address => maps:get(address, maps:get(recipient, Package)),
             completed => false},
    db_routes:add_stop(maps:get(id, Route), Stop).

find_available_driver(_Zone) -> {error, no_driver}.
get_delivery_zone(_Recipient) -> default_zone.
notify_driver(_Driver, _Route) -> ok.
generate_route_id() -> base64:encode(crypto:strong_rand_bytes(8)).
```

---

## 6. Real-Time Driver Tracking

```erlang
%% driver_tracker.erl — real-time GPS position tracking
-module(driver_tracker).
-behaviour(gen_server).

-export([start_link/0, update_position/3, get_position/1,
         get_nearby_drivers/3]).
-export([init/1, handle_call/3, handle_cast/2]).

-define(POSITION_TTL, 300).  % 5 minutes

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

update_position(DriverId, Lat, Lng) ->
    gen_server:cast(?MODULE, {update, DriverId, Lat, Lng}).

get_position(DriverId) ->
    gen_server:call(?MODULE, {get, DriverId}).

get_nearby_drivers(Lat, Lng, RadiusKm) ->
    gen_server:call(?MODULE, {nearby, Lat, Lng, RadiusKm}).

init([]) ->
    ets:new(driver_positions, [named_table, public,
                               {read_concurrency, true}]),
    {ok, #{}}.

handle_cast({update, DriverId, Lat, Lng}, State) ->
    Now = erlang:system_time(second),
    ets:insert(driver_positions, {DriverId, Lat, Lng, Now}),
    %% Broadcast to any observers (e.g., customer tracking WebSocket)
    event_bus:publish(driver_events, #{
        type      => position_updated,
        driver_id => DriverId,
        lat => Lat, lng => Lng,
        timestamp => Now
    }),
    {noreply, State}.

handle_call({get, DriverId}, _From, State) ->
    Now = erlang:system_time(second),
    Result = case ets:lookup(driver_positions, DriverId) of
        [{_, Lat, Lng, UpdatedAt}]
          when Now - UpdatedAt < ?POSITION_TTL ->
            {ok, #{lat => Lat, lng => Lng, updated_at => UpdatedAt}};
        [{_, _, _, _}] ->
            {error, stale_position};
        [] ->
            {error, not_found}
    end,
    {reply, Result, State};

handle_call({nearby, Lat, Lng, RadiusKm}, _From, State) ->
    Now = erlang:system_time(second),
    All = ets:tab2list(driver_positions),
    Nearby = [
        #{driver_id => Id, lat => DLat, lng => DLng,
          distance_km => Dist}
        || {Id, DLat, DLng, UpdatedAt} <- All,
           Now - UpdatedAt < ?POSITION_TTL,
           (Dist = route_optimizer:haversine(#{lat => Lat, lng => Lng},
                                              #{lat => DLat, lng => DLng})) =< RadiusKm
    ],
    Sorted = lists:sort(fun(A, B) ->
        maps:get(distance_km, A) < maps:get(distance_km, B)
    end, Nearby),
    {reply, {ok, Sorted}, State}.

handle_cast(_Msg, State) -> {noreply, State}.
```

---

## 7. แบบฝึกหัด

1. เพิ่ม ETA recalculation เมื่อ driver ติดรถติด (update ด้วย real speed)
2. Implement package clustering algorithm แทน nearest-neighbor
3. สร้าง WebSocket endpoint สำหรับ customer ติดตาม package แบบ real-time
4. เพิ่ม proof-of-delivery: บันทึก photo + signature เมื่อส่งสำเร็จ

---

## สรุป Part 77

✅ Logistics domain model: package, route, stop types  
✅ Package tracking: status FSM, waypoint history, SMS notifications  
✅ Route optimization: nearest-neighbor heuristic, Haversine distance  
✅ Warehouse management: grid-based location assignment, barcode scanning  
✅ Delivery assignment: zone grouping, driver matching, route planning  
✅ Real-time GPS tracking: ETS positions, nearby driver search, event broadcast  

---

*Part 77/100 | [← ก่อนหน้า](../part76/README.md) | [ถัดไป →](../part78/README.md)*
