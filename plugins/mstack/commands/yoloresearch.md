---
description: Fully autonomous ML research loop. Runs cycles, commits, pushes, opens/updates PR, and submits to Kaggle within budget — all without asking. Refuses only what would corrupt scientific integrity. Use only when you're OK with the loop driving end-to-end.
argument-hint: "[focus=auto] [cycles=N] — e.g. features, tuning cycles=10"
allowed-tools: Bash(git:*), Bash(gh:*), Bash(python:*), Bash(uv:*), Bash(make:*), Bash(kaggle:*), Bash(ls:*), Bash(cat:*), Bash(tail:*), Bash(head:*), Bash(find:*), Bash(grep:*), Bash(sha256sum:*), Bash(md5:*), Bash(pip:*), Read, Write, Edit, Grep, Glob, AskUserQuestion, TodoWrite
---

# /yoloresearch

Fully autonomous variant of `/autoresearch`. Same scientific spine (hypothesis → gates → verdict → repro stamp), no human-in-the-loop friction. The loop owns:

- Editing, running, evaluating, documenting
- Committing to the session branch
- **Pushing** to remote after every commit
- **Opening or updating a PR** to staging after the first keep, on every subsequent keep
- **Submitting** to Kaggle (or any leaderboard wired in `project.md`) when a keep beats the current public LB, **within budget**

The user is not consulted per cycle. Use this when you trust the gates and want a long autonomous run.

`$ARGUMENTS` — all optional. `focus` ∈ {`features`, `tuning`, `cv`, `model`, `ensemble`, `eda`, `auto`} (default `auto`). `cycles=N` caps the run; otherwise it runs until interrupted.

## Context

- project.md: !`cat project.md 2>/dev/null | head -150 || echo "MISSING — bootstrap required"`
- Leaderboard tail: !`tail -15 experiments/README.md 2>/dev/null || echo "MISSING — bootstrap required"`
- Branch: !`git branch --show-current`
- Working tree: !`git status --short`
- HEAD: !`git rev-parse --short HEAD 2>/dev/null`
- Remote: !`git remote get-url origin 2>/dev/null || echo "no remote"`

---

## INTEGRITY RULES (non-negotiable, even in yolo)

These aren't asking permission — they protect the loop's outputs from being garbage. If any trigger fires, the loop **aborts the cycle, reverts code, logs the reason, and picks a different hypothesis next cycle.**

- **NEVER modify CV splits, seeds, evaluation code, or target transform mid-session.** A change here invalidates every prior cycle's comparison. If the hypothesis requires it → cycle is auto-abandoned and the loop notes "needs new session with different CV".
- **NEVER push to or merge into a protected branch** (`main`, `master`, `staging`, plus anything listed in `project.md`). The session branch is the only branch the loop writes to. PRs are opened *targeting* staging; merge is still the human's click.
- **NEVER overwrite prior experiment files.** `NNN_<slug>.md` is append-only as a set.
- **NEVER touch frozen files** listed in `project.md`.
- **NEVER submit past the budget ceiling.** Warn threshold = informational. Ceiling = hard stop on submissions for the rest of the session.
- **NEVER continue if dataset version, package versions, or feature-store schema changed silently** since the last cycle. → `infra-fail`, session pauses, single chat message to user.
- **NEVER tune on the public LB.** Trust local CV. Public LB is read-only signal except for the final submission decision.
- **One hypothesis per cycle.** No drive-by refactors.

If a hypothesis genuinely requires touching frozen files or CV/seed/target, the loop **skips that hypothesis** and picks a different one next cycle — it does not stop to ask.

---

## SESSION SETUP (once per invocation)

1. If current branch is protected → **stop and ask once** ("Create `yolo/<YYYY-MM-DD>-<slug>` from here, or switch first?"). This is the only synchronous prompt yolo will issue.
2. If already on a `yolo/*` or `autoresearch/*` branch → resume on it.
3. Otherwise use the current branch.
4. If `git remote` is unset → warn in chat, downgrade to "commit + open PR locally is impossible, no submission either". Continue with local-only commits.
5. Record session start: branch, starting SHA, timestamp. Used in every repro stamp.

---

## BOOTSTRAP (first run only)

If `project.md` is missing or `experiments/` lacks `README.md`/`TEMPLATE.md`:

