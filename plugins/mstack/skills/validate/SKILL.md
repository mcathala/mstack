---
name: validate
description: Run a local check of the current iteration against the declared metric and return pass/fail with a loop-back recommendation. Use when user says "validate", "is this ready", "did it work", "check the metric", or after /start-plan finishes.
---

# validate

Step 6 of the mstack flow. Verify the iteration's output meets the metric declared at step 1, **locally**, before push.

## Inputs

- **The metric.** Declared at the start of the iteration ("ship with AUC > 0.85", "page loads under 1s", "all tests pass and types check"). If you cannot find it in conversation context or the parent PRD, ask **once** — don't proceed without one.
- **The baseline.** For iteration 2+, the previous best result. Without a baseline, "improvement" is meaningless.

## Phase 1 — Classify the metric

Two flavors. Most iterations have a mix:

1. **Deterministic** — "tests pass", "typecheck clean", "endpoint returns 200", "command exits 0". Pass/fail is binary.
2. **Empirical** — "AUC > X", "P95 latency < Y ms", "leaderboard score", "conversion lift". Pass/fail is threshold against measurement.

State both before running so the user can correct your reading of the metric.

## Phase 2 — Run the check locally

- **Deterministic:** run the test suite, typecheck, smoke-test the entry point. Capture exit codes and any failure output.
- **Empirical:** run the eval harness against the holdout / dev set / staging fixture. Capture the raw number.

**Local only.** Do not validate in prod or against the live leaderboard — that's downstream of /ship. The whole point of validate is to catch issues before the world sees them.

If the harness doesn't exist (first iteration, empirical metric), surface that — building the harness is itself a /start-plan task.

## Phase 3 — Report

Produce a short validation report:

```markdown
## Validation report
- Iteration: <name or issue#>
- Metric (deterministic): <pass | fail with one-line failure detail>
- Metric (empirical): <value> vs baseline <value> (Δ <delta>)
- Decision: PASS / FAIL
```

Keep it tight. The report is for humans skimming, not for archival.

## Phase 4 — On fail, recommend a loop-back depth

Don't just say "try again." Diagnose where the failure originates:

| What broke | Loop back to |
|---|---|
| Code error, failing test, runtime crash | step 5 — `/start-plan` |
| Spec was wrong — code does what was asked, but the metric reveals the ask was wrong | step 3 — rewrite PRD |
| Alignment was wrong — even a correct spec wouldn't help; we're solving the wrong problem | step 2 — `/sync` |
| Framing was wrong — the iteration's idea or scope is off | step 1 — new iteration |

Recommend **one** depth and explain in two lines. The user picks; don't auto-act.

If the cause isn't obvious (failing in unexpected ways, mysterious metric regression), invoke `/diagnose` before recommending — diagnosis comes before depth choice.

## Phase 5 — Hand off

- **PASS** → suggest `/ship`. Do not push directly.
- **FAIL** → state the recommended loop-back depth and stop. The user decides.

## Do not

- Lower the threshold to make it pass. The threshold is part of the contract.
- Validate against a metric different from the one declared at step 1. Changing the goalpost mid-iteration is the most common way teams fool themselves.
- Push, commit, or deploy. /ship's job, only after PASS.
- Invoke /start-plan automatically on fail. Recommend it, let the user trigger it.
