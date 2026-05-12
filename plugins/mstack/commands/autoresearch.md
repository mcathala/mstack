---
description: Hypothesis-driven autonomous ML research loop. Runs cycles on one session branch until interrupted; never pushes, merges to main, or submits without approval. Delivery is /ship's job.
argument-hint: "[focus=auto] [cycles=N] — e.g. features, tuning cycles=3"
allowed-tools: Bash(git:*), Bash(python:*), Bash(uv:*), Bash(make:*), Bash(ls:*), Bash(cat:*), Bash(tail:*), Bash(head:*), Bash(find:*), Bash(grep:*), Bash(sha256sum:*), Bash(md5:*), Bash(pip:*), Read, Write, Edit, Grep, Glob, AskUserQuestion, TodoWrite
---

# /autoresearch

Autonomous ML research loop. Each invocation is a **session** that runs many cycles on a single branch. You interrupt and run `/ship` when satisfied; that's how work leaves the loop. Inspired by `karpathy/autoresearch`'s `project.md` pattern (human-editable config + comparable fixed-budget experiments), adapted for tabular / Kaggle-style work where **overfitting is the primary risk**.

`$ARGUMENTS` — all optional. `focus` is one of `features`, `tuning`, `cv`, `model`, `ensemble`, `eda`, `auto` (default). `cycles=N` caps the run at N cycles; the real exit is you hitting Ctrl-C.

## Context

- project.md: !`cat project.md 2>/dev/null | head -150 || echo "MISSING — bootstrap required"`
- Leaderboard tail: !`tail -15 experiments/README.md 2>/dev/null || echo "MISSING — bootstrap required"`
- Branch: !`git branch --show-current`
- Working tree: !`git status --short`
- HEAD: !`git rev-parse --short HEAD 2>/dev/null`

---

## HARD RULES (non-negotiable)

The loop owns iteration. Delivery is `/ship`'s job. So:

- **NEVER `git push`, merge to a protected branch, or submit to any leaderboard / Kaggle without explicit user approval in chat.** Protected branches are `main`, `master`, `staging`, plus anything listed in `project.md`. Commits to the session branch are expected — that's the loop's working memory.
- **NEVER modify CV splits, seeds, evaluation code, or target transform mid-session.** Comparability across cycles is sacred. If a change is needed, stop and ask.
- **One hypothesis per cycle.** No drive-by refactors, no "while I'm here" tweaks.
- **No tuning on the public leaderboard.** Trust local CV.
- **Stop and ask** when:
  - A change would touch data leakage, target encoding, CV strategy, target reversibility, or submission logic
  - Submission budget hit the warn threshold from `project.md` (default 50%) or is exhausted
  - Prior experiment artifacts would be deleted or overwritten
  - A refactor is required to test the hypothesis (refactor escape valve, Phase 3)
  - Dataset version, package versions, or feature-store schema has changed since the last cycle (breaks comparability)

---

## SESSION SETUP (once per invocation, before cycle 1)

