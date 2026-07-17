# ADR-001: August Vertical Slice Architecture Freeze

- Status: Accepted
- Date: 2026-07-15
- Owners: Hathor Planning

## Context

Hathor must deliver a public demonstrable microservice product by August 30, 2026. The architecture must prioritize reliable payment-to-entitlement behavior, secure creator customization, and reproducible local/staging operations without claiming production scale or live-payment settlement.

## Decisions

1. The release uses five independently deployable applications: API gateway, auth, catalog, commerce, and library. Each domain service has its own physical PostgreSQL container and least-privilege database credential locally.
2. RabbitMQ is the only asynchronous domain-event transport. Commerce uses a transactional outbox and library uses an idempotent inbox. Redis Pub/Sub is prohibited for fulfillment.
3. The team develops and validates locally first with Docker Compose. Public staging runs on one hardened VPS when hosting becomes available, using the same Compose topology, a TLS reverse proxy, private Docker networks, persistent service volumes, and only ports 80/443 publicly exposed. SSH is restricted to approved operators.
4. Public user access uses short-lived RS256 access tokens and rotating HttpOnly refresh cookies. Internal service calls use separate auth-issued RS256 client-credential tokens with five-minute expiry, service subject, audience, and scope claims.
5. Public registration creates only gamer accounts. The first administrator is created by a one-time, secret-gated bootstrap command. Thereafter, admins use audited role-management APIs to grant creator/admin roles.
6. The payment UI offers branded Fawry, Vodafone Cash, and InstaPay simulator flows. They use one server-side provider-adapter callback contract and are enabled only in demo/test deployments.
7. Download delivery is limited to one private, immutable, SHA-256-verified ZIP artifact per seeded game stored in Cloudflare R2. Creator build uploads, chunking, and delta patching are deferred.
8. The Store Page Designer AI is a catalog-service provider adapter. It uses an optional OpenAI key in staging and returns only schema-constrained JSON Patch proposals. Manual designer functionality remains available without an AI provider.
9. RabbitMQ consumers retry five times with exponential backoff before moving a message to the DLQ. Reconciliation runs daily and has a documented repair/replay procedure.
10. The VPS takes encrypted nightly off-host backups of PostgreSQL data and release metadata with 14-day retention. A restore drill is required before release.
11. OpenAPI 3.1 and AsyncAPI documents are the reviewed contract source. Implementations add runtime Zod validation that matches the approved schemas.
12. Public registration is protected by rate limits and Cloudflare Turnstile verification. The simulator UI supports only successful and failed payment outcomes; delayed callbacks and expiry remain automated-test scenarios.

## Consequences

The release intentionally defers live provider settlement, high availability, multi-region deployment, creator upload pipelines, arbitrary CSS, and AI buyer assistants. The team gains actual microservice, broker, database-isolation, and deployment experience while keeping the payment and entitlement critical path small enough to test thoroughly.
