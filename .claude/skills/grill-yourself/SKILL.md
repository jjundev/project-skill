---
name: grill-yourself
description: >-
  Autonomously self-interview a plan or design — pose each grilling question AND
  answer it with your own recommended choice, then produce a two-bucket decision
  table (Confident / Assumptions·needs you) plus a plan. No back-and-forth
  dialogue. Terminates on convergence or a 20-decision safety cap. Read-only:
  never edits project code. Invoke explicitly with /grill-yourself.
disable-model-invocation: true
---

This is the autonomous, no-dialogue counterpart to the `grilling` skill. Instead of
interviewing the user one question at a time, walk the design tree yourself: pose
each question AND answer it with your recommended choice, without stopping for input.

## Hard rule: never modify project code
Plan only. Do NOT use Edit, Write, or NotebookEdit on any source file. Codebase
exploration is strictly read-only (Read / Grep / Glob, and Bash for inspection only).
The artifact you produce is a plan — output it in chat. Write it to a `.md` file only
if the user explicitly asks. The design is the deliverable, NOT a green light to build:
even if the resolved design is immediately implementable, do not implement it.
Implementation is a separate, user-initiated next turn.

(Note: this rule is enforced by directive, not by the harness. Honor it strictly.)

## How to run (zero stops, end-to-end)
1. **Surface scan.** Explore the codebase (read-only) and enumerate EVERY decision
   the plan hinges on. This list becomes the decision table — don't stop at the
   obvious few. When a decision can be answered by reading existing code rather than
   reasoning abstractly, read the code.
2. **Resolve in dependency order.** Take the next most-important unresolved decision
   (the one whose reversal would change the most other decisions) and resolve it with
   your recommended answer + a one-line rationale, then resolve its dependents. Skip
   purely cosmetic leaf details (give them sensible defaults in the plan instead of a
   table row).
3. **Build the decision table**, numbered, split into two buckets:
   - **Confident** — recommendable from codebase facts and sound defaults. Cite
     `file:line` in the rationale wherever a code fact backs the pick.
   - **Assumptions / needs you** — you filled these with a *guess* the user should
     confirm (deadlines, product direction, priorities, business constraints). The
     plan is complete, but state the assumption you made and flag it.
4. **Write the plan** that follows from those decisions.
5. **Footer:** "To flip a decision, re-invoke with `#<n>=<value>`."

Run with zero stops. The user course-corrects afterward by overriding numbered rows.

## Termination
Stop resolving decisions when **either** condition is met:
- **Convergence** — an implementer could write code from the resolved decisions
  without needing further design clarification.
- **Safety cap** — 20 decisions have been resolved. Stop immediately at the cap
  regardless of convergence.

If the safety cap fires before convergence, include an `## Unresolved areas` section
listing the design questions that remained genuinely *open* (you could not resolve
them at all). This is distinct from the **Assumptions / needs you** bucket, which
holds decisions you DID resolve with a guess. Rule of thumb: filled-with-a-guess →
Assumptions bucket; could-not-fill → Unresolved areas.

"Stop" means stop resolving decisions and emit the design. It never means "begin
implementation."

## Output format

```
# Plan: <one-line title>

<2–4 sentence summary of the resolved design — what gets built, how it behaves, what
the user will see.>

## Decision table

### Confident
| # | Decision (question) | Recommended answer | Rationale (one line, file:line where applicable) |
|---|---|---|---|

### Assumptions / needs you
| # | Decision (question) | Assumed value | Why only the user can truly decide |
|---|---|---|---|

## Plan

<Concrete design specification that follows from the decisions: components, data
flow, file layout, interfaces, behaviors. Written so an implementer can act on it
directly.>

## Unresolved areas (only if the cap fired before convergence)
- <open question that could not be resolved at all>

## Termination
<"Converged after N decisions." or "Hit safety cap at 20 decisions — see Unresolved areas.">

---
To flip a decision, re-invoke with `#<n>=<value>`.
```

Match the language of the surrounding conversation (e.g., if the user is writing
Korean, output in Korean).

## Overrides
If invoked with one or more `#<n>=<value>` arguments, treat those decisions as locked
to the given values and re-derive everything else around them. This relies on the
prior decision table being present in the conversation context — if it isn't (e.g. a
fresh session), ask the user to restate the plan, then re-run the full scan.

## The one allowed stop
If no plan or target is given, infer it from the conversation and working context.
Only if there is genuinely nothing to grill — no plan stated and none inferable — ask
the user what to grill. That is the sole exception to the zero-stops rule.

## Self-check before ending the turn
After emitting the design message, confirm you did NOT call Edit, Write, NotebookEdit,
or an implementation-oriented Bash command during this turn. If you did, you violated
this skill — the design document is the only allowed output.
