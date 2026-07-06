---
name: security-baseline
description: "Ensure minimum security practices before shipping any code. Use for any new service, API, or authentication flow. Checks: encryption, secrets management, PII handling, auth/authz, compliance scope, and dependency vulnerabilities."
---

# Security Baseline

The **design-time** security gate: it forces the security decisions that are nearly free to make on paper and expensive to retrofit — what's PII, how secrets are sourced, where the auth boundary sits, what compliance scope applies. It validates the *plan*, not the shipped code (that's `deploy-gate`'s job). The stance: security is a design input, not a pre-launch scramble. If a dimension can't be answered, the design isn't done.

## When to Trigger
Before implementation of:
- Authentication / authorization systems
- Any code handling PII
- APIs with external consumers
- Payment / financial flows
- Public-facing services

## When NOT to Trigger
- A local-only script with no PII, no network surface, no secrets
- A throwaway prototype on synthetic data (note that the security gate is deferred until it touches real data/users)
- A change wholly inside an already-baselined boundary that adds no new data, endpoint, or dependency

## Scale to the build
- **Auth / payments / PII / public services** → all dimensions, plus Security/Legal review where required.
- **AI-native (agents, RAG, tool-calling)** → dimensions 1–5 **plus dimension 6 (AI-native threats) is mandatory**, not optional.
- **Internal tool, no sensitive data** → secrets management + dependency check; document why encryption/compliance are out of scope.
- **Prototype on synthetic data** → confirm no real secrets/PII present; defer the rest with a written reason.

## Threat model first (one pass before the dimensions)
Don't jump to controls — name what you're defending. One quick pass:
- **Assets:** what's worth stealing/corrupting here (PII, money movement, credentials, model context)?
- **Attackers & surface:** who (external user, malicious tenant, compromised dependency, hostile content fed to an LLM) and through what entry point?
- **Trust boundaries:** where does untrusted data cross into trusted execution? (user input, third-party APIs, retrieved documents, tool outputs.)
The dimensions below are the controls; this pass decides which ones matter most.

## The Dimensions

### 1. Secrets Management
- ✓ No secrets in code
- ✓ Secrets sourced from [env vars / vault / managed service]
- ✓ Rotation strategy: [every N days]
- ✓ Audit logging: [who accessed this secret?]

### 2. Encryption
PII fields:
- [ ] Identified (what is PII in this system?)
- [ ] At rest: [AES-256 via X / managed service]
- [ ] In transit: [TLS + cert pinning where needed]
- [ ] Key management: [where are keys stored? rotation?]

