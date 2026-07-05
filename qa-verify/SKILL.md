---
name: qa-verify
description: |
  The verification loop that drives the running product through its real user journeys — the gate that proves the thing actually works, with real browser evidence, not passing unit tests. It runs at THREE points: **pre-handoff** (localhost / dev server, the moment the build is complete and a human is about to use the product for the FIRST time — no deploy required), on **staging** after deploy-gate clears an artifact, and on **production** after an approved prod deploy. Use whenever: the build engine reports the build / last story complete and a human is about to use the product, a staging/preview deploy just completed, a production deploy was just approved and executed, the user says "verify the deploy", "QA this", "click through it", "check prod", "is it actually working", or an autonomous build loop reports a story/epic as done. Also trigger for post-deploy monitoring requests ("watch for errors", "any regressions since the deploy"). This skill drives a REAL browser through the critical user journeys defined in the Eval Spec, captures console errors and network failures, compares performance against baseline, and — on pre-handoff and staging (never prod) — runs a bounded fix-and-regression-test loop. It is the evidence-producing gate: no journey is "verified" without a screenshot or console-clean check to prove it. Trigger this even when all tests passed in CI; tests validate code, this validates the running product. The pre-handoff pass is non-negotiable: no artifact reaches deploy-gate until the product has been driven like a user on localhost and a day-0 first-run checklist is green.
---

# QA Verify

The verification loop that drives the running product. Everything upstream validated the plan (gates) and the code (tests + evals in the build loop). This skill validates the **running product** — real browser, real clicks, real evidence. It runs the first time a human is about to use the product, not just after a deploy.

## Position in the chain (three run modes)

```
[build engine: last story done]
        │
        ▼
qa-verify (PRE-HANDOFF · localhost, no deploy)      ← runs BEFORE deploy-gate
        │   journeys + day-0 first-run checklist
        │   ▲ day-0 fix loop (localhost) ─┐
        └───┴──────────────────────────────┘
        │ green
        ▼
deploy-gate (staging auto → prod HARD STOP)
        │
        ▼
qa-verify (STAGING · live preview URL)  ──► fix loop (staging only)
        │ green
        ▼
prod approval ──► qa-verify (PROD + monitoring window)
```

- **Pre-handoff** (localhost, no deploy): runs the moment the build engine reports the build complete and a human is about to use the product for the first time. This is the gate that closes the first-run/integration gap — the product is driven like a user *at the exact moment a human becomes the user*, not later after a deploy. No artifact reaches deploy-gate until this is green.
- **Staging / prod**: deploy-gate's job ends the moment the deploy command succeeds; qa-verify's staging/prod passes start the moment a URL is live.
- Routing between modes is the **orchestrator's** call — this gate signals completion and never routes itself. Findings feed the runtime eval loop defined in the Eval Spec (Step 7) — not a re-run of upstream gates.

## Inputs (read these first)

1. **Eval Spec** (from `eval-spec`): the critical user journeys + binary pass/fail criteria. These ARE the QA script. Do not invent journeys; verify the specced ones.
2. **API contract** (from `api-contract-definition`): expected endpoints + error taxonomy, for network-request verification.
3. **Deploy Gate Report**: the target URL/environment and what changed — scope verification to what shipped.
4. **Performance baseline** (`.pipeline/perf-baseline.json` if it exists): prior load times / Core Web Vitals to diff against. If absent, this run CREATES the baseline.

If the Eval Spec has no Critical User Journeys section (older spec format), do not halt: compile the journey list from its user-facing deterministic tests and guardrails, record the compiled list in `.pipeline/state.yaml` for human review, and flag that the Eval Spec should add a journeys section in its next revision. If neither journeys nor user-facing criteria exist → STOP: the spec was not autonomous-ready; route back to eval-spec.

## The Verify Pass (run in order)

The journey steps (1–5) run in **every** mode against the mode's target (localhost / staging URL / prod URL). Step 0 runs in **pre-handoff mode only** — it is the first-run gate.

### Step 0 — Day-0 first-run checklist (PRE-HANDOFF only)

This is the check the Level-3 distribution-agent build failed: 56/56 tests green, every design gate passed, and the product still broke in the founder's first five minutes — because the failures were first-run/integration failures no earlier gate drives. Run these against `localhost` / the dev server, from the state a brand-new user actually starts in:

