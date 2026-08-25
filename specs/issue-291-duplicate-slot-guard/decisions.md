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
