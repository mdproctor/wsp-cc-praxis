# Slot DB/Disk Integrity — Design Spec

**Issue:** #195 — Slot numbering and DB/disk integrity
**Branch:** issue-195-slot-db-disk-integrity
**Date:** 2026-08-13

## Problem

Slot numbering is derived from disk scans (`allocate_slot_number()` reads
`slots/` and `slots/attic/` for `max + 1`). This has caused:

- **Numbering collisions:** 24 casehub slots exist in both `slots/N/` and
  `slots/attic/N/` — ghost remnant directories at the active path inflated
  the disk scan, then when those ghosts became invisible to a fresh scan,
  the counter reset and reused numbers already in the attic.
- **DB/disk state drift:** DB says active but disk is in attic (and vice
  versa). Gaps in the DB sequence (slots 52, 58, 66 never recorded).
- **Silent write failures:** `@safe` decorator on `record_slot_create()`
  swallows DB write errors, so the worklog silently misses slot creations.

No work has been lost — the ghost directories contain no `.slot` files and
the attic copies have the real data. But the bookkeeping is unreliable.

## Solution Overview

Four changes:

1. **DB-authoritative numbering** — `allocate_slot_number()` queries
   `MAX(slot_number) + 1` from the worklog DB instead of scanning disk.
   Hard fail if the DB is unavailable (D3).

2. **Authoritative writes for slot creation** — `record_slot_create()`
   does not use `@safe`. If the DB write fails, slot creation fails.
   Other worklog functions remain informational/best-effort (D4).

3. **Inline drift detection on list** — `list_slots()` cross-references
   disk state against DB state and emits `WARN=db_drift` lines for
   mismatches. Zero-cost since list already iterates all slots.

4. **Repeatable reconciliation** — enhanced `reconcile_slots.py` with
   three phases: audit (report with taxonomy), strategy (propose actions),
   execute (apply approved actions with quarantine).

Plus the already-implemented fix from this session: `relocate_claude_projects()`
and `remove_claude_projects()` now resolve paths to absolute before encoding,
fixing the bug where conversation directories silently failed to move during
archival when the script was invoked with a relative family root.

## Detailed Design

### 1. `allocate_slot_number()` — DB as authority

Current (disk scan):
```python
def allocate_slot_number(family_root: Path) -> int:
    existing = []
    # scan slots/ and slots/attic/ for max
    return max(existing, default=0) + 1
```

New (DB query with hard fail):
```python
def allocate_slot_number(family_root: Path) -> int:
    if _wl is None:
        print("ERROR=worklog_unavailable")
        print("ERROR_DETAIL=worklog module required for slot numbering")
        sys.exit(1)
    conn = _wl.connect()
    row = conn.execute(
        "SELECT MAX(slot_number) FROM slots WHERE family_root=?",
        (_wl._norm(str(family_root)),),
    ).fetchone()
    conn.close()
    db_max = row[0] if row[0] is not None else 0
    return db_max + 1
```

No fallback to disk scan. If the worklog module is not importable or the
DB connection fails, slot creation is blocked with a clear error message.

### 2. Authoritative writes for slot creation

`create_slot()` currently wraps the DB write in try/except (lines 638-650):
```python
if _wl:
    try:
        _conn = _wl.connect()
        _wl.record_slot_create(...)
        _conn.close()
    except Exception:
        pass
```

New: the DB write is mandatory and errors propagate:
```python
conn = _wl.connect()
_wl.record_slot_create(conn, slot_num, ...)
conn.close()
```

`record_slot_create()` itself must not use `@safe`. Add a new function
`record_slot_create_strict()` that does NOT use the decorator, or remove
`@safe` from `record_slot_create()` and handle the error explicitly in
callers that need best-effort behavior.

**Recommended approach:** remove `@safe` from `record_slot_create()`. The
only caller is `create_slot()`, which now requires it to succeed. No other
caller needs best-effort behavior for this function.

Other worklog functions (`record_slot_archive`, `record_slot_merge`,
`record_work_start`, etc.) keep `@safe` — they are informational and must
not block operations.

### 3. Inline drift detection on `list_slots()`

After building the slot list from disk, query the DB:
```python
if _wl:
    conn = _wl.connect()
    db_slots = _wl.slot_status(conn, family_root=str(family_root))
    conn.close()
    # Compare and emit warnings
```

