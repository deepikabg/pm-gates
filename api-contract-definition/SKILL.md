---
name: api-contract-definition
description: "Define API contracts (OpenAPI/gRPC) before implementation. Use when designing service-to-service APIs or public endpoints. Ensures versioning, pagination, error handling, and schema consistency."
---

# API Contract Definition

The contract is the cheapest place to get the API right. Once consumers integrate, the shape is frozen — a renamed field or an unversioned breaking change becomes everyone else's migration. This skill forces the decisions that are nearly free now and irreversible later: **versioning, error taxonomy, pagination, and backwards-compatibility promise** — agreed and validated against real scenarios *before* a line of implementation.

## When to Trigger
Before coding any API endpoint:
- Public APIs (external consumers)
- Service-to-service APIs
- Webhook specifications
- Event schema definitions

## When NOT to Trigger
- A single internal function call with no network boundary
- A throwaway endpoint in a prototype you'll delete (note the contract is deferred)
- A purely additive, non-breaking change to an endpoint already under contract (just extend the existing spec)

## Scale to the build
- **Public API / external consumers** → full spec, explicit versioning + deprecation policy, 5–10 scenarios, consumer testing guide.
- **Internal service-to-service** → schema + error taxonomy + versioning; lighter on the deprecation ceremony if both sides ship together.
- **Prototype** → sketch the resource shapes and the error envelope; skip formal versioning until it has a real consumer.

## What It Produces
1. OpenAPI spec (or gRPC proto)
2. Versioning strategy
3. Error taxonomy
4. Example requests/responses (5–10 scenarios)
5. Consumer testing guide

## Workflow

### Step 1: Gather Intent
```
For this API, tell me:

1. Primary use case?  (CRUD / async job / real-time stream / file upload-download)
2. Expected throughput?  ([req/sec at launch] → [req/sec at scale])
3. Backwards compatibility required?  (Yes = public/external · No = internal, can break)
4. Consumer types?  (web / mobile / server-to-server / CLIs/scripts)
```

### Step 2: Generate the Spec
```yaml
openapi: 3.0.0
info:
  title: [API Name]
  version: 1.0.0
  description: |
    [What this API does]
    Versioning: Semantic versioning (MAJOR.MINOR.PATCH)
    Deprecation policy: 6-month grace period before breaking changes
paths:
  /v1/[resource]:
    post:
      summary: Create [resource]
      parameters:
        - in: header
          name: Idempotency-Key       # safe-to-retry: dedupe duplicate creates (critical for payments)
          required: true
          schema: { type: string }
      requestBody:
        content:
          application/json:
            schema: [...]
      responses:
        201:
          description: Created
          content:
            application/json:
              schema: [...]
        400:
          description: Validation error
          content:
            application/json:
              schema:
                type: object
                properties:
                  error_code: { type: string }   # e.g., "INVALID_EMAIL"
                  message:    { type: string }   # e.g., "Email format invalid"
                  details:    { type: object }   # e.g., {"field": "email", "reason": "..."}
        429:
          description: Rate limited
          headers:
            X-RateLimit-Remaining: { schema: { type: integer } }
    get:
      summary: List [resource] (paginated)
      parameters:
        - in: query
          name: limit
          schema: { type: integer, default: 25, maximum: 100 }   # cap page size
        - in: query
          name: cursor
          schema: { type: string }        # opaque cursor — prefer over offset for stable paging
      responses:
        200:
          description: OK
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:        { type: array, items: {} }
                  next_cursor: { type: string, nullable: true }   # null = last page

components:
  schemas:
    ErrorResponse:
      type: object
      properties:
        error_code: { type: string }   # Machine-readable
        message:    { type: string }   # Human-readable
        request_id: { type: string }   # For support tickets
        timestamp:  { type: string }   # ISO 8601
  securitySchemes:
    apiKey:
      type: apiKey
      in: header
      name: X-API-Key

security:
  - apiKey: []
```

