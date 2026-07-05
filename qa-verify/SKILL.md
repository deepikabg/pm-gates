---
name: qa-verify
description: |
  The post-ship verification loop that runs AFTER deploy-gate clears an artifact — the gate that proves the deployed thing actually works, with real browser evidence, not passing unit tests. Use whenever: a staging/preview deploy just completed, a production deploy was just approved and executed, the user says "verify the deploy", "QA this", "click through it", "check prod", "is it actually working", or an autonomous build loop reports a story/epic as done. Also trigger for post-deploy monitoring requests ("watch for errors", "any regressions since the deploy"). This skill drives a REAL browser through the critical user journeys defined in the Eval Spec, captures console errors and network failures, compares performance against baseline, and — on staging only — runs a bounded fix-and-regression-test loop. It is the evidence-producing gate: no journey is "verified" without a screenshot or console-clean check to prove it. Trigger this even when all tests passed in CI; tests validate code, this validates the running product.
---

# QA Verify

The verification loop after the ship. Everything upstream validated the plan (gates) and the code (tests + evals in the build loop). This skill validates the **running product** — real browser, real clicks, real evidence.

## Position in the chain

```
… security-baseline → [build engine] → deploy-gate → qa-verify   ← YOU ARE HERE
                                            ↑              │
                                            └── fix loop ──┘  (staging only)
```

- deploy-gate's job ends the moment the deploy command succeeds.
- qa-verify's job starts the moment a URL is live (staging or prod).
- Findings feed the runtime eval loop defined in the Eval Spec (Step 7) — not a re-run of upstream gates.

## Inputs (read these first)

1. **Eval Spec** (from `eval-spec`): the critical user journeys + binary pass/fail criteria. These ARE the QA script. Do not invent journeys; verify the specced ones.
2. **API contract** (from `api-contract-definition`): expected endpoints + error taxonomy, for network-request verification.
3. **Deploy Gate Report**: the target URL/environment and what changed — scope verification to what shipped.
4. **Performance baseline** (`.pipeline/perf-baseline.json` if it exists): prior load times / Core Web Vitals to diff against. If absent, this run CREATES the baseline.

If the Eval Spec has no Critical User Journeys section (older spec format), do not halt: compile the journey list from its user-facing deterministic tests and guardrails, record the compiled list in `.pipeline/state.yaml` for human review, and flag that the Eval Spec should add a journeys section in its next revision. If neither journeys nor user-facing criteria exist → STOP: the spec was not autonomous-ready; route back to eval-spec.

## The Verify Pass (run in order)

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

## The Fix Loop (STAGING ONLY — bounded)

On staging failures, run: **diagnose → fix → write a regression test that would have caught it → re-run the failed journey → re-run the full pass if the fix touched shared code.**

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

- **VERIFIED (staging):** signal the orchestrator — artifact is cleared for deploy-gate's production approval step.
- **VERIFIED (prod) + monitoring clean:** pipeline complete. Findings backlog (flags, perf notes) → route to brainstorm/eval-spec as future stories.
- **FAILED:** report + fix-loop status → orchestrator decides (staging: loop; prod: human).

## Anti-Patterns
- ❌ "Tests pass, ship it" → ✅ tests validate code; this validates the product. Both required.
- ❌ Verifying journeys Claude invented → ✅ verify the Eval Spec's journeys; gaps in the spec are eval-spec findings.
- ❌ Marking a journey passed with console errors → ✅ console-clean is part of pass.
- ❌ Unbounded fix loops → ✅ 3 per finding, 2 full re-runs, then human.
- ❌ Auto-fixing production → ✅ prod findings get a rollback recommendation and a human.
- ❌ Screenshots as decoration → ✅ evidence or it didn't happen: every ✓ has an artifact.
