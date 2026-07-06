---
name: story-map
description: |
  Decompose the approved design artifacts into a dependency-ordered build plan — the explicit owner of the plan→build handoff. Use after security-baseline passes (all design artifacts locked) and before any story is built, or on "break this down", "create the issues", "plan the build", "what's the build order", "show me the board". This skill turns the approved Architecture + Eval Spec + Brief + Prototype into epics and stories with explicit dependencies (a DAG, executed in topological order), builds the TRACEABILITY MATRIX that proves every P0 journey has stories and every story has eval criteria (holes block the gate), marks which stories can run in parallel (in isolated git worktrees) vs. must serialize, and — when a GitHub remote is configured — projects the plan into GitHub Issues + a Project board so the human can watch the build fill in. state.yaml remains the machine source of truth; GitHub is the generated, synced view. The build engine consumes this plan; it never invents stories.
---

# Story Map

The gate that owns "breaking it down." Everything upstream produced approved *design* artifacts; everything downstream *executes stories*. This gate is the explicit bridge: it decomposes the design into a dependency-ordered story DAG, proves the decomposition actually covers the spec (traceability), and renders the plan somewhere a human can watch — GitHub Issues + Project board. Decomposition is real work with a real artifact and a real owner; "the engine will figure it out" is how builds drift.

## Position in the chain

```
… security-baseline ──► spec-readiness check ──► STORY-MAP ──► [build engine]
   (design locked)        (zero open questions)      │              (executes the DAG,
                                                     │               topological order)
                                     Story-Map.md + state.stories
                                     └─► GitHub Issues + Project board (synced view, if remote)
```

## Inputs (read all four)
1. **Architecture** — components, data model, phases → the epic skeleton and the data-dependency order.
2. **Eval Spec** — journeys + per-dimension criteria → what each story must satisfy; the traceability targets.
3. **Brainstorm Brief** — MVP scope + phases → what's in, what's deferred; the use-case list stories must serve.
4. **Prototype (approved triad)** — screens + copy → UI stories reference their approved screens.

## The Method

### Step 1 — Epics from the architecture
One epic per architecture component or build phase. Every epic names the component it builds and the Brief use cases it serves.

### Step 2 — Stories: smallest independently-verifiable units
Split each epic into stories where each story:
- is completable in one fresh-context session (small enough — under ~50% of a context window of work);
- names the **Eval-Spec criteria** it satisfies (≥1, always) and the journey it serves;
- has a binary done (its tests + named criteria, per Phase 3 rules);
- gets an id (`S-001`), an iteration budget, and — for UI stories — the prototype screens it implements.

### Step 3 — Dependencies: build the DAG
For every story pair, name the edges:
- **Data deps:** schema before the API that reads it; API before the UI that calls it.
- **File/merge deps:** two stories touching the same files → serialize them (or restructure), regardless of logical independence — parallel same-file work is merge chaos, not speed.
- **Setup deps:** scaffolding, auth, CI before what needs them.

The result must be a **DAG — no cycles** (a cycle means the stories are mis-split; re-cut them). Derive **topological batches**: batch N+1 starts only when its dependencies in batch N are done. Stories within a batch with no file overlap are marked `parallel` — the engine may run them concurrently **in isolated git worktrees**, merging each back on green.

### Step 4 — Traceability matrix (holes BLOCK this gate)
Prove the plan covers the spec — this is what makes the board trustworthy rather than decorative:

| Check | Rule |
|---|---|
| Every P0 journey (Eval Spec) | → ≥1 story implements it, end-to-end |
| Every story | → ≥1 Eval-Spec criterion (a story no criterion needs is scope creep — cut or justify) |
| Every ship-gate criterion | → covered by a story's tests/evals or named infra |
| Every guardrail | → a story or a standing check owns it |
| Every MVP Must-have (Brief) | → traced to stories; anything untraced is a silent scope drop |

Any hole → the gate FAILS with the hole named; route the gap to its owning artifact (missing journey → eval-spec; unclear component → architecture) — never paper over it in the plan.

### Step 5 — The GitHub projection (the human's view; state stays the truth)
`state.yaml` is the machine source of truth — the loop must survive offline, non-GitHub hosts, and Level 0–1. When a remote is configured (see the `git:` block), project the plan:
- **One issue per story** (`gh issue create`): title `[S-001] <story title>`; body = story, eval criteria, journey, prototype screens, and `Depends on: #<issue>` links; labels for epic, batch, and `parallel`; one milestone per build phase.
- **One Project board** (`gh project`): columns Backlog / In progress / Done; issues added in topological order so the board *reads* as the build order.
- Record issue numbers and the project URL back into state.yaml.
- **Sync rules for the engine:** story `in-progress` → move card; story `done` (green, per Phase 3) → close the issue, checkpoint commit message references `#<issue>`; story parked/red → comment with the ledger entry. If state and GitHub disagree, **state wins** — regenerate the view, never reverse-import.
- No remote → skip projection, note it in the report; the plan is fully usable from state alone.

### Step 6 — Human approval of the plan
Present: the DAG (as an ordered batch list), the traceability matrix, story count + budgets, and the board URL if projected. The human approves the *shape of the build* — this is the "view of how the project is getting broken down" moment. Approval recorded in state; changes loop within this gate.

## Output

```markdown
# Story Map: [Project]
Epics: [N]  Stories: [N]  Batches: [N]  Parallel-safe: [N stories]
Traceability: [COMPLETE / holes listed → gate FAILS]

## Build order (topological batches)
Batch 1: S-001 [schema] · S-002 [auth scaffold]        (parallel — no file overlap)
Batch 2: S-003 [API /x] (needs S-001) · …
…

## Traceability matrix
| Journey / criterion | Stories | Covered |
|---|---|---|

## GitHub projection
Issues: [#1–#N / skipped (no remote)]   Project: [URL]
Approved by: [human] at [ts]
```

Stories land in `state.yaml → build.stories` with `depends_on`, `batch`, `parallel`, `github_issue` populated. `Story-Map.md` is the artifact.

## Handoff
Write Story-Map.md + `passed` + the populated stories to `.pipeline/state.yaml`, then **signal loop-orchestrator — it owns routing; this skill never chooses the next gate.** (Per the orchestrator's rules, the build engine starts Batch 1 next, executing the DAG in topological order.)

## Anti-Patterns
- ❌ The engine inventing stories mid-build → ✅ stories come from this gate only; a needed-but-missing story routes back here
- ❌ A story list with no dependency edges → ✅ a DAG with named data/file/setup deps, executed in topological order
- ❌ Parallelizing stories that touch the same files → ✅ file-overlap check; parallel only in isolated worktrees
- ❌ GitHub Issues as the source of truth → ✅ state.yaml is the truth; GitHub is a regenerable view
- ❌ A story that satisfies no eval criterion → ✅ that's scope creep; cut it or trace it
- ❌ A P0 journey with no story → ✅ traceability holes BLOCK the gate, loudly
- ❌ Stories sized to "however much fits" → ✅ one fresh-context session each, budgeted
- ❌ Skipping the human's look at the plan → ✅ the board/batch-list approval is the visibility moment this gate exists for
