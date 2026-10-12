---
name: sync
description: Interview the user until we share one understanding of a project, feature, fix, refactor or exploration, before /spec.
argument-hint: [brief or request]
disable-model-invocation: true
---

Interview me until we reach a shared understanding of this request. Do not create or edit files, and do not write code.

Request: $ARGUMENTS
If empty, use the brief or my latest message.

## 1. Read first

Read the request and the repo. Never ask what they already answer. Finding facts is your job; decisions are mine.

## 2. Type and design tree

- Detect the type: project, feature, fix, refactor or explore. For a chore, say no sync is needed and stop.
- Build the design tree yourself from the request: the main decisions, most important first. Start with the problem and users. When AI behaviour is involved, include guardrails and evals.
- Show the type (add "?" if unsure) and the top branches, each marked open or already answered. Wait for me to confirm, reorder or skip.

## 3. Ask in rounds

- The frontier is every open question whose prerequisites are settled. Ask 2 to 3 per round, most important first: what blocks other decisions or is costly to get wrong.
- Business before technical.
- Each question has your recommended answer and a short reason. Word it so "yes" accepts it.
- Plain numbered list. No bold labels, no arrows, no emoji:

```
1. Who reads the result: support agents or customers?
   Recommended: support agents, the brief says they sort the emails.
```

- I can reply "yes" to accept all, or answer by number.

## 4. Suggestions

When a branch is done, add "Suggestions:" (max 2) if you see a better option or a risk nobody raised. They are proposals only and count only if I accept them.

## Keywords

- "next": leave this branch. Its open questions become open points.
- "enough": stop asking.

## 5. End

When the tree is done, or on "enough", list the type and the open points. Then say: "Run /spec to write it."
