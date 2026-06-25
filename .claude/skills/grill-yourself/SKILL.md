---
name: grill-yourself
description: Autonomously self-interview a plan or design — pose each grilling
  question AND answer it with your own recommended choice, then produce a numbered
  decision table plus a plan. No back-and-forth dialogue. Read-only: never edits
  project code. Invoke explicitly with /grill-yourself.
disable-model-invocation: true
---

This is the autonomous, no-dialogue counterpart to the `grilling` skill. Instead of
interviewing the user one question at a time, walk the design tree yourself: pose
each question AND answer it with your recommended choice, without stopping for input.

## Hard rule: never modify project code
Plan only. Do NOT use Edit, Write, or NotebookEdit on any source file. Codebase
exploration is strictly read-only (Read / Grep / Glob, and Bash for inspection only).
The artifact you produce is a plan — output it in chat. Write it to a `.md` file only
if the user explicitly asks.

(Note: this rule is enforced by directive, not by the harness. Honor it strictly.)

## How to run (zero stops, end-to-end)
1. **Surface scan.** Explore the codebase (read-only) and enumerate EVERY decision
   the plan hinges on. This list becomes the decision table — don't stop at the
   obvious few.
2. **Deep-fill the consequential decisions.** For each decision whose reversal would
   change other parts of the plan, resolve it with your recommended answer + a
   one-line rationale. Skip purely cosmetic leaf details (give them sensible defaults
   in the plan instead of a table row).
3. **Build the decision table**, numbered, split into two buckets:
   - **Confident** — recommendable from codebase facts and sound defaults.
   - **Assumptions / needs you** — only the user can truly decide (deadlines, product
     direction, priorities, business constraints). State the assumption you made and
     flag it for confirmation.
4. **Write the plan** that follows from those decisions.
5. **Footer:** "To flip a decision, re-invoke with `#<n>=<value>`."

Run with zero stops. The user course-corrects afterward by overriding numbered rows.

## Overrides
If invoked with one or more `#<n>=<value>` arguments, treat those decisions as locked
to the given values and re-derive everything else around them. This relies on the
prior decision table being present in the conversation context — if it isn't (e.g. a
fresh session), ask the user to restate the plan, then re-run the full scan.

## The one allowed stop
If no plan or target is given, infer it from the conversation and working context.
Only if there is genuinely nothing to grill — no plan stated and none inferable — ask
the user what to grill. That is the sole exception to the zero-stops rule.