Divergence types detected (from D2 taxonomy):

| Type | Condition | Warning |
|------|-----------|---------|
| `db-only` | DB record, no disk directory | `WARN=db_drift type=db-only slot=N` |
| `disk-only` | Disk directory, no DB record | `WARN=db_drift type=disk-only slot=N` |
| `state-mismatch` | DB state != disk state | `WARN=db_drift type=state-mismatch slot=N db=X disk=Y` |
| `ghost` | Dir with no `.slot` file | `WARN=db_drift type=ghost slot=N` |

Drift detection is non-blocking — warnings only, never errors. The user
runs `reconcile_slots.py` to fix detected drift.

### 4. Repeatable reconciliation (`reconcile_slots.py`)

Enhance the existing script with three explicit phases.

**Phase 1 — Audit:**

Scan all four disk locations (`slots/`, `slots/attic/`, `worktrees/`,
`worktrees/attic/`) and the DB. For each slot number, report:

```
SLOT 3 (casehub):
  disk:  slots/3/ — NO .slot file, contains: [aml/]
  disk:  slots/attic/3/ — .slot=yes .landed=no
  db:    state=archived, created=2026-08-03, archived=2026-08-04
  class: ghost (active dir has no .slot, attic has real data)
```

Classify each divergence per the taxonomy in D2.

**Phase 2 — Strategy:**

For each divergence, propose an action:

| Class | Proposed action |
|-------|-----------------|
| `ghost` | Quarantine to `slots/quarantine/N/` |
| `db-only` | Remove DB record (slot doesn't exist on disk) |
| `disk-only` | Backfill DB record from `.slot` file metadata |
| `state-mismatch` | Update DB state to match disk reality |
| `collision` | If both have `.slot`: renumber the active one (allocate new number from DB). If only attic has `.slot`: treat active as ghost |

Present the full plan and wait for user approval.

**Phase 3 — Execute:**

Apply approved actions. Log each action. Ghost directories go to
`slots/quarantine/N/` (not deleted) so they can be inspected or
recovered.

### 5. Already implemented: path resolution fix

`relocate_claude_projects()` and `remove_claude_projects()` now call
`.resolve()` on `slot_dir` and `dest_dir` before encoding to match
Claude Code's absolute-path-based project directory names. This fixes
the bug where conversations silently failed to move during archival.

Regression tests added in `TestRelocateClaudeProjectsRelativePath`.

## Files Changed

| File | Change |
|------|--------|
| `work-slot/slot_manager.py` | `allocate_slot_number()` → DB query; `create_slot()` → mandatory DB write; `list_slots()` → inline drift check; `relocate_claude_projects()` / `remove_claude_projects()` → `.resolve()` (done) |
| `scripts/worklog.py` | Remove `@safe` from `record_slot_create()` |
| `scripts/reconcile_slots.py` | Three-phase audit/strategy/execute with taxonomy |
| `tests/test_slot_manager.py` | Tests for DB-authoritative numbering, hard fail, drift detection, path resolution (partially done) |
| `tests/test_reconcile_slots.py` | Tests for enhanced reconciliation phases |

## Testing Strategy

- **`allocate_slot_number()`**: test DB query returns max+1; test hard
  fail when `_wl` is None; test hard fail when DB connection fails
- **`create_slot()`**: test that DB write failure prevents slot creation;
  test that slot directory is not created if DB write fails
- **`list_slots()` drift detection**: test each divergence type emits
  correct warning; test no warnings when aligned
- **`reconcile_slots.py`**: test audit output for each divergence class;
  test quarantine rather than delete; test DB backfill from disk
- **Path resolution**: done — `TestRelocateClaudeProjectsRelativePath`

## Acceptance Criteria (from #195)

- [x] Slot numbering uses DB as authority, not disk scan
- [ ] Archive operation updates DB and disk atomically (or reports failure) — already implemented: `archive_slot()` and `remove_slot()` update both; `@safe` on archive recording is acceptable since the disk move is the authoritative action and drift detection catches any DB lag
- [ ] `work-slot list` flags DB/disk divergences when detected
- [ ] No gaps in slot numbering for new slots going forward
- [ ] Existing divergences documented (reconciliation audit)
