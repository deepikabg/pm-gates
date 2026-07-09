---
name: brainstorm
description: |
  Stop the user from jumping to solutions and force clarity on goal, prioritized problem, decision surface, and MVP scope BEFORE any building begins. Use at the very start of ANY new product, feature, project, side-project, or "I want to build X" conversation — even if the user already sounds confident. Triggers on "I want to build", "help me think through", "let's brainstorm", "new idea", "should I build", "I'm thinking about a product/feature/app/tool", or whenever the user is about to start building without a defined problem and MVP. Runs a gated discovery interview (one question at a time), frames solutions as problems, decomposes the decision surface for AI-native systems, triages by frequency and impact into a true MVP vs. later phases, and produces a Brainstorm Brief. On completion it signals loop-orchestrator, which routes next (eval-spec for AI-native/high-stakes builds, architecture-checkpoint otherwise). If the user arrives with an existing spec/PRD, do NOT run the full interview — loop-orchestrator's spec-intake imports it and this skill interviews only on the gaps it surfaces. Always use before architecture or coding when the problem and MVP are not yet crisp.
---

# Brainstorm

A gated discovery skill that stops you building the wrong thing. It forces clarity on **goal → prioritized problem → user & journey → decision surface → MVP → phases** before any architecture decision. Grounded in a structured PM method (see `references/pm-frameworks.md`).

## The Single Most Important Rule (marked "GOLD")

**At the problem stage: understand needs, desires, emotions, barriers, goals; map user flow → painpoints → prioritize by goal.** Stay in the problem space — when they describe a feature or solution, convert it back into the underlying need before going anywhere near how to build it. This is the discipline the whole skill protects.

## Three operating principles

1. **One question at a time.** Never dump a wall of questions. Ask, listen, build on the answer. Talk *through* your reasoning out loud — even when stuck or when an idea seems boring, narrate the questions you're brainstorming. Never answer casually without structure.
2. **Diverge before you converge.** Brainstorm widely — generate ~10 ideas and invite their input too — then prioritize hard. Wide net first, ruthless triage second.
3. **MVP is a hypothesis test, not a smaller product.** It exists to validate the single riskiest assumption — and to test it as cheaply as possible.

## When to Trigger
- Start of any new product/feature/project/side-project
- "I want to build…", "should I build…", "I have an idea…", "let's brainstorm…"
- About to jump into architecture/code without a defined problem
- Even when they sound confident — confidence about a solution often hides an unexamined problem

## When NOT to Trigger (full interview)
- The user arrived with an existing spec/PRD/brief → **loop-orchestrator's spec-intake** owns this case. It maps the document against the Brainstorm Brief schema and marks this gate `imported`; this skill then runs in **gap-interview mode only** — ask solely about the gaps intake surfaced, one at a time, never the full ceremony. Note: a crisp problem doc does NOT skip eval-spec; intake can never import eval-spec by omission.
- A narrow implementation question mid-build
- Pure research/learning with no build attached

---

## The Gated Discovery Flow

Move through gates **in order**. Don't advance until the current gate's check passes. If they jump to solutions or architecture, acknowledge it, park it, and pull them back. Open by telling them what's happening: *"Before we build anything, let me run a quick discovery — one question at a time — so we lock the problem and MVP before touching architecture. I'll think out loud as we go."*

---

### GATE 0 — Clarify & Frame the Question

Ask clarifying questions first; narrow the scope before exploring. (Give a heads-up about your assumptions, then narrow to a clear scope and the area of largest impact.)

- > "Is this open-ended — 'something in this space' — or a specific bet you've mostly decided and want to pressure-test?"
- > "Any constraints I should keep in mind up front — team size, timeline, budget, regulatory, tech you're tied to?"
- For a strategy/0-to-1 framing, surface the **three components that matter most** before anything else. Example: for an apartment-hunting app, the three that most determine success are **business model, competition, and user segment** — name those before any solution. Name the analogous three for *this* idea.

**Gate check:** You know whether this is explore-wide or pressure-test, the hard constraints, and the 2–3 components that will most determine success.

---

### GATE 1 — Goal & Metrics

Separate the *outcome* from the *thing*. (Define goal and metrics **before** answering — if you can't define success, you can't achieve it.)

1. > "In one sentence: what outcome are you creating? Not the product — the change in someone's world."
2. > "Why this, why now? What makes it worth your time over everything else?" *(opportunity cost — and second-order: what does doing this *prevent* or *unlock* elsewhere?)*
3. > "How will you know it worked? Give me the one metric that says 'right bet.'" Then pressure-test it: *is this metric better than the alternatives? what's the issue tree of drivers underneath it?*

**Gate check:** Restatable as `[outcome] for [someone], measured by [one primary metric]`. Park any solution ideas they blurt — save them for MVP triage.

---

### GATE 2 — Prioritized Problem & (if AI-native) Decision Surface

**This is the core gate. Two moves.**

