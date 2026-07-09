# pm-gates

**The PM governance layer for autonomous build loops.**

The popular approaches to autonomous building all constrain the *engineer* side — code discipline, context hygiene, role separation. None of them answer the product questions: *Is this the right thing to build? How will we know it works? What must never happen without a human?*

pm-gates is that layer: ten connected gates that run across the lifecycle of any build, from "I have an idea" to "this is verified in production." They keep the work honest about what problem we're solving, what "good" measurably means, how it should feel, whether the architecture holds, and whether it's safe to ship.

It's a methodology made executable — each gate encodes the judgment a product lead brings to one moment of a build, so an autonomous engine (or a plain Claude Code session) can move fast without shipping the wrong thing, an unmeasurable thing, or an unsafe thing. It is not a product with a user base; it's a working theory of the AI-native PM's job. The sharpest gates were written *after* real build cycles exposed what a clean spec and green tests still miss — the pre-handoff gate exists because a build passed every design check and 56/56 tests, then broke for its first user in five minutes (the changelog traces each such lesson).

```
DEFINE     brainstorm → eval-spec → prototype
             (design prefs → clickable P0 journeys with real copy → riskiest-assumption test)
             ✋ TRIAD APPROVAL — human approves Brief + Eval-Spec + Prototype together
             decision: proceed / pivot(→brainstorm) / kill(→park)
           → architecture-checkpoint
CONTRACT   api-contract-definition → security-baseline
DECOMPOSE  story-map     (epics → story DAG · traceability matrix · GitHub Issues/Project view
                          · ✋ human approves the build plan)
BUILD      [your engine: any autonomous dev loop / plain Claude Code]
             executes the DAG in topological order · parallel stories in git worktrees
             rules: test-first iron law · fresh context per story
                    done = tests+evals run & stored · green run = git checkpoint
VERIFY⁰    qa-verify    (PRE-HANDOFF · localhost · day-0 first-run checklist · BEFORE any deploy)
SHIP       deploy-gate  (staging autonomous · prod = HARD STOP for a human)
VERIFY     qa-verify    (staging → prod · real browser · bounded fix loop · post-deploy watch)
```

`loop-orchestrator` is the single source of truth for sequence, routing, state, and resumption — gates signal completion and never route themselves. It also owns **spec-intake**: arrive with an existing PRD and the loop imports the gates it satisfies, gap-interviews only what's missing, and never re-runs the full discovery ceremony (eval-spec is the one gate that can't be skipped by omission). `eval-spec` carries the AI-native eval loop end to end: the spec it produces is simultaneously the build engine's definition-of-done and `qa-verify`'s browser script. Each gate bounces cheaply back to the previous one when something isn't clear enough — if you can't write the eval, the problem isn't understood well enough to build.

## The skills

| Skill | When it fires | What it does |
|---|---|---|
| `loop-orchestrator` | Start of any build cycle; on resume; whenever routing is unclear | Owns the end-to-end loop: scale-adaptive routing (Levels 0–3), phase rules, `.pipeline/state.yaml`, human hard stops |
| `brainstorm` | Start of any new product/feature/"I want to build X" | Gated discovery interview. Goal → prioritized problem → decision surface → true MVP. Produces a Brainstorm Brief |
| `eval-spec` | After MVP scope locks, before/alongside architecture | Turns "what good looks like" into measurable pass/fail contracts: deterministic tests vs. probabilistic evals vs. guardrails, gold datasets, ship/no-ship thresholds |
| `prototype` | After eval-spec, before architecture (user-facing builds) | Elicits design preferences, makes the P0 journeys clickable with real copy + empty/error/loading states, tests the riskiest assumption (go/pivot/kill), and runs the **triad approval**: Brief + Eval-Spec + Prototype approved together |
| `architecture-checkpoint` | Before proposing a system design or major change | Validates architecture against PM constraints: scalability, DB/cache, error handling, observability, cost, explicit trade-offs |
| `api-contract-definition` | Designing service-to-service or public APIs | OpenAPI/gRPC contracts before implementation: versioning, pagination, errors, schema consistency |
| `security-baseline` | Any new service, API, or auth flow | Minimum security bar: encryption, secrets, PII, auth/authz, compliance scope, dependency CVEs |
| `story-map` | After design locks, before any story is built | Decomposes the approved design into a dependency-ordered story DAG with a traceability matrix (holes block); projects it as GitHub Issues + a Project board the human watches; human approves the plan |
| `deploy-gate` | Before any deploy / migration / public URL goes live | Re-scans the *built artifact* for secrets/PII/public-surface drift; hard-stops prod deploy and schema migration for explicit human approval |
| `qa-verify` | **Pre-handoff** (build done, on localhost, before any deploy); then after every staging or prod deploy | Drives a real browser through the Eval Spec's user journeys + a **day-0 first-run checklist** (fresh-account onboarding, zero-optional-infra, real user input); bounded fix loop on localhost/staging; post-deploy monitoring window on prod |

