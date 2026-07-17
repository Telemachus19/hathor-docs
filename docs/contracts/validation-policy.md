# Contract Validation Policy

## Source Of Truth

`public-api.openapi.yaml`, `internal-api.openapi.yaml`, and `domain-events.asyncapi.yaml` are reviewed source contracts. Code does not change a route or event shape before its source contract changes in the same pull request.

## Runtime Enforcement

Each service uses Zod or an equivalent runtime schema validator at its transport boundary. Validators must enforce the approved contract before domain logic executes:

- Request body, path, query, and header validation for HTTP handlers.
- Response contract tests for public routes.
- Message payload validation before RabbitMQ publish and immediately after consume.
- Strict theme DSL validation before persistence or preview.

Services may use generated TypeScript types/clients, but generated types do not replace runtime validation.

## CI Gate

Every pull request that changes a contract or consumer must run:

1. OpenAPI lint/validation for both OpenAPI documents.
2. AsyncAPI lint/validation for the RabbitMQ document.
3. Schema compatibility check for changed event versions.
4. Contract tests for every changed HTTP operation or event consumer.
5. Full Compose integration test for payment/entitlement contract changes.

The implementation team should select one maintained OpenAPI linter and one maintained AsyncAPI validator during repository bootstrap, pin them in `package.json`, and run them in CI. A contract cannot be changed through generated code or handwritten route changes alone.

## Versioning Rules

- Additive optional HTTP fields may remain in the same minor draft contract after consumer review.
- Removed/renamed fields, changed validation rules, and changed event meanings require a new version.
- Events use versioned names such as `commerce.order.paid.v1`; a breaking event becomes `commerce.order.paid.v2` with parallel consumer migration.
- The simulator provider contract is versioned like any external provider contract.
