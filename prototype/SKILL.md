---
name: prototype
description: |
  Build a clickable prototype with real copy and thought-through UX as the THIRD design artifact — the one the human can feel, not just read. Use after eval-spec locks the critical user journeys and before architecture is approved, for any Level 2–3 build with a user-facing surface. Triggers on "mock it up", "what would it look like", "prototype this", "let me see it before we build", or whenever the loop reaches the prototype node. This skill first elicits the user's design preferences (mood, colors, reference products — one question at a time), then turns the Eval Spec's P0 journeys into a clickable prototype with real copy (no lorem ipsum), covers empty/error/loading states and a minimal accessibility bar, and then runs the RISKIEST-ASSUMPTION VALIDATION: the prototype in front of target users (or the founder-as-user) produces an explicit validated / invalidated / inconclusive verdict and a proceed / pivot / kill decision recorded in state. It culminates in the TRIAD APPROVAL: Brainstorm Brief + Eval Spec + Prototype approved together by the human before architecture hardens anything. If the prototype can't be built from the spec, the spec isn't clear enough to build from.
---

# Prototype

The third design artifact — and the only one the human can *feel*. The Brief says what and why; the Eval Spec says what "working" measurably means; the prototype is those same P0 journeys made clickable, with the real words on the real screens. It exists for two reasons: (1) copy and UX are product decisions that deserve approval *before* they're expensive, and (2) it is usually the **cheapest possible test of the riskiest assumption** — a go / pivot / kill decision for the price of a mockup instead of a build.

## Position in the chain

```
brainstorm ──► eval-spec ──► PROTOTYPE ──► ✋ TRIAD APPROVAL ──► architecture-checkpoint
                (journeys)      │                (human approves Brief + Eval-Spec + Prototype together)
                                │
                    riskiest-assumption validation
                    verdict: validated / invalidated / inconclusive
                    decision: proceed / pivot (→ brainstorm) / kill (→ park, celebrate the save)
```

One journey definition, three progressively-real encodings: eval-spec defines the journeys → this skill makes them clickable → qa-verify later drives the *real product* through the same journeys. The prototype's screens and copy become qa-verify's expected reference at pre-handoff.

## When to Trigger
- The loop reaches this node (eval-spec passed, Level 2–3, user-facing surface)
- "Mock it up", "show me before we build", "what would it look like", "prototype this"
- A build is about to enter architecture with UX and copy still living in nobody's head

## When NOT to Trigger
- Level 0–1 (patch / quick fix) — no prototype ceremony for a bug fix
- Headless systems with no user-facing surface (API-only, pipelines) — note it's skipped and why; the "prototype" of an API is its contract (api-contract-definition)
- A prototype already exists and was imported via spec-intake (must be genuinely clickable and cover the P0 journeys — a static mock is a gap, not an import)

## Scale to the build
- **Level 3 (new system):** full pass — all P0 journeys, states, a11y bar, user validation, triad approval.
- **Level 2 (feature):** prototype-lite — only the new/changed screens, real copy, clickable through the feature's journey. Run user validation only if this feature carries the riskiest assumption; otherwise founder-review suffices. Triad approval still applies (the three artifacts are approved together).
- **Throwaway prototype builds:** the build IS the prototype; skip this gate, note why.

## The Method

### Step 0 — Design-preference intake (ASK, don't guess)
Before building anything, interview the user — one question at a time, brainstorm-style. Taste is not inferable:
1. > "Three adjectives for how this product should feel?" (start from the Brief's Voice adjectives if present — confirm or adjust)
2. > "Products whose look/feel you admire — 2 or 3 references?"
3. > "Color preferences or brand colors? Light, dark, or both?"
4. > "Dense and information-rich, or spacious and focused?"
5. > "Any accessibility needs beyond the baseline (larger text, reduced motion)?"

Record the answers as `design_prefs` in state.yaml — the build phase inherits them too, so the real product doesn't drift from the approved look.

### Step 1 — Screen map from the Eval Spec's journeys
Every P0 journey from the Eval Spec becomes a sequence of named screens. **Do not invent journeys** — if a screen doesn't serve a specced journey, it doesn't belong in the prototype; if a journey can't be drawn as screens, that's an eval-spec gap (bounce it back, cheaply).

### Step 2 — Real copy, from the Brief's Positioning & Voice
Write the actual words: headlines, button labels, onboarding text, empty states, error messages, confirmation moments. Source the tone from the Brief's Positioning & Voice block. **Lorem ipsum is a FAIL** — placeholder copy hides the hardest product decisions (what do we promise? what do we call things? what happens when it's empty?).

### Step 3 — Make it clickable
Tool-agnostic (static HTML/CSS/JS, Figma, v0 — whatever is fastest to *clickable*), but the bar is fixed:
- Every P0 journey is clickable end-to-end — a user can walk it without narration.
- Artifact lives at `.pipeline/prototype/` (or a recorded URL for hosted tools) with its sha/version in state.
- The prototype is a **design artifact, not a codebase**: it may inform the build but never ships to production without passing through the full build chain.

