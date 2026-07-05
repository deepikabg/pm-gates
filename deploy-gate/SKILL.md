---
name: deploy-gate
description: |
  The runtime safety gate that runs AFTER the build is complete and BEFORE anything is deployed, migrated, or made public. Use whenever code is about to be pushed to a hosting platform (Railway, Vercel, Netlify, Fly, etc.), a database migration is about to run (Supabase, Postgres, Prisma migrate, etc.), a public URL/domain is about to go live, or any irreversible production action is imminent. This gate re-scans the ACTUAL built artifact (not the design) for leaked secrets, PII exposure, and unintended public endpoints, re-checks the installed dependency tree for CVEs, and REQUIRES explicit human approval for the two irreversible actions: production deploy and schema migration. This gate is BLOCKING even when Claude Code is running in bypass-permissions / "don't ask" mode — autonomous flow is allowed through staging, but production deploy and DB migration always stop for a human. Trigger this even if security-baseline already ran at design time; that validated the plan, this validates the shipped code.
---

# Deploy Gate

The last gate before the world sees it. Everything upstream (brainstorm → architecture → eval → API contract → security-baseline) validated the *plan*. This gate validates the *built artifact* and guards the *irreversible actions*.

## Why this exists (the gap it closes)

`security-baseline` is a **design-time** gate: it checks PII handling, auth model, and dependency choices *before code exists*. But in an autonomous build-and-deploy flow — especially Claude Code in bypass-permissions mode — two things go unchecked without this gate:

1. **Drift.** The built code can diverge from the approved design: a hardcoded secret, an added dependency, an endpoint exposed that wasn't specced.
2. **The irreversible moment.** The skill chain otherwise ends at "proceed to implementation." There is no node covering *push to prod / run the migration / go live*. That is the single riskiest moment and it must not be auto-approved.

This gate is the difference between "autonomous through staging, human-gated to production" being aspirational vs. actually enforced.

## The Core Rule (holds even in bypass-permissions mode)

**Reversible actions may proceed autonomously. Irreversible actions always stop for explicit human approval.**

- **Auto-approve (reversible):** build, run tests, lint, deploy to a *staging/preview* environment, run migrations against a *dev/branch* database, dry-run commands.
- **HARD STOP — require explicit human "yes" (irreversible):**
  - **Production deploy** (anything serving real users / a public domain)
  - **Production schema migration** or any destructive DB operation (drop, alter, data backfill)
  - **DNS / custom domain** changes
  - **Anything that sends real money, real emails, or real messages to real users**

A global "don't ask permissions" setting does NOT override these stops. If Claude Code is in bypass mode, this gate re-introduces a confirmation specifically for the actions above.

> **Note — checkpoint commits vs. deploy pushes.** During the build phase the engine commits and pushes a **green-run checkpoint** to the *feature/dev branch* after every green test/eval run (loop-orchestrator Phase 3 rule 8) — that is reversible and autonomous. Promoting any branch to `main`/the prod branch, or a push that triggers a production deploy, is **this gate's** territory and follows the hard-stop rules above. Checkpoint ≠ deploy: the engine never pushes to `main` on its own.

## When to Trigger

Fire this the moment any of these is imminent:
- `railway up`, `vercel --prod`, `netlify deploy --prod`, `fly deploy`, `git push` to a deploy-linked branch
- `supabase db push`, `prisma migrate deploy`, any migration against a non-dev database
- Pointing a custom domain or changing DNS
- Flipping a feature flag that exposes the build to real users

## The Gate (run in order)

### Step 1 — Re-scan the BUILT artifact (not the design)

Run against the actual code that's about to ship:

- **Secrets:** scan for hardcoded keys, tokens, passwords, connection strings, `.env` values committed to the repo. (e.g., `git grep -nE "(api[_-]?key|secret|token|password|bearer)" `, check for committed `.env`, scan for high-entropy strings.) **Any hit → HARD STOP.**
- **PII exposure:** confirm PII fields identified in `security-baseline` are still encrypted/handled as specced; confirm no PII in logs, URLs, or client-side bundles.
- **Unintended public surface:** list every route/endpoint that will be reachable. Flag any that are public but were specced as authed, debug/admin endpoints, or anything not in the approved API contract.
- **Drift check:** diff the shipped endpoints + dependencies against what `security-baseline` and `api-contract-definition` approved. Name anything new.

### Step 2 — Re-check dependencies AS INSTALLED

Design-time checked the *chosen* deps; this checks the *resolved tree*:
- Run the ecosystem audit (`npm audit`, `pip-audit`, `cargo audit`, etc.) against the lockfile.
- **Any critical/high CVE → HARD STOP** until resolved or explicitly accepted by the human with reason logged.
- Flag any dependency present in the lockfile that wasn't in the approved design.

### Step 3 — Confirm environment & secrets management

- Confirm secrets are sourced from the platform's env/secret store (Railway variables, Vercel env, Supabase vault) — **never** from committed files.
- Confirm staging vs. production targets are not crossed (you are not about to push dev keys to prod or vice versa).
- Confirm the deploy target is the intended one (right project, right environment).

### Step 4 — The irreversible-action approval (the hard stop)

For each irreversible action about to run, present it explicitly and wait for a human "yes":

```
⛔ DEPLOY GATE — irreversible action requires your approval:

Action:        [e.g., supabase db push → PRODUCTION]
Target:        [project / environment / URL]
What changes:  [migration summary / files deployed / endpoints going live]
Reversible?    NO — [why: schema change / public URL / etc.]
Rollback plan: [how to undo if it goes wrong, or "NONE — proceed with caution"]

Scan results:  secrets [✓/✗] · PII [✓/✗] · public surface [✓/✗] · CVEs [✓/✗]

Approve this action? (explicit yes required — bypass-permissions does not apply here)
```

Only on explicit human approval does the action run. If denied → stop, report, await direction.

## Output

```markdown
## Deploy Gate Report
Artifact: [commit/branch]   Target: [platform / env]

### Built-artifact scan
- Secrets: [✓ clean / ✗ findings]
- PII handling: [✓ matches design / ✗ drift]
- Public surface: [endpoints + any unexpected]
- Drift vs. approved design: [none / list]

### Dependencies (as installed)
- Audit: [clean / N critical, N high]
- New deps not in design: [none / list]

### Irreversible actions pending approval
- [ ] [action] → [target]  — approved by human? [pending/yes/no]

### Verdict
[ ✅ Cleared for staging — auto-proceeding ]
[ ⛔ Production deploy / migration — HELD for explicit human approval ]
```

## Handoff (→ qa-verify)

- **Staging/preview:** clear to proceed autonomously; report the URL and trigger `qa-verify` for the full verification pass.
- **Production / migration:** only after explicit human approval given in this gate. After a successful prod deploy, confirm rollback path is known, then trigger `qa-verify` for prod verification and the post-deploy monitoring window.

This gate's job ends when the deploy command succeeds. Verification of the *running product* belongs to `qa-verify` — the chain is complete only when its prod monitoring window closes clean. Ongoing findings feed back through the eval/feedback loop, not through re-running this gate.

## Anti-Patterns
- ❌ Trusting the design-time security check to cover the shipped code → ✅ re-scan the actual artifact
- ❌ Letting "don't ask permissions" auto-approve a prod deploy or migration → ✅ those always hard-stop
- ❌ Treating staging and prod the same → ✅ staging auto, prod gated
- ❌ Running a migration with no rollback plan → ✅ state the rollback (or flag its absence) before approval
- ❌ Scanning chosen deps but not the resolved lockfile → ✅ audit as installed
