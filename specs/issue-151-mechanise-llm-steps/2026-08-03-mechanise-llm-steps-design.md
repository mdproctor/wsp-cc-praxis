# Mechanise LLM-Dependent Steps + Epic Lifecycle Awareness

**Issue:** #151
**Date:** 2026-08-03
**Principle:** Scripts do, scripts verify, LLMs decide.

---

## Problem

Two related gaps in the work lifecycle:

1. **Mechanisation:** Five steps rely on the LLM remembering to do them. LLMs
   don't carry state between sessions — "noted for next time" is not enforcement.
2. **Epic awareness:** work-end, merge_slot, work-pause, and work-resume have no
   consistent awareness of epic state. You can accidentally close mid-epic with
   no warning, lose epic context across pause/resume, and see no batch info in
   the stack picker.

Same root cause: scripts should surface state and enforce invariants; LLMs should
make decisions based on what scripts report.

---

## Layer 1 — Foundation

### A. `epic_manager.py` — canonical epic detection + check subcommand

**Problem:** Three independent parsers for "is this an epic?" — ctx.py (checks
`workspace/design/.epic`), work_router.py (checks `.slot` + `.epic`),
slot_manager.parse_slot_md (checks `.slot`). Should be one source of truth.

**Change:** Add two things to `epic_manager.py`:

1. `detect(path: Path) -> dict | None` — given a workspace or slot path, find
   and parse the epic file. Returns `{"epic_path": Path, "is_epic": True, ...}`
   or `None`. Checks both `design/.epic` (single-repo) and `.slot` with
   `Type: epic` (slot). This replaces the ad-hoc parsing in ctx.py and
   work_router.py.

2. `check` CLI subcommand — calls `status()` (already exists) and outputs
   KEY=VALUE lines for consumption by work-end and merge_slot:
   ```
   IS_EPIC=yes
   EPIC_COMPLETE=yes|no
   SAFE_EXIT=yes|no
   CURRENT_BATCH=2
   TOTAL_BATCHES=4
   ACTIVE_ISSUE=83
   COMPLETED_COUNT=5
   TOTAL_COUNT=12
   ```

`status()` already computes all of this (returns `current_batch`,
`current_issue`, `batches` with completion state). The `check` subcommand
is just a CLI wrapper.

**Tests:** Unit tests for `detect()` with both `.epic` and `.slot` epic files,
non-epic `.slot`, and missing files. Integration test for `check` subcommand
output format.

### B. ctx.py enrichment

**Change:** Replace the inline epic detection (lines 165-170) with a call to
`epic_manager.detect()`. Add `EPIC_BATCH` and `EPIC_ACTIVE_ISSUE` to output.
This means work-end (which already runs ctx.py) gets batch context without
needing a separate `check` call for the simple case.

The `check` subcommand (Layer 2) is still needed for `EPIC_COMPLETE` and
`SAFE_EXIT` — ctx.py stays cheap and doesn't compute completion state.

**Tests:** Update existing ctx.py tests for new output fields.

---

## Layer 2 — Gates

### C. work-end epic confirmation gate

**Change:** Add Step 0b in work-end Pre-conditions, after branch divergence
check, before pause stack check.

Read `IS_EPIC` from ctx.py output (already run in Path Resolution). If `no`,
skip silently.

If `yes`, run:
```bash
python3 ~/.claude/skills/work-slot/epic_manager.py check <EPIC_PATH>
```

Three outcomes:

| State | UX |
|-------|----|
| `EPIC_COMPLETE=yes` | Proceed silently — all children done |
| `SAFE_EXIT=yes` | "Batch N/M complete. Safe exit point — close? (y/n)" |
| Neither | "⚠ Mid-batch (issue #X of batch N/M). Partial close loses context. Continue? (y/confirm-partial)" — requires typing `confirm-partial`, not just `y` |

The mid-batch confirmation is deliberately high-friction. Accidentally closing
mid-batch is the bug that surfaced this issue.

**Skill change only** — no new script (uses `check` from Layer 1).

### D. merge_slot() epic check

