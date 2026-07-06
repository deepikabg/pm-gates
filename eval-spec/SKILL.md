---
name: eval-spec
description: |
  Define a rigorous, per-feature/per-dimension evaluation spec BEFORE building — the deliverable that turns "what good looks like" into measurable, testable contracts. Use immediately after the MVP scope is locked (after brainstorm, before or alongside architecture) for any AI-native or high-stakes build. This skill classifies every feature/dimension as deterministic test vs. probabilistic eval vs. guardrail, runs error-analysis-first taxonomy design, builds gold-standard datasets, writes binary pass/fail criteria with rubrics for subjective dimensions, defines LLM-as-judge / trajectory evaluation for agents, slices evals by user segment to catch bias, and sets ship/no-ship thresholds. Triggers on "write evals", "what should we measure", "how do we know it works", "eval set", "test criteria", "before we build", or when a build is about to start without defined success criteria. If you can't write the eval, the problem isn't understood well enough to build.
---

# Eval Spec

Evals are the product spec. This skill produces the eval set *before* the build, so the build has a measurable target instead of a vibe. Grounded in an AI-native eval playbook + Google's Agent Quality framework + the Husain/Shankar error-analysis method (see `references/eval-craft.md`).

## Core stance

- **Evals make failure legible and measurable.** The shortcut: *"if this shipped and went wrong, what screenshot ends up in a Slack escalation thread?"* Those are your eval cases.
- **Error analysis is the most important activity** — it decides *what evals to write in the first place*. Don't start from generic metrics; start from real (or realistic) failures.
- **If you can't write the eval, you don't understand the problem well enough to build it.** That's a signal to bounce back to brainstorm — caught here, cheaply, before architecture.

## Placement in the chain

Runs **after brainstorm locks the MVP**, **before/alongside architecture**. Brainstorm = what & why. Eval-spec = what "working" measurably means. Architecture = how. This ordering means an undefinable eval bounces you back one cheap step (problem clarity), not two expensive ones (rebuild). Revisit lightly after architecture to attach evals to concrete components.

## Scale to the build (don't over-apply)

- **AI-native / high-stakes features** → full eval spec (this whole skill).
- **Deterministic, low-stakes features** → a short test list, not an eval suite.
- **Throwaway prototype** → skip; note that evals are deferred and why.
Pull the **frequency lens** through: a daily-fired AI decision earns a real gold-standard set; a monthly deterministic job earns one smoke test. Don't write 50 evals for something that runs twice a year.

---

## The Method

### Step 1 — Inventory features/dimensions, then CLASSIFY each

List every feature and quality dimension the build must satisfy. Classify each into exactly one bucket — this decides the instrument and prevents both theater and false assurance:

| Bucket | What it is | Instrument |
|---|---|---|
| **Deterministic test** | One correct answer; verifiable (CRUD works, auth blocks, form validates, math is right) | Traditional unit/integration test, pass/fail |
| **Probabilistic eval** | Open-ended quality; many valid outputs (is this summary good? did the agent route right? is retrieval relevant?) | Gold-standard dataset + binary criteria + rubric / LLM-as-judge |
| **Guardrail (must-never)** | Catastrophic if it ever happens (PII leak, toxic output, unauthorized action, hallucinated citation in legal/medical) | Red-team suite, pass/fail, zero-tolerance |

**Trap to avoid:** don't write fuzzy "evals" for deterministic features (overhead) or pass/fail "tests" for probabilistic ones (false assurance — a 99% score means the eval is too easy, not that the system is excellent).

### Step 2 — Error analysis FIRST (build the failure taxonomy)

Don't invent metrics top-down. Derive them from failures:

