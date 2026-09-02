# EP Advance Queue Before Wrap Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #325 — executing-plans: advance queue unconditionally when ALL_DONE before wrap/continue choice

**Goal:** Rewrite executing-plans Step 4 to advance the queue unconditionally before offering wrap/continue choice, preventing wrap from bypassing queue advancement.

**Architecture:** Replace the current two-bullet Step 4 with a three-phase sequence: (4a) unconditional mechanical advance, (4b) session choice after advance, with queue-complete fallback to work-end.

**Tech Stack:** Markdown (SKILL.md edit)

## Global Constraints

- Edit the source at `~/claude/hortora/soredium/executing-plans/SKILL.md`, not the installed copy
- Run `sync-local` after the edit to propagate to `~/.claude/skills/`

---

## Batch 1: Rewrite Step 4

### Task 1: Replace Step 4 routing in executing-plans SKILL.md

**Files:**
- Modify: `executing-plans/SKILL.md:180-192` — replace current Step 4

- [ ] **Step 1: Read the current Step 4 text to confirm exact boundaries**

The text to replace starts at line 180 (`**4. Route to next step:**`) and ends at line 192 (the last line of the work-end bullet, before the blank line at 193).

Current text:
```markdown
**4. Route to next step:**

```bash
python3 ~/.claude/skills/project/ctx.py
```

Read `HAS_PLAN` and `PLAN_POSITION` from the output.

- **If `HAS_PLAN=yes` and the queue has remaining issues** → this plan
  is done but the branch has more work. Run `work next` to advance to
  the next issue — do NOT invoke work-end.
- **If no plan, or the queue is complete** → invoke work-end for final
  review, squash, push, and branch closure.
```

- [ ] **Step 2: Replace with the three-phase Step 4**

Use the Edit tool to replace lines 180-192 with the new text. The replacement:

```markdown
**4. Route to next step:**

```bash
python3 ~/.claude/skills/project/ctx.py
```

Read `HAS_PLAN` and `PLAN_POSITION` from the output.

**4a. Advance queue (unconditional, mechanical):**

If `HAS_PLAN=yes` and the queue has remaining issues:

1. Run `work next` immediately — this advances the queue, closes the
   GitHub issue, and sets the next issue as active.
2. Report: "Advanced to #<next>. Issue #<completed> closed."

This is not a user choice. Queue advancement is a mechanical consequence
of task completion.

**4b. Offer session choice:**

After advancing (or if no advance was needed):

- If queue has remaining issues (just advanced):
  - **Continue** → proceed to next issue's plan
  - **Wrap** → invoke handover (queue already advanced, next session
    starts clean at the new active issue)

- If queue is complete, or no plan exists:
  - Invoke work-end for final review, squash, push, and branch closure.
```

- [ ] **Step 3: Verify the edit didn't corrupt surrounding sections**

Read lines 170-210 of the modified file. Confirm:
- Step 3's "Triage each" bullet (line ~178) still ends cleanly before Step 4
- The "When to Stop and Ask for Help" section follows after Step 4 with proper spacing
- No orphaned code fences or broken markdown

- [ ] **Step 4: Sync to installed skills**

Run: `python3 scripts/claude-skill sync-local --all -y`

Expected: sync completes, `~/.claude/skills/executing-plans/SKILL.md` matches the edit.

- [ ] **Step 5: Commit**

```bash
git add executing-plans/SKILL.md
git commit -m "fix(#325): advance queue unconditionally before wrap/continue choice

Closes #325"
```

## References

- [2026-09-03-ep-advance-queue-before-wrap-design.md] — design spec
- [executing-plans/SKILL.md:180-192] — current Step 4 (the broken path)
- [executing-plans/SKILL.md:96-102] — Step 2 batch completion (already correct, unchanged)
- [GitHub #325] — reproducer and root cause
