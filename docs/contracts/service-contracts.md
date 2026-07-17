# Service Contracts

## Contract Rules

1. Public HTTP routes are documented in OpenAPI before implementation.
2. Internal HTTP routes are private, separately documented, and require service identity.
3. RabbitMQ payloads are documented in AsyncAPI and validated on publish and consume.
4. API and event schemas use runtime validation. TypeScript types alone are insufficient.
5. Breaking changes create a new version; they do not silently modify an existing contract.
6. All timestamps are UTC ISO-8601 strings. All identifiers are UUIDs. Money is a decimal string at HTTP/event boundaries and `NUMERIC(12,2)` in PostgreSQL.

## Public API Rules

All authenticated caller identity is derived from a verified JWT subject, never from a path/body user ID or a gateway-injected identity header. Object endpoints enforce ownership in the owning service.

Mutating endpoints return a correlation ID and require request validation. `POST /api/v1/txn/init` requires a high-entropy `Idempotency-Key` header. Retrying the same key for the same user, payment method, cart version, and normalized item set returns the original result.

Standard application errors use:

```json
{
  "error": {
    "code": "ORDER_EXPIRED",
    "message": "The payment window has expired.",
    "correlationId": "uuid"
  }
}
```

Important codes include `UNAUTHENTICATED`, `FORBIDDEN`, `NOT_FOUND`, `VALIDATION_FAILED`, `IDEMPOTENCY_CONFLICT`, `PRICE_CHANGED`, `ALREADY_OWNED`, `ORDER_EXPIRED`, `PAYMENT_UNAVAILABLE`, and `DEPENDENCY_UNAVAILABLE`.

## Checkout Quote

Commerce calls catalog before creating an order. The catalog quote response contains the game ID, title, sellability, currency, unit price, discount, quote ID, price version, and expiry. Commerce persists this response as immutable order-item snapshots and never accepts a total, price, discount, or currency from the browser.

The display batch endpoint is not a checkout authority and cannot be used to price an order.

## Payment Simulator Contract

The simulator is a server-selected payment-provider adapter available only when the environment is explicitly configured as `demo` or `test`. The public UI may request only `paid` or `failed` outcomes; delayed callbacks and expiry are controlled integration-test fixtures.

The buyer-facing simulation request may operate only on the authenticated caller's own `payment_pending` simulator order. It causes the server-side simulator to send a provider-shaped signed callback to the normal commerce webhook handler. It never updates the order directly and never accepts a client-selected amount or order status.

Every callback includes a unique provider event ID, timestamp, order reference, merchant ID, payment method, EGP amount, currency, and terminal status. Commerce verifies the exact raw body HMAC before parsing, checks timestamp freshness, deduplicates provider event IDs, and validates every correlation field against the locked local order.

## Domain Events

RabbitMQ is the sole transport for payment, entitlement, and release events. The required initial event is `commerce.order.paid.v1`.

```json
{
  "eventId": "uuid",
  "eventType": "commerce.order.paid.v1",
  "schemaVersion": 1,
  "occurredAt": "2026-07-15T12:00:00Z",
  "correlationId": "uuid",
  "producer": "commerce-service",
  "payload": {
    "orderId": "uuid",
    "userId": "uuid",
    "items": [
      {
        "gameId": "uuid",
        "titleSnapshot": "Example Game",
        "pricePaidEgp": "150.00",
        "currency": "EGP"
      }
    ]
  }
}
```

The library publishes `library.entitlement.granted.v1` only after its license transaction commits. Every event has an immutable `eventId`, versioned type, and validated payload.
