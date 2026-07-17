# Implementation Roadmap To August 30, 2026

## Release Scope

The release demonstrates a complete microservice vertical slice:

1. Register, login, refresh, logout, and role enforcement.
2. Seeded public catalog with constrained themes.
3. Persistent cart and server-authoritative checkout quote.
4. Secure simulator payment through normal callback verification.
5. RabbitMQ outbox/inbox entitlement grant.
6. Library list and authorized download of a seeded immutable build.
7. Creator draft/theme editing and admin moderation/transaction audit.
8. Local Docker Compose demonstration, with public staging deployment after VPS access is available.

Deferred: live payments, creator build uploads, chunking/delta patching, arbitrary CSS, AI buyer assistants, recommendations, and high-availability scaling.

## Team Ownership

| Engineer | Primary ownership |
| --- | --- |
| Engineer 1 | Infrastructure, Compose, staging, gateway, observability |
| Engineer 2 | Auth, commerce, payment simulator, outbox |
| Engineer 3 | Catalog, creator ownership, seeded build metadata, library consumer |
| Engineer 4 | Web application, integration, checkout/library/creator/admin user flows |

Every owner writes tests and updates contracts for their work. Another service owner reviews cross-service changes.

## Schedule

### July 15-19: Architecture Freeze

- Consolidate canonical documentation and resolve all conflicting decisions.
- Approve OpenAPI, internal API, AsyncAPI, data, security, reliability, and AI designer contracts; select and pin OpenAPI/AsyncAPI validation tools for CI.
- Create GitHub issues from this roadmap with dependency links and acceptance criteria.

**Exit:** no unresolved transport, ownership, auth, payment, or route ambiguity.

### July 20-26: Local Platform, Gateway, And Auth Baseline

- Create monorepo applications and isolated service database topology.
- Implement health/readiness, secret injection, structured logging, and correlation IDs.
- Implement gateway routing and auth registration/login/JWKS/refresh/logout, Cloudflare Turnstile registration verification, one-time admin bootstrap, and audited admin role management.
- Implement seeded public catalog browse/detail and web auth/store shell.
- Establish the complete local Compose environment, including private internal services, health checks, seeded data, and reproducible startup.

**Exit:** `register -> login -> gateway-authenticated /me -> browse seeded catalog` works in a clean local Compose environment.

### July 27-August 2: Catalog Quotes, Themes, Carts, And Pending Orders

- Implement theme schema, creator ownership checks, game status transition rules, draft revisions, and the optional provider-adapter AI theme proposal flow.
- Implement catalog quote internal API and immutable checkout snapshots.
- Implement cart CRUD, idempotent transaction initialization, and checkout UI.
- Implement public/internal contract tests.

**Exit:** an authenticated gamer can create exactly one valid `payment_pending` order from a server quote.

### August 3-9: Payment Simulator, Outbox, RabbitMQ, And Licenses

- Implement simulator provider adapter, raw-body HMAC verification, payment events, and order transitions.
- Implement RabbitMQ topology, outbox relay, retry/DLQ, and monitoring.
- Implement library inbox, idempotent license grants, entitlement acknowledgments, and library UI.
- Add duplicate callback/event and restart failure tests.

**Exit:** verified simulated payment grants exactly one entitlement after retries or consumer restarts.

### August 10-16: Seeded Build Download, Creator, Admin, And VPS Provisioning

- Seed immutable private Cloudflare R2 build artifact and manifest; record SHA-256 and build state.
- Implement library download authorization and client integrity verification.
- Implement creator theme editor and admin moderation/transaction audit interfaces.
- Add authorization/IDOR/security tests.
- When hosting becomes available, provision the VPS, domain/TLS, private Docker networks, deployment secrets, Cloudflare R2 credentials, and off-host backup destination.

**Exit:** only an entitled user can download a published seeded build; creator/admin boundaries are enforced. If VPS access is available, the same tested Compose images run on staging.

### August 17-23: Integration, VPS Deployment, And Failure Recovery

- Run contract, integration, and E2E tests through the local Compose topology. Run the same suite on public staging as soon as VPS access is available.
- Exercise broker outage, outbox recovery, duplicate event, DLQ replay, expired order, and key-rotation scenarios.
- Implement reconciliation job and runbooks.
- Add Memcached only if catalog correctness and invalidation tests are already green.

**Exit:** payment-to-entitlement recovery paths are exercised and all release-blocking tests pass locally; staging deployment is validated if hosting is available.

### August 24-30: Release Hardening And Demo

- Freeze features on August 24.
- Fix only release-blocking defects.
- Verify backup/restore, clean local Compose startup, staging deployment, and seeded demo accounts/data.
- Rehearse the complete demo and failure-recovery walkthrough.

**Exit:** all release criteria pass on local Compose. If hosting is provisioned by mid-August as planned, the same criteria also pass on public staging before release.

## Release Criteria

1. Local Compose exposes only web/gateway by default. In staging, only gateway/web are public.
2. All services independently verify JWTs and enforce ownership/roles.
3. Checkout never trusts browser pricing or totals.
4. Payment callbacks are raw-body verified, replay-safe, and idempotent.
5. Commerce uses an outbox and library uses an inbox/processed-event ledger.
6. Broker outages and duplicate deliveries cannot lose or duplicate entitlements.
7. Download access is limited to an owned game and published seeded build.
8. Creator/admin object ownership and role boundaries are tested.
9. Contract, integration, E2E, security, and failure-recovery tests pass.
10. Reconciliation, DLQ replay, and entitlement-repair runbooks are present and exercised.
