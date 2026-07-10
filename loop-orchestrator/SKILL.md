---
name: loop-orchestrator
description: |
  The single source of truth for the end-to-end autonomous build loop: which gate runs when, what artifact each hands off, where state lives, and where humans must intervene. Use at the START of any build cycle ("build this", "run the loop", "start the pipeline", "new feature", "ship X"), when the user PROVIDES an existing spec/PRD/brief as the input ("build from this spec", "here's my PRD" — this triggers spec-intake, never a full re-interview), when RESUMING work in a repo that contains .pipeline/state.yaml, when the user asks "where are we" / "what's next" / "pipeline status" / "show me the board" / "break this down into issues", when the build engine (an autonomous dev loop or a plain Claude Code session) finishes a story/epic and needs routing, or whenever it is unclear which gate applies. Also trigger when any gate completes — this skill routes to the next node. This orchestrator enforces scale-adaptive routing (quick fixes do not run the full ceremony; new systems do), the TDD iron law inside the build phase, fresh-context-per-story execution, and the non-negotiable human hard stops at production deploy, schema migration, and prod incident response. If a build is happening without this skill's state file, the loop is unmanaged — initialize it.
---

# Loop Orchestrator

One backbone, one artifact model. This skill defines the holistic end-to-end loop and routes every step. Individual gates own their own logic; this skill owns **sequence, state, routing, and resumption**.

## Design principles

