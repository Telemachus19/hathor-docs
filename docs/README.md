# Hathor Canonical Documentation

This directory is the implementation source of truth for Hathor's August 30, 2026 vertical slice. The root-level planning documents remain valuable background and issue-source material, but these documents resolve their conflicting decisions.

## Canonical Documents

| Document | Authority |
| --- | --- |
| `architecture/system-architecture.md` | Service ownership, topology, trust boundaries, and infrastructure rules |
| `architecture/vps-deployment.md` | Public staging topology, service identity, persistence, and backup rules |
| `contracts/service-contracts.md` | Public, internal, and asynchronous contract rules |
| `contracts/public-api.openapi.yaml` | Gateway-visible HTTP API contract |
| `contracts/internal-api.openapi.yaml` | Scoped service-to-service HTTP API contract |
| `contracts/domain-events.asyncapi.yaml` | RabbitMQ event contract |
| `contracts/validation-policy.md` | Contract source-of-truth, runtime validation, and CI rules |
| `data/service-data-model.md` | Per-service ownership, financial invariants, audit records, and reconciliation data |
| `security/security-architecture.md` | Authentication, authorization, payment, browser, and storage security |
| `reliability/payment-entitlement.md` | Order state machine, outbox/inbox, RabbitMQ, reconciliation, and recovery |
| `reliability/payment-entitlement-sequence.md` | End-to-end purchase sequence and failure behavior |
| `architecture/ai-designer.md` | Safe AI integration for the Store Page Designer |
| `delivery/implementation-roadmap.md` | Team allocation, August 30 schedule, scope, and release gates |
| `delivery/architecture-freeze-checklist.md` | Approved freeze decisions and contract-approval gate |
| `delivery/issue-dependencies.md` | Critical-path dependency graph and team allocation |
| `../GitHub_Projects_Fullstack_Plan.md` | GitHub Projects V2 board source for full-stack delivery |
| `adr/ADR-001-august-vertical-slice.md` | Accepted architecture-freeze decisions |
| `runbooks/` | Admin bootstrap, DLQ replay, and backup/restore procedures |

## Non-Negotiable Decisions

1. Hathor is implemented as independently deployable gateway, auth, catalog, commerce, and library services.
2. Each domain service owns one physical PostgreSQL instance locally and never reads another service database.
3. RabbitMQ is the only asynchronous transport for payment, entitlement, and release events. Redis Pub/Sub is prohibited for these events.
4. Commerce uses a transactional outbox; library uses an idempotent inbox before acknowledging fulfillment events.
5. The gateway is the only public ingress to business services. Databases, caches, RabbitMQ, and service ports are private.
6. RS256 access tokens are short-lived and independently verified by each service. Browser access tokens are memory-only; refresh tokens are rotated HttpOnly cookies.
7. The August release uses a secure payment simulator and a seeded immutable build. It does not process live money or accept creator build uploads.
8. The Store Page Designer and its AI assistant can configure only validated, predefined design options. They cannot emit executable presentation code or publish automatically.

## Documentation Status

All future architectural changes must update the relevant canonical document and include an ADR when they alter one of the non-negotiable decisions.
