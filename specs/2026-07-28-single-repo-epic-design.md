# Single-Repo Epic Support

**Issue:** Hortora/soredium#100
**Also closes:** #101 (validation), #103 (start vs resume)
**Date:** 2026-07-28
**Status:** Approved

## Problem

Epic-driven iteration currently requires the full slot infrastructure —
multi-repo family root, git worktrees, SLOT.md, symlink re-pointing,
Maven isolation. Single-repo projects that want to iterate through an
epic's child issues sequentially have no equivalent.

## Design Decisions

- **Single branch** for the whole epic (no branch-per-child-issue)
- **`.epic` dotfile** in `workspace/design/` for state tracking (consistent
  with `.meta`, `.slot`, `.phase-a-complete`)
- **Optional batching** — flat list for small epics, batch planning for 5+
  children (reuses existing epic_manager.py grouping logic)
- **Entry via `work epic #N`** — new verb in the `work` router
- **Iteration via `work next`** — new verb in the `work` router, context-aware
  (detects slot vs single-repo)

## .epic File Format

Lives at `workspace/design/.epic`, committed to the workspace repo.

```
# Epic #100 — single-repo-epic-support

## Issue
Hortora/soredium#100
Covers:
Type: epic

## What to do
Single-repo epic support — sequential child issue iteration.
Current: Batch 1 — Infrastructure

## Batch Plan

### Batch 1 — Infrastructure <- current
- [ ] #102 — Rename SLOT.md to .slot <- active
- [ ] #103 — Router start vs resume fix

### Batch 2 — Integration
- [ ] #104 — work next for single-repo
- [ ] #105 — work-start epic overlay

## Session State
Current batch: 1
Current issue: #102 — Rename SLOT.md to .slot
Last wrap: epic created
```

Differences from `.slot`:
- No `## Repos` section
- No `## Created` with slot number
- No `Safe exit:` line (batch boundaries still serve as natural exit points)
- Heading: `# Epic #N — slug` instead of `# Slot N — branch`

## Entry Point: `work epic #N`

New verb in `work/SKILL.md` Step 1 routing table.

1. Fetch epic issue from GitHub, parse child issues from `## Scope`
   checklist (`- [ ] #N` entries)
2. Fetch title/labels/body for each child
3. If 5+ children → batch planning (reuse epic_manager.py grouping).
   Otherwise flat ordered list as a single batch.
4. Create branch: `issue-N-<slug>`
5. Scaffold `.meta` and `JOURNAL.md` via existing `scaffold.py`
6. Write `workspace/design/.epic` with batch plan
7. Set `type: epic` in `.meta`
8. Report and instruct to run work-start

## Iteration: `work next`

New verb in `work/SKILL.md`. Detects context automatically:

- `/worktrees/` in project path → slot context, reads `.slot`
- `workspace/design/.epic` exists → single-repo context, reads `.epic`

Steps:

1. Close current child issue on GitHub: `gh issue close <N>`
2. Check off in `.epic`: `- [ ]` → `- [x]`, remove `<- active`
3. Add to `Covers:` in `.meta`
4. Move `<- active` to next issue, update `Current issue:` and `Current:`
5. If batch boundary → log "Batch N complete"
6. If epic complete → "All children done. Run work-end to close."
7. Report new active issue

## Router Fix (#103): Start vs Resume

**Bug:** `work/SKILL.md` Step 4 always shows "resume" as option 1 on a
feature branch, even when `HAS_HANDOFF=no`.

**Fix:** Change Step 4 option 1 to be conditional:

```
If HAS_HANDOFF=yes:
  1. resume — read the last handover and continue
If HAS_HANDOFF=no:
  1. start — invoke work-start (first session on this branch)
```

Add epic-specific option when `IS_EPIC=yes`:

```
  N. next — mark current child issue done, advance to next
```

## work_router.py Changes

Detect `.epic` alongside existing `.slot` detection:

```python
epic_candidate = workspace / "design" / ".epic"
if epic_candidate.exists():
    content = epic_candidate.read_text()
    if "Type: epic" in content:
        is_epic = True
        # parse batch/active issue (same logic as .slot)
        result["EPIC_PATH"] = str(epic_candidate)
```

New output key: `EPIC_PATH` (for single-repo). Existing `SLOT_PATH`
remains for slot context. `IS_EPIC`, `EPIC_BATCH`, `EPIC_ACTIVE_ISSUE`
work identically for both.

## epic_manager.py Generalisation

All functions accept explicit paths instead of deriving from `slot_dir`:

```python
# Before
def parse_batch_plan(slot_dir: Path) -> dict:
    slot_md = slot_dir / ".slot"

# After
def parse_batch_plan(epic_path: Path) -> dict:
    # epic_path is full path to .slot or .epic
```

`advance()` gets a `meta_path` parameter:

```python
def advance(epic_path: Path, meta_path: Path) -> dict:
```

Callers:
- Slot: `advance(slot_dir / ".slot", slot_dir / repo / "design" / ".meta")`
- Single-repo: `advance(wksp / "design" / ".epic", wksp / "design" / ".meta")`

`write_epic_slot_md()` → `write_epic_file()` with explicit output path.
New convenience: `write_epic()` for single-repo (no Repos section, no
slot number).

## work-start Epic Overlay

Generalise the overlay trigger:

```
Before: only fires when /worktrees/ in $PROJECT path
After:  also fires when workspace/design/.epic exists
```

Same display for both contexts — batch progress, active issue, commit
linkage (`Refs #<active-issue>`).

## work-end Epic Closure

When `.epic` exists:
- All batches complete → include epic issue in COVERS, close it
- Mid-epic exit → only close completed children, epic stays open
- `.epic` stays on disk for audit (consistent with `.slot` surviving in attic)

## Change Set

| File | Change |
|------|--------|
| `work/SKILL.md` | Add `work epic`, `work next` verbs; fix Step 4 (#103) |
| `work/work_router.py` | Detect `.epic`, output `EPIC_PATH` |
| `work-slot/epic_manager.py` | Generalise to accept paths; rename write function |
| `work-start/SKILL.md` | Generalise epic overlay trigger |
| `work-end/SKILL.md` | Epic-aware closure |
| `work-slot/SKILL.md` | Update `work-slot next` to pass explicit paths |
| `tests/test_epic_manager.py` | Update for path params; add `.epic` tests |
| `tests/test_work_router.py` | Add `.epic` detection tests |

## Not In Scope

- Batch planning UX changes (reuses existing grouping logic)
- Historical specs (left unchanged)
- `work-slot` creation flow (unchanged — still creates `.slot`)
