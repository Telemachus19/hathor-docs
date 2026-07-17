# VPS Deployment Architecture

## Target And Timing

Hathor is implemented and verified locally first using the same Docker Compose topology described here. Public staging runs on one hardened VPS with Docker Compose once hosting is available, targeted for mid-August. This is a release deployment for the August demo, not a high-availability production topology.

Local Compose is the required source of truth for service networking, service discovery, environment variables, migrations, health checks, and the payment-to-entitlement integration suite. The VPS deployment changes only environment-specific values such as domain, TLS, secrets, persistent volumes, firewall rules, and Cloudflare credentials.

## Network Layout

```text
Internet
  |
  +-- TCP 80/443 --> Caddy reverse proxy --> web / API gateway

Private Docker networks
  +-- auth service ------ auth PostgreSQL
  +-- catalog service --- catalog PostgreSQL --- Memcached
  +-- commerce service -- commerce PostgreSQL -- Redis -- RabbitMQ
  +-- library service --- library PostgreSQL ---- RabbitMQ
```

Caddy terminates TLS and redirects HTTP to HTTPS. Only ports 80 and 443 are public. SSH uses key-only authentication and is restricted by firewall to approved operator addresses. RabbitMQ management, databases, cache services, and internal service ports have no public host bindings.

## Service Identity

Auth issues separate internal client-credential tokens from a distinct RS256 signing key/JWKS set. An internal token has:

```json
{
  "iss": "https://auth.hathor.internal",
  "sub": "commerce-service",
  "aud": "catalog-service",
  "scope": ["catalog.quote.read"],
  "exp": "five minutes after issuance",
  "jti": "uuid",
  "kid": "internal-key-id"
}
```

Services obtain internal tokens using their own secret injected through Docker secrets or an untracked deployment secret file. A callee verifies issuer, audience, scope, expiry, `kid`, and `alg=RS256`. User access tokens cannot call internal routes. mTLS is deferred, but private networks and scoped tokens are both required.

## Persistence And Backups

All PostgreSQL containers and RabbitMQ use named persistent volumes. The VPS runs encrypted nightly PostgreSQL dumps and release-metadata backups to a separate off-host storage location with 14-day retention. The backup encryption key is not stored with the backup destination.

The release gate includes a restore drill to a clean environment: restore all four PostgreSQL databases, verify seeded build metadata, and verify that a user entitlement remains queryable. Object artifacts are versioned/immutable and their manifest hashes are included in backup metadata.

## Cloudflare R2 Delivery

The seeded ZIP artifacts reside in a private Cloudflare R2 bucket through its S3-compatible API. The library service is the only service with R2 signing credentials. It issues a 60-120 second URL for one exact immutable object after entitlement and publication checks. Bucket listing, public ACLs, wildcard object prefixes, and browser-supplied object keys are prohibited.

R2 CORS permits only the configured HTTPS web origin and required download methods/headers. Catalog persists the object key and SHA-256, not a mutable public URL. The artifact SHA-256 is included in the download-token response and verified by the download client.

## Deployment Rules

- Build images in CI or a controlled build host; do not install application dependencies interactively on the VPS.
- Inject secrets at deploy time; never include them in images or source control.
- Use health checks and readiness endpoints before accepting gateway traffic.
- Keep a previous immutable image tag available for rollback.
- Record deployed image versions, migration versions, and timestamp in the release log.
