---
name: eval-integration-gate
description: "For AI-native systems (RAG, agents, agentic commerce): ensure eval loops are embedded in architecture, not bolted on. Use when designing multi-agent systems, RAG pipelines, or any system that must measure its own correctness."
---

# Eval Integration Gate

## When to Trigger

Use when Claude is designing:
- Multi-agent systems (orchestrator + worker agents)
- RAG pipelines with vector stores
- Agentic systems that make decisions or generate content
- Any system where "good output" is probabilistic, not deterministic

## What It Does

Ensures eval loops are **first-class** in the architecture:
- Eval scoring is a service, not a post-hoc script
- Feedback (human corrections) flows back to model/prompt immediately
- Eval metrics are live dashboards, not offline reports

## Workflow

### Step 1: Identify Eval Surface

Claude asks:
```
For this AI system, what must be true for it to work?

1. Decision surface: What decisions is the AI making?
   - Recommendations? (ranking, filtering)
   - Generation? (writing, coding)
   - Classification? (routing, tagging)
   - Routing? (which agent gets this)

2. Failure modes: What would be unacceptable?
   - Hallucination? (AI invents facts)
   - Scope overreach? (answers outside scope)
   - Bias? (systematically wrong for segment X)
   - Format non-compliance? (output doesn't match downstream)

3. Success metric: How do we measure correctness?
   - Automated? (ground truth dataset)
   - Human-rated? (rubric + raters)
   - Business metric? (conversion, user satisfaction)
```

### Step 2: Claude Proposes Eval Architecture

```
## Eval Integration Design

### Eval Service
- Input: [system output to evaluate]
- Output: [score 0-1, error category, confidence]
- Latency: [max acceptable]
- Where it runs: [inline? async? batch?]

### Feedback Loop
- Human correction input: [UI/API for corrections]
- Feedback storage: [database schema]
- Correction categorization: [taxonomy of error types]
- Feedback → improvement: [weekly prompt/data update]

### Dashboard
- Weekly metrics:
  - Eval score over time (trending up?)
  - Error categories (are we fixing the most common?)
  - Feedback velocity (how many corrections this week?)
  - Segment performance (do any segments score < threshold?)

### Guardrails
- Score threshold: [if < X%, escalate to human]
- Confidence threshold: [if > Y%, auto-approve; else route to human]
- Anomaly detection: [if eval score drops > 10% week-over-week, alert]

### Failure Modes
- Eval service down: [fallback behavior]
- Feedback queue full: [drop or buffer?]
- No ground truth available: [how to measure correctness?]
```

### Step 3: PM Approval

PM reviews:
- [ ] Eval is embedded (not separate)
- [ ] Feedback loop is defined (not "we'll measure later")
- [ ] Success metric is explicit (not fuzzy)


## Handoff (Next in the SDLC Chain)

Once the eval architecture is approved by the PM, continue down the chain:

1. **If the system exposes APIs or has service-to-service boundaries** → use the **api-contract-definition** skill next, to lock versioning, pagination, and error schemas before implementation.

2. **Otherwise (or after the API contract is defined), always before implementation** → use the **security-baseline** skill to validate PII handling, secrets, auth, compliance scope, and dependency CVEs.

Tell the user which skill is firing next and why, e.g.:
> "Eval loops are designed in. This system exposes a research-output API, so I'm running API Contract Definition next to lock the schema before we build."
