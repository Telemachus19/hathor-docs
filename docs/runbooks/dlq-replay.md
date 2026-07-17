# RabbitMQ DLQ Replay Runbook

## Trigger

An alert reports a non-zero DLQ depth for a payment or entitlement queue.

## Procedure

1. Freeze automated replay for the affected DLQ.
2. Identify the event ID, event type, source order, correlation ID, retry history, and failure class from structured logs.
3. Confirm the original commerce order and outbox record are valid before replaying.
4. Correct the consumer/dependency/schema defect. Do not alter the original event payload or event ID.
5. Replay the message through the approved retry/replay path.
6. Verify the library `processed_events` and `user_licenses.source_order_id` records after processing.
7. Verify commerce receives an entitlement acknowledgment or reconciliation marks the order fulfilled.
8. Document the incident and root cause.

## Prohibited Actions

- Do not acknowledge or discard a DLQ message without an incident record.
- Do not create a license directly to hide an event failure.
- Do not replay a payment event with a new event ID.
- Do not replay an event if its source order is invalid, revoked, or not payment-confirmed.
