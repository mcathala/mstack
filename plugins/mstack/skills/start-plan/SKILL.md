---
name: start-plan
description: Implement the full PRD from /to-action by walking through every unblocked task issue in dependency order. For each task, walk its sub-tasks one at a time, applying TDD where the work is deterministic and ticking the checkbox in the issue body when each sub-task is done. Tests deterministic code only; defers metric checks to /validate. Use when user says "implement", "start coding", "build the PRD", "let's TDD this", or after /to-action hands off issues.
---

# start-plan

Take a PRD (parent issue from /to-action) and implement **all** its task issues end-to-end. For each task, walk its sub-tasks one at a time, ticking each checkbox as it completes. Stop only when everything is green; hand off to /validate.

## When to skip

- **The PRD is empirical, not deterministic.** "Improve AUC by 2 points" cannot be implemented via TDD — surface this; it belongs as an experiment, not a coding pass.
- **The PRD is honestly one tiny change.** Don't dress up a one-line tweak with a multi-phase ceremony — implement and stop.

## Phase 0 — Read the scope

If invoked without an argument, find the most recent unfinished PRD (`gh issue list --label prd --state open`) and confirm with the user.

**Fetch the children** — `/to-action` links them to the parent in three possible ways. Try each in order; take the first that returns a non-empty list:

1. **Native sub-issues API** (canonical when the repo has sub-issues enabled):
   `gh api repos/{owner}/{repo}/issues/{N}/sub_issues`
2. **Summary comment fallback** — `/to-action` posts a single comment on the parent listing all children. Read parent comments (`gh issue view N --comments`), find the summary, parse `#M` references, then `gh issue view M --json number,title,body,labels,state` for each candidate.
3. **Search fallback** — `gh issue list --search "in:body \"Parent #N\"" --json number,title,body,labels,state`. Verify each result has `## Parent #N` in its body.

If all three return empty, surface to the user — the PRD may not have children yet, or the linking convention is non-standard.

**Compute the order.** Read each task's `## Blocked by` list and `## Priority` (or `priority: high|medium|low` label). Topologically sort: blockers before dependents. Within each tier of unblocked tasks, **high priority first, then medium, then low**.

Show the user a plan with priority annotation:

```
PRD #42 — M unblocked tasks, N total.
  [high] #43  Auth tracer bullet                    [unblocked]
  [med]  #44  GitHub provider                       [blocked by #43]
  [med]  #45  Sign-out + persistence                [blocked by #43]
  [low]  #46  Comments → user_id                    [blocked by #43]

Order: #43 → #44 → #45 → #46
I'll stop on first stuck-red or surprising finding. Confirm?
```

Wait for confirmation before starting.

## Phase 1 — Loop over task issues in dependency order

For each task issue, complete every sub-task before moving to the next task:

### 1a. Read the task

`gh issue view N`. Read body, **Relevant Files**, **Subtasks** list, blockers (verify each blocker is closed/done), and any **Notes**.

### 1b. Identify the testable surface

Two questions, asked explicitly per task when non-obvious:

1. **What is deterministic?** Same input → same output. TDD lives here.
2. **What is empirical?** Output depends on data, model, randomness, latency, the network. Validation lives at /validate, not here.

For ML/data work, tests usually cover plumbing (loading, transforms, splits, metric implementations, submission writers) — *not* model accuracy.

For software work, tests usually cover most of the change.

State the split before starting on the sub-tasks.

### 1c. Walk sub-tasks in order

For each sub-task on the task's checklist:

1. **Read the sub-task description.**
2. **Classify:** does this sub-task require deterministic code (TDD applies) or non-test-coverable work (config, infra, scaffolding, empirical setup)?
3. **Implement:**
   - **Deterministic** — red → green → refactor.
     - **RED:** write one test for the sub-task's behavior. Run it. Watch it fail for the right reason. (Wrong-reason failure = broken test, not a passing TDD step.)
     - **GREEN:** write the minimum code to pass. No speculative features.
     - **REFACTOR:** only when green. Dedupe, deepen modules, tighten interfaces. Re-run tests after each step.
   - **Non-deterministic** — just do the work; verify the result manually or empirically as appropriate.
4. **Tick the checkbox** in the task issue body when the sub-task is complete:
   `gh issue edit N --body-file <updated-body>` — read the current body via `gh issue view N --json body`, replace the matching `- [ ] X.Y` line with `- [x] X.Y`, write it back. Keeps the issue as a live progress indicator.
5. **Move to the next sub-task.**

Rules:

- One test at a time during TDD. Test behavior through the public interface, never implementation details.
- Never refactor while red. Get to green first.
- Don't skip sub-tasks. If one is unclear, surface it before continuing.
- A test that breaks during a pure refactor was testing implementation — fix the test, not the code.

### 1d. Checkpoint and continue

When all sub-tasks for the task are ticked, post a one-line status:
`✓ #N done. Moving to #M.`

Move to the next unblocked task. If a previously-blocked task is now unblocked (its blocker just closed), re-evaluate the queue.

## Stop conditions

Stop the loop and surface to the user — do not push through:

- **Stuck red.** Test refuses to go green after two honest attempts. Invoke `/diagnose`; the bug is now the work.
- **Surprising finding.** Implementation reveals the PRD is wrong, a constraint was missed, or a task depends on something not in scope. Recommend a loop-back depth (step 5 / 3 / 2 / 1) and stop.

Early surfacing is cheaper than five doomed tasks.

## Phase 2 — Hand off

When all tasks of the PRD are done (every sub-task ticked, every task green):

- Post a one-line summary: `All M tasks of PRD #42 implemented. Suggesting /validate.`
- Suggest `/validate` to check the iteration against its metric.
- Do **not** invoke `/ship` or `/validate` directly — recommend, let the user trigger.

## Single-issue mode

If the user invokes `/start-plan #42` with a *task*-issue number (not a PRD parent), implement just that one task — walk its sub-tasks — and stop. Do not auto-walk siblings.

## Do not

- Interview the user. /sync already did. (One targeted question if scope is genuinely ambiguous.)
- Test the empirical parts (model accuracy, latency under load, conversion). That's /validate's job.
- Skip sub-tasks or batch them. One at a time, tick as you go.
- Refactor code outside the current sub-task's scope.
- Push, commit, or deploy. /ship handles that, after /validate.
- Continue past a stuck-red or a surprising finding. Stop and surface.
- Auto-invoke /validate or /ship. Recommend, let the user trigger.