**Move A — Find and rank the real problem.** (Understand motivation, dislikes, daily routine; *break down 3 things*; frame as problems, never solutions yet.)
1. > "What's the problem? Describe the moment it actually bites — when, where, who's feeling it."
2. > "If you do nothing, what happens — who's hurt, how badly?" *(painkiller vs. vitamin)*
3. > "Is this one problem or several? List them — then if you could fix only ONE this month, which unlocks the most, and why is it materially more important than the rest?" *(Customer Problems Stack Rank — it's not enough that you solve a problem; you must know where it ranks.)*
4. > "What do people do today instead?" *(the alternative / workaround)*

**Move B — Decompose the decision surface (ONLY if the system is AI-native: agents, RAG, anything making probabilistic decisions).** This is where "build a chatbot" becomes "triage, flag, and route." Ask, one at a time:
1. > "What decisions is this system actually making? Name them as decisions, not features." *(e.g., 'is this claim complete?', 'route to which review path?')*
2. > "For each: simple or complex? low-risk or high-risk?"
3. > "Which can the AI safely decide alone? Which can it only *support* with a human deciding? Which must it **never** decide alone?"
4. > "When must a case escalate to a human?"

Doing Move B here — at the cheapest stage, before anything exists — is what prevents an expensive mid-build reframe. The decision surface *is* part of the problem definition for an AI product; it is not a later gate.

**Gate check:** ONE prioritized problem with explicit stakes and a known alternative. For AI-native builds, a decision map: each decision tagged risk + who decides + escalation trigger.

---

### GATE 3 — User, Journey & JTBD

1. > "Who exactly is this for? Be specific — not 'PMs' but 'newly-promoted PMs at Series B fintechs in their first 90 days.'"
2. > "What job are they hiring it for? 'When I [situation], I want to [motivation], so I can [outcome].'"
3. **Walk the journey before the screen.** (Map the whole journey before the place to be designed — e.g. for a gym: wait times, busy hours, booking equipment in advance.) > "Walk me through their day around this problem — the whole journey, not just the moment they'd touch your product. Where are the painpoints along it?"
4. > "Does this serve people already in your world, or require winning a new segment?"
5. **Second-order:** > "What do they do *next*, outside the product, after this works? What new problem does that create?"

**Gate check:** Specific segment + JTBD + a journey map with painpoints located along it.

---

### GATE 4 — MVP Triage by Frequency × Impact (the discipline against over-building)

**Move 1 — Riskiest assumption (enumerate → classify → score, don't just ask once).**
Open with: > "What must be TRUE for this to work that you're least sure of? If it's false, the whole thing collapses." — but don't stop at the first answer. Enumerate candidate assumptions across the **four risk classes** (a product bet can collapse in four distinct ways):

| Class | The assumption sounds like | Collapse mode |
|---|---|---|
| **Valuable** (demand) | "People actually have this problem and want it solved" | Nobody comes |
| **Viable** (monetization/strategy) | "They'll pay / the unit economics work / we can reach them" | Usage without a business |
| **Usable** (adoption behavior) | "They can self-serve through it / it fits their workflow" | They try it once and bounce |
| **Feasible** (technical) | "The model/tech can actually do this reliably enough" | It works in the demo, not in the wild |

Then score each candidate **impact-if-false × uncertainty** and pick the max. Two calibration notes: (1) most builds die on **valuable/viable** — founders systematically over-test feasibility because it's comfortable, so when in doubt, weight the strategy-level classes up; (2) the honest exception is AI-native systems, where "can the model do this reliably on real inputs" is frequently a *legitimate* top scorer — don't force a business assumption to win when the model bet is genuinely shakier. Record the winner **with its class** — the class determines the test instrument at the prototype gate (a pricing question can't be answered by a usability walkthrough).

**Move 2 — Frequency triage (apply to EVERY candidate use case).**
> "How often does this use case actually fire — daily, weekly, monthly, or one-time?"
Then route by frequency, because frequency determines whether automation is even justified:
- **Daily / high-frequency** → worth real infrastructure; this is where the product lives.
- **Weekly / monthly** → a scheduled task or routine, not a pipeline. Keep it light.
- **One-time / rare** → a script, a manual step, or just do it by hand. **Building a system here is over-engineering.** Name it and cut it.

This single question kills the "overcomplicating features for no reason" failure mode. High-frequency × high-risk decisions earn real building; everything else gets the lightest mechanism that works.

**Move 3 — Prioritize features.** Pull every idea (incl. parked ones). Score with these tools:
- **ICE** (Impact × Confidence × Ease) for a fast pass, and/or
- **MoSCoW** buckets: **Must** (tests the riskiest assumption OR no value without it) / **Should** (Phase 2) / **Could** (Phase 3+) / **Won't** (cut, name why).
- Sanity-check ambition with **1x/2x/5x/10x**: a good MVP proves one 5x/10x bet with just enough 1x scaffolding to be usable — not all-1x (no reason to switch) nor all-10x (unbuildable).

**Move 4 — Sanity check.** > "If we shipped ONLY the Musts, does it validate your riskiest assumption AND give a real user something they'd use? If not, we cut too much or kept too much."

**Gate check:** A small Must-have list that tests the riskiest assumption, with every use case frequency-tagged and anything rare explicitly deferred or de-automated.

---

### GATE 5 — Phase the Roadmap & GTM

**Phases:** Cut features are *sequenced, not deleted*. For each post-MVP phase: what ships, what it unlocks, and the **signal that triggers it** (gate on learning, not calendar — e.g., "build Phase 2 only after MVP hits >40% weekly return").

**GTM (always close here):** > "Where does your prioritized audience hang out, online and offline?" (e.g. moms visit schools, read parenting blogs.) Then sketch the first use cases the MVP must serve — **use cases, not build tasks**: they live in the Brief as scope, and `story-map` later cuts the actual build issues from the approved architecture, traced back to these.

**Positioning & Voice (one pass, three questions):** the prototype's copy and all later marketing inherit this — don't leave it to be improvised.
1. > "In one line, to your target user: what is this and why should they care?" *(positioning)*
2. > "Three adjectives for how it should sound and feel?" *(voice — the prototype gate will confirm these as design adjectives too)*
3. > "The one message a user must walk away with?" *(key message)*

---

## Output: The Brainstorm Brief

```markdown
# Brainstorm Brief: [Project Name]
Date: [date] | Type: [open-ended / pressure-test] | AI-native: [yes/no]

## Goal & Metric
[Outcome] for [someone], measured by [primary metric].
Why now / opportunity cost: [...]  Second-order effects: [...]

## Top-3 Components That Decide Success
1. [...] 2. [...] 3. [...]

## Prioritized Problem
THE problem: [one sentence + the moment it bites]
Stakes if unsolved: [...]   Current alternative: [...]
Deprioritized (parked): [...]

## Decision Surface  (AI-native only)
| Decision | Simple/Complex | Risk | Who decides | Escalation trigger |
|---|---|---|---|---|

## User, Journey & JTBD
Who: [specific segment]
Job: When I [situation], I want to [motivation], so I can [outcome].
Journey + painpoints: [...]   Segment bet: [existing / new]

## Riskiest Assumption
[The one thing that must be true. MVP exists to test THIS.]
Class: [valuable / viable / usable / feasible]   Score: [impact-if-false × uncertainty vs. runners-up]
Runners-up considered: [...]   Planned test instrument (by class): [...]

## MVP (Must-Have) — frequency-tagged
- [ ] [Feature] — freq: [daily/weekly/…] — tests assumption / enabling value
Validates: [...]   Standalone value: [...]   Success signal → Phase 2: [...]
Cheapest test of the assumption: [landing page / manual version / fake-the-feature / prototype]

## Roadmap (Post-MVP)
### Phase 2 — [theme] | Ships: [...] | Unlocks: [...] | Trigger: [signal]
### Phase 3 — [theme] | Ships: [...] | Unlocks: [...] | Trigger: [signal]

## Cut (Won't build / don't automate)
- [Feature] — [why: low frequency / low impact / distracts]

## GTM & Positioning
Where the audience is (on/offline): [...]
First use cases (scope, not build tasks — story-map cuts issues later): [...]
Positioning (one line): [...]   Voice: [3 adjectives]   Key message: [...]

## Parking Lot
[Solution ideas, open questions to revisit]

## → Next: handed to loop-orchestrator
[MVP scope + constraints + decision surface + positioning/voice feed the next gates: eval-spec, then prototype + triad approval]
```

## Handoff
The Brief's **MVP** + **constraints** + **Decision Surface** feed the next gate. Write the Brief's path + a `passed` entry to `.pipeline/state.yaml`, then **signal loop-orchestrator — it owns routing; this skill never chooses the next gate.** (Per the orchestrator's rules, eval-spec runs next at Levels 2–3 — if you can't write the eval, the problem isn't clear enough to build, which bounces cheaply back here.) Say: *"Brief is locked and logged to state — handing to the orchestrator for the next gate."*

**Log decisions:** any consequential decision, pivot, or deferred/rejected recommendation from this gate → append to `.pipeline/DECISIONS.md` with `Affects:` links, so anything it drifts flips to `stale` (format: loop-orchestrator's Decision Ledger).

## Anti-Patterns
- ❌ Dumping all questions at once → ✅ one at a time, think out loud
- ❌ Accepting the first solution → ✅ frame it back as a problem (GOLD)
- ❌ Adding a separate "decision-surface" gate after MVP → ✅ it lives in Gate 2, before anything's built, so there's no mid-build redo
- ❌ Building a pipeline for a monthly/one-time task → ✅ frequency-triage; lightest mechanism that works
- ❌ MVP = v1 with fewer features → ✅ MVP = cheapest test of the riskiest assumption
- ❌ Deleting cut features → ✅ sequence them, gated on learning signals
- ❌ Unmeasurable goal → ✅ one primary metric + its issue tree

## Reference
`references/pm-frameworks.md` holds the full toolkit: 5C/MECE strategy structure, Customer Problems Stack Rank, ICE, MoSCoW, 1x/2x/5x/10x, the 7 planning themes, Helmer's 7 Powers, DHM, Lean assumption-testing ladder, metrics issue-trees, and the frequency lens. Read it when a gate needs depth.