### Step 4 — States and the accessibility bar
For each key screen: **empty** (first-run, zero data — this mirrors qa-verify's day-0 reality), **loading**, and **error** (use the API contract's error taxonomy once it exists; plain-language messages, no raw codes). Minimal a11y bar: text contrast ≥ 4.5:1, visible focus states, labeled inputs, keyboard-walkable journeys. These aren't polish — they're the states real users hit first.

### Step 5 — Riskiest-assumption validation (the go / no-go this gate exists for)
The Brief names the riskiest assumption **and its class** (valuable / viable / usable / feasible). **Match the test instrument to the class** — a usability walkthrough cannot answer a pricing question:

| Assumption class | Test via this gate | How |
|---|---|---|
| **Valuable** (do they want it) | ✅ prototype + fake door | Cold-drive the prototype; end on a real commitment ask (signup, waitlist, "send me access") — measure the ask, not the compliments |
| **Viable** (will they pay) | ✅ prototype + pricing moment | Put the actual price in the flow (pricing page, fake checkout); the verdict is what they *do* at that screen, not what they *say* about value |
| **Usable** (can they self-serve) | ✅ prototype cold-drive | Target user completes the P0 journey unnarrated; verdict = task completion, hesitation points |
| **Feasible** (can the model/tech do it) | ⚠️ NOT a prototype job | Run a **technical spike against the Eval Spec's gold cases** (real inputs, measured pass rate) before architecture. The prototype gate still runs for UX, but the *validation verdict* comes from the spike — clicking through mock screens proves nothing about model reliability |

Then:
1. Put the prototype in front of **3–5 target users** — or, minimum, the founder driving it cold, no narration.
2. Script the session around the assumption, not the features — using the class-matched instrument above.
3. Record the verdict in state.yaml with the class: **validated / invalidated / inconclusive** — with evidence (quotes, observed behavior, commitment-ask conversion, spike pass rate).
4. Decision (human's call, recorded): **proceed** → triad approval · **pivot** → route back to brainstorm with what was learned (this is the cheap bounce the loop exists for) · **kill** → park the project with reasons. A kill here is a *save*, not a failure — name what it cost (a prototype) vs. what it would have cost (the build).

**Inconclusive is not validated.** If the test didn't produce signal, say so and decide knowingly.

### Step 6 — Triad approval (the human moment)
Present all three design artifacts together for one approval: **Brainstorm Brief + Eval Spec + Prototype**. This is the "is this the right thing, do we know what good means, and does it feel right" checkpoint — the last cheap moment to change any of it.
- Record the approval in state.yaml with the sha of all three artifacts. **Any later edit to any of the three invalidates the triad** (hash mismatch) → re-approve before the loop advances.
- The human may approve with notes (logged), request changes (loop within this gate or bounce to the owning gate), or reject.

## Output

```markdown
# Prototype Review: [Project]
Prototype: [path/URL + sha]   Journeys covered: [P0 list from Eval Spec]
Design prefs: [mood / colors / references / density / mode]
States covered: [empty / loading / error per screen]   A11y bar: [✓/gaps]

## Assumption Validation
Assumption tested: [from Brief]   Method: [N users / founder cold-drive]
Verdict: [validated / invalidated / inconclusive] — evidence: [...]
Decision: [proceed / pivot / kill] — by: [human]

## Triad Approval
Brief [sha] + Eval-Spec [sha] + Prototype [sha] — approved by [human] at [ts]
Notes: [...]
```

## Handoff
Write the Prototype Review + `passed` (with the triad block) to `.pipeline/state.yaml`, then **signal loop-orchestrator — it owns routing; this skill never chooses the next gate.** (Per the orchestrator's rules, architecture-checkpoint runs next on proceed; pivot routes back to brainstorm; kill parks the pipeline.)

**Log decisions:** the proceed/pivot/kill call, any design-direction choice, and any deferred/rejected recommendation → append to `.pipeline/DECISIONS.md` with `Affects:` links, so anything it drifts flips to `stale` (format: loop-orchestrator's Decision Ledger).

## Anti-Patterns
- ❌ Lorem ipsum / placeholder copy → ✅ real words; copy is a product decision needing approval
- ❌ Happy-path-only screens → ✅ empty, loading, and error states — the states day-0 users actually hit
- ❌ Guessing the user's taste → ✅ design-preference intake first, one question at a time
- ❌ Prototyping journeys the Eval Spec doesn't define → ✅ the spec's journeys only; gaps bounce to eval-spec
- ❌ Pixel-polishing before the assumption is validated → ✅ validate first, refine after proceed
- ❌ Treating "inconclusive" as "validated" → ✅ name it and decide knowingly
- ❌ Prototype code quietly becoming production code → ✅ design artifact only; the build goes through the full chain
- ❌ Approving Brief, Eval Spec, and Prototype in three separate drive-bys → ✅ one triad approval, three artifacts, one moment