## Where the flow is defined

Three places, deliberately: **logic** in `loop-orchestrator/SKILL.md`, a **declaration** in your repo's `CLAUDE.md`, and **state** in `.pipeline/state.yaml` (template in `loop-orchestrator/assets/`). State in the repo is what makes the loop resumable across sessions and legible to parallel workers.

Add to any target repo's `CLAUDE.md`:

```markdown
## Build pipeline
This repo uses the pm-gates loop. Before any build/deploy work,
consult the loop-orchestrator skill and read .pipeline/state.yaml.
Never deploy to prod or run migrations without deploy-gate approval.
```

## Install

### As a plugin (recommended — to share)

This repo is a Claude Code plugin marketplace:

```
/plugin marketplace add deepikabg/pm-gates
/plugin install pm-gates@deepikabg
```

The plugin is `pm-gates` (the product); the marketplace is `deepikabg` (the author) — defined in [`.claude-plugin/marketplace.json`](.claude-plugin/marketplace.json).

### As a live skills directory (for editing the source)

```
git clone git@github.com:deepikabg/pm-gates.git ~/.claude/skills
```

Each skill directory at the repo root is auto-discovered by Claude Code from `~/.claude/skills/`.

## Layout

```
<skill-name>/
  SKILL.md            # frontmatter (name, description) + instructions
  references/         # supporting docs the skill pulls in
  assets/             # templates the skill ships (e.g., loop-orchestrator/assets/state-template.yaml)
.claude-plugin/
  marketplace.json    # makes this repo an installable plugin marketplace
```

## Design principles

Test-first story execution, fresh context per story, scale-adaptive routing, browser-verified shipping with a bounded fix loop, and evals as the spine from spec to verification. Autonomous through staging, human-gated at the irreversible moments. One backbone, one artifact model.

## Changelog

### v1.3 — *the decision ledger: specs stay honest across pivots*

Adds the missing "why" layer. Artifacts are the *current* spec and `state.yaml` is the *status*, but until now nothing recorded **why** a spec changed or captured decisions made during build and test — so pivots lived in commit messages and the design artifacts silently went stale.

- **`.pipeline/DECISIONS.md` (new)** — an append-only rationale/drift ledger, beside the artifacts (not replacing them). Every gate and the human log consequential decisions, pivots, and trade-offs — and **recommendations are first-class entries** (a deferred one is the backlog; a rejected one is settled *with its reason*, never silently re-litigated). Each entry's `Affects:` links flip the artifacts it drifts to a new **`stale`** status in state.yaml. "What's drifted, and why?" becomes a query, not a memory test.
- **Refresh is replay, not rewrite** — regenerating a stale Brief / Eval-Spec / Architecture means gathering its open decisions from the ledger, applying them, and re-hashing (which re-invalidates downstream through the normal mechanism). This is what lets a months-long, many-pivot project still produce a current, trustworthy spec on demand.
- **`qa-verify` is the main back-propagation path** — a finding that forces a copy/scope/spec change logs a decision that flips the design artifact `stale`, so build/test reality flows back to the design instead of the two drifting apart.

