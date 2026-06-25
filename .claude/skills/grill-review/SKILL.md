---
name: grill-review
description: Objectively review a plan produced by grill-yourself (or any plan with a
  decision table) from a FRESH context. Spawns a separate read-only subagent that never
  saw the authoring reasoning, evaluates every decision along 5 axes (contradiction,
  hidden assumption, omission, reality mismatch, vagueness) against the real codebase,
  and returns a two-layer verdict — fact (confirmed/false/unverifiable, file:line
  evidence) × severity (Blocker/Advisory) — plus a diff-style revision for Blockers, a
  disposition, and a re-grill list. Flags: --deep (parallel reviewers), auto (hands-free
  fix loop that also folds Advisory into the revision). Optional model arg
  (sonnet/opus/haiku/fable) picks the reviewer model; omitted = inherit this agent's
  model. Invoke explicitly with /grill-review.
disable-model-invocation: true
---

The objective-review companion to `grill-yourself`. The agent that wrote a plan is
the worst judge of it — it carries every rationalization in context. So the actual
review is NEVER done by you (the current agent). You hand the plan to a fresh subagent
and relay its verdict.

This skill complements the grill family:
- `grill-me` / `grill-yourself` → produce a plan
- **`grill-review`** → critique that plan from a fresh context (this skill, pre-implementation)
- `grill-verify` → verify implementation against plan (post-implementation)

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
  (the grill-review orchestrator's) current model. Do not hard-code a model.

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
- **Reality check first (mandatory if the plan touches code).** Identify every concrete
  code/file/API/library/config reference the plan makes and verify each with
  Glob/Grep/Read against the current codebase. Record what was checked and found — this
  is the evidence base for the "reality mismatch" axis. You may NOT raise a reality
  mismatch without first attempting verification; speculation is not a finding. If the
  plan has zero code references, note "Reality check: code-independent plan — skipped".
- **Evaluate each decision along 5 axes** against the REAL codebase (ground truth), not
  against the plan's own claims:
  1. **Contradiction** — does this decision conflict with another in the same plan?
  2. **Hidden assumption** — what unstated premise must hold? Is it actually true?
  3. **Omission** — is a critical sub-decision missing? Is an Unresolved area on a
     critical path that cannot be deferred?
  4. **Reality mismatch** — does the codebase / domain / library actually support this?
     (Must be backed by the reality-check evidence above.)
  5. **Vagueness** — is the decision actionable as-stated, or will the implementer have
     to re-decide it?
  A decision may have findings on several axes (record each), or zero (the expected
  default — do not invent findings to fill a quota).
- **Classify every finding on two orthogonal layers:**
  - **Fact verdict** — `confirmed` / `false` / `unverifiable`, each with **`file:line`
    evidence**. A "false" with no citation is just another unverified opinion —
    forbidden. For **unverifiable**, state the locations inspected or the search scope
    and why the evidence was insufficient — "unverifiable" is not a free pass for not
    looking.
  - **Severity** — `Blocker` (implemented as-is, the plan will fail or behave broken)
    vs `Advisory` (implementable, but worth knowing). The two layers are independent: a
    `false` can be Blocker or Advisory; so can an `unverifiable`.

The subagent works in whatever language it reasons best in (English is fine) — only
the final report YOU relay to the user is translated. Don't constrain the reviewer's
language; constrain only the output.

## Output (relay the subagent's report — translate to Korean)

Translate the subagent's report into **Korean** for the user. Keep code identifiers,
file paths, and `file:line` citations verbatim; render verdicts, reasoning, and
findings in Korean. Do not use emoji.

**Relay faithfully — translation only.** You (the orchestrator) may have authored the
plan and carry its bias, so do NOT soften, upgrade, drop, or add to the reviewer's
verdicts. The reviewer's judgment is authoritative; you only restate it in Korean.

Return, in order:

```
<DISPOSITION> — Blocker N, Advisory M    (append " (auto)" when auto mode produced a revision)

**검토 대상**: <mode> / <plan 제목 또는 첫 결정 요약>
**검토 범위**: 결정 K개, 코드 탐색 X회 (또는 "코드 무관 plan")
**리뷰어**: fresh subagent (<model>), 격리 컨텍스트

## 비판
| # | 결정 | 사실판정 | 등급 | 축 | 비판 (현실 충돌이면 file:line 인용) |
|---|---|---|---|---|---|

## 수정안   (Blocker는 항상 / Advisory는 auto일 때만)
- <#>: <원래 결정> → **<수정 결정>** — <해결되는 결함 한 줄>.
  (auto 모드에서는 각 항목에 [Blocker]/[Advisory] 접두, Blocker 먼저 정렬)

## Re-grill list
- **Auto-fixable** — 코드 기반 실패. grill-yourself가 ground truth 재독으로 재유도 가능.
- **Needs you** — user-only (가정) 실패. 재유도해도 또 다른 추측일 뿐, user(또는 /grill-me)만 해결.

## Reality check trace
- <검증 항목>: <결과>
```

1. **Verdict table** — per decision: fact verdict + severity + axis + evidence.
2. **Revision (수정안)** — diff-style. Always include Blocker findings. In `auto` mode,
   also fold Advisory findings (tagged `[Advisory]`, ordered after Blockers); if an
   Advisory has no concrete actionable revision, still emit
   `[Advisory] #: (no concrete revision) — <원본 비판 한 줄>` so it is not silently
   dropped. If a finding needs a brand-new decision the plan lacked, label it
   `#-new-<suffix>`. Omit this section entirely if there is nothing to revise.
3. **Disposition** — one line: **SHIP / REVISE (which rows) / REJECT**.
4. **Re-grill list** — the false / unverifiable items, split Auto-fixable vs Needs-you.

Then **offer once** (default mode only): "Re-grill the auto-fixable rows now?" Route the
needs-you bucket to the user; never silently resolve it.

### Disposition line (computed mechanically)
- `REJECT` / `REVISE (#…)` — Blocker > 0
- `SHIP` with advisories — Blocker = 0, Advisory > 0 (keep the Advisory rows + reality trace)
- `CLEAN — 결함 없음` — both = 0. Omit the 비판 / 수정안 / Re-grill / Reality trace
  sections; output the header + 검토 대상/범위 lines + one sentence:
  `이 plan은 5개 축(모순/숨겨진 가정/누락/현실 충돌/모호함) 모두에서 결함이 발견되지 않았습니다. 진행을 권장합니다.` Then stop.

## `--deep` (more rigor, slower)
Instead of one reviewer, spawn in parallel on the resolved model:
- **fact-checker** — axes ②③④ + reality check, verifying rows against code. If there
  are many rows, shard this across several fact-checkers by row.
- **critic** — axes ①⑤ + plan-level coherence / feasibility only.

Merge and dedupe their findings into the single output above. (Same-model reviewers
share training blind spots; --deep buys depth-per-lens, not true independence.) For
genuine independence, spawn the reviewers on a **different model** than the one that
authored the plan — pass it as the explicit model arg. That, not just more reviewers,
is what escapes shared blind spots.

## `auto` (hands-free fix loop, Advisory included)
Skip the confirmation and run the fix loop yourself. `auto` does two things at once:
it **folds Advisory findings into the revision** (not just Blockers), and it **drives a
hands-free re-derive loop**:
- **Auto-fixable rows** → re-derive them by invoking the `grill-yourself` skill (via the
  Skill tool) on those rows. It relies on the prior decision table being in context; if
  its instructions aren't loaded, read `~/.claude/skills/grill-yourself/SKILL.md` first.
  Then spawn a **fresh** reviewer (new context, same resolved model) on the revised plan
  and repeat until the disposition is SHIP **or** a **max of 3 iterations** is reached.
  Keep every revision in chat — never write it back over the input `.md`.
- **Oscillation guard** — if a fix introduces a new failure, stop and report rather
  than looping.
- **Needs-you (user-only) rows** → you CANNOT truly resolve these. Re-fill them with a
  best-guess assumption so the plan stays complete, but ALWAYS report them in the final
  output as still-unverified 가정. Never claim they are resolved.

`--deep` and `auto` are orthogonal and combine (`/grill-review --deep auto`). Accept the
flags with or without leading dashes and in any order, and an optional model name
alongside them (`/grill-review sonnet --deep auto`). **Auto mode is opt-in** — only the
explicit `auto` argument activates it; never infer it from the user's surrounding
language, the plan's content, or the finding count.

## Hard constraints
- **Never modify files.** Not the plan, not the codebase, not docs. The output is a
  single message. Editing is the next turn's job, driven by the user.
- **Never re-enter Q&A.** If the plan is ambiguous, log it as a Vagueness finding and
  move on. Do not ask clarifying questions.
- **Never invent Reality mismatches.** If you didn't actually grep/read, you can't claim
  the codebase contradicts the plan.
- **Never fill quota.** No findings on an axis is the correct output when there are none.
- **Default = single reviewer + review-only.** The fix loop waits for your one
  confirmation. The pure objective review lands instantly; mutating the plan is opt-in.
