---
name: to-action
description: Turn an aligned understanding (typically from /sync) into action — a PRD as a parent GitHub issue plus task issues with priority, blocked-by, sub-tasks, and relevant files (kanban-board-shaped). Use when the user says "make this actionable", "turn this into issues", "commit this plan", or follows /sync. Do NOT interview — synthesize what was already discussed.
---

# to-action

Commit a planning conversation into a fanned-out plan on GitHub:

- **Phase A:** PRD as a parent issue
- **Phase B:** Task issues with priority and blocked-by, each containing sub-tasks and a list of relevant files

Each task issue is written so an AI agent or junior developer can pick it up cold and start working.

**Do NOT interview the user.** /sync already did that. If something is genuinely missing, ask a single targeted question — never a questionnaire.

## When to skip

If the work is genuinely one-issue-sized (a bug fix, a small UI tweak, a contained refactor), don't write a PRD. Tell the user, offer to open one issue using the task template (minus Parent), and stop.

## Phase 0 — Confirm understanding

Before drafting anything, summarize back to the user in 5–10 lines:

- The problem
- The proposed solution
- The major modules in scope
- Non-obvious constraints or decisions you picked up from /sync

Ask: "Did I get this right?"

Wait for the user's go-ahead. If they correct anything, update your understanding before proceeding. This is the cheapest place to catch a misread — far cheaper than redrafting a posted PRD.

## Phase A — PRD

### A1. Sketch the modules

Identify the major modules to build or modify. Look for opportunities to extract **deep modules** — simple interfaces hiding substantial functionality, testable in isolation.

Confirm the module list with the user.

### A2. Draft the PRD

Use the template below. The PRD's primary reader is a **junior developer or AI agent** — be explicit, unambiguous, and avoid jargon.

Show the draft. Confirm before posting — `gh issue create` is a shared-state action.

### A3. Post to GitHub

Title: `PRD: <feature-name>`. Ask once whether labels should be applied (e.g. `prd`, `epic`).

Use `gh issue create` with the body via heredoc. Return the parent issue URL.

### PRD template

The template flexes for both user-facing features and research/data-science work. Two sections behave differently:

- **User Stories** is included when there's a user distinct from the developer; omit when the developer *is* the user (research, internal tooling).
- **Success Metrics** absorbs hypothesis / baseline / kill criteria for evidence-driven work.

```markdown
## Introduction / Overview
Briefly describe the feature and the problem it solves. State the goal in one sentence.

## Goals
The specific, measurable objectives for this feature. A short list.

## User Stories  (omit if the developer is the user)
User narratives describing feature usage and benefits.
Format: "As a <actor>, I want <feature>, so that <benefit>."

## Functional Requirements
Numbered list of specific functionalities the feature must have. Clear, concise language.
Example: "1. The system must allow users to upload a profile picture."

## Non-Goals (Out of Scope)
What this feature will NOT include.

## Design Considerations  (optional)
Mockups, UI/UX requirements, relevant components/styles, if applicable.

## Technical Considerations  (optional)
Known technical constraints, dependencies, suggestions.
Example: "Should integrate with the existing Auth module."

## Success Metrics
How success is measured.

- For build work: usage / engagement / quality metrics ("reduce support tickets related to X by 30%").
- For research / evidence-driven work, include explicitly:
  - **Baseline:** the current state we're comparing against
  - **Threshold:** the number that decides PASS vs FAIL
  - **Kill criteria:** when to stop trying (e.g., "after 3 variants without ≥ 0.001 improvement")

## Open Questions
Remaining questions, areas needing further clarification. Leave empty if there are none — not a junk drawer.
```

## Phase B — Tasks

The output is shaped like a kanban board — each task carries a **priority** and a **blocked by** list, so the work surfaces in the right order downstream (`/start-plan` walks high-priority unblocked tasks first).

### B1. Generate the full breakdown (single pass)

Based on the PRD, generate the full task breakdown in one pass — do not pause between parent tasks and sub-tasks.

For each parent task, produce:

- **Title** — short, descriptive
- **Priority** — `high` / `medium` / `low`. High = critical path or unblocks others; low = nice-to-have or can wait.
- **State** — `unblocked` or `blocked by #N`
- **Implements** — which PRD functional requirements (1, 3, ...)
- **Sub-tasks** — actionable items implied by the PRD; one line each
- **Relevant files** — files that will be created or modified, including test files; one line each
- **One-line description**

Use judgment on parent-task count — typically about 5. Each parent should be a cohesive chunk of work, demoable or verifiable on its own when possible. If a task can't stand alone (foundational migration, refactor, dep upgrade), mark it explicitly.

### B2. Confirm the breakdown

Show the full breakdown to the user as a numbered list. Ask **once**:

> "Look right? Anything to change before I post these as issues?"

Iterate if the user pushes back. When approved, proceed to B3. **No mandatory pause between sub-task generation and posting** — generation and confirmation are a single step.

### B3. Create the issues

After approval, create all task issues in one batch — no per-issue confirmation. Create in dependency order so blocker issue numbers exist when referenced.

Use GitHub's native sub-issue feature if the repo has it enabled (`gh api`); otherwise the parent-child link lives in the task body's `## Parent` field plus the summary comment in B4.

Apply priority labels: `priority: high`, `priority: medium`, `priority: low`. Create the labels once if they don't exist.

### B4. Summary comment

After all issues are created, post **one** comment on the parent PRD listing the children with their priority and blocked-by state. Do not modify the parent body or close it.

### Task issue template

```markdown
## Parent
#<parent-issue-number>

## Priority
high  (or medium / low)

## Implements
PRD functional requirements: 1, 3

## Relevant Files
- `path/to/file1.ts` — Brief description of why this file is relevant.
- `path/to/file1.test.ts` — Unit tests for `file1.ts`.
- `path/to/another/file.tsx` — Brief description.
- `path/to/another/file.test.tsx` — Unit tests for `another/file.tsx`.

## Notes
- Brief implementation notes if any (e.g., "Tests colocated with implementation files").

## Subtasks
- [ ] 1.1 Sub-task description
- [ ] 1.2 Sub-task description
- [ ] 1.3 Sub-task description

## Blocked by
- [ ] #<issue-number>

(omit Blocked by if unblocked)
```

## Acceptance criteria conventions

- **Verifiable, not vague.** "Works correctly" is bad. "Button shows confirmation dialog before deleting" is good.
- Each sub-task is a verifiable unit. When all sub-tasks are checked, the parent task is done.
- **Universal gates** (typecheck, tests pass, UI verified in browser for UI changes) are Definition of Done — stated once project-wide, not repeated on every task.

## Pause and resume

The skill can stop after Phase A — the user might want to sit on the PRD overnight, share it, edit it on GitHub. To resume Phase B later, invoke with the parent issue number; pick up from B1.

## Escape hatches

- **Task too big when written up.** Stop and surface to the user. Don't ship a wrong-sized issue.
- **Honestly one issue.** Skip the PRD. Open a single issue (task template minus Parent) and stop.

## Do not

- Interview the user. /sync already did. (One targeted question if something's genuinely missing.)
- Write the PRD as a `.md` file in the repo. It lives as a GitHub issue.
- Auto-post to GitHub without confirmation.
- Modify or close the parent issue. (One summary comment listing children is fine and encouraged.)
- Use HITL/AFK terminology. Use **blocked / unblocked** plus `priority: high|medium|low` — both map to GitHub natively.
- Start implementing. Stop after issues are created.
