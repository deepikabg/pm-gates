---
name: architecture-checkpoint
description: "Validate proposed system architecture against PM constraints before implementation. Use when Claude is about to propose a full system design or major architectural change. Checks for: constraint alignment, scalability assumptions, database/cache strategy, error handling, observability, cost estimates, and explicit trade-offs. Always triggers when designing a new service, migration, or AI-native system."
---

# Architecture Checkpoint

The gate between "we know what to build" and writing code. Its job is to make the architecture's trade-offs **explicit and constraint-justified** before they harden into code — because the expensive mistakes (wrong data model, missing eval loop, an under-spec'd failure mode) are cheap to fix on a whiteboard and brutal to fix in production. The discipline: every choice is tied to a stated constraint, every alternative is named, every assumption is surfaced.

## When to Trigger
- New service / system design
- Major refactoring that breaks existing patterns
- AI-native systems (RAG, agents, eval loops)
- Any system expected to scale beyond the initial MVP

## When NOT to Trigger
- A throwaway script or one-off (frequency lens: rare + low-risk → don't architect it)
- A localized change inside an already-approved architecture
- A prototype explicitly meant to be thrown away (note that architecture is deferred and why)

## Scale to the build
Match the rigor to the stakes — don't make a weekend project file a cost model.
- **High-stakes / scaled / AI-native** → the full checklist below, including the eval/feedback service.
- **Standard internal tool** → constraint alignment, data layer, failure modes, observability; skip the deep cost modeling.
- **Prototype** → name the one architectural bet that would be expensive to reverse, and move on.

## What It Validates
1. **Constraint alignment** — does the design respect the PM's stated constraints?
2. **Scalability** — what throughput/latency is achievable, and when does it break?
3. **Data layer** — schema, caching, consistency model
4. **Failure modes** — how does the system degrade under error?
5. **Observability** — what must be instrumented?
6. **Cost** — cloud spend estimate
7. **Eval loop (AI-native only)** — is the eval/feedback service a first-class component? (see Intake)
8. **AI-native runtime (AI-native only)** — token cost (usually the dominant line), latency variance, model-failure fallback, version pinning, idempotent retries

## Workflow

### Step 1: Gather Context (Claude asks)
Before proposing architecture, get:

1. **The primary constraint** (pick ONE):
   - Startup: team size [N], prefer monolith, launch in [weeks]
   - Scaled: throughput [req/sec], team [N], prefer distributed
   - Open-source: API stability required, community contributions
   - AI-native: eval loops embedded, RAG pipeline, agent orchestration
2. **The success metric** (throughput, latency, cost, time-to-launch)
3. **The hard constraints** (regulatory, team size, timeline)

### Step 2: Claude Proposes (structured output)

```
## Architecture Proposal: [System Name]

### Constraint Alignment
✓ Team size: [N] eng → [why monolith vs. distributed]
✓ Launch timeline: [weeks] → [why this trades speed for scalability]
✓ Regulatory scope: [GDPR/SOX/none] → [how architecture handles it]

### Data Layer
- Primary DB: [choice + rationale]
- Schema: [key tables + rationale]
- Caching: [strategy, if needed]
- Consistency model: [immediate/eventual + when each]
- Trust boundaries & tenancy: [where PII lives; how tenants are isolated — e.g., RLS]

### Scalability
- Current: [P50/P95/P99 latency at target req/sec — name the workload shape, read/write ratio]
- Statefulness: [stateless services + externalized state? what holds session/memory?]
- Bottleneck: [where does it break — compute, DB, or model throughput?]
- Scaling trigger: [throughput at which to split services]
- Timeline to re-architect: [if constraints change]

### Failure Modes
- Database down: [degradation mode]
- External API timeout: [fallback]
- **Model API timeout / outage (AI-native):** [fallback — cached response, smaller/cheaper model, queue, or graceful refusal]
- Rate limit hit: [queue strategy]
- Retries: [exponential backoff + idempotent ("safe-to-retry") operations so retries don't double-charge/double-write]
- Observability: [what you must track]

### Eval / Feedback Service  (AI-native only)
- Eval service: [where it runs — inline/async/batch, latency budget]
- Feedback loop: [correction path → storage → update cadence]
- Live metrics: [what's on the dashboard]
(Must match the runtime eval loop specified in the Eval Spec, Step 7.)

### AI-Native Runtime  (AI-native only)
- Model + version pinning: [which model; how prompt/model versions are pinned and rolled]
- Latency budget: [P50/P99 of the model call — LLM calls are the slowest, highest-variance hop]
- Token-cost control: [prefix/context caching; smaller models for routine steps; prompt/context compaction; batching]
- Provisioned throughput vs. scale-to-zero: [for latency-critical vs. bursty workloads]

### Cost Estimate
- **AI-native: token cost per request × volume is usually the dominant line — model it first**, before compute.
- Compute: $[X]/month (baseline) · Database: $[X]/month · Tokens: $[X]/month (cost per request × volume)
- At 10x throughput: $[Y]/month · Scaling cliff: [is there one?]

### Alternatives Considered
1. [Alternative A] — Pros / Cons
2. [Alternative B] — Pros / Cons
Why I recommended Option 1: [reasoning tied to constraints]

### Diagram
[ASCII or textual architecture diagram]

### Open Assumptions
- [ ] Assumption: [X] (affects: [what if wrong?])
- [ ] Assumption: [Y] (affects: [what if wrong?])
```

### Step 3: PM Review Gate

```
✓ Approve as-is
⚠️ Request changes (specific feedback)
✗ Reject, re-propose
```
If ⚠️ or ✗: incorporate feedback, return to Step 2.

### Step 4: Feedback Capture
If the PM requested changes, categorize so corrections compound:
```
- Category: [over-engineered / missing pattern / wrong tech / scale mis-estimate]
- Original assumption: [what Claude assumed]
- Corrected assumption: [what the PM said]
- Impact: [why this matters]
```

## Guardrail Evals (Auto-Block)
Flag and re-propose if the design:
- ✗ Proposes a deprecated framework (EOL < 1 year)
- ✗ Includes no database/persistence
- ✗ Has no error-handling strategy
- ✗ Has no observability plan
- ✗ Is missing the throughput calc
- ✗ Violates the stated constraint (e.g., microservices for a 2-person startup)
- ✗ Is AI-native but has **no eval/feedback service** in the design
- ✗ Is AI-native but has **no token-cost estimate** or **no model-failure fallback**
- ✗ Has retried/async operations that aren't **idempotent** (double-charge / double-write risk)

## Success Metrics (is this skill earning trust?)
- [ ] Approval rate: 80%+ approved without major changes
- [ ] Tech debt: zero architecture-driven incidents in first 3 months
- [ ] Feedback loop: corrections < 2 per proposal within 4 weeks
- [ ] Time-to-approval: < 15 min for PM review

## Intake (What Feeds This Skill)
For AI-native or high-stakes builds, this skill should receive an **Eval Spec** (from `eval-spec`) alongside the Brainstorm Brief. The eval spec's Step 7 specifies the runtime eval loop — which means the architecture **must include the eval/feedback service as a first-class component**, not an afterthought. If no eval spec exists yet for an AI-native build, pause and run `eval-spec` first.

## Handoff
Once the architecture is approved, write `Architecture.md` + a `passed` entry to `.pipeline/state.yaml`, then **signal loop-orchestrator — it owns routing; this skill never chooses the next gate.** (Per the orchestrator's rules, the contract gates run next: api-contract-definition where the system exposes APIs or service boundaries, then security-baseline — always — before any implementation.)

(The eval-loop check that used to live in a separate gate is now folded in here, validated against the Eval Spec — there's no separate step to forget.)

Say, e.g.:
> "Architecture approved — and since it's AI-native, I confirmed the eval/feedback service is a first-class component per the eval spec. Logged to state — handing to the orchestrator for the next gate."

## Anti-Patterns
- ❌ Choices with no rationale → ✅ every choice tied to a stated constraint
- ❌ One option presented as the answer → ✅ alternatives named with pros/cons and why the winner won
- ❌ Hidden assumptions → ✅ surfaced in "Open Assumptions" with "what if wrong?"
- ❌ Microservices for a 2-person startup → ✅ rigor scaled to constraint (start simple, name the scaling trigger)
- ❌ AI-native design with evals "added later" → ✅ eval/feedback service is first-class, per the eval spec
- ❌ Costing servers while ignoring tokens → ✅ token cost modeled first; caching + right-sized models named
- ❌ Assuming the model API is always up and fast → ✅ fallback + a P99 latency budget for the model hop
- ❌ Retries with no idempotency → ✅ safe-to-retry operations (idempotency keys / dedupe)
- ❌ "It'll scale fine" → ✅ a stated bottleneck and the throughput at which it breaks
