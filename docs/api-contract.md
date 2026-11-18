# API Contract: Restaurant & Order Services (Assignment 01)

This document is the single source of truth for both services. Code and tests are
written to match it, not the other way round.

## 1. Services at a glance

| Service    | Default port | Purpose                                   | State                         |
|------------|--------------|-------------------------------------------|-------------------------------|
| Restaurant | `8081`       | Read-only catalogue of restaurants + menus | Seeded in memory at startup   |
| Order      | `8082`       | Create and fetch orders                    | In-memory map behind a mutex  |

Order depends on Restaurant **over HTTP only**. Restaurant depends on nothing.

```
client ──POST /orders──▶ Order ──GET /restaurants/{id}──▶ Restaurant
       ◀── 201 / 4xx / 503 ──    ◀──── 200 / 404 / (down) ────
```

## 2. Common conventions

- **Content type:** all request and response bodies are `application/json; charset=utf-8`.
- **Correlation ID:** header `X-Correlation-ID`.
  - If the client sends one, it is reused; otherwise Order generates a UUID v4.
  - Order forwards it to Restaurant on the validation call.
  - Both services echo it in the response header and include it in every log line.
- **IDs:** restaurants and menu items use readable string IDs (`rest-1`, `item-101`).
  Orders use a random **UUID v4** (e.g. `3f2b8c1e-9d4a-4c7e-8b1f-2a6d5e9c0b47`).
- **Money:** prices are **integers in BDT (taka)**. No floats, so no rounding errors.
- **Time:** `created_at` is RFC 3339 in UTC, e.g. `2026-11-19T10:15:30Z`.

### Error body (all non-2xx responses)

```json
{
  "error": "items[0].quantity must be between 1 and 100",
  "code": "VALIDATION_FAILED",
  "correlation_id": "7c1e2f4a-0b3d-4e5f-9a8b-6c7d8e9f0a1b"
}
```

| HTTP | `code`                   | When                                                        | Service    |
|------|--------------------------|-------------------------------------------------------------|------------|
| 400  | `INVALID_JSON`           | Body is not valid JSON, has unknown fields, or wrong types  | Order      |
| 400  | `VALIDATION_FAILED`      | Missing/empty field, or quantity outside 1–100              | Order      |
| 404  | `RESTAURANT_NOT_FOUND`   | Restaurant ID does not exist                                | Both       |
| 404  | `MENU_ITEM_NOT_FOUND`    | Menu item is not on that restaurant's menu                  | Order      |
| 404  | `ORDER_NOT_FOUND`        | Order ID does not exist                                     | Order      |
| 503  | `RESTAURANT_UNAVAILABLE` | Restaurant unreachable, timed out, or returned 5xx          | Order      |
| 500  | `INTERNAL_ERROR`         | Unexpected server bug (should never happen in tests)        | Both       |

## 3. Restaurant Service (`:8081`)

### `GET /restaurants`: list all restaurants

**200 OK**
```json
{
  "restaurants": [
    {
      "id": "rest-1",
      "name": "Dhaka Biryani House",
      "cuisine": "Bangladeshi",
      "menu": [
        { "id": "item-101", "name": "Kacchi Biryani", "price": 350 },
        { "id": "item-102", "name": "Chicken Roast",  "price": 220 },
        { "id": "item-103", "name": "Borhani",        "price": 60 }
      ]
    }
  ]
}
```
(The real response contains all three seeded restaurants, ordered by ID.)

### `GET /restaurants/{id}`: one restaurant with its menu

**200 OK**: a single restaurant object, same shape as one element above.

**404 Not Found**
```json
{ "error": "restaurant rest-99 not found", "code": "RESTAURANT_NOT_FOUND", "correlation_id": "..." }
```

### `GET /healthz`

**200 OK**: `{ "status": "ok", "service": "restaurant" }`

## 4. Order Service (`:8082`)

### `POST /orders`: create an order

**Request**
```json
{
  "restaurant_id": "rest-1",
  "items": [
    { "menu_item_id": "item-101", "quantity": 2 },
    { "menu_item_id": "item-103", "quantity": 1 }
  ]
}
```

