---
name: reviewing-plans
description: Objectively review an IMPLEMENTATION plan produced by writing-plans (or any
  task-decomposed plan) from a FRESH context — the execution-layer counterpart to grill-review.
  Where grill-review critiques the DESIGN (decision table), reviewing-plans critiques the
  TRANSLATION of that design into executable tasks. It does NOT re-litigate design
  decisions. Spawns a separate read-only subagent that never saw the authoring reasoning,
  evaluates every task along 5 axes (traceability, decomposition, completeness/placeholder,
  interface consistency, buildability) against the real codebase AND the upstream spec, and
  returns a two-layer verdict — fact (confirmed/false/unverifiable, file:line evidence) ×
  severity (Blocker/Advisory) — plus a diff-style revision for Blockers, a disposition, and
  a re-grill list. Flags: --deep (parallel reviewers), auto (Needs-you first, then hands-free
  fix loop via writing-plans that also folds Advisory into the revision). Optional model arg
  (sonnet/opus/haiku/fable) picks the reviewer model; omitted = cross-match (sonnet↔opus) for
  independence, else inherit this agent's model. Reviewer reasoning effort defaults to high;
  override with an effort arg (low/medium/high/xhigh/max). Optionally pass --viz to render the
  report as an HTML artifact dashboard. Invoke explicitly with /reviewing-plans.
disable-model-invocation: true
---

The execution-layer objective review. `writing-plans` turns a settled design into a
task-by-task implementation plan; this skill checks whether that translation is
complete, faithful, and buildable — before an implementer or subagent touches code. The
agent that wrote the plan is the worst judge of it: it remembers what it *meant* the
tasks to say. So the actual review is NEVER done by you (the current agent). You hand the
plan to a fresh subagent and relay its verdict.

Where this sits in the pipeline:
- `grill-yourself` → produce the DESIGN (decision table + design spec)
- `grill-review` → critique that DESIGN from a fresh context (are the decisions right?)
- `writing-plans` → translate the design into an executable task plan
- **`reviewing-plans`** → critique that EXECUTION PLAN from a fresh context (this skill)
- `subagent-driven-development` / `executing-plans` → build it
- `verifying-implementation` → verify the built CODE against the plan

## Scope boundary — the one rule that makes this skill distinct
**reviewing-plans reviews the TRANSLATION, not the DESIGN.** The design decisions are assumed
settled (grill-review already adjudicated them). Do NOT re-open "should the retry count be
3?" — that is grill-review's job. reviewing-plans asks only: *is the design faithfully,
completely, and buildably expressed as tasks?*

If the reviewer believes a design decision itself is wrong, it does NOT raise it as a plan
Blocker. It emits at most **one** Advisory: `[Advisory] design concern (out of scope) —
route to grill-review: <one line>` and moves on. Re-litigating design here wastes tokens
and blurs the two skills' boundary.

## Input
Take the **implementation plan** from a `.md` path if given as an argument, otherwise from
the most recent plan in the conversation. A reviewing-plans target is a *task-decomposed* plan
(tasks with Files / Interfaces / bite-sized steps), typically a `writing-plans` output —
NOT a decision table. If what you're handed is a decision table, tell the user this is
grill-review's input, not reviewing-plans's, and stop.

If an upstream **spec/design** is available (a `grill-yourself` design or a
`docs/**/specs/*.md`), take its path too — it is the ground truth for the traceability
axis. If none is given, infer it from the conversation; if there is genuinely none, run
the other four axes and note "Traceability: no upstream spec available — checked
internal consistency only."

Note which repo / working directory the plan targets and pass that to the reviewer, so it
verifies file paths and references against the right codebase as ground truth (the plan
`.md` may live in a fixture or describe a different repo than the current cwd).

If neither a path nor an inferable plan exists, ask the user for a plan to review; do not
invent one.

**Never overwrite the input `.md`.** Return the review — and any revised plan from `auto` —
in chat. Write to a file only if the user explicitly asks.

**Hard rule — purity of what you pass.** When you spawn the reviewer, pass ONLY the plan
(and the upstream spec, if any), VERBATIM. Never include your own reasoning, the codebase
exploration that produced the plan, or any defense of it. The reviewer must read it cold,
as a stranger's work. This is the one thing that makes the review objective — the fresh
subagent context is wasted if you leak authoring context into the prompt.

## Model selection (optional arg)
The args may carry a model name — one of **`sonnet` / `opus` / `haiku` / `fable`** —
alongside any flags. Resolve the reviewer model **once** and apply it to **every** Agent
spawn below (the default reviewer, the `--deep` reviewers, and the `auto`-loop reviewers):

