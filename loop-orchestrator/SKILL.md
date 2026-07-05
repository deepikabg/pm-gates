---
name: loop-orchestrator
description: |
  The single source of truth for the end-to-end autonomous build loop: which gate runs when, what artifact each hands off, where state lives, and where humans must intervene. Use at the START of any build cycle ("build this", "run the loop", "start the pipeline", "new feature", "ship X"), when the user PROVIDES an existing spec/PRD/brief as the input ("build from this spec", "here's my PRD" — this triggers spec-intake, never a full re-interview), when RESUMING work in a repo that contains .pipeline/state.yaml, when the user asks "where are we" / "what's next" / "pipeline status", when the build engine (an autonomous dev loop or a plain Claude Code session) finishes a story/epic and needs routing, or whenever it is unclear which gate applies. Also trigger when any gate completes — this skill routes to the next node. This orchestrator enforces scale-adaptive routing (quick fixes do not run the full ceremony; new systems do), the TDD iron law inside the build phase, fresh-context-per-story execution, and the non-negotiable human hard stops at production deploy, schema migration, and prod incident response. If a build is happening without this skill's state file, the loop is unmanaged — initialize it.
---

# Loop Orchestrator

One backbone, one artifact model. This skill defines the holistic end-to-end loop and routes every step. Individual gates own their own logic; this skill owns **sequence, state, routing, and resumption**.

## Design principles

