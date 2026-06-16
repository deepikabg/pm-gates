---
name: architecture-checkpoint
description: "Validate proposed system architecture against PM constraints before implementation. Use when Claude is about to propose a full system design or major architectural change. Checks for: constraint alignment, scalability assumptions, database/cache strategy, error handling, observability, cost estimates, and explicit trade-offs. Always triggers when designing a new service, migration, or AI-native system."
---

# Architecture Checkpoint

## When to Use This Skill

Claude should auto-trigger this **before** proposing architecture for:
- New service/system design
- Major refactoring (breaking existing patterns)
- AI-native systems (RAG, agents, eval loops)
- Any system expected to scale beyond initial MVP

## What It Does

Claude validates proposed architecture against:
1. **Constraint alignment** — Does design respect PM's stated constraints?
2. **Scalability** — What throughput/latency is achievable? When does it break?
3. **Data layer** — Schema, caching, consistency model
4. **Failure modes** — How does system degrade under error?
5. **Observability** — What must be instrumented?
6. **Cost** — Cloud spend estimate

## Workflow (Claude + PM)

### Step 1: Gather Context (Claude asks)
Before I propose architecture, I need context:

1. What's the constraint? (Pick ONE primary)
   - Startup: team size [N], prefer monolith, launch in [weeks]
   - Scaled: throughput [req/sec], team [N], prefer distributed
   - Open-source: API stability required, community contributions
   - AI-native: eval loops embedded, RAG pipeline, agent orchestration

2. What's the success metric? (throughput, latency, cost, time-to-launch)

3. What are the hard constraints? (regulatory, team size, timeline)
```

### Step 2: Claude Proposes (Structured Output)

Claude returns:
```
## Architecture Proposal: [System Name]

### Constraint Alignment
✓ Team size constraint: [N] eng → [why monolith vs. distributed]
✓ Launch timeline: [weeks] → [why this trades speed for scalability]
✓ Regulatory scope: [GDPR/SOX/none] → [how architecture handles it]

### Data Layer
- Primary DB: [choice + rationale]
- Schema: [key tables + rationale]
- Caching: [strategy, if needed]
- Consistency model: [immediate/eventual + when each]

### Scalability
- Current: [P95 latency at 1k req/sec]
- Bottleneck: [where does it break?]
- Scaling trigger: [throughput at which to split services]
- Timeline to re-architect: [if constraints change]

### Failure Modes
- Database down: [degradation mode]
- External API timeout: [fallback]
- Rate limit hit: [queue strategy]
- Observability: [what you must track]

### Cost Estimate
- Compute: $[X]/month (at baseline throughput)
- Database: $[X]/month
- At 10x throughput: $[Y]/month
- Scaling cliff: [is there one?]

### Alternatives Considered
1. [Alternative A] — Pros: [...], Cons: [...]
2. [Alternative B] — Pros: [...], Cons: [...]

Why I recommended Option 1:
[Explicit reasoning tied to constraints]

### Diagram
[ASCII or textual architecture diagram]

### Open Assumptions
- [ ] Assumption: [X] (affects: [what if wrong?])
- [ ] Assumption: [Y] (affects: [what if wrong?])
```

### Step 3: PM Review Gate

PM (Dee) reviews:
```
✓ Approve as-is
⚠️ Request changes (specific feedback)
✗ Reject, re-propose
```

If ⚠️ or ✗: Claude incorporates feedback, returns to Step 2.

### Step 4: Feedback Capture

If PM requested changes:
```
Feedback categorized:
- Category: [over-engineered / missing pattern / wrong tech / scale mis-estimate]
- Original assumption: [what Claude assumed]
- Corrected assumption: [what PM said]
- Impact: [why this matters]
```

This feedback is logged and reviewed weekly.

## Guardrail Evals (Auto-Block)

Claude should flag and re-propose if:
- ✗ Proposes deprecated framework (EOL < 1 year)
- ✗ Includes no database/persistence
- ✗ Missing error handling strategy
- ✗ No observability plan
- ✗ Throughput calc missing
- ✗ Violates constraint (e.g., monolith for startup becomes microservices)

## Success Metrics

For PM to trust this skill:
- [ ] Approval rate: 80%+ of proposals approved without major changes
- [ ] Tech debt: Zero architecture-driven incidents in first 3 months
- [ ] Feedback loop: Corrections < 2 per proposal within 4 weeks
- [ ] Time-to-approval: < 15 min for PM review

## Intake (What Feeds This Skill)

For AI-native or high-stakes builds, this skill should receive an **Eval Spec** (from the `eval-spec` skill) alongside the Brainstorm Brief. The eval spec tells you what the system must measurably achieve — which means the architecture must include an **eval/feedback service** as a first-class component (per the PM architecture checklist), not an afterthought. If no eval spec exists yet for an AI-native build, pause and run `eval-spec` first.

## Handoff (Next in the SDLC Chain)

Once the architecture is approved by the PM, route to the next skill based on what the system actually contains. Evaluate in this order and trigger the FIRST that applies, then continue down the chain:

1. **If the system is AI-native** (RAG pipeline, agents, agentic workflows, anything with probabilistic outputs that must measure its own correctness) → use the **eval-integration-gate** skill next, to ensure eval loops are embedded in the architecture rather than bolted on later.

2. **If the system exposes APIs or has service-to-service boundaries** (public endpoints, webhooks, internal service contracts) → use the **api-contract-definition** skill, to lock versioning, pagination, and error schemas before implementation.

3. **Always, before any implementation begins** → use the **security-baseline** skill to validate PII handling, secrets, auth, compliance scope, and dependency CVEs.

These are not mutually exclusive — an AI-native product with APIs runs eval-integration-gate, THEN api-contract-definition, THEN security-baseline. A simple internal tool with no APIs may skip straight to security-baseline.

Tell the user which skill is firing next and why, e.g.:
> "Architecture approved. Since this is an AI-native system with a RAG pipeline, I'm running the Eval Integration Gate next to make sure eval loops are part of the design — not an afterthought."