**Change:** At the top of `merge_slot()`, after the `.phase-a-complete` check,
call `parse_slot_md()` to check `is_epic`. If epic:
- Call `epic_manager.status()` on the `.slot` file
- Print `EPIC_STATUS=batch N/M, N completed, N remaining`
- Informational only — not blocking (Phase A already involved user confirmation)

**Tests:** Add test for merge_slot with epic slot — verify status is printed.

---

## Layer 3 — Mechanical Enforcement

### E. archive_slot() checkbox verification

**Change:** In `archive_slot()`, before moving to attic, if the slot is an epic
(check via `parse_slot_md`), read the `.slot` file and verify all completed
issues have `- [x]`. If any have `- [ ]` but their GitHub issue is CLOSED,
auto-tick them and rewrite the file. Print `CHECKBOXES_FIXED=N` if any were
fixed.

This is a data consistency fix — the issues are already closed, the checkbox
state is just stale. No user confirmation needed.

**Tests:** Test with `.slot` containing unticked checkboxes for closed issues.

### F. phase_b_gate.py (new script)

**Purpose:** Replace the self-certified markdown checklist for Phase B
completion. Reads state, returns structured pass/fail.

**Location:** `work-end/phase_b_gate.py`

**Checks:**
1. All repos in slot have stamp commits (`chore: branch closed`)
2. All issues in COVERS are CLOSED on GitHub
3. `.artifacts-promoted` stamp exists
4. Slot directory is in `worktrees/attic/` (archived)

**Output:**
```
GATE=pass
```
or:
```
GATE=fail
MISSING=stamps:engine,issues:83,archive
```

**Usage in work-end Phase B:** After B7 (archive), before B8 (post-merge),
run the gate. If `GATE=fail`, hard stop with the missing items listed.

**Tests:** Tests for each check (stamps missing, issues open, no archive).

### G. merge_slot() GitHub epic checkbox tick

**Change:** After `merge_slot()` succeeds (`.landed` written), if the slot is
an epic, tick the completed child issues' checkboxes on the GitHub epic body.

Implementation: read the epic issue body via `gh api`, find `- [ ] #N` lines
for each issue in COVERS, replace with `- [x] #N`, update via `gh api`.

Add a helper function `tick_epic_checkboxes(issue_repo: str, epic_number: int,
completed_issues: list[int])` to `epic_manager.py`. merge_slot calls it after
writing `.landed`.

**Tests:** Mock `gh api` calls, verify checkbox replacement regex handles
various formats (`- [ ] #83`, `- [ ] #83 — title`, `- [ ] https://...`).

### H. update_blog_index.py (new script)

**Purpose:** Append entry to `INDEX.md` after a blog file is written to disk.
Replaces the LLM instruction "After writing, update INDEX.md."

**Location:** `write-content/update_blog_index.py`

**Usage:**
```bash
python3 ~/.claude/skills/write-content/update_blog_index.py <blog_dir> <entry_filename> <summary>
```

**Behaviour:**
1. Read `<blog_dir>/INDEX.md` (create if absent with table header)
2. Extract date from filename (`YYYY-MM-DD-*`)
3. Append: `| [<filename>](<filename>) | YYYY-MM-DD | <summary> |`
4. Write back

**Skill change:** In `write-content/forms/diary.md` Step 6, replace the manual
INDEX.md instruction with a script call.

