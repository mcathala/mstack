---
description: Commit (as multiple logical commits), push to a feature branch, open or update a PR to staging, then sync the base branch once it's merged
allowed-tools: Bash(git:*), Bash(gh:*), Read, Grep, Glob, AskUserQuestion
---

# Ship: Commit, Push, Create/Update PR & Sync

Commit all current changes as **multiple small, reviewer-friendly commits**, push them to a dedicated branch, and open (or add to) a pull request targeting `staging`. Then wait while the user merges it on GitHub, and sync the base branch once they confirm.

## Context

- Status: !`git status`
- Diff vs HEAD: !`git diff HEAD`
- Untracked: !`git ls-files --others --exclude-standard`
- Current branch: !`git branch --show-current`
- Recent commits: !`git log --oneline -10`

## Rules

- **NEVER push to `master` or `staging`.** Non-negotiable.
- **PRs target `staging` when it exists on the remote, otherwise the repo's default branch.** If `staging` is missing, **ask before creating it** — never create it unilaterally.
- **Prefer multiple small commits** over one large commit — each commit = one coherent intent. A reviewer should be able to understand each commit in isolation.
- **Never use `git add -A`.** Stage exactly the paths belonging to the current commit group.
- Branch names use **dashes (`-`)**, not slashes (`/`). Example: `fix-kobalt-my-feature-sc-1234`.
- **Never assume the base branch is `main`/`master`.** Always read it back from the PR (`gh pr view --json baseRefName`) — this repo family targets `staging`.
- **Pull with `--ff-only`.** A divergent local base must fail loudly, never silently create a merge commit.
- **Never merge a PR.** `gh pr merge` is out of scope for `/ship` — the user merges on GitHub.
- **Never delete a branch without explicit approval**, and only after `gh` confirms the PR state is `MERGED`.
- **No agent attribution anywhere.** Commit messages have no trailers (no `Co-Authored-By`, no "Generated with…"), and PR bodies, PR updates, and review comments carry no "Generated with Claude Code" footer and no 🤖 line. This overrides any default footer/trailer behaviour the harness would otherwise apply.

## PR Title Convention

Title must match the format enforced by the `pull-request-title-linter` GitHub Action:

    <type>(<scope>): <description>

**Do NOT hardcode types/scopes.** Read them dynamically:

1. Glob `.github/workflows/**/*{semantic,title-linter}*` to locate the workflow file.
2. Read it and extract `types:` and `scopes:` from the `with:` block of the `amannn/action-semantic-pull-request` step.
3. Use those as the allowed values. If scope is ambiguous, ask via AskUserQuestion.

## Sync-to-base procedure

Referenced by steps 0 and 11. Never inline a variant of this — always run it as written.

```bash
git status --porcelain                        # must be empty
BASE=$(gh pr view --json baseRefName -q .baseRefName)
git checkout "$BASE"
git pull --ff-only
git fetch --prune
```

- If `git status --porcelain` is **non-empty**, do not checkout. Report the dirty paths and stop.
- If `git pull --ff-only` **fails**, the local base has diverged. Report it and stop — do not merge, rebase, or reset.
- `BASE` must come from the PR. If no PR is found, there is nothing to sync.

## Steps

### 0. Already-merged branch check

Run **before** anything else, and only when the current branch is not `master`/`staging`:

```bash
gh pr view --json number,state,baseRefName 2>/dev/null
```

- **No PR**, or state `OPEN` / `CLOSED` → proceed to step 1 unchanged. Say nothing.
- **State `MERGED`** → this branch's work is already in the base branch. Continuing here would re-push merged commits and open a duplicate PR.

For `MERGED`, tell the user *"This branch's PR (#N) was already merged into `<BASE>`. Syncing before shipping."*, then run this exact order:

