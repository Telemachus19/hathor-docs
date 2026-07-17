# Payment And Entitlement Reliability

## Guarantee

Hathor provides at-least-once event delivery and exactly-once entitlement effect per source event. A paid order is recorded durably before publication, and a duplicate webhook or event cannot grant duplicate licenses.

## Order State Machine

```text
created -> payment_pending -> payment_confirmed -> fulfillment_pending -> fulfilled
payment_pending -> expired | payment_failed | cancelled
fulfilled -> revoked
```

Transitions are conditional, monotonic updates inside PostgreSQL transactions. Each transition is appended to `order_state_transitions`. An expiry worker handles overdue `payment_pending` orders but cannot override a confirmed payment.

## Commerce Transaction

For a verified terminal callback:

```text
BEGIN
  lock order
  reject invalid state or mismatched payment facts
  insert unique payment event receipt
  update order to payment_confirmed then fulfillment_pending
  append state-transition records
  insert commerce.order.paid.v1 into outbox_events
COMMIT
```

The outbox worker reads pending rows, publishes persistent messages to RabbitMQ with publisher confirms, and marks a row published only after confirmation. A crash may cause a later duplicate publish; this is safe by design.

## Library Transaction

For `commerce.order.paid.v1`:

```text
BEGIN
  insert eventId into processed_events
  if duplicate, commit successfully without re-granting
  insert all licenses with source_order_id and fulfillment_event_id
  append entitlement audit records
  insert library.entitlement.granted.v1 into local outbox
COMMIT
ACK RabbitMQ only after commit
```

The commerce service consumes the grant acknowledgment idempotently and moves the order from `fulfillment_pending` to `fulfilled`.

## Retry And Recovery

Consumers retry transient failures five times with exponential backoff through a bounded retry queue. Malformed or repeatedly failing messages move to a DLQ. DLQ messages are not discarded; the runbook requires diagnosis, correction, and controlled replay using the same event ID.

A reconciliation job finds:

1. Confirmed or fulfillment-pending orders with no matching license source order.
2. Licenses with no valid source order.
3. Unpublished outbox rows past the allowed delay.
4. Acknowledged payment events with no terminal order transition.

Reconciliation runs daily, creates an alert, and uses idempotent repair/replay actions. It never inserts uncorrelated licenses manually.

## Release-Blocking Failure Tests

1. Duplicate checkout initiation returns the original order.
2. Duplicate, stale, invalid, or mismatched callbacks do not complete an order.
3. Commerce restart after payment commit still yields a later entitlement.
4. RabbitMQ outage retains the outbox record and later publishes it.
5. Library crash after commit and before acknowledgment creates one license after redelivery.
6. Duplicate event delivery creates one license set.
7. A poison event reaches the DLQ and is safely replayed.
8. Reconciliation detects a completed order with no entitlement.