1. Detect from the repo: metric, target column, data files, CV strategy, source dir.
2. Generate drafts of `project.md`, `experiments/README.md`, `experiments/TEMPLATE.md`, `experiments/001_baseline.md`.
3. **One synchronous ask**: show the generated `project.md` and ask "Validate / edit / abort?". This is the only other allowed prompt — it's a setup step, not a per-cycle decision.
4. Do not run cycle 1 until validated.

Ensure `.gitignore` excludes large per-experiment artifacts but keeps the markdown.

---

## THE LOOP

Read `project.md` at the start of every cycle. After each cycle: commit, push, maybe open/update PR, maybe submit, continue. Stops only on interrupt, `cycles=N` cap, integrity-fail, or budget exhaustion.

Session-branch tip always reflects the **current best**. Non-keep cycles revert their code (Phase 5).

### Phase 1 — REVIEW

Same as `/autoresearch`. Read in order: `project.md`, leaderboard, current best, last cycle, nearest prior hypothesis. Write a 3-bullet diagnosis. Pick the dominant signal:

- High fold variance (`fold_std / |fold_mean| > 0.10`) → **CV instability**
- Train-val gap > 0.5 × val baseline → **overfit**
- Last 3 deltas all `< 0.5 × CV std` → **plateau** — try a structurally different model
- Errors concentrated in a slice → **coverage gap**
- One fold consistently worst → **hidden leakage / shift**
- Nothing clear → `eda` cycle

`focus` arg overrides.

### Phase 2 — HYPOTHESIZE

Stamp `experiments/NNN_<slug>.md` with:
- **Hypothesis** — falsifiable, quantitative ("≥1% CV MAE vs cycle NNN")
- **Rationale** — 2–3 bullets from Phase 1
- **Expected magnitude**
- **Overfitting risk** — low / med / high + why
- **Kill criteria** — abandon-this-direction trigger
- **Scope** — files, rough line count
- **Reversibility** — if it touches CV / seed / target transform / frozen files → **auto-abandon, pick next hypothesis**

### Phase 3 — IMPLEMENT

Smallest change that tests the hypothesis. Same seed, same folds. If implementation reveals the change crosses an integrity rule mid-edit → revert, mark cycle aborted, pick next hypothesis next cycle.

### Phase 4 — VALIDATE

Run evaluation. Capture:
- Per-fold scores (full table)
- CV mean ± std vs baseline mean ± std
- Train vs val gap (new vs old, abs and rel)
- Repro stamp: git SHA, data hash, Python/key-package versions
- Kill criteria firings
- Feature importance delta + residual diff vs prior best where applicable

#### Auto-verdict gates

`keep` only if all hold:
1. **Magnitude**: mean CV improved by `≥ max(0.5 × baseline_cv_std, min_delta)`.
2. **Direction agreement**: `≥ ⌈0.6 × n_folds⌉` folds improved.
3. **Stability**: worst fold did not regress by more than `1 × baseline_cv_std`.
4. **Generalization**: train-val gap did not grow beyond the `project.md` threshold.
5. **Kill criteria**: none fired.

Otherwise: `discard` (clearly worse / kill fired) or `inconclusive` (within noise).

#### Failure modes

- Non-finite metric, crash, OOM, eval didn't complete → `infra-fail`. Record, do not add leaderboard row, return to Phase 1.
- Mid-run detected CV/seed/data drift → `infra-fail`, **single chat ping**, session pauses for user.

#### Inconclusive auto-follow-up

If `inconclusive`: yolo **automatically** re-runs with `robustness_seeds` from `project.md` (default `[7, 99]`). Verdict becomes:
- `keep` if median across {primary, 7, 99} would have been keep
- `discard` otherwise

No human required. Record all three runs in the experiment file.

#### Robustness check (periodic)

Every `robustness_check_every` cycles, before any submission, and after every keep that beats public LB by more than `1 × baseline_cv_std`: re-validate the current best on the two robustness seeds. If 3-seed mean drops below the prior best by more than `min_delta` → flag seed-overfit and **roll back** the current best (revert the keep commit, mark the experiment `seed-overfit`).

### Phase 5 — DOCUMENT, COMMIT, PUSH, MAYBE-PR, MAYBE-SUBMIT, CONTINUE