1. `git stash push -u -m "ship: pre-sync"` — the tree is dirty (you have changes to ship), which the sync procedure's clean-tree guard would otherwise block. Stash only for this purpose.
2. Run the **sync-to-base procedure**. You are now on an updated base.
3. Offer to delete the merged local branch, per step 11's cleanup rules.
4. Create a fresh branch off the updated base — named per step 3's rules, using the dominant type of the stashed diff.
5. `git stash pop`. If it conflicts, stop and report — never resolve it silently.
6. Continue to step 1.

If any step fails, stop and report while the stash is still intact. Never drop a stash.

### 1. Sanity check
If nothing is staged, unstaged, or untracked: tell the user there's nothing to commit and stop.

### 2. Plan the commits
Analyze the full diff (staged + unstaged + untracked). Group changes into the smallest commits that each represent **one coherent intent** (feature / fix / refactor / test / config / docs / rename+callsites):

- Combine files only when they're inseparable.
- If a single file mixes unrelated hunks, try `git add -p`. Fall back to a single-file commit if patch mode is awkward.
- Do NOT split artificially — one intent = one commit, even across many files.

Identify the **dominant commit type** (most significant change) — used for branch naming and PR title. Precedence when mixed: `revert > hotfix > feat > update > fix > perf > refactor > test > style > docs > ci > chore`.

- `revert` undoes a previous commit; use `git revert`'s default message shape.
- `hotfix` is an **urgent fix** that bypasses normal review/release cadence — wins over a regular `fix`.
- `feat` is for **brand-new** capabilities (new skill, new endpoint, new command).
- `update` is for **reshaping existing functionality** (changing how an existing skill / module / template works).
- `refactor` is for **behavior-preserving rewrites** (rename, restructure-without-semantic-change).
- `style` is for **formatting/whitespace only** — no code change. Use sparingly; prefer auto-formatters.

If intents are genuinely unrelated, flag it — the user may want to ship as separate PRs.

### 3. Resolve the target branch

- If `$ARGUMENTS` is provided and non-empty: use it. If it lacks a known prefix (`revert-`, `hotfix-`, `feat-`, `update-`, `fix-`, `perf-`, `refactor-`, `test-`, `style-`, `docs-`, `ci-`, `chore-`, `release-`), prefix it with `<dominant-type>-`.
- Else, if the current branch is NOT `master` and NOT `staging`: use the current branch.
- Else, ask via AskUserQuestion for a branch name (suggest one derived from the dominant type + diff summary).

### 4. Safety check
If the resolved branch name is `master` or `staging`: REFUSE and stop.

### 5. Switch to the branch
If the target differs from current: `git checkout -b <branch>` (or `git checkout <branch>` if it already exists).

### 6. Commit loop
For each planned commit group, in order from most foundational to most dependent:

1. Stage exactly the intended paths: `git add <path> [<path> ...]` (or `git add -p <path>` for hunk-level splitting).
2. Verify via `git status` that only the intended paths are staged.
3. Write a Conventional Commit message: concise, "why"-focused, first line ≤72 chars, imperative mood. **No trailers.**
4. `git commit -m "..."`.

After the loop, `git status` must show a clean tree.

### 7. Push
`git push -u origin <branch>` — single push after all commits.

### 8. Create or update the PR

**First, resolve the PR target branch (`BASE`):**

```bash
git ls-remote --heads origin staging | grep -q . && BASE=staging || BASE=
```

