---
name: api-contract-definition
description: "Define API contracts (OpenAPI/gRPC) before implementation. Use when designing service-to-service APIs or public endpoints. Ensures versioning, pagination, error handling, and schema consistency."
---

# API Contract Definition

## When to Trigger

Before coding any API endpoint:
- Public APIs (for external consumers)
- Service-to-service APIs
- Webhook specifications
- Event schema definitions

## What It Does

Claude generates:
1. OpenAPI spec (or gRPC proto)
2. Versioning strategy
3. Error taxonomy
4. Example requests/responses (5–10 scenarios)
5. Consumer testing guide

## Workflow

### Step 1: Gather Intent

Claude asks:
```
For this API, tell me:

1. What's the primary use case?
   - CRUD operations?
   - Async job submission?
   - Real-time streaming?
   - File upload/download?

2. Expected throughput?
   - [req/sec at launch]
   - [req/sec at scale]

3. Backwards compatibility required?
   - Yes (public API, external consumers)
   - No (internal, can break)

4. Consumer types?
   - Web frontend?
   - Mobile?
   - Server-to-server?
   - CLIs/scripts?
```

### Step 2: Claude Generates Spec

```yaml
openapi: 3.0.0
info:
  title: [API Name]
  version: 1.0.0
  description: |
    [Clear description of what this API does]
    
    Versioning: Semantic versioning (MAJOR.MINOR.PATCH)
    Deprecation policy: 6-month grace period before breaking changes
    
paths:
  /v1/[resource]:
    post:
      summary: Create [resource]
      parameters: [...]
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
                  error_code: string  # e.g., "INVALID_EMAIL"
                  message: string     # e.g., "Email format invalid"
                  details: object     # e.g., {"field": "email", "reason": "..."}
        429:
          description: Rate limited
          headers:
            X-RateLimit-Remaining: integer

components:
  schemas:
    ErrorResponse:
      type: object
      properties:
        error_code: string      # Machine-readable
        message: string         # Human-readable
        request_id: string      # For support tickets
        timestamp: string       # ISO 8601

  securitySchemes:
    apiKey:
      type: apiKey
      in: header
      name: X-API-Key

security:
  - apiKey: []
```

### Step 3: Validate Against Use Cases

Claude walks through 5–10 scenarios:
```
Scenario 1: "Create new user"
  Request: POST /v1/users { "email": "...", "name": "..." }
  Response: 201 { "id": "usr_123", "email": "...", "created_at": "..." }

Scenario 2: "Email already exists"
  Request: POST /v1/users { "email": "[existing@example.com](mailto:existing@example.com)" }
  Response: 400 { "error_code": "EMAIL_ALREADY_EXISTS", "message": "..." }

...
```

### Step 4: PM Approval

PM reviews:
- [ ] Errors are well-defined (error_code + message)
- [ ] Pagination strategy is clear (if applicable)
- [ ] Versioning path is explicit
- [ ] Backwards compatibility promise is documented

## Handoff (Next in the SDLC Chain)

Once the API contract is approved by the PM, always run the **security-baseline** skill next — before any implementation begins — to validate PII handling, secrets management, auth/authz, compliance scope, and dependency CVEs against the contract you just defined.

Tell the user the skill is firing and why, e.g.:
> "API contract locked. Before we write a line of implementation, I am running the Security Baseline to make sure the endpoints we just specified handle PII and auth correctly."
