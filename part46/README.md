# Part 46: Real-World E-Commerce System

> **"Build for real scale — then optimize for actual load"**  
> สร้างสำหรับ scale จริง — แล้วค่อย optimize สำหรับ load จริง

---

## สารบัญ

1. [System Architecture](#1-system-architecture)
2. [Product Catalog](#2-product-catalog)
3. [Shopping Cart](#3-shopping-cart)
4. [Order Processing](#4-order-processing)
5. [Inventory Management](#5-inventory-management)
6. [Payment Flow](#6-payment-flow)
7. [แบบฝึกหัด](#7-แบบฝึกหัด)

---

## 1. System Architecture

```
E-Commerce Architecture:

  Browser / Mobile
        |
   [API Gateway]  ← rate limit, auth, routing
        |
  ┌─────┼────────────────────────────┐
  │     │                            │
[Catalog]  [Cart Service]  [Order Service]
  │            │                   │
  ETS       Redis-like           [Inventory]
 + DB        ETS               [Payment GW]
                                [Shipping]
```

---

## 2. Product Catalog

```erlang
%% catalog.erl — product catalog with search and caching
-module(catalog).
-export([list/1, get/1, search/2, update_stock/2]).

-define(CACHE_TTL, 300).

list(#{page := Page, per_page := PerPage, category := Cat}) ->
    CacheKey = {list, Page, PerPage, Cat},
    case ttl_cache:get(catalog_cache, CacheKey) of
        {ok, Cached} -> {ok, Cached};
        miss ->
            {ok, Products} = db:query(
                "SELECT id, name, price, stock, category "
                "FROM products WHERE ($1::text IS NULL OR category = $1) "
                "ORDER BY name LIMIT $2 OFFSET $3",
                [Cat, PerPage, (Page-1)*PerPage]
            ),
            Enriched = [enrich(P) || P <- Products],
            ttl_cache:put(catalog_cache, CacheKey, Enriched, ?CACHE_TTL),
            {ok, Enriched}
    end.

get(ProductId) ->
    case ttl_cache:get(catalog_cache, {product, ProductId}) of
        {ok, P} -> {ok, P};
        miss ->
            case db:query(
                "SELECT * FROM products WHERE id = $1", [ProductId]) of
                {ok, [P | _]} ->
                    Enriched = enrich(P),
                    ttl_cache:put(catalog_cache, {product, ProductId},
                                  Enriched, ?CACHE_TTL),
                    {ok, Enriched};
                {ok, []} -> {error, not_found}
            end
    end.

search(Query, Opts) ->
    Limit = maps:get(limit, Opts, 20),
    {ok, Results} = db:query(
        "SELECT id, name, price, ts_rank(search_vector, query) rank "
        "FROM products, plainto_tsquery('english', $1) query "
        "WHERE search_vector @@ query "
        "ORDER BY rank DESC LIMIT $2",
        [Query, Limit]
    ),
    {ok, Results}.

update_stock(ProductId, Delta) ->
    {ok, _} = db:execute(
        "UPDATE products SET stock = stock + $2 "
        "WHERE id = $1 AND stock + $2 >= 0 "
        "RETURNING stock",
        [ProductId, Delta]
    ),
    ttl_cache:invalidate(catalog_cache, {product, ProductId}).

enrich(#{<<"id">> := Id} = P) ->
    P#{
        url      => <<"/api/products/", (integer_to_binary(Id))/binary>>,
        in_stock => maps:get(<<"stock">>, P, 0) > 0
    }.
```

---

## 3. Shopping Cart

```erlang
%% cart.erl — session-based shopping cart using ETS
-module(cart).
-export([get/1, add/3, remove/2, update_qty/3, clear/1, checkout_data/1]).

-define(CART_TABLE, carts).
-define(CART_TTL, 3600 * 24).   %% 24 hours

get(SessionId) ->
    case ets:lookup(?CART_TABLE, SessionId) of
        [{SessionId, Items, _Expires}] -> {ok, Items};
        [] -> {ok, []}
    end.

add(SessionId, ProductId, Qty) ->
    {ok, Items} = get(SessionId),
    NewItems = case lists:keyfind(ProductId, 1, Items) of
        false ->
            [{ProductId, Qty} | Items];
        {ProductId, ExistingQty} ->
            lists:keyreplace(ProductId, 1, Items,
                             {ProductId, ExistingQty + Qty})
    end,
    save(SessionId, NewItems).

remove(SessionId, ProductId) ->
    {ok, Items} = get(SessionId),
    NewItems = lists:keydelete(ProductId, 1, Items),
    save(SessionId, NewItems).

update_qty(SessionId, ProductId, NewQty) when NewQty =< 0 ->
    remove(SessionId, ProductId);
update_qty(SessionId, ProductId, NewQty) ->
    {ok, Items} = get(SessionId),
    NewItems = lists:keyreplace(ProductId, 1, Items, {ProductId, NewQty}),
    save(SessionId, NewItems).

clear(SessionId) ->
    ets:delete(?CART_TABLE, SessionId).

checkout_data(SessionId) ->
    {ok, Items} = get(SessionId),
    ProductIds = [Id || {Id, _} <- Items],
    %% Load product details for checkout
    Products   = [catalog:get(Id) || Id <- ProductIds],
    LineItems  = [#{
        product_id => Id,
        quantity   => Qty,
        price      => maps:get(<<"price">>, P),
        subtotal   => Qty * maps:get(<<"price">>, P)
    } || {{Id, Qty}, {ok, P}} <- lists:zip(Items, Products)],
    Total = lists:sum([maps:get(subtotal, L) || L <- LineItems]),
    {ok, #{items => LineItems, total => Total}}.

save(SessionId, Items) ->
    Expires = os:system_time(second) + ?CART_TTL,
    ets:insert(?CART_TABLE, {SessionId, Items, Expires}),
    {ok, Items}.
```

---

## 4. Order Processing

```erlang
%% order_service.erl — create and manage orders
-module(order_service).
-export([create/2, get/1, list/1, cancel/2, update_status/3]).

create(UserId, SessionId) ->
    case cart:checkout_data(SessionId) of
        {ok, #{items := [], total := 0}} ->
            {error, empty_cart};
        {ok, #{items := Items, total := Total}} ->
            %% Create order in transaction
            case db:transaction(fun() ->
                {ok, #{<<"id">> := OrderId}} = db:execute(
                    "INSERT INTO orders (user_id, total, status) "
                    "VALUES ($1, $2, 'pending') RETURNING id",
                    [UserId, Total]),
                [db:execute(
                    "INSERT INTO order_items "
                    "(order_id, product_id, quantity, price) "
                    "VALUES ($1, $2, $3, $4)",
                    [OrderId,
                     maps:get(product_id, Item),
                     maps:get(quantity, Item),
                     maps:get(price, Item)])
                 || Item <- Items],
                {ok, OrderId}
            end) of
                {ok, OrderId} ->
                    cart:clear(SessionId),
                    %% Trigger order processing saga
                    order_saga:start(#{id => OrderId, items => Items,
                                       user_id => UserId, total => Total}),
                    {ok, OrderId};
                {error, _} = E -> E
            end
    end.

get(OrderId) ->
    case db:query(
        "SELECT o.*, "
        "  json_agg(row_to_json(i.*)) as items "
        "FROM orders o "
        "JOIN order_items i ON i.order_id = o.id "
        "WHERE o.id = $1 "
        "GROUP BY o.id",
        [OrderId]) of
        {ok, [Order | _]} -> {ok, Order};
        {ok, []}          -> {error, not_found}
    end.

list(UserId) ->
    db:query(
        "SELECT id, total, status, created_at "
        "FROM orders WHERE user_id = $1 "
        "ORDER BY created_at DESC LIMIT 50",
        [UserId]).

cancel(OrderId, UserId) ->
    case db:execute(
        "UPDATE orders SET status = 'cancelled' "
        "WHERE id = $1 AND user_id = $2 AND status = 'pending' "
        "RETURNING id",
        [OrderId, UserId]) of
        {ok, [_]} ->
            %% Release inventory reservations
            inventory:release_order(OrderId),
            {ok, cancelled};
        {ok, []} ->
            {error, cannot_cancel}
    end.

update_status(OrderId, Status, _Actor) ->
    db:execute(
        "UPDATE orders SET status = $2, updated_at = NOW() "
        "WHERE id = $1",
        [OrderId, Status]).
```

---

## 5. Inventory Management

```erlang
%% inventory.erl — stock management with reservations
-module(inventory).
-export([check/2, reserve/2, release/2, release_order/1, commit/2]).

-define(RESERVE_TTL, 900).  %% 15 minutes

check(ProductId, Qty) ->
    case db:query(
        "SELECT stock FROM products WHERE id = $1", [ProductId]) of
        {ok, [#{<<"stock">> := Stock}]} when Stock >= Qty -> ok;
        {ok, [#{<<"stock">> := Stock}]} -> {error, {insufficient_stock, Stock}};
        {ok, []} -> {error, product_not_found}
    end.

reserve(OrderId, Items) ->
    %% Check and reserve all items atomically
    db:transaction(fun() ->
        Results = [reserve_item(OrderId, maps:get(product_id, I),
                                maps:get(quantity, I))
                   || I <- Items],
        case lists:all(fun(ok) -> true; (_) -> false end, Results) of
            true  -> ok;
            false ->
                %% Rollback handled by transaction
                error(reservation_failed)
        end
    end).

reserve_item(OrderId, ProductId, Qty) ->
    case db:execute(
        "UPDATE products SET stock = stock - $3, reserved = reserved + $3 "
        "WHERE id = $2 AND stock >= $3 "
        "RETURNING id",
        [OrderId, ProductId, Qty]) of
        {ok, [_]} -> ok;
        {ok, []}  -> {error, {product, ProductId, insufficient_stock}}
    end.

release(OrderId, Items) ->
    [db:execute(
        "UPDATE products SET stock = stock + $3, reserved = reserved - $3 "
        "WHERE id = $2",
        [OrderId, maps:get(product_id, I), maps:get(quantity, I)])
     || I <- Items].

release_order(OrderId) ->
    db:execute(
        "UPDATE products p "
        "SET stock = stock + oi.quantity, reserved = reserved - oi.quantity "
        "FROM order_items oi "
        "WHERE oi.order_id = $1 AND p.id = oi.product_id",
        [OrderId]).

commit(OrderId, Items) ->
    [db:execute(
        "UPDATE products SET reserved = reserved - $2 "
        "WHERE id = $1",
        [maps:get(product_id, I), maps:get(quantity, I)])
     || I <- Items].
```

---

## 6. Payment Flow

```erlang
%% payment_service.erl — payment processing
-module(payment_service).
-export([process/3, refund/2, get/1]).

process(OrderId, Amount, #{method := Method} = PaymentInfo) ->
    case charge_provider(Method, Amount, PaymentInfo) of
        {ok, ProviderRef} ->
            {ok, _} = db:execute(
                "INSERT INTO payments "
                "(order_id, amount, status, provider_ref, method) "
                "VALUES ($1, $2, 'completed', $3, $4) "
                "RETURNING id",
                [OrderId, Amount, ProviderRef, atom_to_binary(Method, utf8)]),
            order_service:update_status(OrderId, <<"paid">>, system),
            %% Commit inventory (stock already decremented, just release reservation)
            {ok, Items} = order_service:get_items(OrderId),
            inventory:commit(OrderId, Items),
            {ok, ProviderRef};
        {error, declined} ->
            db:execute(
                "INSERT INTO payments "
                "(order_id, amount, status, method) "
                "VALUES ($1, $2, 'declined', $3)",
                [OrderId, Amount, atom_to_binary(Method, utf8)]),
            order_service:update_status(OrderId, <<"payment_failed">>, system),
            inventory:release_order(OrderId),
            {error, payment_declined}
    end.

charge_provider(credit_card, Amount, #{card_token := Token}) ->
    %% Call Stripe/Braintree etc
    case stripe:charge(Token, Amount) of
        {ok, Ref} -> {ok, Ref};
        {error, _} = E -> E
    end;

charge_provider(paypal, Amount, #{paypal_order_id := PayPalId}) ->
    paypal:capture(PayPalId, Amount).

refund(PaymentId, Amount) ->
    {ok, Payment} = get(PaymentId),
    case refund_provider(Payment, Amount) of
        {ok, RefundRef} ->
            db:execute(
                "INSERT INTO refunds (payment_id, amount, provider_ref) "
                "VALUES ($1, $2, $3)",
                [PaymentId, Amount, RefundRef]),
            {ok, RefundRef};
        {error, _} = E -> E
    end.

get(PaymentId) ->
    case db:query(
        "SELECT * FROM payments WHERE id = $1", [PaymentId]) of
        {ok, [P | _]} -> {ok, P};
        {ok, []}      -> {error, not_found}
    end.

refund_provider(#{<<"provider_ref">> := Ref, <<"method">> := <<"credit_card">>}, Amount) ->
    stripe:refund(Ref, Amount);
refund_provider(_, _) ->
    {error, refund_not_supported}.
```

---

## 7. แบบฝึกหัด

1. เพิ่ม promo code system: `promo:apply/2` ลด discount จาก cart total
2. Implement order fulfillment queue: ส่ง order ไปยัง warehouse worker
3. เขียน EUnit test สำหรับ `checkout_data/1` ด้วย mock catalog
4. เพิ่ม webhook: ส่ง POST ไปยัง merchant URL เมื่อ order status เปลี่ยน

---

## สรุป Part 46

✅ Product catalog ด้วย full-text search  
✅ Session-based shopping cart  
✅ Order processing ด้วย DB transactions  
✅ Inventory management ด้วย reservations  
✅ Payment flow ด้วย provider abstraction  

---

*Part 46/100 | [← ก่อนหน้า](../part45/README.md) | [ถัดไป →](../part47/README.md)*
