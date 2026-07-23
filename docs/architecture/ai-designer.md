# AI Designer Architecture

## Role

The AI designer is a constrained suggestion capability inside the catalog domain. It helps a creator express store-page intent without receiving authority to execute code, modify protected business data, or publish content.

For the August release, the AI is an optional catalog-service provider adapter using an OpenAI key only when one is configured in staging. It is not a new domain-owning microservice. The Store Page Designer remains fully usable when the AI provider is unavailable.

## Safe Flow

```text
Creator prompt
  -> catalog verifies creator ownership of selected draft
  -> AI adapter receives minimum safe context
  -> AI returns a JSON Patch proposal only
  -> deterministic schema/policy validation
  -> preview and diff
  -> creator explicitly accepts
  -> catalog stores a draft revision
  -> normal admin publication workflow
```

The AI cannot publish, suspend, price, upload, download, authenticate, grant licenses, process payments, or access another creator's game.

## August Recommendation Capability

The August release keeps recommendations inside the catalog domain. It is not a new microservice and is not behavioral personalization. The catalog service asynchronously refreshes a bounded local cache from approved seeded game metadata such as tags, price band, publication state, and safe AI-generated catalog summaries when available.

`GET /api/v1/store/recommendations` returns ranked published games, a short explainable reason, and a deterministic curated fallback. Storefront browsing reads the cached result and never waits for an LLM call. The recommendation path must not receive payment history, entitlement records, roles, emails, refresh tokens, or direct access to another service database.

Cold-start, provider outage, invalid provider output, empty catalog, and cache miss states return the curated fallback. Behavioral profiling, cross-service personalization, a recommendation microservice, and buyer-facing AI chat remain deferred.

## Theme DSL

The designer stores a versioned theme document containing only allowlisted values:

```json
{
  "schemaVersion": 1,
  "palette": {
    "background": "#111827",
    "surface": "#1F2937",
    "text": "#F9FAFB",
    "accent": "#F97316"
  },
  "typography": {
    "headingFont": "cairo",
    "bodyFont": "inter",
    "headingScale": "large"
  },
  "layout": {
    "template": "cinematic",
    "heroAlignment": "left",
    "cardStyle": "elevated",
    "showTrailer": true
  },
  "contentOrder": ["hero", "description", "screenshots", "systemRequirements"]
}
```

The frontend maps validated values to controlled CSS variables and predefined components. It never renders model output as raw HTML, CSS, JavaScript, iframe markup, or inline style strings.

## Permitted AI Actions

| Capability | Output |
| --- | --- |
| Theme proposal | JSON Patch over the theme DSL |
| Copy assistance | Plain text description, feature bullets, FAQ outline, alt-text suggestions |
| Accessibility advice | Explanatory suggestions; deterministic code calculates contrast |
| Tag suggestions | Candidate tags requiring creator confirmation |

The server validates every proposal. The creator sees a diff and must explicitly accept it before a draft revision is saved.

## Forbidden AI Actions

- Raw HTML, CSS, JavaScript, SVG markup, or iframe output.
- Calls to SQL, storage, RabbitMQ, payment, entitlement, admin, or authentication operations.
- Direct access to user/payment/session data.
- Cross-creator data access.
- Automatic publication or moderation.
- Price, discount, role, or publication-state changes.

## Data And Abuse Controls

The adapter sends only the creator prompt and minimum selected-game draft context. It does not send credentials, payment data, private user data, or other creators' records. Prompts and model output are size-limited, rate-limited, and logged with redaction. Creator-provided text is treated as data, never as tool instructions. Provider failure returns a safe unavailable response and does not block manual designer editing.