**Tests:** Test append to existing INDEX.md, creation of new INDEX.md,
idempotency (running twice doesn't duplicate).

---

## Layer 4 — Context Propagation

### I. work-pause stack enrichment

**Change:** In `stack.py`, when pushing an entry, detect epic state:
- If IS_EPIC (from ctx.py or by checking `.epic`/`.slot`): add `epic_batch`
  and `epic_active_issue` fields to the stack entry
- If not epic: omit (backward compatible — existing stack entries without
  these fields are handled gracefully)

Stack entry format becomes:
```
branch=issue-151-...
issue=151
paused=2026-08-03T14:00:00
wip_project=yes|no
wip_workspace=yes|no
slot=72
epic_batch=2/4
epic_active_issue=83
```

**Tests:** Test push with and without epic state, pop with missing fields
(backward compat).

### J. Display improvements (skill documentation)

**J1. Router "end" option epic annotation** (`work/SKILL.md` Step 4):
When `IS_EPIC=yes` (from work_router output), change:
```
> N+1. **end** — close this branch, merge, push, return to main
```
to:
```
> N+1. **end** — ⚠️ epic Batch N of M — close this branch, merge, push, return to main
```

**J2. work-start Epic Overlay numbered** (`work-start/SKILL.md`):
Number the Epic Overlay section as Step 3d in the resume path. Change "After
the standard resume steps complete" to an explicit numbered step so it can't
be skipped.

**J3. work-resume epic context** (`work-resume/SKILL.md`):
Add Step 9b after Step 9 (work-start pre-checks): "If stack entry has
`epic_batch`, display: `Epic — Batch N, active: #M`."

**J4. work-pause/work-resume confirmation messages** (SKILL.md changes):
- work-pause confirmation: include "Epic Batch N/M, active: #X" when epic
- work-resume confirmation: include epic context from stack entry

---

## Layer 5 — Audit

### K. Stamp verification

**Finding:** `land_branch.py cmd_stamp()` (line 144-160) always calls
`verify_stamp.py` before writing the stamp. There is no code path that
skips verification. `merge_slot()` (line 706-713) stamps directly without
calling `verify_stamp.py` — but it stamps in the clone after push, and
the push itself is the verification (content must be on main for the push
to succeed).

**Verdict:** No code change needed. The slot path has a different verification
mechanism (push success = content landed). Document this difference in the
work-end skill for clarity.

---

## Execution Order

```
A (epic_manager detect+check) → B (ctx.py enrichment)
  → C (work-end gate) + D (merge_slot check)
  → E (archive checkbox) + F (phase_b_gate) + G (github tick) + H (blog index)
  → I (stack enrichment)
  → J1-J4 (skill display changes)
  → K (audit — verify only, note in spec)
```

A→B is the foundation everything depends on. C+D use the foundation for gates.
E-H are independent mechanical tasks. I needs A for detection. J is
documentation. K is read-only.

---

## Test Strategy

Every new script and every modified function gets unit tests per the
`externalised-scripts-require-tests` protocol.

| Item | Test file | Tests |
|------|-----------|-------|
| A | test_epic_manager.py | detect() with .epic, .slot, non-epic, missing |
| A | test_epic_manager.py | check subcommand output format |
| B | test_ctx.py (if exists) | EPIC_BATCH, EPIC_ACTIVE_ISSUE in output |
| D | test_slot_manager.py | merge_slot prints EPIC_STATUS for epic slot |
| E | test_slot_manager.py | archive_slot auto-ticks stale checkboxes |
| F | test_phase_b_gate.py (new) | pass, fail-stamps, fail-issues, fail-archive |
| G | test_epic_manager.py | tick_epic_checkboxes regex for various formats |
| H | test_update_blog_index.py (new) | append, create, idempotency |
| I | test_stack.py (or test_work_pause.py) | push/pop with epic fields, backward compat |

---

## Files Changed

**Scripts (new or modified):**
- `work-slot/epic_manager.py` — detect(), check subcommand, tick_epic_checkboxes()
- `work-slot/slot_manager.py` — archive_slot() checkbox fix, merge_slot() epic check + tick
- `work-end/phase_b_gate.py` — new
- `write-content/update_blog_index.py` — new
- `project/ctx.py` — epic enrichment
- `work-pause/stack.py` — epic fields (if stack.py exists; else work-pause SKILL.md instruction for the script)

**Skills (documentation):**
- `work-end/SKILL.md` — Step 0b epic gate, stamp verification note
- `work/SKILL.md` — Step 4 "end" epic annotation
- `work-start/SKILL.md` — Epic Overlay → Step 3d
- `work-resume/SKILL.md` — Step 9b epic context
- `work-pause/SKILL.md` — epic state recording
- `write-content/forms/diary.md` — INDEX.md script call

**Tests (new or modified):**
- `tests/test_epic_manager.py`
- `tests/test_slot_manager.py`
- `tests/test_phase_b_gate.py` (new)
- `tests/test_update_blog_index.py` (new)
