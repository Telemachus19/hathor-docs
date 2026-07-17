# Backup And Restore Runbook

## Backup Policy

- Run encrypted nightly PostgreSQL dumps for auth, catalog, commerce, and library databases.
- Back up catalog build metadata and immutable artifact manifest hashes.
- Store backups off-host with 14-day retention.
- Keep the backup encryption key separate from the backup destination.

## Restore Drill

Before release, restore to an isolated environment:

1. Restore all four databases from the same backup window.
2. Verify migration versions and service readiness.
3. Verify seeded game, build, user, order, and license relationships.
4. Verify a known fulfilled order maps to a license source order.
5. Verify the seeded object key and SHA-256 in catalog metadata match object storage.
6. Record restore duration, data integrity outcome, and issues.

## Recovery Rule

Application recovery uses forward fixes and migration-safe repair procedures. Never use destructive database resets on the public VPS to resolve an application incident.