### Step 2b: Patterns to Lock (pick the ones that apply)
The template above is sync CRUD. Most real APIs need one or more of these — decide them now, not after a consumer integrates:
- **Idempotency** (any create / payment / state-changing POST): require an `Idempotency-Key` header; the server dedupes retries so a network retry never double-charges or double-writes.
- **Pagination** (any list endpoint): cursor-based by default (stable under inserts), with a capped `limit`. Offset pagination only for small, static sets.
- **Async / long-running jobs:** return `202 Accepted` + a job id; consumer polls `GET /jobs/{id}` or receives a webhook on completion. Don't hold a request open.
- **Webhooks** (events to consumers): **HMAC signature** so the receiver can verify authenticity, documented retry policy with backoff, and at-least-once delivery → tell consumers to **dedupe by event id** (their idempotency).
- **Streaming** (AI token output, live feeds): Server-Sent Events (SSE) for one-way server→client; document the event format and how the stream terminates/errors.
- **Rate limits & quotas:** document the actual limits and the `429` + `Retry-After` / `X-RateLimit-*` contract — including cost-based limits on expensive (e.g. LLM-backed) endpoints.
- **AuthZ in the contract:** if using scopes/permissions, state which scope each endpoint requires (not just "needs a key").

### Step 3: Validate Against Use Cases
Walk 5–10 concrete scenarios — happy path, every error in the taxonomy, and pagination:
```
Scenario 1 — Create new user
  Request:  POST /v1/users { "email": "new@example.com", "name": "..." }
  Response: 201 { "id": "usr_123", "email": "new@example.com", "created_at": "..." }

Scenario 2 — Email already exists
  Request:  POST /v1/users { "email": "existing@example.com" }
  Response: 400 { "error_code": "EMAIL_ALREADY_EXISTS", "message": "..." }

Scenario 3 — Rate limited
  Response: 429 { "error_code": "RATE_LIMITED" } + X-RateLimit-Remaining: 0
...
```

### Step 4: PM Approval
- [ ] Errors are well-defined (error_code + message + request_id)
- [ ] Pagination strategy is clear and capped (list endpoints)
- [ ] Idempotency defined for state-changing POSTs (esp. payments)
- [ ] Async/webhook/streaming patterns specified where the use cases need them
- [ ] Webhooks are signed + dedupe-able; rate limits documented
- [ ] Versioning path is explicit
- [ ] Backwards-compatibility promise is documented

## Handoff
Once the contract is approved, write the contract files' paths + a `passed` entry to `.pipeline/state.yaml`, then **signal loop-orchestrator — it owns routing; this skill never chooses the next gate.** (Per the orchestrator's rules, security-baseline runs next — always, before any implementation — validating PII handling, secrets, auth/authz, compliance scope, and dependency CVEs against the contract just defined.)

> "API contract locked and logged to state — handing to the orchestrator for the next gate."

**Log decisions:** versioning/pagination/error-taxonomy choices and any deferred/rejected recommendation → append to `.pipeline/DECISIONS.md` with `Affects:` links, so anything it drifts flips to `stale` (format: loop-orchestrator's Decision Ledger).

## Anti-Patterns
- ❌ Designing the contract after the code exists → ✅ contract first, code to the contract
- ❌ Unversioned endpoints → ✅ explicit version path + deprecation policy from day one
- ❌ Free-text error strings → ✅ machine-readable `error_code` + human `message` + `request_id`
- ❌ Pagination "we'll add when it's slow" → ✅ cursor-based, capped, decided up front for any list endpoint
- ❌ Non-idempotent create/payment endpoints → ✅ `Idempotency-Key` so retries never double-charge
- ❌ Unsigned webhooks / "consumer can just trust it" → ✅ HMAC-signed, dedupe-by-event-id, documented retries
- ❌ Holding a request open for a long job → ✅ `202` + job id + poll/webhook
- ❌ Validating only the happy path → ✅ a scenario for every error in the taxonomy
- ❌ Silent breaking changes → ✅ a documented backwards-compatibility promise consumers can rely on