| Principle | Where it lives here |
|---|---|
| Hard pushback on framing before anything is built | `brainstorm` gate |
| No build without the landscape; the wedge is named before specs harden | `market-scan` (recon + full, no-wedge human stop) |
| Spec as measurable contract; evals as termination criteria | `eval-spec`, Phase 1 |
| Runtime eval loop embedded in architecture, not bolted on | `eval-spec` Step 7, verified by `architecture-checkpoint` |
| Scale-adaptive routing (don't run full ceremony on a bug fix) | Routing levels below |
| Design is approved as a felt artifact, not just documents | `prototype` gate → triad approval |
| The riskiest assumption is tested before the build commits | `prototype` validation → proceed/pivot/kill |
| Decomposition is owned, dependency-ordered, and provably covers the spec | `story-map` gate (DAG + traceability) |
| The human can watch the build plan fill in | `story-map` GitHub Issues/Project projection |
| Fresh context per story; shard documents, don't stuff context | Build-phase rules |
| Test-first iron law: no production code without a failing test | Build-phase rules |
| Real-browser verification, first on localhost before any deploy | `qa-verify` pre-handoff mode |
| A phase closes only on stored test + eval evidence | Phase 3 rules 4–7 · `test_runs` ledger |
| Every green run is a recoverable git checkpoint | Phase 3 rule 8 |
| Every consequential decision + recommendation is logged, with what it drifts | `DECISIONS.md` ledger + `stale` status |
| Stale specs are refreshed by replaying decisions, not rewritten from memory | Decision-ledger refresh flow |
| Irreversible actions always stop for a human | `deploy-gate` hard stops |

## The Canonical Loop

```
                    ┌─────────────── PHASE 1: DEFINE ───────────────┐
  idea ──► brainstorm ──► market-scan ──► eval-spec ──► prototype
           (+ market-scan   (landscape + teardowns       (journeys)    (design prefs → clickable P0 journeys → assumption test)
            RECON at Gate 0) → wedge verdict, fills
                             Brief's differentiator;
                             no wedge = ✋ proceed/pivot/kill —
                             the cheapest kill in the loop)
                                              │ verdict: validated/invalidated/inconclusive
                                              │ decision: proceed / pivot(→brainstorm) / kill(→park)
                                              ▼
                       ✋ TRIAD APPROVAL — human approves Brief + Eval-Spec + Prototype TOGETHER
                                              │
                                              ▼
                                   architecture-checkpoint
                                              │
                    ┌────────── PHASE 2: CONTRACT & DECOMPOSE ──────┤
                    ▼                                               ▼
            api-contract-definition ──────────► security-baseline
                                                                    │
                                              spec-readiness check ─┤
                                                                    ▼
                          story-map  (epics → story DAG · traceability matrix
                                      · GitHub Issues/Project view · ✋ human approves the plan)
                                                                    │
                    ┌────────── PHASE 3: BUILD ─────────────────────┤
                    ▼                                               ▼
        [BUILD ENGINE: your autonomous dev loop / Claude Code]
          executes the story-map DAG in topological order
          parallel batch stories run in isolated git worktrees
          rules: TDD iron law · fresh context per story · done = tests+evals run & stored · green run = checkpoint
                                                                    │
                    ┌────────── PHASE 4: SHIP & VERIFY ─────────────┤
                    ▼                                               ▼
        qa-verify (PRE-HANDOFF · localhost, no deploy)  ← first-run gate
          journeys + day-0 checklist · day-0 fix loop (localhost)
                    │ green
                    ▼
        deploy-gate (staging auto → prod HARD STOP) ──► qa-verify (staging)
                    ▲                                        │
                    └───── fix loop (staging only) ◄─────────┘
                                                             │
        prod approval ──► qa-verify (prod + monitoring) ──► DONE ──► findings → new loop
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
3. **`.pipeline/state.yaml` in the repo** — the *status*: which gates passed, artifact paths, hashes, approvals. Context windows die; this file is how any future session (or parallel worker) knows exactly where the loop stands. Template: see `assets/state-template.yaml` in this skill.
4. **`.pipeline/DECISIONS.md` in the repo** — the *why*: the append-only rationale/drift ledger (see the next section). state.yaml says what state we're in; the artifacts say what the spec currently is; this says why it says that and how it has drifted.

Logic in the skill, declaration in CLAUDE.md, status in state.yaml, rationale in DECISIONS.md. All four or the loop is unmanaged.

## The Decision Ledger (`.pipeline/DECISIONS.md`)

The three memory layers divide cleanly: **artifacts** are the *current* spec, **state.yaml** is the *status*, and this ledger is the ***why***. It is append-only, it sits *beside* the artifacts (it does not replace them), and it is what turns "refresh the eval spec" from an archaeology dig into a mechanical replay. Without it, pivots and trade-offs made during build and test live in commit messages and dead chat threads, and the design artifacts silently rot while the code moves on.

**Who writes:** every gate and the human append; this skill owns the format. **When (the bar — keep it from becoming noise):** log a decision only if it changes an artifact, pivots scope, accepts a named trade-off, or defers/rejects a recommendation — NOT routine implementation choices the spec already covers (same discipline as "don't write an eval for a one-line fix").

**Entry format:**
```markdown
## D-014 · [gate/phase] · [ISO date] · status: accepted
Decision: [what was decided]
Trigger: [qa-verify finding / build discovery / prototype pivot / user ask / Claude recommendation]
Rationale + alternatives: [why this; what else was weighed]
Affects: [Eval-Spec ✎stale · Architecture]     Decided by: [human/claude] · approved: [y/n]
Supersedes: [D-009 or —]
```

- **status:** `proposed | accepted | deferred | rejected | superseded | applied`. **Recommendations are first-class entries** — a deferred recommendation IS the backlog; a rejected one is settled *with its reason*, so it is never silently re-litigated.
- **`Affects:` drives drift.** Any artifact named there flips to `stale` in state.yaml, tagged with the decision id. "What has drifted, and why?" becomes a query, not a memory test.

**Refresh flow (why the ledger earns its keep):** to refresh a stale artifact — gather its open (`accepted`, not-yet-`applied`) decisions from the ledger → regenerate the artifact incorporating them → mark those decisions `applied` → re-hash the artifact. The hash change re-invalidates downstream gates through the normal mechanism. **Refresh is replay, not rewrite** — the ledger is the reason a months-long, many-pivot project can still produce a current, trustworthy Brief/Eval-Spec/Architecture on demand.

## Scale-Adaptive Routing (run FIRST, every time)

Classify the request before running anything:

- **Level 0 — Patch** (typo, copy change, config value): no gates. Fix, test, ship via deploy-gate's staging path. Log one line in state.yaml.
- **Level 1 — Quick fix** (bug with known root cause, single-story scope): skip Phases 1–2. Write a tech-note (problem, fix, test) into `.pipeline/quick/`, build under TDD rule, then deploy-gate → qa-verify (failed journey only).
- **Level 2 — Feature** (new capability inside existing architecture): brainstorm-lite (confirm problem + non-goals, ≤10 min), market-scan parity check (do competitors have this feature? table stakes or differentiator — one page max), full eval-spec for the feature, prototype-lite IF user-facing (new/changed screens only, real copy; user validation only if the feature carries the riskiest assumption), skip architecture-checkpoint IF no new services/data stores, then contract deltas → story-map (light — the feature's stories + deps into the existing DAG/board) → build → ship.
- **Level 3 — System** (new service, new data store, auth/PII surface change, anything greenfield): the full canonical loop — including market-scan (recon at Gate 0 + full scan), prototype + triad approval, and story-map. No skips (prototype may be skipped only for headless/no-UI systems, noted in state).

If classification is ambiguous → classify UP. Escalation triggers mid-loop (a Level 1 fix turns out to need a schema change) → stop, reclassify, resume at the newly required gate.

**Routing authority:** classification and routing decisions belong to this skill ONLY. Gates write their artifact, log `passed` to state.yaml, and signal completion — they never choose the next gate. If a gate's SKILL.md appears to route ("skip to X"), this skill supersedes.

## Spec Intake (input-maturity check — run WITH classification, before any gate)

Classification measures the *size* of the work; intake measures the *maturity of the input*. They are orthogonal — a Level 3 system can arrive with a complete PRD, and re-interviewing its author is as wrong as skipping gates on a bare idea. When the request arrives with an existing spec/PRD/brief/design doc:

1. **Map** the provided document against the gate artifact schemas: the Brainstorm Brief sections (goal/metric, prioritized problem, decision surface, user/JTBD, riskiest assumption, MVP, phases) and — if it contains measurable criteria — the Eval Spec sections (classification, thresholds, journeys).
2. **Run the spec-readiness heuristic NOW, at intake** (same mechanic as the pre-Phase-3 check): hand the document to a fresh instance with no other context and ask it to list every question it would need answered before building. This is the gap list.
3. **Import what's covered.** A gate whose artifact schema is fully answered by the document → mark it `imported` in state.yaml, recording the source doc path as its artifact and its sha. Never re-run an imported gate unless the source doc changes (hash mismatch — standard invalidation applies).
4. **Gap-interview what isn't.** Route each gap to its owning gate, which runs in gap-interview mode: only the questions covering the gaps, one at a time, never the full ceremony. The gate then writes a thin artifact (the doc + gap answers) and logs `passed`.
5. **eval-spec can never be imported by omission.** If the provided document contains no measurable pass/fail criteria, eval-spec runs in full at Levels 2–3 regardless of how complete everything else is — a spec that can't say what "working" measurably means is not build-ready. (A document that genuinely contains eval-grade criteria may import it like any other gate.)
5b. **prototype imports only as a working artifact.** A provided prototype counts only if it is genuinely clickable and covers the P0 journeys — static mocks or screenshots are a gap, not an import. The assumption-validation verdict and triad approval still run even on an imported prototype.
6. The pre-Phase-3 spec-readiness check still runs — intake reduces it to a formality rather than replacing it.

Result: "here's my PRD, build it" costs a gap check measured in minutes, and every artifact-model guarantee downstream (qa-verify's journey script, the build engine's definition-of-done, hash invalidation) holds identically for imported and interviewed gates.

## Phase rules

### Phase 1–2 (Define, Contract & Decompose)
- Each gate reads its predecessor's artifact from the path in state.yaml and writes its own artifact + a `passed` entry before the loop advances. No artifact, no advance.
- Artifact handoffs: brainstorm → `Brainstorm-Brief.md` · market-scan → `Market-Landscape.md` (cited, `valid_until` ~90 days; it also fills the Brief's Market & Differentiator section and re-hashes both) · eval-spec → `Eval-Spec.md` (journeys + thresholds; this becomes the build's definition-of-done AND qa-verify's script) · prototype → `.pipeline/prototype/` + `Prototype-Review.md` (clickable P0 journeys, real copy; qa-verify's expected reference at pre-handoff) · architecture-checkpoint → `Architecture.md` · api-contract-definition → `openapi.yaml` / contract files · security-baseline → `Security-Baseline.md` · story-map → `Story-Map.md` + populated `build.stories`.
- **Triad approval (human):** after prototype, the human approves **Brief + Eval-Spec + Prototype together** — one moment, three artifacts, recorded in state with all three shas. Any later edit to any of the three invalidates the triad (hash mismatch) → re-approve before advancing. The prototype's assumption-validation verdict (validated/invalidated/inconclusive) and decision (proceed/pivot/kill) are recorded with it; **pivot routes back to brainstorm, kill parks the pipeline** — both are successes of the gate, not failures of the loop.
- Planning may run in web bundles / flat-rate contexts; only artifacts enter the repo.
- **Spec-readiness heuristic after security-baseline, before story-map:** hand the combined artifacts to a fresh Claude instance with no other context and ask it to list every question it would need answered before building. Zero questions = the decision surface is closed and decomposition may start. Any question = a gap; route it to the owning gate before any story is cut.
- **Decompose (story-map, the final pre-build node):** the approved artifacts become an epic → story DAG with explicit dependencies, topological batches, a traceability matrix (every P0 journey → stories; every story → eval criteria; holes BLOCK), and — when a remote is configured — a GitHub Issues + Project projection the human can watch. The human approves the plan's shape. state.yaml stays the source of truth; GitHub is the regenerable view.

### Phase 3 (Build) — rules imposed on ANY engine
The orchestrator is engine-agnostic — any autonomous dev loop or plain Claude Code session — but the engine must obey:
1. **Stories come from the approved story-map DAG**, never invented ad hoc. The engine executes batches in topological order; a story whose dependency isn't `done` is not startable. A needed-but-missing story routes back to story-map — the engine may not add stories itself.
2. **TDD iron law:** a story begins by writing the failing test(s) that encode its acceptance criteria. Production code exists only to make them pass. No test → the story is not startable.
3. **Fresh context per story; worktrees for parallel stories.** Workers load: the story, the relevant Architecture/contract shards, and the prototype screens it implements — nothing else. Never the full PRD. Stories marked `parallel` in the same batch run concurrently **in isolated git worktrees** (no shared-file writes by construction), each merging back on green; anything else serializes.
3b. **GitHub sync (when projected):** story in-progress → move the card; story done → close the issue (the checkpoint commit references `#<issue>`); parked/red → comment with the ledger entry. state.yaml wins every disagreement — regenerate the view, never reverse-import.
4. **Story done = evidence, not existence.** A story is `done` only when its tests **and** its named Eval-Spec criteria have been **run in this session** and the results are **recorded** — never merely "tests exist." The engine appends a `test_runs` entry to state.yaml and writes the runner's JSON report to `.pipeline/results/<ts>.json`. "The suite exists / passed once in CI last week" does not close a story.
5. **Evidence is committed, not gitignored.** `.pipeline/results/` is the audit trail — it is version-controlled. Eval runners write their JSON there; the `test_runs` ledger in state.yaml is append-only (the orchestrator never overwrites prior entries).
6. **A full-loop integration test per completed phase is mandatory.** Following the pattern proven in distribution-agent, each phase ships a `tests/loop-phase{N}.test.ts` (or the stack's equivalent) that drives the phase end-to-end; the **phase is not `done` until it passes** and its run is in the ledger.
7. **Provisional gold sets are marked, not trusted.** Evals may run against a provisional/synthetic gold set, but until it is human-curated the stored report must carry `ship_gate: NOT BINDABLE`. This enforces eval-spec's anti-theater rule at storage time — a NOT-BINDABLE result can inform, never gate.
8. **Green run → git checkpoint.** After every test/eval run recorded as **green** in the ledger, the engine commits `git add -A && git commit -m "checkpoint(<story-id>): tests <n>/<n> green, evals <summary>"` and, if a remote is configured, pushes to the **feature/dev branch**. Rules: **never commit secrets** (`.env*` stays gitignored — run a secrets scan before every commit); **never push to `main`/prod branch autonomously** (that is deploy-gate territory); a **red run commits nothing**. If no remote exists, ask the human **once** whether to create one (`gh repo create`), record the answer under `git:` in state.yaml, and stop asking. Purpose: every green state is recoverable — a failing later story rolls back to the last checkpoint instead of thrashing (pairs with the story budget rule).
9. **Boundary violations route up, not around:** a story that needs a new dependency, endpoint, or PII touch stops and routes to the owning gate (security-baseline / api-contract-definition / eval-spec). The engine may not self-approve scope.
10. **Budgets:** max token/iteration budget per story set in state.yaml; exhausted → park the story, report, continue others.

### Phase 4 (Ship & Verify)
- **Pre-handoff first (new — closes the first-run gap):** when the build engine reports the last story done, the orchestrator routes FIRST to **qa-verify in pre-handoff mode against localhost / the dev server — before deploy-gate**. The pass is the Eval-Spec journey script PLUS the day-0 first-run checklist (fresh-account onboarding end-to-end; every P0 journey works with zero optional infra; onboarding consumes the user's real input). **No artifact reaches deploy-gate until this pass is green.** This exists because a real Level-3 build (distribution-agent) passed every design gate and 56/56 tests yet failed in the founder's first five minutes — the failures were first-run/integration failures, exactly qa-verify's class, and they were caught by no earlier gate because journey verification previously ran only after a deploy.
- deploy-gate rules unchanged: staging autonomous; **production deploy, schema migration, DNS, real money/emails = HARD STOP for explicit human yes**, even in bypass-permissions mode.
- qa-verify then runs on every staging deploy (full pass) and after every approved prod deploy (verification + monitoring window). Its fix loop is pre-handoff/staging-only and bounded; prod findings get a rollback recommendation and a human.
- The full chain is therefore: build engine → **qa-verify (pre-handoff, localhost)** → deploy-gate → qa-verify (staging) → prod approval → qa-verify (prod + monitoring). The orchestrator owns every hop; gates only signal completion.
- **Correction to prior chain doc:** deploy-gate is no longer the terminal gate. qa-verify's clean monitoring window is. If deploy-gate's SKILL.md says "terminal," this skill supersedes.

## Resumption protocol
On any session start in a repo with `.pipeline/state.yaml`:
1. Read state.yaml **and skim recent `DECISIONS.md` entries**. 2. Report: level, last passed gate, pending approvals, parked stories, **and any `stale` artifacts with the decisions that drifted them**. 3. Resume at the first incomplete node. Never re-run passed gates unless their input artifact changed (hash mismatch) — then invalidate downstream and re-run from there. A `stale` artifact is refreshed by replaying its open decisions (see the Decision Ledger), not rebuilt from scratch.

## Human hard stops (the complete list — nothing else stops the loop)
1. deploy-gate irreversibles (prod deploy, migration, DNS, real money/emails/messages)
2. qa-verify production findings (rollback decision)
3. **Market-scan no-wedge verdict** — proceed / pivot / kill, decided by the human; the cheapest kill in the loop (research-priced, pre-build)
4. **Triad approval** — Brief + Eval-Spec + Prototype approved together, including the assumption-validation decision (proceed / pivot / kill)
5. **Story-map plan approval** — the human signs off the build's shape (batches, traceability, board)
6. Story budget exhaustion with no completable stories remaining
7. Mid-loop reclassification to Level 3
8. Any gate FAILING twice on the same artifact (loop is thrashing → human)

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
- ❌ Handing a human a "done" build verified only by green unit tests → ✅ qa-verify pre-handoff drives the real first-run journey on localhost before deploy-gate
- ❌ "Works if you start the worker / set the key first" shipped as done → ✅ every P0 journey runs from a clean clone with zero optional infra
- ❌ Marking a story done because the test file exists → ✅ tests + named evals run THIS session and recorded in the `test_runs` ledger
- ❌ A green run left uncommitted, then thrashing to recover it → ✅ green run → checkpoint commit (never secrets, never main)
- ❌ Trusting a provisional gold set to gate a ship → ✅ stored report carries `ship_gate: NOT BINDABLE` until human-curated
- ❌ Approving a build from documents alone → ✅ the triad: the human clicks the prototype before architecture hardens anything
- ❌ Building before the riskiest assumption is tested → ✅ prototype validation first; pivot and kill are cheap here, brutal later
- ❌ "The engine will figure out the tasks" → ✅ story-map owns decomposition: a DAG, a traceability matrix, an approved plan
- ❌ A board that's decoration → ✅ traceability makes it trustworthy; sync rules keep it current; state.yaml stays the truth
- ❌ Decisions and pivots buried in commit messages and dead chat threads → ✅ one append-only DECISIONS.md with what each drifts
- ❌ Design artifacts silently rotting while the build moves on → ✅ `Affects:` flips them `stale`; refresh replays the decisions
- ❌ Re-litigating a settled call, or losing a deferred idea → ✅ recommendations logged with status (deferred = backlog, rejected = settled with reason)
- ❌ Logging every trivial choice until the ledger is noise → ✅ only consequential decisions; the spec already covers the routine
- ❌ Building without once looking at the market → ✅ recon at idea time; full landscape scan before specs harden
- ❌ Deep-scanning the solution category before the problem is framed → ✅ recon early is an existence check; the full scan targets the locked JTBD + segment
- ❌ Proceeding past a no-wedge verdict on optimism → ✅ no-wedge is a human stop: proceed / pivot / kill, logged in DECISIONS.md
