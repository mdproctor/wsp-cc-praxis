# Slot DB/Disk Integrity Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #195 — Slot numbering and DB/disk integrity
**Issue group:** #195

**Goal:** Make the worklog DB authoritative for slot numbering, add inline drift detection on list, and provide a repeatable reconciliation tool.

**Architecture:** Reserve-first pattern for slot creation (DB row inserted before directory), single-transaction confirmation after disk operations, inline drift check on list_slots comparing disk vs DB.

**Tech Stack:** Python 3, SQLite (worklog.db), pytest

## Global Constraints

- All paths stored in DB must be normalized via `Path.resolve()`
- `@safe` stays on all worklog functions EXCEPT slot creation critical path
- Quarantine directory (`slots/quarantine/`) excluded from all automation
- Path resolution fix already implemented and tested

---

### Task 1: Worklog DB — reserve and confirm functions

**Files:**
- Modify: `scripts/worklog.py`
- Create: `tests/test_worklog.py`

**Interfaces:**
- Produces: `reserve_slot_number(conn, family_root) -> int`
- Produces: `confirm_slot_create(conn, slot_number, family_root, repos, branch, issue_number, issue_repo, covers=None) -> int`
- Produces: `_ensure_repo_strict(conn, path, family_root=None) -> int`

TDD: write tests for each function, implement, verify. See spec section 1-2 for signatures. Commit after all three functions pass.

### Task 2: Slot manager — DB-authoritative numbering and reserve-first creation

**Files:**
- Modify: `work-slot/slot_manager.py`
- Modify: `tests/test_slot_manager.py`

**Depends on:** Task 1

**Interfaces:**
- Consumes: `worklog.reserve_slot_number`, `worklog.confirm_slot_create`

Rewrite `allocate_slot_number()` to call `reserve_slot_number`. Update `create_slot()` to use reserve-first + confirm. Fix existing tests that assume disk-based allocation. Commit.

### Task 3: Inline drift detection on `list_slots`

**Files:**
- Modify: `work-slot/slot_manager.py`
- Modify: `tests/test_slot_manager.py`

**Interfaces:**
- Produces: `WARN=db_drift` lines on stdout

Add drift check at end of `list_slots()`. Test each divergence type. Commit.

### Task 4: Reconciliation script rewrite

**Files:**
- Modify: `scripts/reconcile_slots.py`
- Create: `tests/test_reconcile_slots.py`

Three functions: `audit()`, `strategy()`, `execute()`. CLI phases via `--strategy` and `--execute` flags. Test each phase independently. Quarantine moves to `slots/quarantine/N/`. Commit.

### Task 5: Integration verification

Run full test suite, commit-tier validators, verify path resolution tests. Fix any breakage from Task 2's `_wl` requirement change. Commit fixups.