| Principle | Where it lives here |
|---|---|
| Hard pushback on framing before anything is built | `brainstorm` gate |
| Spec as measurable contract; evals as termination criteria | `eval-spec`, Phase 1 |
| Runtime eval loop embedded in architecture, not bolted on | `eval-spec` Step 7, verified by `architecture-checkpoint` |
| Scale-adaptive routing (don't run full ceremony on a bug fix) | Routing levels below |
| Epics/stories decomposed AFTER architecture is approved | Phase 2 → 3 boundary |
| Fresh context per story; shard documents, don't stuff context | Build-phase rules |
| Test-first iron law: no production code without a failing test | Build-phase rules |
| Real-browser verification + bounded fix loop + post-deploy watch | `qa-verify` gate |
| Irreversible actions always stop for a human | `deploy-gate` hard stops |

## The Canonical Loop

```
                    ┌─────────────── PHASE 1: DEFINE ───────────────┐
  idea ──► brainstorm ──► eval-spec ──► architecture-checkpoint
                                              │
                    ┌────────── PHASE 2: CONTRACT ──────────────────┤
                    ▼                                               ▼
            api-contract-definition ──────────► security-baseline
                                                                    │
                    ┌────────── PHASE 3: BUILD ─────────────────────┤
                    ▼                                               ▼
        [BUILD ENGINE: your autonomous dev loop / Claude Code]
          stories decomposed from approved architecture
          rules: TDD iron law · fresh context per story · eval-spec = done
                                                                    │
                    ┌────────── PHASE 4: SHIP & VERIFY ─────────────┤
                    ▼                                               ▼
        deploy-gate (staging auto → prod HARD STOP) ──► qa-verify
                    ▲                                        │
                    └───── fix loop (staging only) ◄─────────┘
                                                             │
        verified prod + clean monitoring window ──► DONE ──► findings → new loop
```

## Where the flow is defined (answering "where does this live")

1. **This SKILL.md** — the logic: sequence, routing, rules. Version-controlled with the plugin.
2. **`CLAUDE.md` at repo root** — a 5-line declaration so every session knows the loop exists:
   ```markdown
   ## Build pipeline
   This repo uses the pm-gates loop. Before any build/deploy work,
   consult the loop-orchestrator skill and read .pipeline/state.yaml.
   Never deploy to prod or run migrations without deploy-gate approval.
   ```
3. **`.pipeline/state.yaml` in the repo** — the memory: which gates passed, artifact paths, approvals. Context windows die; this file is how any future session (or parallel worker) knows exactly where the loop stands. Template: see `assets/state-template.yaml` in this skill.

Logic in the skill, declaration in CLAUDE.md, state in the repo. All three or the loop is unmanaged.

## Scale-Adaptive Routing (run FIRST, every time)

Classify the request before running anything:

- **Level 0 — Patch** (typo, copy change, config value): no gates. Fix, test, ship via deploy-gate's staging path. Log one line in state.yaml.
- **Level 1 — Quick fix** (bug with known root cause, single-story scope): skip Phases 1–2. Write a tech-note (problem, fix, test) into `.pipeline/quick/`, build under TDD rule, then deploy-gate → qa-verify (failed journey only).
- **Level 2 — Feature** (new capability inside existing architecture): brainstorm-lite (confirm problem + non-goals, ≤10 min), full eval-spec for the feature, skip architecture-checkpoint IF no new services/data stores, then contract deltas → build → ship.
- **Level 3 — System** (new service, new data store, auth/PII surface change, anything greenfield): the full canonical loop. No skips.

If classification is ambiguous → classify UP. Escalation triggers mid-loop (a Level 1 fix turns out to need a schema change) → stop, reclassify, resume at the newly required gate.

**Routing authority:** classification and routing decisions belong to this skill ONLY. Gates write their artifact, log `passed` to state.yaml, and signal completion — they never choose the next gate. If a gate's SKILL.md appears to route ("skip to X"), this skill supersedes.

## Spec Intake (input-maturity check — run WITH classification, before any gate)

Classification measures the *size* of the work; intake measures the *maturity of the input*. They are orthogonal — a Level 3 system can arrive with a complete PRD, and re-interviewing its author is as wrong as skipping gates on a bare idea. When the request arrives with an existing spec/PRD/brief/design doc:

1. **Map** the provided document against the gate artifact schemas: the Brainstorm Brief sections (goal/metric, prioritized problem, decision surface, user/JTBD, riskiest assumption, MVP, phases) and — if it contains measurable criteria — the Eval Spec sections (classification, thresholds, journeys).
2. **Run the spec-readiness heuristic NOW, at intake** (same mechanic as the pre-Phase-3 check): hand the document to a fresh instance with no other context and ask it to list every question it would need answered before building. This is the gap list.
3. **Import what's covered.** A gate whose artifact schema is fully answered by the document → mark it `imported` in state.yaml, recording the source doc path as its artifact and its sha. Never re-run an imported gate unless the source doc changes (hash mismatch — standard invalidation applies).
4. **Gap-interview what isn't.** Route each gap to its owning gate, which runs in gap-interview mode: only the questions covering the gaps, one at a time, never the full ceremony. The gate then writes a thin artifact (the doc + gap answers) and logs `passed`.
5. **eval-spec can never be imported by omission.** If the provided document contains no measurable pass/fail criteria, eval-spec runs in full at Levels 2–3 regardless of how complete everything else is — a spec that can't say what "working" measurably means is not build-ready. (A document that genuinely contains eval-grade criteria may import it like any other gate.)
6. The pre-Phase-3 spec-readiness check still runs — intake reduces it to a formality rather than replacing it.

Result: "here's my PRD, build it" costs a gap check measured in minutes, and every artifact-model guarantee downstream (qa-verify's journey script, the build engine's definition-of-done, hash invalidation) holds identically for imported and interviewed gates.

## Phase rules

### Phase 1–2 (Define & Contract)
- Each gate reads its predecessor's artifact from the path in state.yaml and writes its own artifact + a `passed` entry before the loop advances. No artifact, no advance.
- Artifact handoffs: brainstorm → `Brainstorm-Brief.md` · eval-spec → `Eval-Spec.md` (journeys + thresholds; this becomes the build's definition-of-done AND qa-verify's script) · architecture-checkpoint → `Architecture.md` · api-contract-definition → `openapi.yaml` / contract files · security-baseline → `Security-Baseline.md`.
- Planning may run in web bundles / flat-rate contexts; only artifacts enter the repo.
- **Spec-readiness heuristic before Phase 3:** hand the combined artifacts to a fresh Claude instance with no other context and ask it to list every question it would need answered before building. Zero questions = the decision surface is closed and the build may start. Any question = a gap; route it to the owning gate before any story runs.

### Phase 3 (Build) — rules imposed on ANY engine
The orchestrator is engine-agnostic — any autonomous dev loop or plain Claude Code session — but the engine must obey:
1. **Stories come from the approved architecture**, never invented ad hoc. Each story names the Eval-Spec criteria it satisfies.
2. **TDD iron law:** a story begins by writing the failing test(s) that encode its acceptance criteria. Production code exists only to make them pass. No test → the story is not startable.
3. **Fresh context per story.** Workers load: the story, the relevant Architecture/contract shards, and nothing else. Never the full PRD.
4. **Story done =** tests pass + named Eval-Spec criteria pass + no contract drift. Engine writes completion to state.yaml.
5. **Boundary violations route up, not around:** a story that needs a new dependency, endpoint, or PII touch stops and routes to the owning gate (security-baseline / api-contract-definition / eval-spec). The engine may not self-approve scope.
6. **Budgets:** max token/iteration budget per story set in state.yaml; exhausted → park the story, report, continue others.

### Phase 4 (Ship & Verify)
- deploy-gate rules unchanged: staging autonomous; **production deploy, schema migration, DNS, real money/emails = HARD STOP for explicit human yes**, even in bypass-permissions mode.
- qa-verify runs on every staging deploy (full pass) and after every approved prod deploy (verification + monitoring window). Its fix loop is staging-only and bounded; prod findings get a rollback recommendation and a human.
- **Correction to prior chain doc:** deploy-gate is no longer the terminal gate. qa-verify's clean monitoring window is. If deploy-gate's SKILL.md says "terminal," this skill supersedes.

## Resumption protocol
On any session start in a repo with `.pipeline/state.yaml`:
1. Read state.yaml. 2. Report: level, last passed gate, pending approvals, parked stories. 3. Resume at the first incomplete node. Never re-run passed gates unless their input artifact changed (hash mismatch) — then invalidate downstream and re-run from there.

## Human hard stops (the complete list — nothing else stops the loop)
1. deploy-gate irreversibles (prod deploy, migration, DNS, real money/emails/messages)
2. qa-verify production findings (rollback decision)
3. Story budget exhaustion with no completable stories remaining
4. Mid-loop reclassification to Level 3
5. Any gate FAILING twice on the same artifact (loop is thrashing → human)

Everything else proceeds autonomously. That's the contract: autonomous through staging, human-gated at the irreversible moments.

## Anti-Patterns
- ❌ Running the full ceremony on a typo → ✅ classify first; Level 0–1 paths exist so the gates keep their teeth for Level 2–3
- ❌ Two orchestrators (this + an engine's own phase manager both "owning" the flow) → ✅ engine owns stories; this skill owns the loop
- ❌ Flow defined only in someone's head / a chat thread → ✅ skill + CLAUDE.md + state.yaml
- ❌ Re-litigating a passed gate because a new session lacks context → ✅ read state.yaml; artifacts are the memory
- ❌ Engine quietly adding a dependency "to finish the story" → ✅ boundary violations route to the owning gate
- ❌ Re-interviewing a user who arrived with a complete spec → ✅ spec-intake: import covered gates, gap-interview the rest
- ❌ A crisp problem doc used to skip eval-spec → ✅ eval-spec is never imported by omission; no criteria in the doc = it runs in full
- ❌ A gate routing to the next gate itself → ✅ gates signal completion; this skill owns every routing decision