1. Finalize `experiments/NNN_<slug>.md` — Result, Verdict, Kill-criteria check, Repro stamp, Learnings, Follow-ups.
2. Append one row to `experiments/README.md` *unless* `infra-fail`.
3. If `keep`, update `project.md`'s **Current best** section.
4. **Commit** (by verdict):
   - `keep` → stage code + experiment + leaderboard + `project.md` update. One commit. Message: `exp NNN: <slug> — keep (CV <mean>±<std>)`.
   - `discard` / `inconclusive` → revert code, commit only experiment + leaderboard row. Message: `exp NNN: <slug> — <verdict>`.
   - `infra-fail` → revert code, commit only experiment file. Message: `exp NNN: <slug> — infra-fail`.
5. **Push** the session branch to `origin` (`git push -u origin <branch>` first cycle, `git push` after).
6. **PR maintenance** (only on `keep`):
   - First keep of session: `gh pr create --base staging --head <branch>` with title `yoloresearch: <session-slug> (N keeps)` and body summarising the session (cycles run, keeps, current CV vs starting CV, link to leaderboard tail).
   - Subsequent keeps: `gh pr edit <pr#> --body <regenerated body>` — keep the body fresh; don't open new PRs.
7. **Submission** (only on `keep` that beats current public LB if known, AND submissions remaining > 0):
   - Submit via the entry point declared in `project.md` (e.g. `kaggle competitions submit ...`).
   - Increment `submissions_used` in `project.md`, commit the increment as a separate commit.
   - If new submission_used / total ≥ warn threshold (default 50%): log a one-line warning in the report. If at ceiling: stop submitting for the rest of the session.
8. Report to chat in exactly this shape:

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
Pushed: <sha>  PR: <url or "no keeps yet">  Submissions: <used>/<total>
Continuing to cycle NNN+1 — interrupt to stop.
```

9. Begin cycle NNN+1.

The loop **only stops when**:
- The user interrupts,
- `cycles=N` cap is reached,
- An integrity rule fires that *requires* user input (only: mid-run data/package drift detected, or no remote configured and user asked for push),
- All resources exhausted (submissions at ceiling AND no further cycles can improve, or wall-clock budget reached).

It does **not** stop for verdicts, gate failures, normal commits, PR updates, or successful submissions.

---

## ROTATION (when focus=auto)

Same as `/autoresearch`. Pick by Phase 1 diagnosis; when ambiguous prioritise:

1. CV validation 2. EDA / error analysis 3. Feature engineering (one family/cycle)
4. Data cleaning 5. Target transformation 6. Model diversity
7. Hyperparameter tuning (only after architecture & features stable) 8. Ensembling
9. Calibration / post-processing

Skip any stage that doesn't apply per `project.md`.

---

## `project.md` ADDITIONS FOR YOLO MODE

`/yoloresearch` reads the same `project.md` as `/autoresearch` plus these optional keys:

```markdown
## Yolo
- PR target: staging              # branch to open PRs against
- Auto-submit: true               # submit to leaderboard on keeps that beat public LB
- Submission entry point: <e.g. `make submit` or `kaggle competitions submit ...`>
- Submission ceiling behavior: stop  # stop | warn-only
- Robustness seeds: [7, 99]       # used for inconclusive auto-follow-up and seed-overfit checks
```

If the `Yolo` section is missing, yolo defaults to: PR target = `staging`, auto-submit = false (commits and pushes only), ceiling behavior = stop.

---

## DIVERGENCE FROM `/autoresearch`

| Behavior | `/autoresearch` | `/yoloresearch` |
|---|---|---|
| Per-cycle approval | required to continue | none |
| Commits to session branch | yes (auto) | yes (auto) |
| Push to remote | requires approval | auto every commit |
| Open / update PR | manual via `/ship` | auto on keeps |
| Submit to leaderboard | requires approval | auto on LB-beating keeps within budget |
| Inconclusive verdict | logs follow-up, user decides | auto re-runs with robustness seeds, decides |
| Hypothesis touches CV/seed/frozen | stops and asks | auto-abandons, picks next |
| Refactor needed | escape valve, stops and asks | auto-abandons cycle (no in-loop refactors) |
| Stop trigger | interrupt OR per-cycle prompt | interrupt OR integrity-fail OR cap |

If you want the safety net back, use `/autoresearch`. If you want to walk away, use this.

---

## INVOCATION

```
/yoloresearch                  # focus auto, run until interrupted
/yoloresearch features         # focus features
/yoloresearch cycles=20        # cap at 20 cycles
/yoloresearch tuning cycles=10
```

Begin by: (1) session setup, (2) bootstrap if needed, (3) read `project.md`, (4) Phase 1 of cycle 1. From there: no synchronous prompts unless an integrity rule fires.
