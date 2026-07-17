# Security Architecture

## Authentication And Sessions

Auth issues RS256 access tokens with `iss`, `aud`, `sub`, `roles`, `iat`, `nbf`, `exp`, `jti`, `kid`, and an authorization/session version. All services pin `alg=RS256`, validate every claim, and fetch JWKS by `kid` with bounded caching and refresh on unknown keys.

Access tokens expire in 10-15 minutes and remain only in browser memory. Refresh tokens are opaque, stored as hashes, rotate on every refresh, and are sent in `HttpOnly`, `Secure`, `SameSite=Lax` cookies scoped to the refresh path. Refresh-token reuse revokes the token family. Password reset, logout, role removal, account disablement, and incident response revoke relevant token families and authorization versions.

Signing keys are securely injected and persisted outside source control. They are never generated on production or staging container startup. JWKS supports active and retiring keys under distinct `kid` values and has a documented emergency rotation procedure.

Registration assigns `gamer` only. The first administrator is created through a one-time bootstrap command gated by a deployment secret. Thereafter, creator and admin roles are assigned by protected auth-service admin workflows that write an audit record and increment the target user's authorization version. Privileged operations check that version so a demoted creator or admin cannot retain access until ordinary access-token expiry.

Public registration is protected by IP and account-creation rate limits plus server-side Cloudflare Turnstile verification. The Turnstile token is one-time, bound to the registration request, and is never treated as user authentication or stored as a reusable credential.

## Authorization

Every owning service enforces authorization:

| Actor | Allowed scope |
| --- | --- |
| Guest | Published catalog browse/detail only |
| Gamer | Own cart, own orders, own library, own download authorization |
| Creator | Only games with `creator_id` equal to JWT subject; cannot publish/suspend | 
| Admin | Catalog moderation and commerce financial audit |

Creator and admin actions write immutable audit data: actor ID, action, target, before/after state, correlation ID, timestamp, and authorization result.

## Browser Security

Production CORS uses exact HTTPS origins and never wildcard credentials. Cookie-authenticated refresh/logout operations require origin validation and CSRF protection. Helmet/CSP must block object embeds, unsafe base URLs, and unapproved script sources. Game descriptions are plain text or sanitized limited rich text. Media URLs use HTTPS and an allowlisted host set.

Themes are a validated data schema, not CSS. Allowed values are colors, bundled font identifiers, enumerated layout templates, visibility flags, and an ordered list of known page sections. Raw HTML, CSS, JavaScript, selectors, remote fonts, `@import`, arbitrary URLs, and iframe markup are rejected.

## Payment Security

Commerce verifies payment callbacks against the exact raw request body with constant-time HMAC comparison. It validates signature version, timestamp freshness, provider event ID uniqueness, merchant ID, payment reference, method, amount, currency, and terminal state. Callback receipts store a payload hash and verification outcome, not secrets.

The payment simulator is allowed only in demo/test deployments. It is not enabled through a client flag and cannot grant production entitlements.

## Infrastructure And Secrets

Only gateway/web ports are public. Internal services, PostgreSQL, caches, broker ports, and RabbitMQ management are private. Secrets are injected from untracked environment files locally and a managed secret store in staging. Runtime services use least-privilege database and RabbitMQ credentials. No key, password, webhook secret, or object-storage credential is committed to the repository.

## Seeded Build Delivery

Cloudflare R2 object storage is private. The release uses team-seeded immutable artifacts only. Library issues a 60-120 second URL for one exact object after verifying user entitlement, game publication state, and build publication state. The client verifies the recorded SHA-256. Previously issued URLs are bearer credentials until expiry; immediate revocation of issued URLs is a post-release CDN authorization-edge capability.