- **Model name given** → pass it as the Agent tool's `model` override on every spawn.
- **No model name given** → cross-match for maximum independence:
  - Orchestrator is `sonnet` → reviewer uses `opus`
  - Orchestrator is `opus` → reviewer uses `sonnet`
  - Otherwise (haiku, fable, or unknown) → OMIT the `model` parameter so the reviewer inherits your current model.

Parse args leniently: a token matching a known model name sets the reviewer model;
`deep`/`--deep` and `auto`/`--auto` set the flags (below); a token matching a known effort
level sets the reviewer effort (see *Effort selection*); a `viz`/`--viz` token turns on
HTML visualization (see *Visualization*); a `.md` token is a path — the first is the plan,
a second (or one under `specs/`) is the upstream spec. Order doesn't matter, dashes are
optional.

## Effort selection (optional arg)
The args may also carry an effort level — one of **`low` / `medium` / `high` / `xhigh` /
`max`**. Resolve the reviewer effort **once** and apply it to **every** Agent spawn:
- **Effort level given** → pass it as the Agent tool's `effort` override on every spawn.
- **No effort level given** → default to **`high`**.

## How to run (default: fast single reviewer)
Use the Agent tool with `subagent_type=general-purpose`, passing the model resolved in
*Model selection* and the effort resolved in *Effort selection*, to run the review in a
fresh context.

(Note: `general-purpose` retains write tools — read-only here is enforced by **directive**,
not by the harness. State the read-only rule explicitly and forbid the reviewer from
spawning further agents. If you want harness-enforced read-only at the cost of audit depth,
`subagent_type=Explore` removes the write tools.)

Instruct that subagent to:

- Treat the plan as a stranger's work — do NOT defend it. Assume the design is settled;
  review only the translation into tasks (see *Scope boundary*).
- Work **read-only**: Read / Grep / Glob and Bash for inspection only. Never Edit, Write,
  or NotebookEdit anything; never run mutating shell commands; never spawn further subagents.
- **Reality check first (mandatory if the plan touches code).** Identify every concrete
  file path, function, type, API, library, and command the plan references and verify each
  with Glob/Grep/Read against the current codebase: does the file exist (or is its parent
  creatable)? does the function/type it consumes already exist or is it produced by an
  earlier task? is the library importable? Record what was checked and found — this is the
  evidence base for the buildability axis. You may NOT raise a buildability finding without
  first attempting verification; speculation is not a finding.
