---
name: security-baseline
description: "Ensure minimum security practices before shipping any code. Use for any new service, API, or authentication flow. Checks: encryption, secrets management, PII handling, auth/authz, compliance scope, and dependency vulnerabilities."
---

# Security Baseline

## When to Trigger

Before implementation of:
- Authentication/authorization systems
- Any code handling PII
- APIs with external consumers
- Payment/financial flows
- Public-facing services

## What It Does

Claude validates security posture across 5 dimensions:

### 1. Secrets Management
✓ No secrets in code
✓ Secrets sourced from [env vars / vault / managed service]
✓ Rotation strategy: [every N days]
✓ Audit logging: [who accessed this secret?]

### 2. Encryption
PII fields:
- [ ] Identified (what is PII in this system?)
- [ ] At rest: [AES-256 via X / managed service]
- [ ] In transit: [TLS + cert pinning where needed]
- [ ] Key management: [where are keys stored? rotation?]

### 3. Authentication/Authorization
- [ ] Auth model defined: [OAuth2 / API key / JWT / session]
- [ ] Scope boundaries: [what endpoints need auth?]
- [ ] Error handling: [don't leak whether user exists]
- [ ] Rate limiting: [protect against brute force]

### 4. Compliance Scope
If applicable:
- [ ] GDPR: [PII handling, retention, deletion]
- [ ] SOX: [audit trails, access controls]
- [ ] HIPAA: [data classification, encryption]
- [ ] PCI-DSS: [card data handling]

### 5. Dependency Vulnerabilities
- [ ] All dependencies checked against CVE database
- [ ] No critical CVEs
- [ ] Transitive dependencies audited
- [ ] Update strategy: [how often?]

## Guardrail Evals (Auto-Block)

Claude flags and requires PM + Security review if:
- ✗ PII detected without encryption spec
- ✗ Secrets in code/config
- ✗ Compliance scope stated but not addressed
- ✗ Critical CVE in dependencies
- ✗ Auth boundary undefined
- ✗ Error messages leak sensitive info

## Approval Workflow

Claude surfaces proposal → Dee **+ Security/Legal review** → Approve/reject/revise

## Handoff (End of the Design Chain → Build)

The security baseline is the final design gate. Once the PM (plus Security/Legal where required) approves:

**All specs are locked: MVP scope (brainstorm) → architecture (architecture-checkpoint) → [eval design] → [API contract] → security baseline.**

Proceed to implementation. Tell the user explicitly that the design chain is complete and what they're building against, e.g.:
> "Security baseline approved. The full design chain is locked — MVP scope, architecture, eval loops, API contract, and security are all signed off. Ready to build against these specs. Want me to start implementation?"

During the build, the remaining quality gates (test strategy, self code review) run as HOTL checks — Claude proposes, you spot-check — rather than as blocking design gates.

**Important — security-baseline is a DESIGN-time gate. It validated the plan, not the shipped code.** Before anything is deployed, migrated, or made public, the **deploy-gate** skill must run. It re-scans the actual built artifact for leaked secrets/PII/exposed endpoints, audits the installed dependency tree, and hard-stops for explicit human approval on irreversible actions (production deploy, schema migration) — even if Claude Code is in bypass-permissions mode. Tell the user:
> "Design-time security is approved and we're clear to build. When the build is ready to ship, the Deploy Gate will run before any push to Railway/Supabase/production — staging proceeds autonomously, but production deploy and DB migration will stop for your explicit approval."
