# Admin Bootstrap Runbook

## Purpose

Create exactly one initial administrator before the audited admin role-management API is available.

## Preconditions

- The target user already exists as a gamer account.
- A high-entropy `INITIAL_ADMIN_BOOTSTRAP_SECRET` is injected only into the deployment environment.
- The operation is performed through the auth-service bootstrap command, not direct SQL.
- The command writes a `role_change_audit` entry with `bootstrap` as the action source.

## Procedure

1. Confirm the target user ID through an authenticated operator process.
2. Run the one-time auth-service bootstrap command with the target user ID and bootstrap secret.
3. Verify the command atomically grants `admin`, increments authorization version, records audit data, and invalidates the bootstrap secret state.
4. Log the operator, target ID, correlation ID, and deployment version outside the application repository.
5. Remove the bootstrap secret from the running environment after successful initialization.
6. Confirm a second invocation is rejected.

## Safety Rules

- Never grant administrator through direct SQL, a public registration field, or a seeded password committed to source control.
- After bootstrap, all creator/admin role changes use `POST /api/v1/admin/users/{userId}/roles` and create an audit record.
- If bootstrap is suspected to be compromised, rotate the secret, review auth logs, increment affected authorization versions, and rotate signing keys if token exposure is suspected.
