# writing-plans Decision Checkpoint Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a "Decision Checkpoint" to the `writing-plans` skill so the planner pauses after mapping file structure and asks the user, via `AskUserQuestion`, about genuine execution-level forks the spec/codebase can't settle — defaulting to zero questions.

**Architecture:** Insert one new `## Decision Checkpoint` section into `writing-plans/SKILL.md`, placed immediately after `## File Structure` (where decomposition locks in) and before `## Task Right-Sizing`. The section borrows `grill-yourself`'s AskUserQuestion format rules (recommended-first, no yes/no, 1–4 per call) but changes the timing (after file-structure mapping, not end-of-plan) and the scope (execution-level decisions only). The skill exists in two identical copies that must be kept byte-identical.

**Tech Stack:** Markdown skill file (no code, no test harness). Verification is by `grep`/`diff` on file content.

## Global Constraints

- The skill lives in **two locations that must stay byte-identical**:
  - `.claude/skills/writing-plans/SKILL.md` (git-tracked, edited via the worktree)
  - `~/.claude/skills/writing-plans/SKILL.md` (global, NOT git-tracked — sync manually)
- The two copies are currently identical (`diff` returns nothing). They must remain identical after this change.
- Match the existing skill's prose voice: imperative, terse, no emoji, `**bold**` for directives.
- Do not alter any other section, the YAML frontmatter, or `plan-document-reviewer-prompt.md`.
- Worktree path (git-tracked copy root): `/Users/hyunjun_macbook_pro/Documents/Project/project_skill/.claude/worktrees/writing-plans-user-questions-367675`
- Global copy path: `/Users/hyunjun_macbook_pro/.claude/skills/writing-plans/SKILL.md`

---

### Task 1: Add the Decision Checkpoint section to the git-tracked copy

**Files:**
- Modify: `.claude/skills/writing-plans/SKILL.md` (insert between `## File Structure` block ending at the line `This structure informs the task decomposition. Each task should produce self-contained changes that make sense independently.` and the following `## Task Right-Sizing` heading)

**Interfaces:**
- Consumes: nothing (leaf edit to a markdown file)
- Produces: a new `## Decision Checkpoint` heading present in the file, relied on by Task 2 (sync) and Task 3 (verification)

- [ ] **Step 1: Read the file and locate the insertion point**

Run: `grep -n "## Task Right-Sizing" ".claude/skills/writing-plans/SKILL.md"`
Expected: one match — this heading is where the new section is inserted *before*. Confirm the line immediately above it is the blank line following the `## File Structure` paragraph that ends `...make sense independently.`

- [ ] **Step 2: Insert the new section**

Insert the following block so it sits after the `## File Structure` section's final paragraph and before `## Task Right-Sizing` (one blank line separating each heading from surrounding text):

```markdown
## Decision Checkpoint

Once the File Structure is mapped, pause and scan for decisions that (a) the
spec and the codebase cannot settle and (b) would change the file structure or
task decomposition if reversed. These are execution-level forks — a library
choice with real trade-offs, a schema or data model the spec left open, an
ambiguous scope boundary. **Cosmetic defaults and anything resolvable by reading
the code are not checkpoint decisions — decide those yourself and move on.**

If — and only if — such decisions exist, ask the user before writing tasks,
using the `AskUserQuestion` tool:

- Convert each decision into a concrete, answerable question. Never yes/no —
  force a specific choice or a concrete value. (The user can always pick "Other"
  to type their own answer.)
- Lead with your recommended option: place it first in the options list and
  append `(Recommended)` to its label, chosen from codebase facts and sound
  defaults.
- `AskUserQuestion` takes 1–4 questions per call. If more than 4 decisions
  remain, batch them across sequential calls.
- After the answers land, fold each choice into the File Structure and the tasks
  that follow. If an answer diverges from your recommendation, propagate the
  change to every decision that depended on it.

**Default to zero questions.** A well-settled spec should leave nothing to ask —
the design was resolved during brainstorming/grilling, and this checkpoint only
catches genuine execution forks the spec never covered. Do not manufacture
questions; if nothing qualifies, proceed straight to task decomposition.
```

- [ ] **Step 3: Verify the section landed in the right place with the right neighbors**

Run: `grep -n -e "^## File Structure$" -e "^## Decision Checkpoint$" -e "^## Task Right-Sizing$" ".claude/skills/writing-plans/SKILL.md"`
Expected: three matches in this exact order — `## File Structure`, then `## Decision Checkpoint`, then `## Task Right-Sizing` (ascending line numbers).

- [ ] **Step 4: Verify the AskUserQuestion format rules are present**

Run: `grep -c -e "AskUserQuestion" -e "(Recommended)" -e "Default to zero questions" ".claude/skills/writing-plans/SKILL.md"`
Expected: each pattern found (non-zero counts) — confirms the recommended-first rule, the tool reference, and the zero-questions default all made it in.

- [ ] **Step 5: Commit**

```bash
git add .claude/skills/writing-plans/SKILL.md
git commit -m "feat(writing-plans): add Decision Checkpoint for user decisions during planning"
```

---

### Task 2: Sync the change to the global (non-git) copy

