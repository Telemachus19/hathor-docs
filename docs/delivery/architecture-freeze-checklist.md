# Architecture Freeze Checklist

## Approved Decisions

- [x] Four domain microservices plus gateway remain mandatory.
- [x] Four physical local PostgreSQL containers and least-privilege service users.
- [x] RabbitMQ only for durable fulfillment events; Redis Pub/Sub excluded.
- [x] Commerce transactional outbox and library inbox/idempotent consumer.
- [x] Local Compose is the August delivery target. VPS, Caddy, and public staging are deferred until hosting is budgeted.
- [x] Auth-issued five-minute RS256 scoped service tokens for internal HTTP calls.
- [x] One-time secret-gated initial admin bootstrap; audited admin role workflow afterward.
- [x] Three branded payment simulator flows through one secure provider callback contract.
- [x] One immutable, SHA-256-verified private ZIP per seeded game.
- [x] Five RabbitMQ retries, DLQ, and daily reconciliation.
- [x] Optional OpenAI-backed designer copilot through a provider adapter.
- [x] Encrypted nightly off-host backups with 14-day retention and restore drill.
- [x] Rate-limited public registration with Cloudflare Turnstile verification.
- [x] Simulator UI supports paid and failed outcomes only; expiry and delayed callbacks are test fixtures.
- [x] Private Cloudflare R2 bucket provides the seeded immutable ZIP artifact.
- [x] OpenAPI and AsyncAPI are the reviewed contract source; Zod validates runtime inputs.

## Contract Approval Actions

| Artifact | Owner | Approval condition |
| --- | --- | --- |
| `public-api.openapi.yaml` | Auth, catalog, commerce, library, web leads | Every public path has an owner, role rule, request/response schema, and error behavior |
| `internal-api.openapi.yaml` | Auth, catalog, commerce, library leads | Every synchronous dependency has audience, scope, timeout, and fail-closed policy |
| `domain-events.asyncapi.yaml` | Commerce and library leads | Event schema, retry, DLQ, publisher confirm, and inbox behavior are agreed |
| `service-data-model.md` | Domain service leads | State constraints, audit records, outbox/inbox, and source-order provenance are complete |
| `security-architecture.md` | Security/planning lead | Threats, session lifecycle, simulator protection, secrets, and browser controls are accepted |
| `payment-entitlement.md` | Commerce and library leads | Failure tests and reconciliation process are accepted |
| `vps-deployment.md` | Infrastructure lead | Firewall, TLS, secrets, backups, rollback, and restore drill are feasible |

## Implementation Readiness Gate

Implementation starts when every item below is true:

1. OpenAPI and AsyncAPI YAML pass a chosen linter/validator in CI.
2. Each endpoint/event has a GitHub issue, service owner, reviewer, and test acceptance criteria.
3. Every database mutation has a migration owner and rollback/forward-fix policy.
4. Every external secret has a named staging storage location and rotation owner.
5. The payment simulator, webhook, and entitlement flow have an end-to-end sequence diagram reviewed by commerce and library owners.
6. The first-admin bootstrap procedure is documented, tested in a disposable environment, and cannot run after initialization.
7. Backup restore and DLQ replay procedures have named owners and pre-release test dates.
