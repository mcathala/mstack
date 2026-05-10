---
description: Hypothesis-driven autonomous ML research loop. Edits, runs, evaluates, documents. Never commits, pushes, or submits without approval.
argument-hint: "[cycles=1] [focus=auto] — e.g. 3 features, 1 tuning, auto"
allowed-tools: Bash(git:*), Bash(python:*), Bash(uv:*), Bash(make:*), Bash(ls:*), Bash(cat:*), Bash(tail:*), Bash(head:*), Bash(find:*), Bash(grep:*), Bash(sha256sum:*), Bash(md5:*), Bash(pip:*), Read, Write, Edit, Grep, Glob, AskUserQuestion, TodoWrite
---

# /autoresearch

Autonomous ML research loop. Inspired by `karpathy/autoresearch`'s `program.md` pattern (human-editable instruction layer + comparable fixed-budget experiments), adapted for tabular / Kaggle-style work where **overfitting is the primary risk**.

`$ARGUMENTS` — first token is a cycle **budget** (default `1`), second is an optional focus (`features`, `tuning`, `cv`, `model`, `ensemble`, `eda`, `auto`). The budget is a ceiling: the loop **always stops for approval after each cycle**, regardless of budget remaining.

## Context

- program.md: !`cat program.md 2>/dev/null | head -80 || echo "MISSING — bootstrap required"`
- Leaderboard tail: !`tail -15 experiments/README.md 2>/dev/null || echo "MISSING — bootstrap required"`
- Branch: !`git branch --show-current`
- Working tree: !`git status --short`
- HEAD: !`git rev-parse --short HEAD 2>/dev/null`

---

## HARD RULES (non-negotiable)

- **NEVER `git commit`, `git push`, or submit to any leaderboard / Kaggle without explicit user approval in chat.**
- **NEVER modify CV splits, seeds, or evaluation code mid-series.** If a change is needed, stop, propose, and require approval — comparability is sacred.
- **One hypothesis per cycle. One branch per cycle.** Branch: `exp/NNN-<short-slug>`.
- **No tuning on the public leaderboard.** Trust local CV.
- **Stop and ask** when:
  - A change touches data leakage, target encoding, CV strategy, target transform reversibility, or submission logic
  - Submission budget hit the warn threshold from `program.md` (default 50%) or is exhausted
  - Prior experiment artifacts would be deleted or overwritten
  - A refactor is required to test the hypothesis (see *Refactor escape valve*)
  - The dataset version, package versions, or feature-store schema has changed since the last cycle (breaks comparability)

---

## BOOTSTRAP (first run only)

If `program.md` is missing **or** `experiments/` lacks `README.md` and `TEMPLATE.md`, do this before cycle 1:

1. Detect from the repo: metric, target column, data files, CV strategy if any, source dir.
2. Generate drafts of the files below.
3. Show the user the generated `program.md` and ask via AskUserQuestion: *"Validate / edit / abort?"*
4. Do not run cycle 1 until validated.

```
program.md                   # human-editable config (template at end of this file)
experiments/
  README.md                  # leaderboard table — append-only
  TEMPLATE.md                # hypothesis → result template
  001_baseline.md            # captures current state as cycle 001
src/                         # if missing and project lacks one
```

Ensure `.gitignore` excludes large per-experiment artifacts (model binaries, predictions) but keeps the markdown.

---

## THE LOOP

Read `program.md` at the **start of every cycle** — it is the source of truth for metric, CV, budgets, and frozen files.

### Phase 1 — REVIEW

Read in this order, stop early if you have what you need:
1. `program.md` (config may have changed since last cycle)
2. `experiments/README.md` (leaderboard)
3. The **current best** experiment file in full
4. The **most recent experiment** (last cycle)
5. Any prior experiment whose hypothesis is closest to the candidates you're considering

Then write a 3-bullet diagnosis of the current best's biggest weakness, using this rubric:

| Signal | Diagnosis | Suggested focus |
|---|---|---|
| `fold_std / |fold_mean| > 0.10` | CV instability | `cv` or feature-stability work |
| `train_val_gap > 0.5 × val_baseline` (metric-aware) | Overfit | regularization, fewer features, more data |
| Last 3 cycles' deltas all `< 0.5 × CV std` | Plateau | structurally different model family |
| Errors concentrated in an identifiable slice | Coverage gap | targeted FE on that slice |
| One fold consistently worst | Hidden leakage / shift | `cv` revalidation |
| None clear | Unknown | `eda` cycle |

If `focus` arg is set, override the rubric's pick.

### Phase 2 — HYPOTHESIZE

Before touching code, write `experiments/NNN_<slug>.md` from `TEMPLATE.md` containing:

- **Hypothesis** — one sentence, falsifiable, quantitative target ("≥1% CV MAE reduction vs cycle NNN").
- **Rationale** — 2–3 bullets grounded in the Phase 1 diagnosis.
- **Expected magnitude** — rough estimate.
- **Overfitting risk** — low / medium / high + why.
- **Kill criteria** — concrete result that would mean abandoning this *direction* (not just this cycle).
- **Scope** — files to be touched, rough line count. Anything beyond this scope = a new cycle.
- **Reversibility** — does this change CV / seed / target transform / leakage surface? If yes → STOP and ask.

### Phase 3 — IMPLEMENT

- `git checkout -b exp/NNN-<slug>` from current `main`.
- Make the **smallest change** that tests the hypothesis. No drive-by refactors. No "while I'm here" tweaks.
- Reuse the same seed and CV folds as the prior cycle.
- **Refactor escape valve**: if the hypothesis genuinely cannot be tested without restructuring, abort the cycle, surface the refactor as a separate proposal, get approval, do it on its own labeled branch, then restart the cycle.

### Phase 4 — VALIDATE