**Files:**
- Modify: `/Users/hyunjun_macbook_pro/.claude/skills/writing-plans/SKILL.md` (must end byte-identical to the git-tracked copy)

**Interfaces:**
- Consumes: the finished git-tracked copy from Task 1 (source of truth)
- Produces: a global copy identical to the git-tracked copy, relied on by Task 3's identity check

- [ ] **Step 1: Copy the git-tracked copy over the global copy**

Run:
```bash
cp ".claude/skills/writing-plans/SKILL.md" "/Users/hyunjun_macbook_pro/.claude/skills/writing-plans/SKILL.md"
```
Expected: no output (success).

- [ ] **Step 2: Verify the two copies are byte-identical**

Run:
```bash
diff ".claude/skills/writing-plans/SKILL.md" "/Users/hyunjun_macbook_pro/.claude/skills/writing-plans/SKILL.md" && echo IDENTICAL
```
Expected: `IDENTICAL` (diff prints nothing, exit 0). No commit — the global copy is outside any git repo.

---

### Task 3: Full-file verification pass

**Files:**
- Read only: both copies of `SKILL.md`

**Interfaces:**
- Consumes: the edited git-tracked copy (Task 1) and the synced global copy (Task 2)
- Produces: confirmation that the change is complete, correctly placed, and identical across both locations

- [ ] **Step 1: Confirm no other section was disturbed**

Run: `git diff --stat HEAD~1 -- .claude/skills/writing-plans/SKILL.md`
Expected: only `.claude/skills/writing-plans/SKILL.md` changed, with insertions only (no deletions of existing lines other than adjacent blank-line reflow). Review `git diff HEAD~1 -- .claude/skills/writing-plans/SKILL.md` and confirm the sole change is the added `## Decision Checkpoint` section.

- [ ] **Step 2: Confirm section ordering and identity together**

Run:
```bash
grep -n "^## " ".claude/skills/writing-plans/SKILL.md" && \
diff ".claude/skills/writing-plans/SKILL.md" "/Users/hyunjun_macbook_pro/.claude/skills/writing-plans/SKILL.md" && echo SYNCED
```
Expected: the heading list shows `## Decision Checkpoint` sitting between `## File Structure` and `## Task Right-Sizing`, followed by `SYNCED`.

---

### Task 4 (optional): Record the dual-location fact in memory

**Files:**
- Create: `/Users/hyunjun_macbook_pro/.claude/projects/-Users-hyunjun-macbook-pro-Documents-Project-project-skill/memory/writing-plans-dual-location.md`
- Modify: `/Users/hyunjun_macbook_pro/.claude/projects/-Users-hyunjun-macbook-pro-Documents-Project-project-skill/memory/MEMORY.md` (add one index line)

**Interfaces:**
- Consumes: nothing
- Produces: a durable note that `writing-plans` (like the grill-* skills) is dual-located and edits must hit both copies

- [ ] **Step 1: Write the memory file**

```markdown
---
name: writing-plans-dual-location
description: writing-plans skill lives in project (git) + global copies; edit both
metadata:
  type: reference
---

The `writing-plans` skill exists in two byte-identical copies that must be kept
in sync when edited: `.claude/skills/writing-plans/SKILL.md` (git-tracked) and
`~/.claude/skills/writing-plans/SKILL.md` (global, not in git). Same pattern as
[[grill-skills-dual-location]]. The global copy will not be committed — sync it
manually with `cp` after editing the git-tracked one.
```

- [ ] **Step 2: Add the index pointer to MEMORY.md**

Append under the existing memory list:

```markdown
- [writing-plans dual location](writing-plans-dual-location.md) — writing-plans lives in project (git) + global; edit both
```

- [ ] **Step 3: (No commit)** — the memory directory is outside the project repo; nothing to commit.

---

## Self-Review

**1. Spec coverage** (spec = the agreed design from the prior turn):
- "Pause during plan creation at important decision moments" → Decision Checkpoint section, placed after File Structure (before tasks are written). ✓
- "Ask via AskUserQuestion" → Step 2 content references the tool with format rules. ✓
- "Only genuine execution-level forks; default to zero questions" → `**Default to zero questions.**` paragraph + cosmetic/code-resolvable exclusions. ✓
- "Reuse grill-yourself format rules (recommended-first, no yes/no, 1–4/call)" → bullet list in Step 2. ✓
- "Keep both skill copies in sync" → Task 2 + Task 3 identity checks + Global Constraints. ✓

**2. Placeholder scan:** No TBD/TODO/"handle edge cases"/"similar to Task N". The full section text is inline in Task 1 Step 2 — no "fill in later". ✓

**3. Type consistency:** N/A for prose, but cross-references are consistent — the section heading name `## Decision Checkpoint`, the tool name `AskUserQuestion`, and the marker string `Default to zero questions` are used identically in the insert (Task 1 Step 2) and in every verification grep (Tasks 1, 3). ✓

## Notes on adaptation

This is a markdown skill edit, not code, so there is no pytest/red-green cycle. Verification steps use `grep`/`diff` with exact expected output in the TDD slot's spirit (exact commands, expected results, no placeholders). Task 4 is optional and touches only the out-of-repo memory store.