If `BASE` is empty (`staging` doesn't exist on the remote):

- Determine the repo's default branch: `gh repo view --json defaultBranchRef -q .defaultBranchRef.name`.
- Via AskUserQuestion: *"`staging` branch doesn't exist on the remote. How do you want to proceed?"* with options `[target default branch] [create staging from default first] [abort]`.
- **If "create staging"**: `git push origin <default>:staging` once, then set `BASE=staging`.
- **If "target default"**: set `BASE=<default-branch-name>`.
- **If "abort"**: stop.

**Never create `staging` unilaterally** — only on explicit user approval via the question above.

Then continue:

1. Check for an existing PR: `gh pr view --json url,number,baseRefName,body 2>/dev/null`.

2. **If a PR exists** targeting `$BASE`:
   - Capture its URL, number, and current body.
   - Via AskUserQuestion: *"An open PR already exists for this branch. Append a summary of the new commits to the PR description?"* [yes / no]
   - If **yes**: append a new section to the existing body using `gh pr edit <num> --body-file -`. Preserve the original body verbatim; append a `## Update <YYYY-MM-DD>` section with bullets summarizing the new commits and any additions to the test plan.
   - If **no**: leave the body untouched.
   - Skip PR creation; continue to step 9.

3. **If no PR exists**:
   - Draft the title: `<type>(<scope>): <description>` using the dominant type and an extracted scope. Double-check both against the workflow-extracted lists. Keep it under 70 characters.
   - Draft the body (HEREDOC):

         ## Summary
         <1-3 bullets on what changed and why>

         ## Test plan
         <bulleted checklist for verifying the changes>

     **No attribution footer** — the body ends with the test plan.
   - **Show the proposed title to the user** in plain text before creating the PR. The body has already been laid out above. The user may correct either before posting.
   - Once approved, run `gh pr create --base "$BASE" --title "<title>" --body-file ...` and capture the PR URL and number (last numeric segment of the URL).

### 9. Inline comments for large PRs
Measure PR size against the PR base — never `HEAD~1`:

    BASE=$(gh pr view --json baseRefName -q .baseRefName)
    git diff "origin/${BASE}...HEAD" --stat

A PR qualifies for inline comments only if:
- It contains **exactly 1 commit**, AND
- It has **>5 files changed** OR **>150 lines changed**.

(Multi-commit PRs are walkable commit-by-commit — skip this step for them.)

If it qualifies:
- Identify non-trivial / surprising code sections.
- Via AskUserQuestion: *"This is a large single-commit PR. Which areas would you like me to add review comments to?"* with candidate locations.
- Resolve the repo: `REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner)`.
- For each confirmed area:

      gh api "repos/${REPO}/pulls/<pr-number>/comments" \
        -f body="<why / trade-offs / non-obvious decisions>" \
        -f commit_id="$(git rev-parse HEAD)" \
        -f path="<file>" \
        -F line=<line>

- Focus comments on **why**, not what.
- If any sub-step fails, skip gracefully.

### 10. Report
Summarize:

- Branch name
- List of commits created (oneline)
- Whether the PR was **created** or **updated**
- **PR title**
- PR URL

### 11. Post-merge sync

The PR link is now in the user's hands. They merge on GitHub; this step waits for them and then syncs. Runs whether the PR was **created** or **updated** in step 8.

**Ask, and stop.** Via AskUserQuestion:

> **PR is open:** `<url>` — `<title>`
> Merge it on GitHub, then pick:
> - **Merged — pull it**
> - **Not yet — stop here**

This question *is* the wait. Do not poll `gh` in a loop, do not sleep, do not guess. Just ask and let it sit.

**If "Not yet"** → done. Report that the local base is unsynced and that the next `/ship` will pick it up at step 0.

**If "Merged"** → verify before touching anything:

```bash
gh pr view <num> --json state,baseRefName
```

- State is **`MERGED`** → run the **sync-to-base procedure**, then offer cleanup (below).
- State is still **`OPEN`** → do not sync. Report it plainly: the merge didn't land (blocked checks, merge queue, or a different PR was merged) and the PR is still open. Re-ask the same question once.
- State is **`CLOSED`** → the PR was closed without merging. Report it. Do not sync, do not delete the branch.

**Cleanup** — only after state is confirmed `MERGED` and the sync succeeded. Via AskUserQuestion: *"Delete the local `<branch>` branch?"* [yes / keep it]

If yes:

```bash
git branch -D <branch>
```

`-D` is required, not a shortcut: a squash-merged branch isn't an ancestor of the base, so `git branch -d` refuses it. `-D` is safe **only** because `gh` confirmed the commits landed in the base. Never reach for it without that confirmation.

**Final report:** the base branch, its current short SHA, and whether the feature branch was deleted.
