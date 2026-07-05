# pm-gates

**The PM governance layer for autonomous build loops.**

The popular approaches to autonomous building all constrain the *engineer* side — code discipline, context hygiene, role separation. None of them answer the product questions: *Is this the right thing to build? How will we know it works? What must never happen without a human?*

pm-gates is that layer: eight connected gates that run across the lifecycle of any build, from "I have an idea" to "this is verified in production." They keep the work honest about what problem we're solving, what "good" measurably means, whether the architecture holds, and whether it's safe to ship.

```
DEFINE     brainstorm → eval-spec → architecture-checkpoint
CONTRACT   api-contract-definition → security-baseline
BUILD      [your engine: any autonomous dev loop / plain Claude Code]
             rules: test-first iron law · fresh context per story · eval-spec = done
SHIP       deploy-gate  (staging autonomous · prod = HARD STOP for a human)
VERIFY     qa-verify    (real browser · bounded fix loop · post-deploy watch)
```

`loop-orchestrator` is the single source of truth for sequence, routing, state, and resumption — gates signal completion and never route themselves. It also owns **spec-intake**: arrive with an existing PRD and the loop imports the gates it satisfies, gap-interviews only what's missing, and never re-runs the full discovery ceremony (eval-spec is the one gate that can't be skipped by omission). `eval-spec` carries the AI-native eval loop end to end: the spec it produces is simultaneously the build engine's definition-of-done and `qa-verify`'s browser script. Each gate bounces cheaply back to the previous one when something isn't clear enough — if you can't write the eval, the problem isn't understood well enough to build.

## The skills

| Skill | When it fires | What it does |
|---|---|---|
| `loop-orchestrator` | Start of any build cycle; on resume; whenever routing is unclear | Owns the end-to-end loop: scale-adaptive routing (Levels 0–3), phase rules, `.pipeline/state.yaml`, human hard stops |
| `brainstorm` | Start of any new product/feature/"I want to build X" | Gated discovery interview. Goal → prioritized problem → decision surface → true MVP. Produces a Brainstorm Brief |
| `eval-spec` | After MVP scope locks, before/alongside architecture | Turns "what good looks like" into measurable pass/fail contracts: deterministic tests vs. probabilistic evals vs. guardrails, gold datasets, ship/no-ship thresholds |
| `architecture-checkpoint` | Before proposing a system design or major change | Validates architecture against PM constraints: scalability, DB/cache, error handling, observability, cost, explicit trade-offs |
| `api-contract-definition` | Designing service-to-service or public APIs | OpenAPI/gRPC contracts before implementation: versioning, pagination, errors, schema consistency |
| `security-baseline` | Any new service, API, or auth flow | Minimum security bar: encryption, secrets, PII, auth/authz, compliance scope, dependency CVEs |
| `deploy-gate` | Before any deploy / migration / public URL goes live | Re-scans the *built artifact* for secrets/PII/public-surface drift; hard-stops prod deploy and schema migration for explicit human approval |
| `qa-verify` | After every staging or prod deploy | Drives a real browser through the Eval Spec's user journeys; bounded fix loop on staging; post-deploy monitoring window on prod |

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

## Acknowledgments

No code or text in this project is copied from other projects, but several of its principles stand on the shoulders of the frameworks that pioneered them: the test-first "iron law" is Superpowers' (Jesse Vincent); fresh-context-per-story execution and scale-adaptive planning tracks come from GSD and the BMAD Method; browser-driven QA with a fix→regression-test loop and post-deploy monitoring come from gstack (Garry Tan, MIT). The eval-spec spine, the gate chain, and the irreversible-action hard stops are original to this project. Credit where it's due — go star them.

## License

[MIT](LICENSE)
