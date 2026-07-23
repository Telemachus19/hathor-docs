# Service Data Model

## Rules

Each service owns its schema and migrations. All service databases use UTC `TIMESTAMPTZ`, generated UUID primary keys where appropriate, and explicit constraints for financial and state fields. Cross-service IDs are validated through contracts; they are not database foreign keys.

## Auth Service

`users` stores email, normalized email, password hash, display name, account state, authorization version, and timestamps. Public registration always assigns `gamer`.

`refresh_token_families` and `refresh_tokens` store only token hashes, family linkage, expiry, revocation state, reuse detection data, and timestamps. A refresh-token reuse revokes the entire family.

`role_change_audit` stores actor ID, target user ID, granted/revoked role, prior and new authorization version, correlation ID, and timestamp. The one-time bootstrap command creates the initial admin and writes the same audit record.

## Catalog Service

`games` stores creator ID, title, slug, descriptions, status, price, currency, discount, approved theme document, and timestamps. New games default to `draft`.

`game_status_transitions` records actor, prior status, next status, reason, correlation ID, and timestamp.

`theme_revisions` stores the creator-owned validated theme document, creator/AI origin metadata, parent revision, acceptance timestamp, and draft status. AI proposals are not persisted as active revisions until the creator explicitly accepts them.

`catalog_recommendation_cache` stores bounded, non-personalized recommendation sets keyed by optional source game context, ranked published game IDs, short explainable reasons, source (`cached` or `curated_fallback`), and refresh timestamp. It contains catalog metadata only and does not store payment, entitlement, account, or behavioral-profile data.

`builds` stores game ID, semantic version, platform, private immutable object key, manifest SHA-256, state, creation/publish/revocation timestamps, and creator/administrator actor IDs. A published game references one explicit published build for the seeded MVP.

## Commerce Service

`cart_items` is authoritative PostgreSQL cart state. It includes cart version or updated timestamp to support checkout snapshotting.

`orders` stores user ID, state, payment method, payment reference, provider transaction ID, idempotency key, total amount, currency, expiry, and state timestamps. Idempotency keys are unique within a user scope.

`order_items` stores immutable game ID, title snapshot, catalog quote ID, catalog price version, unit price, discount snapshot, currency, and line total.

`payment_events` stores unique provider event ID, order ID, payload hash, signature outcome, provider timestamp, processing result, and received time.

`order_state_transitions` is append-only and records state changes, actor/system, reason, correlation ID, and timestamp.

`outbox_events` stores a unique event ID, event type/version, payload, publish attempts, status, last error, and published timestamp.

Required constraints include:

- Amounts are non-negative.
- Currency is `EGP` for this release.
- Discount is between 0 and 100.
- Order, payment, game, and build state values use PostgreSQL enums or CHECK constraints.
- Provider event IDs are unique per provider.
- Payment references are unique but are not the sole security correlation identifier.

## Library Service

`processed_events` is the inbox ledger. `event_id` is unique and records event type, correlation ID, and processing timestamp.

`user_licenses` stores user ID, game ID, source order ID, fulfillment event ID, paid price/currency, acquired timestamp, and optional revocation fields. It prevents duplicate game ownership while retaining the payment/event evidence needed for reconciliation.

`entitlement_audit` records grants, revocations, download authorization, actor/system, correlation ID, and timestamps.

## Reconciliation Queries

The services expose or schedule reconciliation queries for:

1. Payment-confirmed orders without a published outbox event.
2. Fulfillment-pending orders without a license source order after the allowed delay.
3. Licenses without a valid source order/event.
4. Revoked games/builds with newly issued download authorizations.
