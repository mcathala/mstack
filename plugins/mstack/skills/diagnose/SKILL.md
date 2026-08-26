---
name: diagnose
description: Disciplined debug loop for stuck-red tests, runtime errors, and mysterious metric regressions. Reproduce → hypothesise → instrument → fix → regression-test. Use when a test won't go green during implementation, when /validate reports a metric regression with no obvious cause, when the user says "diagnose this", "debug this", "why is this failing", or describes a bug or unexpected metric drop.
---

# diagnose

A discipline for hard failures. Two trigger flavors:

- **Stuck-red bug** — implementation hit a test that refuses to go green, or runtime code throws unexpectedly.
- **Metric regression** — `/validate` reports a fail with no obvious cause: empirical metric dropped, became flaky, or moved in an unexpected direction.

Skip phases only when explicitly justified.

## Phase 1 — Build a feedback loop

**This is the skill.** Everything else is mechanical. With a fast, deterministic, agent-runnable pass/fail signal you will find the cause — bisection, hypothesis-testing, and instrumentation all consume that signal. Without one, staring at code won't save you.

Spend disproportionate effort here. **Be aggressive, be creative, refuse to give up.**

### Ways to construct one

Software-flavored:

1. **Failing test** at whatever seam reaches the bug (unit, integration, e2e).
2. **Scripted invocation** — curl/HTTP script, CLI with fixture input diffed against a snapshot, or a headless browser script (Playwright/Puppeteer) that drives the UI and asserts on DOM/console/network.
3. **Replay a captured trace** — save a real network request / payload / event log to disk; replay it through the code path in isolation.
4. **Throwaway harness** — minimal subset of the system (one service, mocked deps) that exercises the bug code path with a single function call.
5. **Bisection harness** — automate "boot at state X, check, repeat" so `git bisect run` works.

Data-flavored:

6. **Fixture-scale repro** — shrink the data to N=10 rows where you can hand-check the answer.
7. **Stratified subsample** — held-out 1% of the data for fast iteration; verify the regression reproduces there.
8. **Differential pipeline** — same input through old (working) vs new (broken) pipeline; diff intermediate outputs at each transform.
9. **Single-fold isolation** — serialize the failing CV split, run just that fold, inspect.

Last resort:

10. **HITL bash script** — if a human must click, drive *them* with a structured loop so captured output still feeds back to you.

### Iterate on the loop itself

Treat the loop as a product. Once you have *a* loop, ask:

- Can I make it faster? (Cache setup, skip unrelated init, narrow scope.)
- Sharper signal? (Assert on the specific symptom, not "didn't crash".)
- More deterministic? (Pin time, seed RNG, isolate filesystem, freeze network.)

A 30-second flaky loop is barely better than no loop. A 2-second deterministic loop is a debugging superpower.

### Non-deterministic failures

Goal is not a clean repro but a **higher reproduction rate**. Loop the trigger 100×, parallelise, add stress, narrow timing windows. A 50%-flake bug is debuggable; 1% is not — keep raising the rate until it is.

### When you genuinely cannot build a loop

Stop. List what you tried. Ask the user for: (a) access to whatever environment reproduces it, (b) a captured artifact (HAR file, log dump, dataset slice, screen recording with timestamps), or (c) permission to add temporary production instrumentation. **Do not proceed to hypothesise without a loop.**

Do not proceed to Phase 2 until you have a loop you believe in.

## Phase 2 — Reproduce

Run the loop. Watch the failure appear.

Confirm:

- [ ] The loop produces the failure mode the **user / triggering skill** described — not a different one nearby. Wrong bug = wrong fix.
- [ ] The failure is reproducible across multiple runs (or, for stochastic failures, at a debuggable rate).
- [ ] The exact symptom is captured (error text, wrong output, regressed metric value) so later phases can verify the fix actually addresses it.

## Phase 3 — Hypothesise

Generate **3–5 ranked, falsifiable hypotheses** before testing any. Single-hypothesis generation anchors on the first plausible idea.

Each hypothesis must state its prediction:

> "If X is the cause, then changing Y will make the failure disappear / changing Z will make it worse."

If you cannot state the prediction, the hypothesis is a vibe — discard or sharpen it.

For metric regressions, common suspects:

- Train/test leakage introduced by a new transform
- Fold composition changed (different seed, different stratification)
- Distribution shift between training and eval data
- Encoder fitted on the wrong slice (whole dataset instead of train fold)
- Hyperparameter changed silently (default updated, config drifted)
- Eval set or metric definition changed under the hood

**Show the ranked list to the user before testing.** They often have domain knowledge that re-ranks instantly ("we just deployed a change to #3"). Cheap checkpoint, big time saver. Don't block on it — proceed with your ranking if the user is AFK.

## Phase 4 — Instrument

Each probe must map to a specific prediction from Phase 3. **Change one variable at a time.**

Tool preference:

1. **Debugger / REPL inspection** if the env supports it. One breakpoint beats ten logs.
2. **Targeted logs** at the boundaries that distinguish hypotheses.
3. Never "log everything and grep".

**Tag every debug log** with a unique prefix, e.g. `[DEBUG-a4f2]`. Cleanup at the end becomes a single grep. Untagged logs survive; tagged logs die.

**Perf branch.** For performance regressions, logs are usually wrong. Establish a baseline measurement (timing harness, `performance.now()`, profiler, query plan), then bisect. Measure first, fix second.

## Phase 5 — Fix + regression test

Write the regression test **before the fix** — but only if there is a **correct seam** for it.

A correct seam is one where the test exercises the **real failure pattern** as it occurs at the call site. If the only available seam is too shallow (single-caller test when the bug needs multiple callers, unit test that can't replicate the chain that triggered the bug), a regression test there gives false confidence.

**If no correct seam exists, that itself is the finding.** Note it. The codebase architecture is preventing the bug from being locked down. Surface the architectural friction to the user as part of the post-fix summary.

If a correct seam exists:

1. Turn the minimised repro into a failing test at that seam.
2. Watch it fail.
3. Apply the fix.
4. Watch it pass.
5. Re-run the Phase 1 feedback loop against the original (un-minimised) scenario.

For metric regressions, the regression "test" is often a tiny eval harness that runs against a fixture and asserts the metric is within an envelope of the baseline.

## Phase 6 — Cleanup + hand back

Required before declaring done:

- [ ] Original repro no longer reproduces (re-run the Phase 1 loop)
- [ ] Regression test passes (or absence of correct seam is documented)
- [ ] All `[DEBUG-...]` instrumentation removed (`grep` the prefix)
- [ ] Throwaway prototypes deleted, or moved to a clearly-marked debug location
- [ ] The hypothesis that turned out correct is stated in the commit / PR message — so the next debugger learns

**Then hand back to the triggering skill:**

- Triggered during implementation (stuck red) → return to the implementation and continue from the task that was blocked.
- Triggered by `/validate` (metric regression) → return to `/validate`, re-run the metric, then recommend a loop-back depth based on whether the fix moved it.

## What would have prevented this?

After the fix is in, ask: what architectural change would have made this impossible? If the answer involves a real shift (no good test seam, tangled callers, hidden coupling, fragile data pipeline), surface the specifics to the user as a post-mortem finding. Make the recommendation **after** the fix, not before — you have more information now than when you started.

## Do not

- Hypothesise without a feedback loop. The order matters.
- Test multiple variables at once.
- Skip the post-fix re-run of the original repro. "Test passes" is not "bug is gone."
- Leave tagged debug logs in place after declaring done.
- Continue if you genuinely cannot build a loop. Surface to the user instead.