- **Fresh-account onboarding completes end-to-end.** Sign up / sign in with a brand-new account and get all the way to first value. **FAIL on:** an email-confirmation dead-end (a link that never arrives or can't be clicked in dev), an unconfigured OAuth provider, or any step that assumes an already-provisioned account.
- **Every P0 journey works with ZERO optional infrastructure running.** No background worker, no LLM/API key, no third-party dashboard configured. A P0 journey that needs a degraded-but-working fallback must *have* one. **"Start these N processes / set these keys first" is a FAIL**, not a setup note — the default-clone-and-run state is the test.
- **Onboarding consumes the user's REAL input.** If the flow asks for a URL, a file, a handle — the product must actually fetch/read/process *that input*, not return placeholder or canned output. Enter a real value the founder would enter and verify it flows through.

Any Step 0 failure blocks handoff to deploy-gate. On localhost this is fixable in the bounded fix loop below; record each finding with evidence.

### Step 1 — Smoke: is it even up?
- Load the root URL and every top-level route from the API contract.
- HARD FAIL: any 5xx, blank render, or infinite spinner → skip to Step 5 (report) immediately for prod; enter fix loop for staging.

### Step 2 — Journey verification (the core)
For each critical journey in the Eval Spec, in priority order:
- Drive it in a real browser (claude-in-chrome tools / Playwright in CI): real clicks, real form input, real navigation. No mocked responses.
- At each journey step capture: screenshot, console messages, failed network requests.
- **Pass = the Eval Spec's binary criterion met AND console clean AND no unexpected 4xx/5xx.** A journey that "looks done" with console errors is a FAIL.
- Test at two viewport widths minimum (mobile ~380px, desktop ~1280px). Layout breakage at either = FAIL.

### Step 3 — Negative-path spot checks
- Submit each specced form empty and with malformed input; confirm errors match the API contract's error taxonomy (no raw stack traces, no 500s for user error).
- Hit one authed endpoint unauthenticated; confirm it denies. (This re-proves deploy-gate's public-surface scan against the live system.)

### Step 4 — Performance diff
- Measure page load and Core Web Vitals for the top 3 journeys.
- Compare to baseline. Regression >20% on any metric = FLAG (not fail); >50% = FAIL.
- Write the new numbers to `.pipeline/perf-baseline.json` only if the run passes.

### Step 5 — Report (always produced)

```markdown
## QA Verify Report
Target: [URL / env]   Commit: [sha]   Run: [timestamp]

### Journeys (from Eval Spec)
| # | Journey | Result | Evidence |
|---|---------|--------|----------|
| 1 | [name]  | ✓/✗    | [screenshot path / console note] |

### Negative paths: [✓ / findings]
### Console: [clean / N errors — listed]
### Network: [clean / unexpected responses — listed]
### Performance vs baseline: [within band / regressions listed]

### Verdict
[ ✅ VERIFIED — evidence attached ]
[ ⛔ FAILED — N findings, fix loop status below ]
```

## The Fix Loop (PRE-HANDOFF & STAGING — never prod — bounded)

On pre-handoff (localhost) or staging failures, run: **diagnose → fix → write a regression test that would have caught it → re-run the failed journey → re-run the full pass if the fix touched shared code.** A pre-handoff Step 0 finding writes the regression test into the day-0 checklist's coverage so the same first-run failure can never recur silently.

Bounds (hard, to keep the loop from thrashing):
- Max **3** fix iterations per finding, max **2** full-pass re-runs per deploy.
- A fix may not add dependencies, change the API contract, or touch auth/PII handling — those route back through the relevant upstream gate.
- Budget exhausted → stop, report all findings, await human direction.

**PRODUCTION failures are NEVER auto-fixed.** Prod finding → report immediately, state rollback recommendation (from deploy-gate's rollback plan), and wait. The fix ships through the full chain.

## Post-Deploy Monitoring Window (prod only)

After a verified prod deploy, monitor for a defined window (default 30 min active, then hand off to platform alerting):
- Re-poll console/network on the top journey every ~10 min.
- Watch platform logs (Railway/Vercel/Supabase) for new error signatures.
- Any new error signature or availability drop → alert the human immediately with the rollback recommendation. Do not act autonomously in prod.

At window close: append monitoring summary to the QA Verify Report and log journey-level results as the first entries in the runtime eval dashboard specified in the Eval Spec (Step 7).

## Handoff

- **VERIFIED (pre-handoff):** signal the orchestrator — journeys + day-0 checklist are green on localhost; the artifact is cleared to proceed to deploy-gate. This is what "the build is ready for a human" means.
- **VERIFIED (staging):** signal the orchestrator — artifact is cleared for deploy-gate's production approval step.
- **VERIFIED (prod) + monitoring clean:** pipeline complete. Findings backlog (flags, perf notes) → route to brainstorm/eval-spec as future stories.
- **FAILED:** report + fix-loop status → orchestrator decides (pre-handoff/staging: loop; prod: human). A failed pre-handoff pass NEVER advances to deploy-gate — first-run has to work before the world sees it.

## Anti-Patterns
- ❌ "Tests pass, ship it" → ✅ tests validate code; this validates the product. Both required.
- ❌ Verifying journeys Claude invented → ✅ verify the Eval Spec's journeys; gaps in the spec are eval-spec findings.
- ❌ Marking a journey passed with console errors → ✅ console-clean is part of pass.
- ❌ Unbounded fix loops → ✅ 3 per finding, 2 full re-runs, then human.
- ❌ Auto-fixing production → ✅ prod findings get a rollback recommendation and a human.
- ❌ Screenshots as decoration → ✅ evidence or it didn't happen: every ✓ has an artifact.
- ❌ Handing the founder a "done" build verified only by unit tests → ✅ pre-handoff drives the real first-run journey on localhost before any deploy.
- ❌ "Works once you start the worker / set the key / configure the dashboard" → ✅ every P0 journey works from a clean clone with zero optional infra, or it's a FAIL.