1. **Get traces.** Real interactions if any exist; otherwise generate **structured synthetic data via dimensions** (e.g., for a recipe app: Dietary Restriction × Cuisine × Query Complexity → tuples → natural-language queries). Structured beats "give me test queries."
2. **Open coding.** One domain expert (a "benevolent dictator") reviews 20–50 (scale to 50–100 for production) traces and writes open-ended notes on what's wrong — qualitative journaling, no fixed categories yet.
3. **Axial coding → taxonomy.** Cluster notes into a failure taxonomy that is: **mutually exclusive** (a failure fits one category), **mechanism-distinct** (can't collapse two), and **root-cause not symptom** ("retrieved wrong context," not "hallucination").
4. **Prioritize by frequency × impact.** Fix obvious bugs immediately (don't write an eval for something a one-line prompt fix solves). Write evals for the high-frequency/high-impact categories that remain.
5. **The taxonomy also becomes executable state:** each high-frequency data-corruption category (duplicate records, empty/partial maps, half-migrated rows) ships as a **dirty-state fixture** (`tests/fixtures/dirty/`) that qa-verify re-runs the P0 journeys against — the failure catalog tested as *starting conditions*, not just as outputs.

### Step 3 — For PROBABILISTIC features: gold standard + binary criteria

- **Gold-standard dataset:** 50–200 hand-curated cases that reflect *real* scenarios (not sanitized demos), covering typical + edge + adversarial. The PM/domain expert writes these — this forces clarity on what "good" means and prevents eval theater. Quality over quantity: 50 sharp cases beat 10,000 synthetic ones.
- **Binary pass/fail, not 1–5 Likert.** Prefer clear pass/fail criteria — Likert scales bury disagreement and drift. If a dimension feels inherently scalar, decompose it into specific binary questions ("Does the summary include the decision? Y/N", "Does it omit PII? Y/N").
- **Rubric** for any judged dimension: explicit, with reasoning required.

### Step 4 — For AGENTS: evaluate the trajectory, not just the output

(Google Agent Quality — "the trajectory is the truth": an agent can produce a right-looking answer via broken reasoning.)
- **Outside-In:** start with end-to-end (black box) — did the final outcome meet the goal? Then go **Inside-Out** (glass box) — inspect the execution trace.
- **Four Pillars** to spec per agent: **Effectiveness** (did it achieve the goal), **Efficiency** (tokens/cost, latency, # steps), **Robustness** (graceful under API timeout, ambiguous input, missing data — retries, asks, reports vs. crashes/hallucinates), **Safety** (the non-negotiable gate: stays in bounds, refuses harmful, resists prompt injection / data leakage).
- **LLM-as-Judge** for qualitative outputs and intermediate "thoughts": give the judge the input, output, golden reference (if any), and a detailed rubric; require reasoning. Use it for *hypothesis/pattern detection*, never as the final ship decision.
- **Agent-as-Judge** for process: plan quality, tool selection/arguments, context handling.
- Metrics are the cheap **first filter** to catch obvious failures at scale before escalating to LLM-judge then human.

### Step 5 — Slice by segment (bias is a product risk, not an ethics appendix)

Global averages hide segment failures. For each eval, define slices by the sensitive/important segments for this product (user type, geo, language, content type). Ask: **"Who is most likely to be harmed by a wrong output here?"** Add eval slices for those segments, not just aggregate scores. Watch for distribution/representation/objective bias (model) and where human reviewers might amplify bias (selection/confirmation/attention).

### Step 6 — Thresholds, HITL tie-in, and instrumentation

- **Ship/no-ship threshold per dimension:** the score that gates release (e.g., "≥90% recall on high-risk cases", "0 guardrail violations on red-team suite", "median expert pass on gold set"). Suspiciously high (99%+) = eval too easy, not system excellent.
- **Tie thresholds to HITL level by risk:** high-risk → human approves (and for high-stakes tool calls, an interruption/approval step before execution); medium → AI acts, humans sample + handle escalations; low → AI acts with monitoring.
- **Instrumentation to spec now** (so it's built in, not bolted on): online metrics (acceptance/override rates, per-segment performance, error rates), offline (eval scores, calibration, drift), and the incident→eval-case flow. New systems: re-run error analysis weekly until patterns stabilize; mature: monthly + after every incident/complaint-spike/drift.

### Step 7 — Specify the runtime eval loop (build it in, don't bolt it on)

The spec above defines *what* to measure. This step defines the *living system* that measures it in production — so the architecture treats evals as a first-class component, not a post-hoc script. For AI-native systems (RAG, agents, agentic commerce), specify these now; `architecture-checkpoint` then verifies they're actually in the design.

- **Eval service (not a script).** Specify it as a real component: input = the system output to evaluate; output = score + error category + confidence; where it runs = inline / async / batch; max acceptable latency. "We'll run a notebook later" is the failure this prevents.
- **Feedback loop (closed, not aspirational).** Where do human corrections enter (UI/API)? How are they stored (schema)? How are they categorized (reuse the failure taxonomy from Step 2)? And the cadence by which corrections become prompt/data/model updates. A feedback loop with no path back to the system is just logging.
- **Live dashboard (not an offline report).** Eval score over time (trending up?), error categories (are the most common ones shrinking?), feedback velocity, and per-segment performance against threshold. If the only time anyone sees eval numbers is in a quarterly deck, the loop isn't running.
- **Runtime guardrails.** Score threshold → escalate to human below it; confidence threshold → auto-approve above it, else route to human; anomaly detection → alert if eval score drops week-over-week beyond a set delta.
- **Loop failure modes.** Eval service down → fallback behavior? Feedback queue full → drop or buffer? No ground truth available → how is correctness measured at all?

**Gate check:** the eval loop is specified as buildable architecture (service + feedback path + dashboard + guardrails), tied to the gold set and taxonomy above — ready for `architecture-checkpoint` to design in.

---

## Output: The Eval Spec

```markdown
# Eval Spec: [Project / Feature]
Linked MVP: [from Brainstorm Brief]   AI-native: [y/n]

## Feature/Dimension Inventory & Classification
| Feature/Dimension | Bucket (test/eval/guardrail) | Frequency | Instrument |
|---|---|---|---|

## Failure Taxonomy (from error analysis)
| Category (root-cause) | Mechanism | Freq×Impact | Bucket | Eval that catches it |
|---|---|---|---|---|

## Critical User Journeys (qa-verify's post-deploy script)
| # | Journey (end-to-end user path) | Steps | Binary pass criterion | Priority |
|---|---|---|---|---|

## Gold-Standard Dataset (probabilistic features)
- Size & coverage: [typical / edge / adversarial]
- Source: [real traces / structured synthetic — dimensions used]
- Owner: [domain expert]
- Sample cases: [input → expected/pass-criteria]

## Per-Dimension Eval Definitions
For each:
- Type: objective / subjective / guardrail
- Pass criteria: [binary]
- Rubric (if judged): [...]
- Judge: [code metric / LLM-as-judge / human]   Segment slices: [...]
- Threshold (ship/no-ship): [...]

## Agent Trajectory Evals (if agentic)
- Effectiveness / Efficiency / Robustness / Safety criteria
- Trajectory checks: plan quality, tool use, context handling

## Bias / Segment Slices
- Segments: [...]   Who's most harmed by a wrong output: [...]
- Per-segment thresholds: [...]

## Thresholds, HITL & Instrumentation
- Ship gates: [...]
- HITL level per high-risk decision: [...]
- Online/offline metrics + incident→eval-case flow: [...]
- Error-analysis cadence: [weekly until stable → monthly]

## Runtime Eval Loop (architecture must build this)
- Eval service: [input / output / inline-async-batch / latency]
- Feedback loop: [correction UI/API → storage → taxonomy → update cadence]
- Dashboard: [score-over-time / error categories / feedback velocity / per-segment]
- Runtime guardrails: [score & confidence thresholds, anomaly alert]
- Loop failure modes: [service down / queue full / no ground truth]

## → Feeds: architecture-checkpoint (must build the eval/feedback service as a first-class component)
```

## Handoff

The Eval Spec is a primary input to **architecture-checkpoint**: the runtime eval loop specified in Step 7 (eval service + feedback path + dashboard + guardrails) must be designed in as a first-class component, not bolted on later. Say:
> "Eval spec is locked — every feature classified, failure taxonomy built from error analysis, gold set defined, thresholds set, and the runtime eval loop specified. This now feeds Architecture so the eval/feedback service is designed in, not bolted on."

## Anti-Patterns
- ❌ Top-down generic metrics (latency, token count) as your evals → ✅ error-analysis-first, derived from real failures
- ❌ Fuzzy evals on deterministic features / pass-fail on probabilistic ones → ✅ classify first, right instrument per bucket
- ❌ 1–5 Likert that hides disagreement → ✅ binary pass/fail, decompose scalar feelings into Y/N questions
- ❌ Judging only the final output of an agent → ✅ trajectory eval (the process is the truth)
- ❌ Global averages only → ✅ segment slices; ask who's most harmed
- ❌ 99% score = great → ✅ 99% = eval too easy
- ❌ Gold set of sanitized demos → ✅ real/edge/adversarial cases a domain expert curates
- ❌ Writing an eval for a bug you can fix in one line now → ✅ fix it, reserve evals for recurring failure modes

## Reference
`references/eval-craft.md` — the full method: error analysis (open/axial coding), structured synthetic data, three eval levels, binary-vs-Likert rationale, LLM/Agent-as-Judge setup, Google's Four Pillars + Outside-In, objective/subjective/guardrail buckets, segment-slice bias method, and thresholds/cadence.