**Validation rules** (checked in this order, first failure wins):

| # | Rule                                                        | Failure            |
|---|-------------------------------------------------------------|--------------------|
| 1 | Body is valid JSON, no unknown fields, correct types, ≤ 1 MB | 400 `INVALID_JSON` |
| 2 | `restaurant_id` is present and non-empty                    | 400 `VALIDATION_FAILED` |
| 3 | `items` has at least one entry                              | 400 `VALIDATION_FAILED` |
| 4 | Every `items[i].menu_item_id` is non-empty                  | 400 `VALIDATION_FAILED` |
| 5 | Every `items[i].quantity` is an integer from 1 to 100       | 400 `VALIDATION_FAILED` |
| 6 | Restaurant exists (Order calls `GET /restaurants/{id}`)     | 404 `RESTAURANT_NOT_FOUND` |
| 7 | Every `menu_item_id` is on that restaurant's menu           | 404 `MENU_ITEM_NOT_FOUND` |
| - | Restaurant down / timeout (default 2 s) / 5xx               | 503 `RESTAURANT_UNAVAILABLE` |

Rules 1–5 are checked **locally, before** any network call, so bad input never
reaches Restaurant.

**201 Created** (header `Location: /orders/{order_id}`)
```json
{
  "order_id": "3f2b8c1e-9d4a-4c7e-8b1f-2a6d5e9c0b47",
  "correlation_id": "7c1e2f4a-0b3d-4e5f-9a8b-6c7d8e9f0a1b",
  "status": "CONFIRMED",
  "restaurant_id": "rest-1",
  "items": [
    { "menu_item_id": "item-101", "name": "Kacchi Biryani", "quantity": 2, "unit_price": 350 },
    { "menu_item_id": "item-103", "name": "Borhani",        "quantity": 1, "unit_price": 60 }
  ],
  "total_price": 760,
  "created_at": "2026-11-19T10:15:30Z"
}
```
`name` and `unit_price` are copied from Restaurant **at order time**, so a later
price change does not alter an existing order.

### `GET /orders/{id}`: fetch an order

**200 OK**: the same object returned by `POST /orders`.
**404 Not Found**: `ORDER_NOT_FOUND`.

### `GET /healthz`

**200 OK**: `{ "status": "ok", "service": "order" }`
This is a **liveness** check: it does not call Restaurant, so Order reports healthy
even while Restaurant is down (Order is still running and can return 503s correctly).

## 5. Configuration (environment variables)

| Variable             | Service    | Default        | Notes                                           |
|----------------------|------------|----------------|-------------------------------------------------|
| `PORT`               | both       | `8081` / `8082`| Port the HTTP server listens on                 |
| `RESTAURANT_URL`     | Order      | none (required)| e.g. `http://localhost:8081` locally, `http://restaurant:8081` in Docker. Order exits at startup if unset. |
| `RESTAURANT_TIMEOUT` | Order      | `2s`           | Go duration; upper bound on the validation call |

## 6. Seed catalogue (3 restaurants, 8 menu items)

| Restaurant ID | Name                 | Cuisine     | Item ID    | Item           | Price (BDT) |
|---------------|----------------------|-------------|------------|----------------|-------------|
| `rest-1`      | Dhaka Biryani House  | Bangladeshi | `item-101` | Kacchi Biryani | 350 |
|               |                      |             | `item-102` | Chicken Roast  | 220 |
|               |                      |             | `item-103` | Borhani        | 60  |
| `rest-2`      | Chittagong Mezban    | Bangladeshi | `item-201` | Mezbani Beef   | 400 |
|               |                      |             | `item-202` | Kala Bhuna     | 380 |
|               |                      |             | `item-203` | Plain Rice     | 40  |
| `rest-3`      | Old Town Fuchka      | Street Food | `item-301` | Fuchka (8 pcs) | 80  |
|               |                      |             | `item-302` | Chotpoti       | 70  |

Menu item IDs are globally unique, but an item is only valid for **its own**
restaurant: ordering `item-201` from `rest-1` returns 404 `MENU_ITEM_NOT_FOUND`.
