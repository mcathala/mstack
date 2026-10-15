---
name: spec
description: Turn the decisions from /sync into a one-page spec at docs/specs/<n>-<slug>.md, with eval and guardrails and a task checklist. Use only when the user asks for a spec by name.
disable-model-invocation: true
---

Synthesize this conversation into a spec file. Do not interview me. Running /spec is the go to write this file, not to write code.

## Rules

- Write only what was decided. Everything else goes in Assumptions.
- No code, file paths or schemas. Describe behaviour in words.
- If nothing was decided, say so, suggest /sync and stop.
- Short bullets, plain words. One page max.
- Structure: "##" for sections, "###" for groups inside a section. Never more than one level of bullets.

## File

- Path: docs/specs/<n>-<slug>.md, where n is the highest existing number + 1, with no leading zeros.
- A new step or requirement gets a new spec that links the previous one in "Related".
- A change to the same feature edits its spec and adds a line to the Changes log.

## Template

Skip sections meant for other types.

<spec-template>
# <n>. <Title>
Type: project | feature | fix | refactor | explore · Status: draft · Related: <spec, brief or issue>

## Summary
- Problem: one sentence: what we are trying to solve, and why if mentioned.
- Solution: one sentence: what we build.
- Done when: one sentence: how we know it works.

## Original request
Only the part of the brief or my message this spec covers, verbatim, in a quote block.

## Decisions
### <Group>
- One decision per line.

## Eval and guardrails
Project, or a feature or fix that changes AI behaviour.
How we check it works, and what must never happen.

## Safety
Refactor only. What must not change and how we check it.

## Findings
Explore only. Time box. Findings and decision stay empty until done.

## Tasks
Grouped by area, numbered in build order across groups. Each task ends with something to run or see.
### <Area>
- [ ] 1. ...

## Out of scope

## Assumptions
Open points and anything not decided.

## Changes log
Added only when the spec is edited: one line per change, what and why.
</spec-template>

## Tasks and status

- Tick a task only after it ran and its check passed.
- Set Status to done when all tasks are ticked and I confirm.

## After writing

Print the path, the Summary and the task list. Then say: "Review the spec. Say go to implement all tasks, starting from task 1."