- **Evaluate each task along 5 axes** against the REAL codebase and the upstream spec
  (ground truth), not against the plan's own claims:
  1. **Traceability** — does every spec/design requirement map to at least one task? Is any
     requirement silently dropped? (If no upstream spec, check internal consistency only.)
  2. **Decomposition** — are tasks right-sized and independently testable, in correct
     dependency order? Does a task depend on something a later task produces? Does one task
     bundle unrelated deliverables that a reviewer couldn't approve/reject as a unit?
  3. **Completeness / placeholder** — any `TODO`, `TBD`, "implement later", "add appropriate
     error handling", "similar to Task N", or a code step with no actual code / a test step
     with no actual test? These are plan failures (see writing-plans' No-Placeholders rule).
  4. **Interface consistency** — do the Consumes/Produces signatures line up across tasks? A
     function called `clearLayers()` in Task 3 but `clearFullLayers()` in Task 7 is a defect.
     Are types/params/return values used in later tasks actually defined in earlier ones?
  5. **Buildability** — could an implementer follow this task without getting stuck? Are the
     referenced paths/functions/libs real (per the reality check)? Are the commands runnable
     with the stated expected output?
  A task may have findings on several axes (record each), or zero (the expected default — do
  not invent findings to fill a quota).
- **Calibration.** Only flag what would cause a real problem during implementation: an
  implementer building the wrong thing, getting stuck, or a task that can't be acted on.
  Minor wording and stylistic preference are NOT findings.
- **Classify every finding on two orthogonal layers:**
  - **Fact verdict** — `confirmed` / `false` / `unverifiable`, each with **`file:line`
    evidence** (the plan location, and the codebase location it does/doesn't match). A
    "false" with no citation is just another unverified opinion — forbidden. For
    **unverifiable**, state the locations inspected or the search scope and why the evidence
    was insufficient — "unverifiable" is not a free pass for not looking.
  - **Severity** — `Blocker` (implemented as-is, the implementer fails, builds the wrong
    thing, or gets stuck) vs `Advisory` (buildable, but worth knowing). The layers are
    independent: a `false` can be Blocker or Advisory; so can an `unverifiable`.

The subagent works in whatever language it reasons best in (English is fine) — only the
final report YOU relay to the user is translated.

## Output (relay the subagent's report — translate to Korean)

Translate the subagent's report into **Korean** for the user. Keep code identifiers, file
paths, and `file:line` citations verbatim; render verdicts, reasoning, and findings in
Korean. Do not use emoji.

**Relay faithfully — translation only.** You (the orchestrator) may have authored the plan
and carry its bias, so do NOT soften, upgrade, drop, or add to the reviewer's verdicts. The
reviewer's judgment is authoritative; you only restate it in Korean.

Return, in order:

```
<DISPOSITION> — Blocker N, Advisory M    (append " (auto)" when auto mode produced a revision)

**검토 대상**: <mode> / <plan 제목 또는 첫 태스크 요약>
**검토 범위**: 태스크 K개, 코드 탐색 X회, 스펙 대조 <있음/없음>
**리뷰어**: fresh subagent (<model>, effort=<effort>), 격리 컨텍스트

## 비판
| # | 태스크·스텝 | 사실판정 | 등급 | 축 | 비판 (파일/함수 언급 시 file:line 인용) |
|---|---|---|---|---|---|

## 수정안   (Blocker는 항상 / Advisory는 auto일 때만)
- <태스크 #>: <원래 스텝/인터페이스> → **<수정 내용>** — <해결되는 결함 한 줄>.
  (auto 모드에서는 각 항목에 [Blocker]/[Advisory] 접두, Blocker 먼저 정렬)

## Re-grill list
- **Auto-fixable** — 계획 표현 결함. writing-plans 재호출로 해당 태스크만 재생성 가능.
- **Needs you** — 스펙 자체의 공백/모호. 계획을 다시 짜도 또 다른 추측일 뿐, user만 해결.
- **Out of scope (design)** — grill-review로 라우팅할 설계 우려(있으면).

## Reality check trace
- <검증 항목(파일/함수/라이브러리)>: <존재/부재 결과>
```

1. **Verdict table** — per task: fact verdict + severity + axis + evidence.
2. **Revision (수정안)** — diff-style. Always include Blocker findings. In `auto` mode, also
   fold Advisory findings (tagged `[Advisory]`, ordered after Blockers); if an Advisory has
   no concrete actionable revision, still emit `[Advisory] #: (no concrete revision) —
   <원본 비판 한 줄>` so it is not silently dropped. If a finding needs a brand-new task the
   plan lacked, label it `#-new-<suffix>`. Omit this section entirely if there is nothing to
   revise.
3. **Disposition** — one line: **SHIP / REVISE (which tasks) / REJECT**.
4. **Re-grill list** — the false / unverifiable items, split Auto-fixable vs Needs-you, plus
   any out-of-scope design concern routed to grill-review.

When Needs-you items exist, resolve them first. Then **offer once** (default mode only):
"Re-generate the auto-fixable tasks now?"

**Needs-you grilling (both modes, when Needs-you items exist):** After the 1st review, before
any re-generate loop, use the `AskUserQuestion` tool to ask the user about each Needs-you
item (a spec gap the plan can't resolve on its own). In `auto`, if a later loop iteration
discovers a new Needs-you item, pause the loop, resolve it immediately, then resume.
- Convert each gap into a concrete, answerable question. Never ask yes/no — force a specific
  choice or concrete value. Users can always pick "Other" to type a custom answer.
- AskUserQuestion accepts 1–4 questions per call. If there are more than 4, batch across
  sequential calls.
- For each question, include a recommended option — place it first and append `(Recommended)`.
- After the user answers, patch the affected task inline in chat, marked **[confirmed]**,
  noting which item it resolves. Do not silently absorb the answer — make it visible.

### Disposition line (computed mechanically)
- `REJECT` / `REVISE (#…)` — Blocker > 0
- `SHIP` with advisories — Blocker = 0, Advisory > 0 (keep the Advisory rows + reality trace)
- `CLEAN — 결함 없음` — both = 0. Omit the 비판 / 수정안 / Re-grill / Reality trace sections;
  output the header + 검토 대상/범위 lines + one sentence:
  `이 계획은 5개 축(추적성/분해/완결성/인터페이스 정합성/실행가능성) 모두에서 결함이 발견되지 않았습니다. 구현 진행을 권장합니다.` Then stop.

## Visualization (optional, `--viz`)
When the args carry a `viz`/`--viz` token, ALSO render the review as a self-contained HTML
artifact dashboard — but only **after** the normal chat report is emitted and translated.
The translated chat report stays the primary deliverable and comes first; the artifact is
supplementary. Read the shared procedure at
`<this skill's base directory>/../_shared/grill-viz.md` (the `_shared` sibling of this
skill's directory; read it by path, not via the Skill tool) and follow it.

Map the review into the data contract: `title` = plan title / first-task summary; `subtitle`
= `<mode> / 태스크 K개 / <model>, effort=<effort>`; `disposition` = the disposition line and
`dispositionKind` = its keyword (`ship`/`revise`/`reject`/`clean`); ONE `buckets` entry
(name e.g. `"비판"`) whose rows set `n`, `question` = the full 태스크·스텝 text (OMIT
`answer`), `axis`, `verdict`, `severity`, and `rationale`; `revisions` = the 수정안;
`realityTrace` = the reality-check trace lines; `reGrillList` = the Auto-fixable / Needs-you
/ Out-of-scope split. Favicon 🧱.

- **CLEAN disposition** → emit a minimal artifact: title + the green CLEAN banner, no cards.
- **`auto` mode** → update the SAME artifact once, after the fix loop finishes, to the final
  revised state; note the iteration count in `subtitle`. Do not redeploy per iteration.

If the Artifact tool is unavailable, the shared procedure prints the HTML file path instead
— visualization never blocks or delays the report.

## `--deep` (more rigor, slower)
Instead of one reviewer, spawn in parallel on the resolved model and effort:
- **completeness-checker** — axes ①③④ (traceability, completeness/placeholder, interface
  consistency), verifying tasks against the spec and cross-task references. If there are many
  tasks, shard this across several checkers by task range.
- **buildability-critic** — axes ②⑤ (decomposition, buildability) + reality check against the
  codebase.

Merge and dedupe their findings into the single output above. (Same-model reviewers share
training blind spots; --deep buys depth-per-lens, not true independence — genuine independence
comes from a **different reviewer model**, which the default cross-match already gives you.)

## `auto` (Needs-you first, then hands-free fix loop, Advisory included)
Skip the confirmation and run the fix loop yourself. `auto` does two things at once: it
**folds Advisory findings into the revision** (not just Blockers), and it **drives a
hands-free re-generate loop**. The loop is hands-free only after spec gaps are confirmed:
- **Step 1 — Needs-you rows first** → Do NOT best-guess fill a spec gap. After the 1st
  review, if any Needs-you rows exist, resolve them interactively via AskUserQuestion (see
  *Needs-you grilling*) and place the confirmed answers in context before any re-generate
  loop starts. If there are none, skip this step.
- **Step 2 — Auto-fixable rows** → re-generate the affected tasks by invoking the
  `writing-plans` skill (via the Skill tool) on those tasks, feeding it the findings and the
  settled design as context. If its instructions aren't loaded, read
  `~/.claude/skills/writing-plans/SKILL.md` first. Then spawn a **fresh** reviewer (new
  context, same resolved model and effort) on the revised plan and repeat until the
  disposition is SHIP **or** a **max of 3 iterations** is reached. The 3-iteration cap counts
  re-generate loop iterations only; Needs-you resolution steps do not count. Keep every
  revision in chat — never write it back over the input `.md`.
- **Step 3 — New Needs-you during the loop** → if a later iteration surfaces a new spec gap,
  pause the loop, resolve it immediately via AskUserQuestion, place the confirmed answer in
  context, then resume.
- **Oscillation guard** — if a fix introduces a new auto-fixable failure, stop and report
  rather than looping. A newly discovered Needs-you row is not oscillation; resolve it via
  Step 3 instead.

`--deep` and `auto` are orthogonal and combine (`/reviewing-plans --deep auto`). Accept the flags
with or without leading dashes and in any order, and an optional model name and/or effort
level alongside them (`/reviewing-plans sonnet high --deep auto`). **Auto mode is opt-in** — only
the explicit `auto` argument activates it; never infer it from the user's surrounding
language, the plan's content, or the finding count.

## Hard constraints
- **Never modify files.** Not the plan, not the codebase, not docs. The output is a single
  message. Editing is the next turn's job, driven by the user. (The sole exception: a `--viz`
  run writes ONE self-contained HTML file to the scratchpad directory for the visualization.)
- **Never review the design.** Design decisions are grill-review's domain. A wrong-looking
  decision gets at most one out-of-scope Advisory routed to grill-review — never a plan Blocker.
- **Never re-enter Q&A (reviewer only).** The reviewer subagent must not ask clarifying
  questions — if a task is ambiguous, log it as a Completeness/Buildability finding and move
  on. (The orchestrator's Needs-you grilling via AskUserQuestion is the one deliberate
  exception.)
- **Never invent buildability findings.** If you didn't actually grep/read, you can't claim a
  path or function is missing.
- **Never fill quota.** No findings on an axis is the correct output when there are none.
- **Default = single reviewer + review-only.** The fix loop waits for your one confirmation.
  The pure objective review lands instantly; mutating the plan is opt-in.
