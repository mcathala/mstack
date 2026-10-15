---
name: sync
description: Interview the user until we share one understanding of a project, feature, fix, refactor or exploration, before /spec. Use only when the user asks for sync by name.
argument-hint: [request, or your planned approach]
disable-model-invocation: true
---

Align with me on what to build before any spec or code. Do not create or edit files, and do not write code.

Request: $ARGUMENTS
If empty, use my latest message and any linked brief, issue or doc.

## 1. Read first

Read the request, the repo and every linked doc. Never ask what they already answer. Finding facts is your job; decisions are mine.

## 2. Start from my approach

If I stated an approach, start from it: say what holds, then challenge the weak points. Do not replace it with yours.
If I gave no approach, propose a plan, then ask around it.

## 3. Design tree

The choices to make before building, grouped by topic.
- Leaf: one choice only I can make, with its options. Facts are found, not asked.
- Branch: a topic built from the request.
- Status: open, answered, assumed (I can override), later. Write "after <ID>" when a leaf depends on another.
- Each round asks every open choice that can be decided now (nothing it depends on is still open), most important first.
- At the start, show the branches so I can check every topic is covered.

## 4. First message

- Type: project, feature, fix, refactor or explore. For a chore, say no sync is needed and stop. Add "?" if unsure.
- The branches of the design tree. Start with the problem and users. When AI behaviour is involved, include guardrails and evals.
- Then the first round of questions.

## 5. Questions

- Ask only real choices. Anything with one obvious answer goes in one line: "Assumed: <ID> ...".
- Most important first: a choice that blocks other choices, or is costly to get wrong.
- Questions only someone else can answer are marked "(ask <who>)".
- Each question uses its leaf ID: one short line with all the information needed to answer, then 2 or 3 options of one line each with their trade-off, then your recommendation and why. The recommendation can be any option, and "do nothing" can be an option. No yes/no questions unless the choice really is binary. No bold, arrows or emoji. Format:

```
<Branch>
<ID> <Question>?
   A. <Option> (<trade-off>)
   B. <Option> (<trade-off>)
   Recommended: <letter>, <why>.
```

- How I answer: "ok all" accepts every recommendation of this round; "2.1 ok" accepts one; "2.1B" picks another option; the same form overrides an assumed choice. I can mix them. Any question I don't mention takes your recommendation.

## 6. Keywords

- "next": leave this branch. Its open choices become open points.
- "enough": stop asking.

## 7. End

When every choice is settled, or on "enough", list the type, the decisions in one line each, and the open points. Then say: "Run /spec to write it."