1. If the current branch is protected (`main`/`master`/`staging` or anything in `project.md`'s protected list), **stop and ask**: "Create `autoresearch/<YYYY-MM-DD>-<slug>` from here, or switch to an existing branch first?" Never run on a protected branch.
2. If already on an `autoresearch/*` branch, treat this as a resumed session and keep going on it.
3. Otherwise use the current branch as the session branch.
4. Record session start: branch name, starting SHA, timestamp. These go into every cycle's repro stamp.

The session branch is the unit of work. Every cycle commits to it. When the user interrupts and runs `/ship`, that branch's history is the PR.

---

## BOOTSTRAP (first run only)

If `project.md` is missing **or** `experiments/` lacks `README.md` and `TEMPLATE.md`:

1. Detect from the repo: metric, target column, data files, CV strategy if any, source dir.
2. Generate drafts of `project.md`, `experiments/README.md`, `experiments/TEMPLATE.md`, `experiments/001_baseline.md`.
3. Show the generated `project.md` and ask via AskUserQuestion: *"Validate / edit / abort?"*
4. Do not run cycle 1 until validated.

Ensure `.gitignore` excludes large per-experiment artifacts (model binaries, predictions) but keeps the markdown.

---

## THE LOOP

Read `project.md` at the **start of every cycle** — it is the source of truth for metric, CV, budgets, frozen files. After each cycle, commit and start the next. Continue until interrupted, until `cycles=N` is reached, or until a "stop and ask" trigger fires.

The session branch's tip should always reflect the **current best**. Non-keep cycles record the attempt in the experiment file but revert their code changes (Phase 5 handles this).

### Phase 1 — REVIEW

Read in order, stop early if you have what you need:
1. `project.md` (config may have changed)
2. `experiments/README.md` (leaderboard)
3. The current best experiment file in full
4. The most recent experiment (last cycle)
5. Any prior experiment whose hypothesis is closest to candidates you're considering

Write a 3-bullet diagnosis of the current best's biggest weakness. Pick the dominant signal:

- High fold variance (`fold_std / |fold_mean| > 0.10`) → **CV instability** — focus `cv` or feature stability
- Large train-val gap (> 0.5 × val baseline, metric-aware) → **overfit** — regularize, fewer features, more data
- Last 3 deltas all `< 0.5 × CV std` → **plateau** — structurally different model family
- Errors concentrated in an identifiable slice → **coverage gap** — targeted FE on that slice
- One fold consistently worst → **hidden leakage / shift** — `cv` revalidation
- Nothing clear → `eda` cycle

If the `focus` arg is set, it overrides this pick.

### Phase 2 — HYPOTHESIZE

Before touching code, stamp `experiments/NNN_<slug>.md` from `TEMPLATE.md` with:

- **Hypothesis** — one sentence, falsifiable, quantitative target ("≥1% CV MAE reduction vs cycle NNN")
- **Rationale** — 2–3 bullets grounded in Phase 1
- **Expected magnitude** — rough estimate
- **Overfitting risk** — low / medium / high + why
- **Kill criteria** — concrete result that would mean abandoning this *direction*, not just this cycle
- **Scope** — files to be touched, rough line count. Anything beyond = a new cycle.
- **Reversibility** — does this change CV / seed / target transform / leakage surface? If yes → STOP and ask.

### Phase 3 — IMPLEMENT

- Make the **smallest change** that tests the hypothesis. No drive-by refactors.
- Reuse the same seed and CV folds as the prior cycle.
- **Refactor escape valve**: if the hypothesis genuinely cannot be tested without restructuring, abort the cycle, surface the refactor as a separate proposal, get approval, do it on its own branch, then restart the cycle.

### Phase 4 — VALIDATE

Run the project's evaluation. Capture and report all of:

- **Per-fold scores** (full table — never just the mean)
- **CV mean ± std** alongside baseline mean ± std
- **Train vs val gap** (new vs old, absolute and relative)
- **Reproducibility stamp**: git SHA, data hash (sha256 of train file or feature manifest), Python/key-package versions
- Whether each pre-registered kill criterion fired
- Where applicable: feature-importance delta and residual-distribution diff vs prior best

#### Auto-verdict gates

A cycle is `keep` **only if all hold**:

1. **Magnitude**: mean CV improved by `≥ max(0.5 × baseline_cv_std, min_delta)` where `min_delta` comes from `project.md`.
2. **Direction agreement**: `≥ ⌈0.6 × n_folds⌉` folds improved.
3. **Stability**: worst fold did not regress by more than `1 × baseline_cv_std`.
4. **Generalization**: train-val gap did not grow beyond the threshold in `project.md` (default: max of +20% relative OR +0.5 × baseline gap absolute).
5. **Kill criteria**: none fired.

Otherwise: `discard` (clearly worse, or kill fired) or `inconclusive` (within noise, no red flags).

#### Failure modes

- Non-finite metric, crash, OOM, evaluation didn't complete → `infra-fail`. Record in the experiment file, do not add a leaderboard row, return to Phase 1.
- Detected mid-run that CV / seed / data version changed silently → `infra-fail` + flag to user.

#### Inconclusive follow-up

Write a one-line follow-up plan in the experiment file — typically "re-run with seed=99 and seed=7, treat as keep iff median verdict is keep across all 3."

#### Robustness check (periodic)

Every `robustness_check_every` cycles (`project.md`, default 10), OR before any leaderboard submission, OR when requested: re-validate the **current best** on two additional seeds. If the 3-seed mean drops below the prior best by more than `min_delta`, flag seed-overfit and propose a rollback.

### Phase 5 — DOCUMENT, COMMIT, CONTINUE

1. Finalize `experiments/NNN_<slug>.md` — Result, Verdict, Kill-criteria check, Repro stamp, Learnings, Follow-ups.
2. Append one row to `experiments/README.md` *unless* verdict is `infra-fail`.
3. If verdict is `keep`, update `project.md`'s **Current best** section.
4. **Commit to the session branch**, by verdict:
   - `keep` — stage code changes + experiment file + leaderboard row + `project.md` update. One commit. Message: `exp NNN: <slug> — keep (CV <mean>±<std>)`.
   - `discard` / `inconclusive` — revert code changes (`git checkout -- <files>`), commit only the experiment file + leaderboard row. Message: `exp NNN: <slug> — <verdict>`.
   - `infra-fail` — revert code changes, commit only the experiment file. Message: `exp NNN: <slug> — infra-fail`.
5. Report to chat in exactly this shape:

```
## Cycle NNN — <slug>

Hypothesis: <one line>
Result:     CV <mean> ± <std>  (baseline: <mean> ± <std>)  — folds: [...]
Gap:        train-val <new> (was <old>, Δ <relative>%)
Verdict:    keep | discard | inconclusive | infra-fail
Gates:      magnitude pass/fail  agreement pass/fail  stability pass/fail  generalization pass/fail  kill-criteria pass/fail
Repro:      git=<sha>  data=<hash[:8]>  pkg=<key versions>
Diff:       <files changed>, <+net lines>

Session: <M> cycles run, <K> keeps. Branch tip CV: <mean ± std>.
Continuing to cycle NNN+1 — interrupt to stop and run /ship.
```

6. Begin cycle NNN+1.

The loop **only stops when**:
- The user interrupts (Ctrl-C / new message),
- `cycles=N` cap is reached,
- A hard-rule "stop and ask" trigger fires.

It does **not** stop for per-cycle approval. Per-cycle decisions are made by the gates, not the user.

---

## ROTATION (when focus=auto)

Pick by Phase 1 diagnosis. When ambiguous, prioritise:

1. **CV validation** — first cycle, and any time leakage / shift is suspected
2. **EDA / error analysis** — slice errors, find systematic mistakes
3. **Feature engineering** — one *family* per cycle (lags / rolling / interactions / encodings)
4. **Data cleaning** — outliers, missingness, target sanity
5. **Target transformation** — log / rank / clip / residual
6. **Model diversity** — different family before going deeper on one
7. **Hyperparameter tuning** — only after architecture and features are stable; small Optuna budget first
8. **Ensembling / stacking** — only with diverse, individually strong bases
9. **Calibration / post-processing** — last-mile

Skip any stage that doesn't apply per `project.md`.

---

## `project.md` TEMPLATE

```markdown
# project.md — config for /autoresearch

## Project
- Name: <e.g. kaggle-f1-pitstop>
- Goal: <regression / classification / ranking>
- Metric: <RMSE / LogLoss / MAP@k>  (lower-better: true|false)

## Data
- Train: <path>
- Test: <path>
- Target column: <...>
- Group/time column for CV: <e.g. raceId, season>
- Data version / hash: <sha256 of train file, or manifest hash>

## CV
- Strategy: <GroupKFold by raceId | TimeSeriesSplit | StratifiedKFold>
- n_splits: <set to fit your dataset — 5 is a common default; 7-10 for high-variance metrics, 3 for slow models>
- Random seed: 42
- Robustness seeds (for periodic re-validation): [7, 99]

## Thresholds (auto-verdict gates)
- min_delta: <smallest improvement in metric units that counts as signal — try 0.25 × baseline_cv_std if unsure>
- max_gap_increase_relative: 0.20
- max_gap_increase_absolute: <e.g. 0.5 × current gap>
- robustness_check_every: 10  # cycles

## Budgets
- Wall-clock per experiment: <e.g. 10 min>
- Submissions: used 0 / 30, warn at 50%
- Compute: <CPU / 1×GPU / etc>

## Current best
- Branch / commit: <...>
- CV: <mean ± std>  (per-fold: [...])
- Public LB: <if known>
- Last robustness check: <cycle NNN, seeds [42,7,99] → ...>

## Conventions
- Source dir: src/
- Models: models/
- Protected branches (loop refuses to run on these): [main, master, staging]
- Frozen files (do not touch without approval): [<...>]
- Evaluation entry point: <e.g. `make cv` or `python -m src.cv`>

## Open questions / hunches  (free text — human edits this)
- ...
```

---

## INVOCATION

```
/autoresearch                  # focus auto, run until interrupted
/autoresearch features         # focus features, run until interrupted
/autoresearch cycles=3         # cap at 3 cycles, focus auto
/autoresearch tuning cycles=5  # focus tuning, cap at 5
```

When satisfied with the session's accumulated improvement, interrupt and run `/ship`. The session branch's history is your PR.

Begin by: (1) running session setup, (2) running bootstrap if needed, (3) reading `project.md`, (4) entering Phase 1 of cycle 1.
