# Decisions — #291 Duplicate Slot Guard

## D1: Rollback strategy for partial slot creation failures

**Choice:** Internal rollback — wrap the risky section of `create_slot()` in try/except, clean up directory and DB `pending` row before re-raising
**Alternatives:**
- External cleanup command — adds a `cleanup-failed-slot` subcommand callable from SKILL.md or reconciliation; more complex, problem should be prevented at source
**Rationale:** The failure should be cleaned up where it happens. `reconcile_slots.py` already handles orphaned state after the fact for edge cases that slip through.
**Trade-offs:** If the process is killed (SIGKILL, power loss), internal cleanup won't run — but that's an edge case `reconcile_slots.py` already covers.
**Sources:** `work-slot/slot_manager.py:599-747` (create_slot), `scripts/reconcile_slots.py`
**Exploration:** quick
**Status:** captured

## D2: Pending slot number reuse on retry

**Choice:** Auto-reuse — if the most recent DB entry for this family_root is `state='pending'`, reuse that number instead of allocating a new one. Before reusing, verify the pending slot's directory is empty or absent; if it has partial clones from the failed attempt, remove the directory first.
**Alternatives:**
- Warn and ask — push the decision to the SKILL.md layer with a user prompt; adds friction for a case where the user has already decided to retry
- Always allocate new — wastes numbers, leaves orphan pending rows accumulating in the DB
**Rationale:** The pending row is definitionally abandoned: allocate succeeded, confirm never ran, so no work exists on that number. Retrying is the user's explicit intent. Cleaning the directory before reuse prevents stale .m2/ or half-cloned repos from the old attempt contaminating the new slot.
**Trade-offs:** If two concurrent sessions create slots for the same family_root, both could see the same pending row. Mitigated by the fact that slot creation is single-session and the DB write is atomic.
**Sources:** `worklog.py:220-235` (reserve_slot_number), `slot_manager.py:331-343` (allocate_slot_number)
**Exploration:** quick
**Depends on:** D1 (rollback creates the pending orphans that D2 cleans up on retry)
**Status:** captured