Run the project's evaluation. **Capture and report all of:**

- **Per-fold scores** (full table — never just the mean)
- **CV mean ± std** alongside baseline mean ± std
- **Train vs val gap** (new vs old, absolute and relative)
- **Reproducibility stamp**: git SHA, data file hash (sha256 of train file or feature manifest), Python/key-package versions
- Whether each pre-registered kill criterion fired
- Where applicable: feature importance delta and residual-distribution diff vs prior best

#### Auto-verdict gates

A cycle is `keep` **only if all hold**:

1. **Magnitude**: mean CV improved by `≥ max(0.5 × baseline_cv_std, min_delta)` where `min_delta` comes from `program.md`.
2. **Direction agreement**: `≥ ⌈0.6 × n_folds⌉` folds improved (e.g. 3/5, 4/7).
3. **Stability**: worst fold did not regress by more than `1 × baseline_cv_std`.
4. **Generalization**: train-val gap did not increase by more than the threshold in `program.md` (default: max of +20% relative OR +0.5 × baseline gap absolute).
5. **Kill criteria**: none fired.

Otherwise verdict is `discard` (clearly worse / kill criterion fired) or `inconclusive` (within noise but no red flags).

#### Failure modes (do NOT pollute leaderboard)

- Non-finite metric, crash, OOM, or evaluation didn't complete → mark experiment `infra-fail`, write the file with the failure, do **not** add a leaderboard row, return to Phase 1.
- Detected mid-run that CV/seed/data version changed silently → `infra-fail` + flag to user.

#### Inconclusive follow-up

If `inconclusive`: write a one-line follow-up plan in the experiment file — typically "re-run with seed=99 and seed=7, treat as keep iff median verdict is keep across all 3."

#### Robustness check (periodic)

Every 10 cycles, OR before any leaderboard submission, OR when the user requests it: re-validate the **current best** on two additional seeds. If mean across the 3 seeds drops below the prior best by more than `min_delta`, flag seed-overfit and propose a rollback.

### Phase 5 — DOCUMENT & REPORT

Update:
- `experiments/NNN_<slug>.md` — fill in **Result**, **Verdict**, **Kill-criteria check**, **Reproducibility stamp**, **Learnings**, **Follow-ups**.
- `experiments/README.md` — append one row only if verdict is not `infra-fail`.
- If `program.md` tracks `submissions_used` and a submission happened (with approval), increment it.

Then report to the user **in chat** in exactly this shape:

```
## Cycle NNN — <slug>

Hypothesis: <one line>
Result:     CV <mean> ± <std>  (baseline: <mean> ± <std>)  — folds: [f1, f2, f3, f4, f5]
Gap:        train-val <new> (was <old>, Δ <relative>%)
Verdict:    keep | discard | inconclusive | infra-fail
Gates:      magnitude ✅/❌  agreement ✅/❌  stability ✅/❌  generalization ✅/❌  kill-criteria ✅/❌
Repro:      git=<sha>  data=<hash[:8]>  pkg=<key versions>
Diff:       <files changed>, <+net lines>

Recommendation: merge to main | keep branch for follow-up | discard branch | rollback current best

Working tree is on branch exp/NNN-<slug>, clean.
Awaiting approval to: <commit | merge | submit | discard | none>.
Cycles remaining in budget: <N-1>.
```

**Then STOP.** Do not start the next cycle until the user responds, even if the budget allows more. If the user approves the verdict, proceed. If they request changes, treat as new instructions.

---

## ROTATION (when focus=auto)

Pick by Phase 1 diagnosis. When ambiguous, rotate roughly in this priority order, weighted by current weakness:

1. **CV strategy validation** — first cycle, and any time leakage / shift is suspected
2. **EDA / error analysis** — slice errors, find systematic mistakes
3. **Feature engineering** — one *family* per cycle (lags / rolling / interactions / encodings)
4. **Data cleaning** — outliers, missingness, target sanity
5. **Target transformation** — log / rank / clip / residual
6. **Model diversity** — different family before going deeper on one
7. **Hyperparameter tuning** — only after architecture and features are stable; small Optuna budget first
8. **Ensembling / stacking** — only with diverse, individually strong bases
9. **Calibration / post-processing** — last-mile

Skip any stage that doesn't apply per `program.md`.

---

## ANTI-PATTERNS

- ❌ Tuning hyperparameters before features and CV are stable
- ❌ Chasing public LB while local CV regresses
- ❌ Random KFold on time-series or grouped data — use the strategy in `program.md`
- ❌ Adding many feature families in one cycle (can't attribute the delta)
- ❌ Touching the seed or fold definition mid-series without approval
- ❌ Refactoring during an experiment cycle
- ❌ Comparing cycles across different data versions or package versions (always log the repro stamp; mismatch → infra-fail)
- ❌ Stacking experiments without merging — keep `main` as the current best
- ❌ Reporting only the mean (always per-fold table)

---

## `program.md` TEMPLATE

```markdown
# program.md — project config for /autoresearch

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
- n_splits: 5
- Random seed: 42
- Robustness seeds (for periodic re-validation): [7, 99]

## Thresholds (auto-verdict gates)
- min_delta: <smallest improvement in metric units that counts as signal>
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
- Frozen files (do not touch without approval): [<...>]
- Evaluation entry point: <e.g. `make cv` or `python -m src.cv`>

## Open questions / hunches  (free text — human edits this)
- ...
```

---

## INVOCATION

```
/autoresearch              # 1 cycle, focus auto, stops for approval
/autoresearch 3            # budget of 3, still stops each cycle for approval
/autoresearch 1 features   # 1 cycle focused on feature engineering
```

Begin by: (1) reading `program.md`, (2) running the bootstrap check if anything is missing, (3) entering Phase 1 of cycle 1.
