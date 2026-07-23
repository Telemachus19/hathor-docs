# Implementation Issue Dependencies

## Critical Path

```text
Architecture freeze
  -> Compose/private networks/secrets/health
  -> Auth/JWKS/service-token issuer
  -> Gateway and public/internal contract validation
  -> Catalog quote + library ownership check
  -> Cart + idempotent order initialization
  -> Simulator callback + commerce outbox
  -> RabbitMQ library inbox + entitlement grant
  -> Seeded build metadata + download authorization + bounded AI features
  -> End-to-end failure tests, reconciliation, restore drill
  -> Release hardening
```

The creator designer and AI proposal flow can proceed in parallel after auth, catalog ownership, and theme schema validation exist. Cached recommendations begin only after catalog metadata is stable. Neither AI capability may delay the commerce-to-entitlement critical path.

## Milestone Dependencies

| Work item | Depends on | Blocks |
| --- | --- | --- |
| Milestone 0 contracts | Planning approval | All implementation work |
| Private Compose topology | Approved deployment/security architecture | All service implementation and integration tests |
| Auth user/session flow | Auth schema, signing keys, gateway path | Protected routes and service-token issuance |
| Internal service tokens | Auth service client credentials and JWKS | Catalog quotes, ownership checks, build lookup |
| Catalog quote | Catalog schema, internal token validation | Checkout initialization |
| Library ownership check | Library schema, internal token validation | Checkout initialization |
| Idempotent order initialization | Cart, quote, ownership check | Payment simulator |
| Payment callback/outbox | Order schema and RabbitMQ topology | Entitlement grant |
| Library inbox/consumer | Event contract, library schema | Fulfillment completion and library UI |
| Seeded build/download token | Catalog build record, entitlement check, storage setup | Download demonstration |
| Admin role workflow | Auth session authorization version and audit schema | Creator onboarding/admin controls |
| AI designer proposal | Catalog theme schema and creator ownership | Constrained designer copilot demonstration |
| Cached recommendations | Stable catalog metadata and provider adapter | Recommendation UI and local AI acceptance |
| Reconciliation/DLQ/restore | Outbox/inbox and local backups | Release approval |

## Parallel Team Allocation

| Engineer | July 25-28 | July 29-August 6 | August 7-13 |
| --- | --- | --- | --- |
| Engineer 1 | Compose, observability, contract CI | RabbitMQ, monitoring, backup automation | Recovery drills and local-demo hardening |
| Engineer 2 | Auth, JWKS, refresh, service tokens, bootstrap admin | Commerce cart/order/simulator/outbox | Reconciliation, transaction audit |
| Engineer 3 | Catalog schema, public store, quote, theme validation | Library ownership, inbox, entitlement APIs | Build metadata, download authorization, moderation, AI cache |
| Engineer 4 | Web shell, auth/store pages, generated client | Cart/checkout/order status/library UX | Creator/admin UI, AI proposal/recommendation UI, E2E coverage |

## Feature-Freeze Rule

After August 14, the team may not add a route, event type, database table, payment state, role, or external provider. Changes are restricted to release-blocking defects, security fixes, test corrections, documentation/runbook fixes, and local-demo hardening.