### 3. Authentication / Authorization
AuthN (who you are) ≠ AuthZ (what you may do). Most real breaches are AuthZ failures.
- [ ] **AuthN** model defined: [OAuth2 / API key / JWT / session]
- [ ] **Object-level AuthZ (the #1 API risk — IDOR/BOLA):** every access checks *this caller owns this object*, not just "is logged in." `GET /orders/123` must verify 123 belongs to the caller.
- [ ] **Multi-tenant isolation:** how is tenant A prevented from reading tenant B's rows? (e.g., **Supabase RLS / row-level policies**, enforced in the DB, default-deny — not just in app code.)
- [ ] **Least privilege:** each principal (user, service, **agent**) gets only the scopes its job needs; deny by default.
- [ ] Error handling: [don't leak whether a user exists]
- [ ] Rate limiting: [brute-force protection **and** cost-DoS protection on expensive endpoints]

### 4. Compliance Scope
If applicable:
- [ ] GDPR: [PII handling, retention, deletion]
- [ ] SOX: [audit trails, access controls]
- [ ] HIPAA: [data classification, encryption]
- [ ] PCI-DSS: [card data handling]

### 5. Dependency Vulnerabilities
- [ ] All dependencies checked against the CVE database
- [ ] No critical CVEs
- [ ] Transitive dependencies audited
- [ ] Update strategy: [how often?]

### 6. AI-Native Threats (mandatory for any agent / RAG / tool-calling system)
The model itself is an attack surface. You **cannot** rely on the model's judgment — defense-in-depth with deterministic guardrails *outside* the model is the baseline.
- [ ] **Prompt injection:** treat *all* external content as untrusted — user input, retrieved documents, tool outputs, third-party data. Separate system instructions from user/retrieved content; consider trusted vs. untrusted planner separation.
- [ ] **Treat model output as untrusted:** never `eval`/exec it, never render it as raw HTML/Markdown, validate it before it drives a tool call or DB write. Output-filter for leaked secrets, PII, tokens, and active content (HTML/JS/URLs) before it reaches a user or downstream system.
- [ ] **Tool-call authorization:** the tool's own logic enforces policy — "no matter what the model reasons or a malicious prompt suggests, the tool refuses out-of-policy actions." Allowlist tools/resources; least-privilege per tool.
- [ ] **Irreversible-action chokepoint:** deterministic rule outside the model requires explicit human confirmation for high-stakes actions (move money, delete data, external writes). Pairs with `deploy-gate`.
- [ ] **Data exfiltration / context leakage:** what enters the prompt and conversation context? Keep PII out of prompts and logs; don't let proprietary data reach third-party tools/servers.
- [ ] **Agent identity & blast radius:** if agents act, each gets its own scoped, least-privilege identity (not a shared broad API key) so a compromised agent is contained.
- [ ] **Classic input/output handling still applies:** validate input (path traversal, SQLi), encode output (XSS), guard server-side request forgery (SSRF) on any URL the system fetches.

## Guardrail Evals (Auto-Block)
Flag and require PM + Security review if:
- ✗ PII detected without an encryption spec
- ✗ Secrets in code/config
- ✗ Compliance scope stated but not addressed
- ✗ Critical CVE in dependencies
- ✗ AuthN defined but **object-level AuthZ / tenant isolation undefined** (IDOR risk)
- ✗ Error messages leak sensitive info
- ✗ AI-native system with **no prompt-injection handling** or that **trusts raw model output** (renders/executes it)
- ✗ Agent can invoke a privileged or irreversible tool with **no authorization check outside the model**

## Approval Workflow
Claude surfaces the proposal → PM (plus Security/Legal where required) reviews → approve / reject / revise.

## Handoff
The security baseline is the final *design* gate. Once the PM (plus Security/Legal where required) approves, write `Security-Baseline.md` + a `passed` entry to `.pipeline/state.yaml`, then **signal loop-orchestrator — it owns routing; this skill never chooses the next gate.** (Per the orchestrator's rules, the spec-readiness check and then **story-map** run next — the design is decomposed into a dependency-ordered, traceable build plan before any story is built. Implementation starts only from that approved plan.)

**All design specs are now locked: MVP scope (brainstorm) → eval spec → prototype (triad-approved) → architecture → [API contract] → security baseline.** Say, e.g.:
> "Security baseline approved — the full design chain is locked and logged to state. Handing to the orchestrator: next is the spec-readiness check, then story-map breaks this into the dependency-ordered build plan for your approval."

During the build, the remaining quality gates (test strategy, self code review) run as human-on-the-loop checks — Claude proposes, you spot-check — not as blocking design gates.

**Important — this is a DESIGN-time gate. It validated the plan, not the shipped code.** Before anything is deployed, migrated, or made public, the **deploy-gate** skill must run: it re-scans the actual built artifact for leaked secrets/PII/exposed endpoints, audits the installed dependency tree, and hard-stops for explicit human approval on irreversible actions (production deploy, schema migration) — even in bypass-permissions mode. Tell the user:
> "Design-time security is approved and we're clear to build. When the build is ready to ship, the Deploy Gate runs before any push to production — staging proceeds autonomously, but production deploy and DB migration will stop for your explicit approval."

## Anti-Patterns
- ❌ Security as a pre-launch checklist → ✅ security decided at design time, as an input
- ❌ "We'll figure out PII later" → ✅ PII fields identified and encryption specified now
- ❌ Compliance scope named but not addressed → ✅ each named regime has concrete handling
- ❌ Checking only direct dependencies → ✅ transitive tree audited too
- ❌ Auth bolted onto specific endpoints ad hoc → ✅ one defined auth model + explicit scope boundary
- ❌ "Logged in" treated as "authorized" → ✅ object-level AuthZ + tenant isolation (RLS), default-deny
- ❌ Trusting the model to police itself against injection → ✅ deterministic guardrails outside the model; treat all external content and all model output as untrusted
- ❌ Agents holding a broad shared API key → ✅ per-agent least-privilege identity, tool-side authz
- ❌ Treating this as the last security step → ✅ deploy-gate re-checks the built artifact before it ships
