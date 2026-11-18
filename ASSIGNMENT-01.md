# Assignment 01. Containerized Order and Restaurant Services

> Source: https://notes.mohibul.dev/courses/cse-4115l-distributed-systems-lab-2026/assignment-01
> Retrieved: 2026-11-18

**Course:** CSE 4115L Distributed Systems Lab (2026)
**Week:** 2
**Marks:** 5 / 40 lab marks
**Dependency:** New system foundation
**Deadline:** Not listed on the page

## Scenario and assignment objective

Customers browse restaurants and create orders. Two isolated services must communicate reliably across an HTTP boundary.

## Required system features

- **Restaurant Service:** `GET /restaurants` and `GET /restaurants/{id}`; seed >= 3 restaurants and >= 6 items.
- **Order Service:** `POST /orders` and `GET /orders/{id}`; validate restaurant and item IDs by calling Restaurant Service.
- Reject bad quantities, missing fields, nonexistent entities and unavailable downstream service with correct HTTP responses.
- Generate unique `order_id` and `correlation_id`; return JSON with `order_id`, `status` and `created_at`.
- Write separate Dockerfiles for both services; use a non-hardcoded `RESTAURANT_URL` env variable.
- Accept >= 20 concurrent requests without corrupting order state; `/healthz` for both services.

## Implementation guide (suggested steps)

1. Create folders `services/order` and `services/restaurant` with separate main entrypoints and a minimal README.
2. Write an API contract, request/response JSON examples and the seed catalogue first.
3. Implement the Restaurant read-only endpoints, then Order validation and an Order in-memory store protected by a mutex (A1 only).
4. Call Restaurant using an HTTP client with a short configured timeout and explicit 4xx/5xx mapping.
5. Write one Dockerfile per service; build images and run containers on a user-defined Docker network.
6. Create automated API tests for success, validation, concurrency and unavailable dependency.

## Hints (not complete solutions)

- The Order process must not call a Restaurant function directly; use an actual HTTP request.
- Run the two services independently with different ports. In Docker, `localhost` inside Order is **not** Restaurant.
- Store order records behind a mutex or use a concurrency-safe map; the state will migrate to PostgreSQL in A2.
- Propagate correlation IDs in requests and logs so a cross-service request is traceable.

## API / data contracts

`POST /orders`

```json
// Request
{
  "restaurant_id": "...",
  "items": [
    { "menu_item_id": "...", "quantity": 1 }
  ]
}

// Response
{
  "order_id": "...",
  "status": "...",
  "created_at": "..."
}
```

Recommended errors:

| Status | Meaning |
|---|---|
| 400 | Malformed input |
| 404 | Restaurant or item missing |
| 503 | Dependency (Restaurant Service) unavailable |

## Mandatory verification and failure experiments

- [ ] 20 concurrent POSTs yield 20 unique IDs, no race/panic.
- [ ] Stop Restaurant; order submission returns a bounded failure (no indefinite hanging).
- [ ] Start both containers without locally installed Go and retrieve created orders.

## Required individual submission

- Individual private repository; service code; two Dockerfiles; README; API examples.
- At least six automated test cases plus example logs of the outage experiment.

## Acceptance checklist

- [ ] Both APIs run independently in containers.
- [ ] Order requests call Restaurant through HTTP and handle failure predictably.
- [ ] Concurrency, request validation and reproducible startup pass.

## Grading rubric (common)

| Component | Weight |
|---|---|
| Implementation | 40% |
| Tests | 25% |
| Fault experiment | 20% |
| Documentation / viva | 15% |
