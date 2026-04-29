---
description: Commit (as multiple logical commits), push to a feature branch, and open or update a PR to staging
allowed-tools: Bash(git:*), Bash(gh:*), Read, Grep, Glob, AskUserQuestion
---

# Ship: Commit, Push & Create/Update PR

Commit all current changes as **multiple small, reviewer-friendly commits**, push them to a dedicated branch, and open (or add to) a pull request targeting `staging`.

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
- Commit messages have **no trailers** (no `Co-Authored-By`, no "Generated with…").

## PR Title Convention

Title must match the format enforced by the `pull-request-title-linter` GitHub Action:

    <type>(<scope>): <description>

**Do NOT hardcode types/scopes.** Read them dynamically:

1. Glob `.github/workflows/**/*{semantic,title-linter}*` to locate the workflow file.
2. Read it and extract `types:` and `scopes:` from the `with:` block of the `amannn/action-semantic-pull-request` step.
3. Use those as the allowed values. If scope is ambiguous, ask via AskUserQuestion.

## Steps

### 1. Sanity check
If nothing is staged, unstaged, or untracked: tell the user there's nothing to commit and stop.

### 2. Plan the commits
Analyze the full diff (staged + unstaged + untracked). Group changes into the smallest commits that each represent **one coherent intent** (feature / fix / refactor / test / config / docs / rename+callsites):

- Combine files only when they're inseparable.
- If a single file mixes unrelated hunks, try `git add -p`. Fall back to a single-file commit if patch mode is awkward.
- Do NOT split artificially — one intent = one commit, even across many files.

Identify the **dominant commit type** (most significant change) — used for branch naming and PR title. Precedence when mixed: `feat > fix > perf > refactor > test > docs > ci > chore`.

If intents are genuinely unrelated, flag it — the user may want to ship as separate PRs.

### 3. Resolve the target branch

- If `$ARGUMENTS` is provided and non-empty: use it. If it lacks a known prefix (`feat-`, `fix-`, `chore-`, `refactor-`, `docs-`, `hotfix-`, `release-`, `test-`, `ci-`, `perf-`), prefix it with `<dominant-type>-`.
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

3. **If no PR exists**: `gh pr create --base "$BASE"` with:
   - **Title**: `<type>(<scope>): <description>` using the dominant type and an extracted scope. Double-check both against the workflow-extracted lists.
   - **Body** (HEREDOC):

         ## Summary
         <1-3 bullets on what changed and why>

         ## Test plan
         <bulleted checklist for verifying the changes>

   - Capture the PR URL and number (last numeric segment of the URL).

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
Summarize: branch name, the list of commits created (oneline), whether the PR was **created** or **updated**, and the PR URL.
