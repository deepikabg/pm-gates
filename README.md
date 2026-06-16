# claude-skills

Personal [Claude Code](https://claude.com/claude-code) skills — a connected set of product gates that run across the lifecycle of any build, from "I have an idea" to "this is going to production." They keep the work honest about *what problem we're solving*, *what "good" measurably means*, *whether the architecture holds*, and *whether it's safe to ship*.

These are installed as global skills under `~/.claude/skills/`. This repo is that directory under version control.

## Skills

| Skill | When it fires | What it does |
|-------|---------------|--------------|
| [`brainstorm`](brainstorm/SKILL.md) | Start of any new product/feature/"I want to build X" | Gated discovery interview (one question at a time). Forces clarity on goal → prioritized problem → user journey → decision surface → MVP → phases. Produces a Brainstorm Brief. |
| [`eval-spec`](eval-spec/SKILL.md) | After MVP scope locks, before/alongside architecture | Turns "what good looks like" into measurable pass/fail contracts. Classifies each feature as deterministic test vs. probabilistic eval vs. guardrail; builds gold datasets; sets ship/no-ship thresholds. |
| [`architecture-checkpoint`](architecture-checkpoint/SKILL.md) | Before proposing a full system design or major change | Validates architecture against PM constraints: scalability, DB/cache strategy, error handling, observability, cost, explicit trade-offs. |
| [`api-contract-definition`](api-contract-definition/SKILL.md) | Designing service-to-service or public APIs | Defines API contracts (OpenAPI/gRPC) before implementation — versioning, pagination, error handling, schema consistency. |
| [`security-baseline`](security-baseline/SKILL.md) | Any new service, API, or auth flow | Minimum security bar: encryption, secrets management, PII handling, auth/authz, compliance scope, dependency CVEs. |
| [`deploy-gate`](deploy-gate/SKILL.md) | Before any deploy / migration / public URL goes live | Runtime safety gate on the *built artifact*: re-scans for leaked secrets/PII/public endpoints, re-checks deps for CVEs, requires explicit human approval for prod deploy and schema migration. |

## How they chain

```
brainstorm ──▶ eval-spec ──▶ architecture-checkpoint ──▶ ...build... ──▶ security-baseline ──▶ deploy-gate
                                    │
                       api-contract-definition (for APIs)
```

`eval-spec` carries the AI-native eval loop end to end: it now specifies the runtime eval service/feedback/dashboard (Step 7), and `architecture-checkpoint` verifies that loop is built in as a first-class component.

Each gate bounces cheaply back to the previous one when something isn't clear enough: if you can't write the eval, the problem isn't understood well enough to build.

## Layout

```
<skill-name>/
  SKILL.md            # frontmatter (name, description) + instructions
  references/         # supporting docs the skill pulls in
                      #   brainstorm/references/pm-frameworks.md, eval-spec/references/eval-craft.md
```

## Install

### As a plugin (recommended — to share)

This repo is a Claude Code plugin marketplace. Anyone can install all six skills in two commands:

```
/plugin marketplace add deepikabg/claude-skills
/plugin install pm-gates@deepika-pm-skills
```

The plugin is `pm-gates`; the marketplace is `deepika-pm-skills` (defined in [`.claude-plugin/marketplace.json`](.claude-plugin/marketplace.json)).

### As a live skills directory (for editing the source)

```bash
git clone git@github.com:deepikabg/claude-skills.git ~/.claude/skills
```

Each skill directory is auto-discovered by Claude Code from `~/.claude/skills/`.
