---
name: spec
description: Turn the decisions from /sync into a spec at docs/specs/<n>-<slug>.md, with eval and guardrails and a task checklist.
disable-model-invocation: true
---

Synthesize this conversation into a spec. Do not interview me. Running /spec is the go to write this file, not to write code.

## Rules

- Write only what was decided. Everything else goes in Assumptions.
- No code, file paths or schemas. Describe behaviour in words.
- If nothing was decided in the conversation, say so, suggest /sync and stop.
- Short bullets. One page max for a project, half a page for other types.

## File

- Path: docs/specs/<n>-<slug>.md, where n is the highest existing number + 1, with no leading zeros.
- To change an existing spec, edit it and add a line to a "Changes log" section at the end: what changed and why.

## Template

Skip sections meant for other types.

<spec-template>
# <n>. <Title>
Type: project | feature | fix | refactor | explore
Status: draft
Related: <other spec, if any>

## Original request
The brief or my message, verbatim, in a quote block. Never rewrite it.

## Problem
Who, the pain, what good looks like. For a fix: the symptom and how to reproduce it.

## Decisions
Grouped by branch of the design tree.

## Eval and guardrails
Project, or a feature or fix that changes AI behaviour. Behaviour matrix (kind of input, then answer, send to a human, or refuse), never-do rules, must-pass cases, threshold (score and number of runs), cost of a wrong answer. For a feature, also the existing cases that must not regress.

## Safety
Refactor only. What must not change and how we check it.

## Findings
Explore only. Time box. Findings and decision stay empty until the exploration is done.

## Tasks
- [ ] Small steps in build order, each ending with something to run or see. Eval work is a task too.

## Out of scope

## Assumptions
Open points and anything not decided.
</spec-template>

## Tasks and status

- Tick a task only after it ran and its check passed.
- Set Status to done when all tasks are ticked and I confirm.

## After writing

Print the path and the task list. Then say: "Review the spec. Say go to start task 1."
