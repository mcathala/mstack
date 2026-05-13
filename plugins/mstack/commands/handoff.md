---
description: Compact the current conversation into a handoff doc at a temp path so a fresh agent can pick up the work.
argument-hint: "[focus] — optional description of what the next session will focus on"
allowed-tools: Bash(git:*), Bash(mktemp:*), Bash(date:*), Read, Write
---

# /handoff — Session handoff

Write a handoff document summarising the current session so a fresh agent can continue the work. Save it to the temp path produced in the Context block below (it already exists as an empty file — `Read` it first, then `Write` over it).

This is **not a transcript summary**. It is a state dump answering: what are we building, where do we stand, what's stuck, what's next.

If `$ARGUMENTS` is non-empty, treat it as a description of what the next session will focus on and tailor the doc accordingly.

## Context

- Time: !`date '+%Y-%m-%d %H:%M %Z'`
- Branch: !`git branch --show-current`
- HEAD: !`git log -1 --oneline`
- Working tree: !`git status --short`
- Modified files: !`git diff --name-only HEAD`
- Untracked files: !`git ls-files --others --exclude-standard`
- Recent commits on this branch: !`git log $(git merge-base HEAD main 2>/dev/null || echo HEAD~10)..HEAD --oneline 2>/dev/null | head -20`
- Diff stat vs main: !`git diff --stat $(git merge-base HEAD main 2>/dev/null || echo HEAD~10)..HEAD 2>/dev/null | tail -15`
- Output path: !`mktemp "${TMPDIR:-/tmp}/handoff-XXXXXX.md"`

## What to write

Synthesize from **(a) the conversation we just had** and **(b) the git context block above**. Do **not** invent state you have no evidence for — if you don't know something, write `unknown` rather than guessing.

Do **not** duplicate content already captured in other artifacts (PRDs, plans, ADRs, issues, commits, diffs). Reference them by path or URL instead.

Use this exact structure. Keep sections tight — bullets, not paragraphs. Trim ruthlessly.

```markdown
# Handoff — <YYYY-MM-DD HH:MM> — <branch>

## Goal
<1–3 bullets: what we're trying to build, and why. The durable "north star,"
stable across sessions — not the current sub-task. If a PRD issue / design
doc exists, link to it rather than re-summarising.>

## Status right now
<Where the work actually stands. Concrete. e.g., "Mid-way through issue #42,
sub-task 3/5 — the parser refactor is done, callsites are half-migrated."
One short paragraph OR 3–5 bullets.>

## Active files
<Files with uncommitted changes RIGHT NOW (from `git status` above). For
each, one line on *what* is being changed and *why*. Path → intent.>

## What didn't work
<Approaches tried and abandoned, with the *reason*. Format:
- Tried X → didn't work because Y.
So the next session doesn't re-attempt them. If nothing was tried-and-
rejected, write "nothing notable".>

## Open questions
<Decisions pending, things we don't yet know, or external blockers.
Omit the section if there are none.>

## Suggested skills
<Which skills the next session should consider invoking, if any. e.g.,
`mstack:start-plan` to continue implementation from the PRD, `mstack:diagnose`
to debug the failing test, `mstack:validate` to check the metric. Omit
the section if none apply.>

## Resume here
<The next concrete action. Be specific enough that the next session can
execute it without re-reading the rest of the document. Two examples:

Debug context:
"Run `pytest tests/parser_test.py::test_nested_quotes` and address the
assertion on line 42 — the fix is probably to handle escaped backslashes
in `Parser._consume_quote()`."

Design context:
"Decide between approach A (factor parser logic into a new `tokenizer.rs`,
sketched at src/parser.rs:1-40) and approach B (keep inline, add a state
machine around src/parser.rs:200). A is simpler; B handles the unicode
escape case raised in conversation."

NOT: "continue working on the parser".>

## Repro
- Branch: <branch>
- HEAD: <short SHA + subject>
- Working tree: <clean | N modified, M untracked>
- Last command run: <if known>
```

## Steps

1. Use the **Output path** from the Context block. `Read` it first (it exists but is empty), then `Write` the composed handoff to that same path.
2. Compose using the structure above, populated from:
   - The conversation in this session.
   - The Context block — use the `Time:` and git values **verbatim**, do not guess.

## Report

Compact chat output:
- Path written (so the user can copy/paste it into the next session — e.g., "feed this to a fresh agent: `<path>`").
- Verbatim copy of the **Resume here** line.
- If "What didn't work" was filled from a transcript that may be truncated due to compaction, flag it explicitly — e.g., `Note: earlier session content may be lost to compaction — "What didn't work" partial.`

## Rules

- **No invented state.** If a section has nothing real to say, write `unknown` or `none` and move on. Padding defeats the purpose.
- **Reference, don't duplicate.** PRDs, plans, ADRs, issues, commits, diffs already exist — link to them, don't re-summarise their contents.
- **Resume here is the single most important line.** Spend disproportionate care on it. A vague "Resume here" makes the whole handoff useless.
- **Flag compaction loss.** If the transcript may be truncated, say so rather than confidently writing "nothing notable" in "What didn't work".
