# AI Architecture & Integration Specification

## Role & Scope

AI capabilities in Hathor span three distinct architectural paradigms: **Direct LLM Integration**, **Retrieval-Augmented Generation (RAG)**, and **Agentic AI Workflows**. All AI capabilities operate as isolated provider adapters or worker services. They do not hold direct database-write permissions, payment authority, or auto-publication capability.

```text
                                  +---------------------------------------+
                                  |            Client / Web               |
                                  +---------------------------------------+
                                                      |
                                          API Gateway (/api/v1/*)
                                                      |
         +--------------------+-----------------------+-----------------------+
         |                    |                       |                       |
+------------------+ +------------------+   +-------------------+   +--------------------+
|   Auth Service   | | Commerce Service |   |  Library Service  |   |  Catalog Service   |
+------------------+ +------------------+   +-------------------+   +--------------------+
                                                      |                       |
                                             (Read-only Metadata)      +--------------+
                                                      |                | AI Adapter   |
                                                      v                +--------------+
                                            +-------------------+             |
                                            |  AI Domain Worker | <-----------+
                                            | (RAG & Agent Engine)
                                            +-------------------+
                                                      |
                                        +-------------+-------------+
                                        |                           |
                                +---------------+           +---------------+
                                |  LLM Provider |           |  Vector Store |
                                | (OpenAI/Anth) |           |  (pgvector)   |
                                +---------------+           +---------------+
```

---

## 1. Direct LLM Integration (Structured Output Generation)

Direct LLM calls handle single-turn prompt-to-structure generation without external retrieval loops or autonomous tool execution.

### Capabilities:
1. **Store Page Theme Customizer Copilot**:
   - Accepts creator prompts (e.g. *"Dark synthwave vibe with neon gold accents and Cairo font"*).
   - Invokes LLM provider adapter using strict JSON Schema enforcement.
   - Outputs RFC 6902 JSON Patch operations over the allowlisted `ThemeDocument` DSL.
2. **Store Copy & Marketing Assistance**:
   - Generates game descriptions, feature bullet lists, FAQ outlines, tag suggestions, and Arabic/English localized store copy from creator notes.

---

## 2. Retrieval-Augmented Generation (RAG Integration)

RAG grounds LLM responses in Hathor's domain data using vector embeddings stored in PostgreSQL (`pgvector`).

### Capabilities:
1. **Semantic Storefront Search & Discovery** (`catalog-service`):
   - **Index**: Game titles, long descriptions, tags, and system requirements embedded via `text-embedding-3-small` into `catalog_db.game_embeddings`.
   - **Retrieval Flow**: Gamer inputs natural query (e.g. *"tactical turn-based RPG with dark space atmosphere"*) → API embeds query → Cosine similarity vector search retrieves Top-K games → LLM ranks and attaches short explainable reasons (*"Selected because of tactical combat tags and sci-fi theme"*).
   - **Fallback**: Search defaults to standard PostgreSQL text/tag ILIKE search if vector indexing or LLM retrieval times out.
2. **Creator Guidelines & Store Policy Assistant** (`support-service`):
   - **Index**: Official Hathor creator documentation, image size limits, fee structures, and MENA payment payout rules.
   - **Retrieval Flow**: Creator asks *"What are the header banner dimensions and Fawry payout rules for Egypt?"* → System retrieves relevant policy chunks → LLM returns verified answers complete with markdown doc citations.

---

## 3. Agentic AI Workflows (Tool-Calling Autonomous Agents)

Agentic AI operates as multi-step autonomous decision loops using registered internal tool registries. All agent actions require **Human-in-the-Loop (HITL)** approval before mutating application state.

```text
[Goal Request] -> [Agent Loop] -> [Tool Call: Metadata/Palette/Copy] -> [Validate Schema] -> [HITL Approval Card] -> [State Change]
```

### Agent A: Auto-Storefront Builder Agent (Creator Assistant)
- **Goal**: Build a complete, valid store page draft from raw assets and text.
- **Tool Registry**:
  - `get_game_metadata(gameId)`: Pull draft title, price, and category.
  - `extract_asset_palette(gameId)`: Analyze screenshot color distributions.
  - `generate_theme_patch(palette, template)`: Generate theme JSON Patch.
  - `validate_theme_schema(themeDoc)`: Server-side schema verification.
  - `draft_marketing_copy(language)`: Generate store copy.
- **Execution**: Agent executes iterative tool loop, self-corrects any JSON validation errors, and presents a complete ready-to-review storefront draft to the creator.

### Agent B: Pre-Moderation & Compliance Agent (Admin Assistant)
- **Goal**: Audit games submitted for review (`pending_review`) and generate a risk report before human Admin publication.
- **Tool Registry**:
  - `fetch_draft_content(gameId)`: Fetch game theme, store copy, and asset links.
  - `check_text_safety(text)`: Run toxicity/safety classification.
  - `verify_policy_compliance(gameId)`: Query RAG policy index for content/pricing violations.
- **Execution**: Agent generates a **Moderation Audit Card** (Risk Score, Policy Check Results, Copyright Risk) for the Admin Dashboard. Only a human Admin clicking "Approve" triggers the `published` status change.

---

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

---

## Permitted AI Actions

| Paradigm | Capability | Output |
| :--- | :--- | :--- |
| **Direct LLM** | Theme & Copy Proposal | JSON Patch over theme DSL, plain-text copy |
| **RAG** | Semantic Search & Policy QA | Ranked game lists with reasons, cited policy answers |
| **Agentic AI** | Store Builder & Pre-Moderation | Multi-step drafts, Moderation Audit Cards |

---

## Forbidden AI Actions

- Executing raw HTML, CSS, JavaScript, SVG markup, or iframe output.
- Calls to SQL, storage, RabbitMQ, payment, entitlement, admin, or authentication operations.
- Direct access to user, payment, or session credentials.
- Cross-creator data access.
- Automatic publication or status state changes without human approval.
- Direct price, discount, or role modifications.

---

## Data And Abuse Controls

The adapter sends only the prompt and minimum required draft context. Prompts and model outputs are size-limited, rate-limited, and redacted in logs. Creator-provided text is treated as data, never as tool instructions. Provider failure returns a safe unavailable response and does not block manual workflows.
