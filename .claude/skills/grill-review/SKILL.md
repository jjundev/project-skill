---
name: grill-review
description: Objectively review a plan produced by grill-yourself (or any plan with
  a decision table) from a FRESH context. Spawns a separate read-only subagent that
  never saw the authoring reasoning, verifies every decision against the real
  codebase, and returns per-row verdicts + a disposition + a re-grill list. Flags:
  --deep (parallel reviewers), auto (hands-free fix loop). Optional model arg
  (sonnet/opus/haiku/fable) picks the reviewer model; omitted = inherit this agent's
  model. Invoke explicitly with /grill-review.
disable-model-invocation: true
---

The objective-review companion to `grill-yourself`. The agent that wrote a plan is
the worst judge of it — it carries every rationalization in context. So the actual
review is NEVER done by you (the current agent). You hand the plan to a fresh
subagent and relay its verdict.

## Input
Take the plan + decision table from a `.md` path if given as an argument, otherwise
from the most recent plan in the conversation. If neither exists — no path and no plan
inferable from the conversation — ask the user for a plan to review; do not invent one.

Note which repo / working directory the plan targets and pass that to the reviewer, so
it verifies against the right codebase as ground truth (the plan `.md` may live in a
fixture or describe a different repo than the current cwd).

**Never overwrite the input `.md`.** Return the review — and any revised plan from
`auto` — in chat. Write to a file only if the user explicitly asks.

**Hard rule — purity of what you pass.** When you spawn the reviewer, pass ONLY the
plan and its decision table, VERBATIM. Never include your own reasoning, the codebase
exploration that produced the plan, or any defense of it. The reviewer must read it
cold, as a stranger's work. This is the one thing that makes the review objective —
the fresh subagent context is wasted if you leak authoring context into the prompt.

## Model selection (optional arg)
The args may carry a model name — one of **`sonnet` / `opus` / `haiku` / `fable`** —
alongside any flags. Resolve the reviewer model **once** and apply it to **every** Agent
spawn below (the default reviewer, the `--deep` reviewers, and the `auto`-loop reviewers):

- **Model name given** → pass it as the Agent tool's `model` override on every spawn.
- **No model name given** → OMIT the `model` parameter so each reviewer inherits *your*
  (the grill-review orchestrator's) current model — if you are running on Opus, the
  reviewer runs on Opus too; on Sonnet, Sonnet too. Do not hard-code a model.

Parse args leniently: a token matching a known model name sets the reviewer model;
`deep`/`--deep` and `auto`/`--auto` set the flags (below); a `.md` token is the plan
path (see *Input*). Order doesn't matter, dashes are optional.

## How to run (default: fast single reviewer)
Use the Agent tool with `subagent_type=general-purpose`, passing the model resolved in
*Model selection*, to run the review in a fresh context.

(Note: `general-purpose` retains write tools — read-only here is enforced by
**directive**, not by the harness. So state the read-only rule explicitly and forbid
the reviewer from spawning further agents. If you ever want harness-enforced read-only
at the cost of audit depth, `subagent_type=Explore` removes the write tools.)

Instruct that subagent to:

- Treat the plan as a stranger's work — do NOT defend it.
- Work **read-only**: Read / Grep / Glob and Bash for inspection only. Never Edit,
  Write, or NotebookEdit anything; never run mutating shell commands; never spawn
  further subagents.
- Apply three lenses and verify against the REAL codebase (ground truth), not against
  the plan's own claims:
  1. **Assumptions vs code** — every「Assumptions / needs you」row: is the assumption
     true against the actual code?
  2. **Confident decisions vs code** — every「Confident」row: a confident pick can
     still be wrong against reality. Check it too.
  3. **Plan-level coherence / feasibility** — missing steps, internal contradictions,
     unrealistic premises that no single row captures.
- Return, for each reviewed item, a **verdict: confirmed / false / unverifiable**,
  each with **`file:line` evidence**. A "false" with no citation is just another
  unverified opinion — forbidden. For **unverifiable**, state the locations inspected
  or the search scope and why the evidence was insufficient — "unverifiable" is not a
  free pass for not looking.

The subagent works in whatever language it reasons best in (English is fine) — only
the final report YOU relay to the user is translated. Don't constrain the reviewer's
language; constrain only the output.

## Output (relay the subagent's report — translate to Korean)

Translate the subagent's report into **Korean** for the user. Keep code identifiers,
file paths, and `file:line` citations verbatim; render verdicts, reasoning, and
findings in Korean.

**Relay faithfully — translation only.** You (the orchestrator) may have authored the
plan and carry its bias, so do NOT soften, upgrade, drop, or add to the reviewer's
verdicts. The reviewer's judgment is authoritative; you only restate it in Korean.

Return, in order:
1. **Verdict table** — per row: verdict + evidence, plus lens-③ plan-level findings.
2. **Disposition** — one line: **SHIP / REVISE (which rows) / REJECT**.
3. **Re-grill list** — the false / unverifiable items, split into two buckets:
   - **Auto-fixable** — code-grounded failures; grill-yourself can re-derive these
     correctly by re-reading ground truth.
   - **Needs you** — user-only (「가정」) failures; re-grilling only yields another
     guess. Only the user (or a `/grill-me` session) can truly resolve these.

Then **offer once**: "Re-grill the auto-fixable rows now?" Route the needs-you bucket
to the user; never silently resolve it.

## `--deep` (more rigor, slower)
Instead of one reviewer, spawn in parallel:
- **fact-checker** — lenses ① + ②, verifying rows against code. If there are many
  rows, shard this across several fact-checkers by row.
- **critic** — lens ③, plan-level coherence / feasibility only.

Spawn every reviewer on the model resolved in *Model selection*. Merge and dedupe their
findings into the single output above. (Same-model reviewers share training blind spots;
--deep buys depth-per-lens, not true independence.) For genuine independence, spawn the
reviewers on a **different model** than the one that authored the plan — pass it as the
explicit model arg, or set the Agent tool's `model` override per reviewer. That, not just
more reviewers, is what escapes shared blind spots.

## `auto` (hands-free fix loop)
Skip the confirmation and run the fix loop yourself:
- **Auto-fixable rows** → re-derive them by invoking the `grill-yourself` skill (via the
  Skill tool) on those rows. It relies on the prior decision table being in context; if
  its instructions aren't loaded, read `~/.claude/skills/grill-yourself/SKILL.md` first.
  Then spawn a **fresh** reviewer (new context, same model resolved in *Model selection*)
  on the revised plan and repeat until the disposition is SHIP **or** a **max of 3
  iterations** is reached. Keep every revision in chat — never write it back over the
  input `.md`.
- **Oscillation guard** — if a fix introduces a new failure, stop and report rather
  than looping.
- **Needs-you (user-only) rows** → you CANNOT truly resolve these. Re-fill them with a
  best-guess assumption so the plan stays complete, but ALWAYS report them in the
  final output as still-unverified「Assumptions」. Never claim they are resolved.

`--deep` and `auto` are orthogonal and combine (`/grill-review --deep auto`). Accept the
flags with or without leading dashes and in any order (`deep`, `--auto`, etc.), and an
optional model name alongside them (`/grill-review sonnet --deep auto`).

## Default
No flags = single reviewer + review-only (the fix loop waits for your one
confirmation). The pure objective review lands instantly; mutating the plan is opt-in.
