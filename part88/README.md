# Part 88: gRPC and High-Performance Binary APIs

> **"Binary protocols speak louder than words — they just speak more efficiently"**  
> Binary protocol พูดดังกว่า text — แต่พูดได้มีประสิทธิภาพกว่ามาก

---

## สารบัญ

1. [gRPC vs REST vs GraphQL](#1-grpc-vs-rest-vs-graphql)
2. [Protocol Buffers Definition](#2-protocol-buffers-definition)
3. [gRPC Server in Erlang](#3-grpc-server-in-erlang)
4. [Streaming RPCs](#4-streaming-rpcs)
5. [gRPC Client](#5-grpc-client)
6. [Interceptors](#6-interceptors)
7. [แบบฝึกหัด](#7-แบบฝึกหัด)

---

## 1. gRPC vs REST vs GraphQL

```
API Protocol Comparison
═══════════════════════════════════════════════════════════

                REST        GraphQL     gRPC
Protocol:       HTTP/1.1    HTTP/1.1    HTTP/2
Format:         JSON        JSON        Protobuf (binary)
Schema:         OpenAPI     SDL         .proto file
Streaming:      SSE/WS      Subscript.  Native (bi-di)
Type safety:    Optional    Strong      Strong
Browser:        Native      Native      grpc-web only
Latency:        Medium      Medium      Low (binary, multiplex)
Throughput:     Medium      Medium      High
Code gen:       Optional    Optional    Required
Discovery:      Manual      Introspect  Reflection API

Use gRPC when:
  - Service-to-service (not browser)
  - High throughput / low latency (>10k RPS)
  - Strongly typed contracts critical
  - Bidirectional streaming needed
  - Polyglot environment (Go + Erlang + Python)

Erlang gRPC library: grpc (hex.pm/packages/grpc)
```

---

## 2. Protocol Buffers Definition

```protobuf
// order_service.proto
syntax = "proto3";

package orders.v1;
option erlang_package = "order_service";

// Core message types
message Order {
  string id = 1;
  string customer_id = 2;
  repeated OrderItem items = 3;
  OrderStatus status = 4;
  int64 total_cents = 5;
  int64 created_at = 6;  // Unix timestamp
}

message OrderItem {
  string product_id = 1;
  int32 quantity = 2;
  int64 unit_price_cents = 3;
}

enum OrderStatus {
  ORDER_STATUS_UNSPECIFIED = 0;
  ORDER_STATUS_PENDING = 1;
  ORDER_STATUS_CONFIRMED = 2;
  ORDER_STATUS_SHIPPED = 3;
  ORDER_STATUS_DELIVERED = 4;
  ORDER_STATUS_CANCELLED = 5;
}

// RPC request/response messages
message CreateOrderRequest {
  string customer_id = 1;
  repeated OrderItem items = 2;
}

message GetOrderRequest {
  string order_id = 1;
}

message ListOrdersRequest {
  string customer_id = 1;
  int32 page_size = 2;
  string page_token = 3;
}

message ListOrdersResponse {
  repeated Order orders = 1;
  string next_page_token = 2;
}

message OrderEvent {
  string order_id = 1;
  OrderStatus new_status = 2;
  int64 timestamp = 3;
}

// Service definition
service OrderService {
  // Unary RPCs
  rpc CreateOrder(CreateOrderRequest) returns (Order);
  rpc GetOrder(GetOrderRequest) returns (Order);
  rpc ListOrders(ListOrdersRequest) returns (ListOrdersResponse);

  // Server streaming: subscribe to order status updates
  rpc WatchOrderStatus(GetOrderRequest) returns (stream OrderEvent);

  // Client streaming: batch create orders
  rpc BatchCreateOrders(stream CreateOrderRequest) returns (ListOrdersResponse);

  // Bidirectional streaming: real-time order processing
  rpc ProcessOrders(stream CreateOrderRequest) returns (stream OrderEvent);
}
```

---

## 3. gRPC Server in Erlang

```erlang
%% order_service_server.erl — gRPC server implementation
-module(order_service_server).
-behaviour(grpc_server).

%% grpc_server callbacks
-export([create_order/2, get_order/2, list_orders/2,
         watch_order_status/2, batch_create_orders/2, process_orders/2]).

%% Unary RPC: CreateOrder
create_order(#{customer_id := CustomerId, items := Items}, _Stream) ->
    case orders:create(CustomerId, Items) of
        {ok, Order} ->
            {ok, order_to_proto(Order)};
        {error, invalid_customer} ->
            {error, {not_found, <<"Customer not found">>}};
        {error, insufficient_inventory} ->
            {error, {failed_precondition, <<"Insufficient inventory">>}};
        {error, Reason} ->
            {error, {internal, io_lib:format("~p", [Reason])}}
    end.

%% Unary RPC: GetOrder
get_order(#{order_id := OrderId}, _Stream) ->
    case orders:get(OrderId) of
        {ok, Order} ->
            {ok, order_to_proto(Order)};
        {error, not_found} ->
            {error, {not_found, <<"Order not found">>}}
    end.

%% Unary RPC: ListOrders with pagination
list_orders(#{customer_id := CustomerId, page_size := PageSize,
              page_token := PageToken}, _Stream) ->
    Opts = #{
        limit      => max(1, min(PageSize, 100)),
        page_token => decode_page_token(PageToken)
    },
    case orders:list_by_customer(CustomerId, Opts) of
        {ok, OrderList, NextToken} ->
            {ok, #{
                orders           => [order_to_proto(O) || O <- OrderList],
                next_page_token  => encode_page_token(NextToken)
            }};
        {error, Reason} ->
            {error, {internal, io_lib:format("~p", [Reason])}}
    end.

%% Server streaming RPC: watch order status changes
watch_order_status(#{order_id := OrderId}, Stream) ->
    %% Subscribe to order events
    {ok, Sub} = order_events:subscribe(OrderId),
    stream_events(OrderId, Sub, Stream).

stream_events(OrderId, Sub, Stream) ->
    receive
        {order_event, OrderId, Status, Timestamp} ->
            Event = #{
                order_id   => OrderId,
                new_status => status_to_proto(Status),
                timestamp  => Timestamp
            },
            case grpc_stream:send(Stream, Event) of
                ok     -> stream_events(OrderId, Sub, Stream);
                closed -> order_events:unsubscribe(Sub)
            end;
        {status_change, closed} ->
            order_events:unsubscribe(Sub),
            ok
    after 30000 ->
        %% Send keepalive
        grpc_stream:send(Stream, #{order_id => OrderId,
                                   new_status => 0,
                                   timestamp => erlang:system_time(second)}),
        stream_events(OrderId, Sub, Stream)
    end.

%% Proto conversion helpers
order_to_proto(Order) ->
    #{
        id          => maps:get(id, Order),
        customer_id => maps:get(customer_id, Order),
        items       => [item_to_proto(I) || I <- maps:get(items, Order, [])],
        status      => status_to_proto(maps:get(status, Order)),
        total_cents => maps:get(total_cents, Order, 0),
        created_at  => maps:get(created_at, Order, 0)
    }.

item_to_proto(Item) ->
    #{
        product_id       => maps:get(product_id, Item),
        quantity         => maps:get(quantity, Item),
        unit_price_cents => maps:get(unit_price_cents, Item)
    }.

status_to_proto(pending)   -> 1;
status_to_proto(confirmed) -> 2;
status_to_proto(shipped)   -> 3;
status_to_proto(delivered) -> 4;
status_to_proto(cancelled) -> 5;
status_to_proto(_)         -> 0.

decode_page_token(<<>>) -> undefined;
decode_page_token(Token) -> base64:decode(Token).

encode_page_token(undefined) -> <<>>;
encode_page_token(Token)     -> base64:encode(Token).

batch_create_orders(_Request, _Stream) -> {ok, #{orders => []}}.
process_orders(_Request, _Stream) -> ok.
```

---

## 4. Streaming RPCs

```erlang
%% grpc_streaming.erl — bidirectional streaming example
-module(grpc_streaming).
-export([process_orders/2]).

%% Bidirectional streaming: receive orders, emit events in real time
process_orders(InitialMsg, Stream) ->
    %% Process first message
    Result = handle_order_request(InitialMsg),
    grpc_stream:send(Stream, Result),
    streaming_loop(Stream).

streaming_loop(Stream) ->
    case grpc_stream:recv(Stream) of
        {ok, Msg} ->
            Result = handle_order_request(Msg),
            grpc_stream:send(Stream, Result),
            streaming_loop(Stream);
        {error, closed} ->
            ok;
        {error, Reason} ->
            logger:warning("Stream error: ~p", [Reason]),
            ok
    end.

handle_order_request(#{customer_id := _CustId, items := Items}) ->
    %% Validate and process each order
    OrderId = generate_order_id(),
    #{
        order_id   => OrderId,
        new_status => 1,  %% pending
        timestamp  => erlang:system_time(second)
    }.

generate_order_id() ->
    binary:encode_hex(crypto:strong_rand_bytes(8)).
```

---

## 5. gRPC Client

```erlang
%% order_client.erl — gRPC client for order service
-module(order_client).
-export([start/0, create_order/2, get_order/1, watch_order/2]).

-define(TARGET, "orders-service:50051").

start() ->
    grpc_client:connect(?TARGET, [
        {transport, tls},
        {tls_options, [{verify, verify_peer},
                       {cacertfile, "/etc/ssl/certs/ca-bundle.crt"}]}
    ]).

create_order(CustomerId, Items) ->
    Req = #{customer_id => CustomerId, items => Items},
    grpc_client:call(order_service, 'CreateOrder', Req).

get_order(OrderId) ->
    grpc_client:call(order_service, 'GetOrder', #{order_id => OrderId}).

%% Server streaming with callback
watch_order(OrderId, CallbackFun) ->
    Req = #{order_id => OrderId},
    {ok, Stream} = grpc_client:stream(order_service, 'WatchOrderStatus', Req),
    collect_stream(Stream, CallbackFun).

collect_stream(Stream, Fun) ->
    case grpc_client:recv(Stream) of
        {ok, Event} ->
            Fun(Event),
            collect_stream(Stream, Fun);
        {error, closed} ->
            ok;
        {error, Reason} ->
            {error, Reason}
    end.
```

---

## 6. Interceptors

```erlang
%% grpc_interceptors.erl — server interceptors for cross-cutting concerns
-module(grpc_interceptors).
-export([auth_interceptor/3, logging_interceptor/3, metrics_interceptor/3]).

%% Authentication interceptor
auth_interceptor(Method, Request, Handler) ->
    case grpc_stream:get_metadata(<<"authorization">>) of
        undefined ->
            {error, {unauthenticated, <<"Missing authorization">>}};
        <<"Bearer ", Token/binary>> ->
            case jwt_auth:verify(Token) of
                {ok, Claims} ->
                    grpc_stream:set_context(claims, Claims),
                    Handler(Method, Request);
                {error, _} ->
                    {error, {unauthenticated, <<"Invalid token">>}}
            end;
        _ ->
            {error, {unauthenticated, <<"Invalid authorization format">>}}
    end.

%% Structured logging interceptor
logging_interceptor(Method, Request, Handler) ->
    Start = erlang:monotonic_time(microsecond),
    logger:info("gRPC request", #{method => Method}),
    Result = Handler(Method, Request),
    DurationUs = erlang:monotonic_time(microsecond) - Start,
    Status = case Result of {ok, _} -> ok; {error, _} -> error end,
    logger:info("gRPC response", #{
        method      => Method,
        status      => Status,
        duration_us => DurationUs
    }),
    Result.

%% Metrics interceptor
metrics_interceptor(Method, Request, Handler) ->
    Start  = erlang:monotonic_time(millisecond),
    Result = Handler(Method, Request),
    DurMs  = erlang:monotonic_time(millisecond) - Start,
    Status = case Result of {ok, _} -> <<"ok">>; _ -> <<"error">> end,
    telemetry:execute(
        [grpc, request],
        #{duration_ms => DurMs},
        #{method => Method, status => Status}
    ),
    Result.
```

---

## 7. แบบฝึกหัด

1. Implement client-side load balancing สำหรับ gRPC: round-robin across multiple endpoints
2. เพิ่ม deadline propagation: ถ้า client ให้ deadline 2s ให้ cancel RPC เมื่อหมดเวลา
3. สร้าง gRPC health check service ตาม grpc.health.v1 spec
4. Implement retry interceptor: retry idempotent calls on UNAVAILABLE status

---

## สรุป Part 88

✅ Protocol comparison: เลือก gRPC เมื่อ service-to-service, high throughput  
✅ Protobuf schema: messages, enums, service definition with streaming  
✅ Server implementation: unary, server streaming, bidirectional  
✅ Client: unary calls, server streaming with callback  
✅ Interceptors: auth, logging, metrics เป็น chain pattern  

---

*Part 88/100 | [← ก่อนหน้า](../part87/README.md) | [ถัดไป →](../part89/README.md)*
