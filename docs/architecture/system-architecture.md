# System Architecture

## Scope

Hathor's August 30, 2026 release is a distributed vertical slice: a gamer can authenticate, browse seeded games, create an order, complete a secure simulator payment, receive an entitlement, and download one seeded immutable build. A creator can edit a constrained store theme for an owned draft. An admin can moderate games and inspect transaction records.

Live payment providers, creator build uploads, delta patching, AI buyer assistants, recommendations, and arbitrary CSS are outside this release.

## Service Ownership

| Component | Owns | Does not own |
| --- | --- | --- |
| API gateway | Public ingress, TLS, CORS, rate limits, request IDs, edge policy, routing | Domain authorization, business data, database access |
| Auth service | Users, credentials, roles, sessions, refresh-token families, JWKS signing keys | Games, orders, licenses |
| Catalog service | Games, tags, prices, publication state, creator ownership, themes, build metadata | Carts, orders, payments, licenses |
| Commerce service | Carts, catalog quotes, orders, payment events, payment audit, outbox | Catalog records, licenses, credentials |
| Library service | Licenses, wishlists, processed events, entitlement audit, download authorization | Payment finalization, mutable catalog data |
| Object storage | Private immutable seeded artifact and manifest objects | Entitlements, publication state, payment decisions |
| RabbitMQ | Durable domain-event transport | Source-of-truth business state |
| Redis | Rate limits, short-lived revocation metadata, optional disposable cache | Orders, carts, payments, entitlement data |
| Memcached | Optional anonymous catalog response cache | User-specific or authorization-sensitive data |

## Public Routes

| Path | Owner | Access |
| --- | --- | --- |
| `/api/v1/user/*` | Auth | Public and authenticated user operations |
| `/api/v1/store/*` | Catalog | Public browse; creator mutations are protected |
| `/api/v1/cart/*` | Commerce | Authenticated owner only |
| `/api/v1/txn/*` | Commerce | Authenticated owner; webhook route has provider authentication |
| `/api/v1/inventory/*` | Library | Authenticated owner only |
| `/api/v1/creator/*` | Catalog | Creator ownership or admin only |
| `/api/v1/admin/users/*` | Auth | Admin only |
| `/api/v1/admin/games/*` | Catalog | Admin only |
| `/api/v1/admin/transactions/*` | Commerce | Admin only |

`gameId` is the canonical UUID for a game everywhere: public APIs, internal APIs, database records, and events. `appId` is not used for this release.

## Trust Boundaries

```text
Internet
  |
  | HTTPS
  v
Web application and API gateway
  |
  | private service network
  +-- auth service ------ auth PostgreSQL
  +-- catalog service --- catalog PostgreSQL --- Memcached
  +-- commerce service -- commerce PostgreSQL -- Redis -- RabbitMQ
  +-- library service --- library PostgreSQL ---- RabbitMQ
  +-- private object storage
```

Only the web application and API gateway are host/publicly reachable. The payment webhook is routed through the gateway to commerce but is not a buyer-authenticated route. Service ports, PostgreSQL, Redis, Memcached, RabbitMQ AMQP, and RabbitMQ management are private. Development-only debug ports may bind to `127.0.0.1` through a Compose profile.

The gateway is not a trusted identity source. Every domain service verifies the bearer access token and performs its own authorization. Internal APIs require private networking and a scoped, short-lived service credential with an audience and calling-service identity. The browser cannot call an `/internal/*` route.

## Database Isolation

Local development uses four PostgreSQL containers and four named volumes: `auth-postgres`, `catalog-postgres`, `commerce-postgres`, and `library-postgres`. Each service receives a distinct non-superuser database account and only its own connection string. Runtime services never receive the PostgreSQL superuser credential.

Cross-service identifiers are opaque UUIDs. There are no cross-database foreign keys, joins, migrations, or direct reads. Required cross-domain data is obtained through a documented internal API or a local event-driven projection.

## Required Internal Calls

| Caller | Callee | Scope | Purpose |
| --- | --- | --- | --- |
| Commerce | Catalog | `catalog.quote.read` | Server-authoritative sellability and checkout quote |
| Commerce | Library | `library.ownership.read` | Reject already-owned items before payment |
| Library | Catalog | `catalog.build.read` | Resolve the published immutable build for an owned game |

Internal calls use bounded timeouts and fail closed for checkout, ownership, and download decisions. The caller returns a retryable unavailable response rather than guessing data during a dependency outage.

When hosting is available in mid-August, public staging uses one hardened VPS with Docker Compose and a TLS reverse proxy. See `vps-deployment.md` for firewall, service-token, persistence, backup, and rollback requirements.

## Catalog And Build Lifecycle

Games default to `draft`. Only an admin can publish or suspend a game.

```text
draft -> pending_review -> published -> suspended
                 |              |
                 +-> rejected   +-> suspended
```

For the release, trusted team members seed immutable private build artifacts. Catalog owns the build record:

```text
draft -> ready -> published -> revoked
```

Each build has an immutable object key, manifest SHA-256, version, platform, game ID, state, and publication timestamp. `manifest_url` is not an authority record. Library rechecks entitlement, game status, and build state before issuing a short-lived exact-object download URL.

## Operational Requirements

Every service exposes `/health/live` and `/health/ready`. Readiness checks required dependencies: database for all services, RabbitMQ for library, and the outbox worker/database for commerce. Each request and event carries a correlation ID. Logs are structured and never include passwords, JWTs, payment secrets, or signed URLs.
