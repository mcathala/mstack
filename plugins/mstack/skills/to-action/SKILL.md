---
name: to-action
description: Turn an aligned understanding (typically from /sync) into action — a PRD as a parent GitHub issue plus vertical-slice child issues an AI can pick up cold. Use when the user says "make this actionable", "turn this into issues", "commit this plan", or follows /sync. Do NOT interview — synthesize what was already discussed.
---

# to-action

Commit a planning conversation into a fanned-out plan on GitHub:

- **Phase A:** PRD as a parent issue
- **Phase B:** vertical-slice child issues, linked to the parent

Each child issue is written so an AI agent can pick it up cold and start working.

**Do NOT interview the user.** /sync already did that. If something is genuinely missing, ask a single targeted question — never a questionnaire.

## When to skip

If the work is genuinely one-issue-sized (a bug fix, a small UI tweak, a contained refactor), don't write a PRD. Tell the user, offer to open one issue using the issue template (minus Parent), and stop.

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

Confirm the module list with the user. Confirm which modules they want tests for.

### A2. Draft the PRD

Use the template below. Show the draft. Confirm before posting — `gh issue create` is a shared-state action.

### A3. Post to GitHub

Title: `PRD: <feature-name>`. Ask once whether labels should be applied (e.g. `prd`, `epic`).

Use `gh issue create` with the body via heredoc. Return the parent issue URL.

### PRD template

```markdown
## Problem
The problem from the user's perspective. One tight paragraph.

## Solution
The solution from the user's perspective. One tight paragraph.

## User Stories
A small, well-chosen list. Numbered (US-001, US-002, ...) so Phase B can reference them. Quality over volume.

Format: As a <actor>, I want <feature>, so that <benefit>.

User stories describe value to a user. If a "story" is really an implementation task ("add column to DB"), put it in Implementation Decisions instead.

## Implementation Decisions
- Modules and their interfaces
- Architectural decisions
- Schema changes / API contracts / specific interactions

No file paths or code snippets — they go stale.

## Testing Decisions
Which modules will be tested. Reference prior art (similar tests in the codebase) instead of re-litigating testing philosophy.

## Out of Scope
Explicit non-goals.

## Open Questions
Genuinely unresolved items. Leave empty if there are none — not a junk drawer.
```

## Phase B — Slices

### B1. Draft vertical slices

Each slice is a thin path through every layer the feature touches (schema → API → UI → tests), demoable on its own.

**Vertical is the default.** Horizontal is allowed when honestly justified — schema migrations, refactors, dep upgrades, foundation work. Mark these explicitly and explain why.

**Size:** thin enough to demo independently *and* ship in roughly one focused session. Finer than that is bookkeeping.

Mark each slice **unblocked** or **blocked by #N**. Prefer unblocked.

### B2. Quiz the user

Show the proposed breakdown as a numbered list. For each slice:

- **Title**
- **State**: unblocked / blocked by #N
- **Implements**: PRD user stories (US-001, ...)
- **One-line description**

Ask:
- Granularity right? Too coarse / too fine?
- Dependencies right?
- Anything missing?
- Any labels to apply to all child issues?

Iterate until approved.

### B3. Create the issues

After approval, create all issues in one batch — no per-issue confirmation. Create in dependency order so blocker issue numbers exist when referenced.

Use GitHub's native sub-issue feature if the repo has it enabled (`gh api`); otherwise use task-list syntax for blockers (`- [ ] #N`) so the checkbox auto-ticks when the referenced issue closes.

After all issues are created, post **one** comment on the parent PRD listing the children. Do not modify the parent body or close it.

### Issue template

```markdown
## Parent
#<parent-issue-number>

## Implements
PRD user stories: US-001, US-003

## What to build
The end-to-end behavior of this slice from a user's perspective. Not layer-by-layer steps. Written so an agent with no prior conversation context can understand the goal.

## Acceptance criteria
- [ ] Specific, verifiable criterion
- [ ] Another verifiable criterion
- [ ] [for UI changes] Verified in browser

Universal gates (typecheck, tests pass) apply to every issue and are not repeated here.

## Blocked by
- [ ] #<issue-number>

(omit this section if unblocked)
```

## Acceptance criteria conventions

- **Verifiable, not vague.** "Works correctly" is bad. "Button shows confirmation dialog before deleting" is good.
- **Universal gates** (typecheck, tests, UI verified in browser for UI changes) are Definition of Done — stated once project-wide, not repeated on every story.

## Pause and resume

The skill can stop after Phase A — the user might want to sit on the PRD overnight, share it, edit it on GitHub. To resume Phase B later, invoke with the parent issue number; pick up from B1.

## Escape hatches

- **Slice too big when written up.** Stop and surface to the user. Don't ship a wrong-sized issue.
- **Honestly one issue.** Skip the PRD. Open a single issue (issue template minus Parent) and stop.

## Do not

- Interview the user. /sync already did. (One targeted question if something's genuinely missing.)
- Write the PRD as a `.md` file in the repo. It lives as a GitHub issue.
- Auto-post to GitHub without confirmation.
- Modify or close the parent issue. (One summary comment listing children is fine and encouraged.)
- Use HITL/AFK terminology. Use **blocked / unblocked** — it maps to GitHub natively.
- Start implementing. Stop after issues are created.
