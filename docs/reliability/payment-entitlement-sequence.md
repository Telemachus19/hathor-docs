# Payment To Entitlement Sequence

## Successful Simulator Purchase

```text
Gamer -> Gateway: POST /txn/init + Idempotency-Key
Gateway -> Commerce: authenticated request
Commerce -> Catalog: internal quote (scoped service token)
Catalog --> Commerce: sellable price snapshot
Commerce -> Library: internal ownership check (scoped service token)
Library --> Commerce: owned game IDs
Commerce -> Commerce DB: lock cart, snapshot order, create payment_pending order
Commerce --> Gamer: order ID, branded simulator instructions, expiry

Gamer -> Gateway: POST /txn/{orderId}/simulate-payment
Gateway -> Commerce: authenticated owner request
Commerce -> Simulator Adapter: create server-side provider callback
Simulator Adapter -> Gateway: POST /txn/webhooks/simulator + raw-body HMAC
Gateway -> Commerce: raw body preserved
Commerce -> Commerce DB: lock order, deduplicate event, verify amount/reference/state,
                           confirm payment, write outbox event
Commerce --> Simulator Adapter: idempotent success

Outbox Worker -> RabbitMQ: commerce.order.paid.v1 + publisher confirm
RabbitMQ -> Library: durable order-paid message
Library -> Library DB: inbox insert, license insert, audit, local outbox
Library -> RabbitMQ: acknowledge only after commit
Library -> RabbitMQ: library.entitlement.granted.v1
RabbitMQ -> Commerce: durable grant acknowledgement
Commerce -> Commerce DB: fulfillment_pending -> fulfilled
Gamer -> Gateway: GET /inventory/apps
Gateway -> Library: authenticated owner request
Library --> Gamer: license visible
```

## Failure Rules

| Failure | Required behavior |
| --- | --- |
| Duplicate checkout request | Return the original order for the same user/idempotency key/cart version/input set |
| Catalog or library check unavailable | Fail checkout with retryable `DEPENDENCY_UNAVAILABLE`; do not create a payable order |
| Invalid/late/replayed callback | Store verification evidence where safe, reject completion, and do not write outbox event |
| Commerce commits but RabbitMQ is unavailable | Outbox record remains pending and retries later |
| Commerce crashes after broker confirmation | Event may be republished; library inbox deduplicates it |
| Library crashes before DB commit | RabbitMQ redelivers the unacknowledged message |
| Library crashes after DB commit before ACK | RabbitMQ redelivers; `processed_events.event_id` deduplicates it |
| Five consumer failures | Route to DLQ, alert, diagnose, and replay with original event ID |
| Daily reconciliation finds missing license | Re-drive the existing source event/outbox record; never manually insert an uncorrelated license |

## Simulator Rules

The simulator endpoint is enabled only in `demo`/`test` environments. It accepts only the authenticated order owner, only a non-expired `payment_pending` order, and only a simulator payment method. The UI may select `paid` or `failed`, but cannot supply a price, reference, callback time, or arbitrary state. Its generated callback traverses the same HMAC, deduplication, and state-transition code path used by a real provider adapter. Delayed callbacks and expiry remain controlled test fixtures.