### v1.2 — *approve what you'll feel, not just what you'll read; build from a traced plan, not a vibe*

Closes the two weakest links: design was approved as documents only, and the plan→build handoff ("break it down into tasks") had no owner.

- **`prototype` gate (new)** — after eval-spec, the P0 journeys become a clickable prototype with real copy (design-preference interview first — mood, colors, references; no guessing taste, no lorem ipsum), covering empty/error/loading states and a minimal a11y bar. It doubles as the **riskiest-assumption test**: an explicit validated / invalidated / inconclusive verdict and a human **proceed / pivot / kill** decision, recorded in state. Culminates in the **triad approval** — Brief + Eval-Spec + Prototype approved together, sha-locked; editing any of the three invalidates the approval.
- **`story-map` gate (new)** — the explicit owner of decomposition: epics → story DAG with data/file/setup dependency edges, executed in topological batches, parallel-safe stories isolated in git worktrees. A **traceability matrix** blocks the gate on holes (every P0 journey → stories; every story → eval criteria). When a remote is configured, the plan projects into **GitHub Issues + a Project board** — the human-visible view of how the build is broken down. `state.yaml` stays the source of truth; GitHub is the regenerable view; the engine closes issues as stories go green.
- **Routing-authority fixes** — architecture-checkpoint, api-contract-definition, and security-baseline no longer route themselves (a v1.1 inconsistency); all eight gates now signal the orchestrator. Brainstorm gains a **Positioning & Voice** block (the prototype's copy and later marketing inherit it) and its GTM "user stories" are clarified as use cases — build issues come from story-map.

### v1.1 — *no handoff without a driven journey; no done without stored evidence; no green without a checkpoint*

Hardened from a real Level-3 build (`distribution-agent`, 2026-07-05) that passed every design gate and shipped 56/56 green tests, yet handed the founder a product that failed in the first five minutes — email-confirmation dead-end on first signup, a background worker required to see the first result, extraction that ignored the founder's real URL. All three were first-run/integration failures caught by no earlier gate, because journey verification only ran *after* a deploy.

- **qa-verify gains a `pre-handoff` mode** — the Eval-Spec journeys **plus a day-0 first-run checklist** run against `localhost` the moment the build is done and a human is about to become the user, *before* deploy-gate. Fresh-account onboarding must complete end-to-end, every P0 journey must work with zero optional infrastructure, and onboarding must consume the user's real input. Chain is now: build → **qa-verify (pre-handoff)** → deploy-gate → qa-verify (staging) → prod approval → qa-verify (prod).
- **Evidence at every phase, stored in state** — a story/phase is `done` only when its tests *and* named Eval-Spec criteria were run this session and recorded in an append-only `test_runs` ledger, with JSON reports committed under `.pipeline/results/`. A full-loop `tests/loop-phase{N}` test is mandatory per phase; provisional gold sets are stored `ship_gate: NOT BINDABLE` until human-curated.
- **Git checkpoint after every green run** — the engine commits `checkpoint(<story-id>): tests n/n green…` and pushes to the feature/dev branch (never `main`; never with secrets; a red run commits nothing), so every green state is recoverable instead of thrashed toward.

### v1.0 — the eight-gate loop

brainstorm → eval-spec → architecture-checkpoint → api-contract-definition → security-baseline → deploy-gate → qa-verify, orchestrated by `loop-orchestrator` with scale-adaptive routing, spec-intake, and human hard stops at irreversible actions.

## Acknowledgments

No code or text in this project is copied from other projects, but several of its principles stand on the shoulders of the frameworks that pioneered them: the test-first "iron law" is Superpowers' (Jesse Vincent); fresh-context-per-story execution and scale-adaptive planning tracks come from GSD and the BMAD Method; browser-driven QA with a fix→regression-test loop and post-deploy monitoring come from gstack (Garry Tan, MIT). The eval-spec spine, the gate chain, and the irreversible-action hard stops are original to this project. Credit where it's due — go star them.

## License

[MIT](LICENSE)
