---
description: Sync work state with HANDOFF.md — reads the prior handoff if one exists (resume), then writes the current state (snapshot). One command for session start, end, or mid-checkpoint.
argument-hint: "[path] — optional file path, defaults to HANDOFF.md at repo root"
allowed-tools: Bash(git:*), Bash(mv:*), Bash(test:*), Bash(date:*), Bash(grep:*), Read, Write, Glob, Grep
---

# /handoff — Session handoff

Synchronize work state with `HANDOFF.md` (or `$ARGUMENTS` if provided). One command, both directions:

- **Read** the prior handoff if one exists — re-grounds the assistant (and the user) in goal, status, and the prior "Resume here" line before doing anything new.
- **Write** the current state — snapshot of where we are, anchored against what was just read.

This is **not a transcript summary**. It is a state dump answering: what are we building, where do we stand, what's stuck, what's next. Run it at session start to resume, at session end to save, or mid-session to checkpoint.

## Context

- Time: !`date '+%Y-%m-%d %H:%M %Z'`
- Branch: !`git branch --show-current`
- HEAD: !`git log -1 --oneline`
- Working tree: !`git status --short`
- Modified files: !`git diff --name-only HEAD`
- Untracked files: !`git ls-files --others --exclude-standard`
- Recent commits on this branch: !`git log $(git merge-base HEAD main 2>/dev/null || echo HEAD~10)..HEAD --oneline 2>/dev/null | head -20`
- Diff stat vs main: !`git diff --stat $(git merge-base HEAD main 2>/dev/null || echo HEAD~10)..HEAD 2>/dev/null | tail -15`
- Existing handoff: !`test -f HANDOFF.md && echo "present" || echo "none"`
- .gitignore covers HANDOFF: !`grep -qE '^HANDOFF\.(prev\.)?md$' .gitignore 2>/dev/null && echo "yes" || echo "no"`

## Output path

- If `$ARGUMENTS` is non-empty: treat it as the output path.
- Else: `HANDOFF.md` at the repo root.

If a file already exists at the resolved path, **rename it to `<path>.prev.md`** (overwriting any existing `.prev.md`) before writing the new one. One generation of backup is enough.

## What to write

Synthesize from **(a) the conversation we just had** and **(b) the git context block above**. Do **not** invent state you have no evidence for — if you don't know something, write `unknown` rather than guessing.

Use this exact structure. Keep sections tight — bullets, not paragraphs. Trim ruthlessly.

```markdown
# Handoff — <YYYY-MM-DD HH:MM> — <branch>

## Goal
<1–3 bullets: what we're trying to build, and why. The durable "north star,"
stable across sessions — not the current sub-task. If a PRD issue / design
doc exists, link to it.>

## Status right now
<Where the work actually stands. Concrete. e.g., "Mid-way through issue #42,
sub-task 3/5 — the parser refactor is done, callsites are half-migrated."
One short paragraph OR 3–5 bullets.>

## Active files
<Files with uncommitted changes RIGHT NOW (from `git status` above). For
each, one line on *what* is being changed and *why*. Path → intent.>

## Touched this session
<Files read, edited, or commanded-on during this session — even if no
longer mid-edit. Group as:
- **Edited:** <paths>
- **Read for context:** <paths>
- **Commands run:** <e.g. `pytest tests/foo_test.py`, `make cv`>
This gives the next session a map of the surface area we touched.>

## What didn't work
<Approaches tried and abandoned, with the *reason*. Format:
- Tried X → didn't work because Y.
So the next session doesn't re-attempt them. If nothing was tried-and-
rejected, write "nothing notable".>

## Open questions
<Decisions pending, things we don't yet know, or external blockers. Omit
the section if there are none.>

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

### Phase 0 — READ (if a prior handoff exists)

If the Context block shows `Existing handoff: present` at the resolved path:

1. `Read` the file.
2. Echo to chat, verbatim from the file: **Goal**, **Status right now**, and **Resume here**. This re-grounds the user and the assistant in prior state before any new authorship.
3. Carry prior content forward as the **starting point** for the write — do not re-author from scratch. Append new state; mark stale lines as superseded; **append-only for "What didn't work"** (never delete a prior dead end, even if you can no longer recall its original context).

If no prior handoff exists, skip Phase 0.

### Phase 1 — WRITE

1. **Resolve path.** Use `$ARGUMENTS` if non-empty, else `HANDOFF.md` at the repo root.
2. **Back up existing.** If a file exists at the resolved path, `mv <path> <path>.prev.md` (overwrite any existing `.prev.md`). Skip silently if none.
3. **Compose** using the structure above, populated from:
   - Prior file content carried forward from Phase 0.
   - The conversation in this session (new state since the prior write).
   - The Context block — use the `Time:` value **verbatim**, do not guess.
4. **Pickup-only mode.** If Phase 0 ran AND this session has produced no new evidence (working tree clean against prior HEAD, no in-context work yet), the write is a **pickup stamp only**: copy prior content forward, update the timestamp, and add one line to "Status right now" — e.g., `Picked up at <Time> — no new work yet.` Do not re-author other sections.
5. **Write** the file.

### Phase 2 — REPORT

Compact chat output:
- Path written.
- Verbatim copy of the **Resume here** line.
- If a `.prev.md` backup was created: one line — `Previous handoff preserved at <path>.prev.md`.
- If the Context block shows `.gitignore covers HANDOFF: no`: suggest adding `HANDOFF.md` and `HANDOFF.prev.md` to `.gitignore`. Otherwise stay silent.
- If any section was filled with `unknown` because earlier-session content appears lost to context compaction, flag it explicitly — e.g., `Note: "What didn't work" may be incomplete — earlier session content was not available in context.`

## Rules

- **Always read before writing.** If a prior handoff exists, Phase 0 is mandatory. Writing without reading silently loses prior state — exactly the failure mode this command exists to prevent.
- **Overwrite at the file level.** The file represents *current* state, not history. The one-generation `.prev.md` backup is the safety net.
- **Append-only at the section level for "What didn't work."** Carry prior dead ends forward; only add new ones. Re-attempting failed approaches is the main thing this section prevents.
- **No invented state.** If a section has nothing real to say, write `unknown` or `none` and move on. Padding defeats the purpose.
- **Flag compaction loss.** If "Touched this session" or "What didn't work" is being filled from a transcript that may be truncated, write `earlier session content may be lost to compaction — partial` rather than confidently writing `nothing notable`.
- **Resume here is the single most important line.** Spend disproportionate care on it. A vague "Resume here" makes the whole handoff useless.
