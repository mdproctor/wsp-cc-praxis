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

## D3: Duplicate branch guard — detection mechanism

**Choice:** Disk scan of active slots — iterate `slots/*/.slot`, parse branch name from the `# Slot N — <branch>` header, refuse creation if a match exists. Error message: "Slot N already has branch `<name>`. Use that slot or archive it first."
**Alternatives:**
- DB join via `work_items` table — branch exists in `work_items.branch`, no schema change needed. But the branch isn't recorded until `confirm_slot_create`, so the guard misses the exact retry-after-failure scenario we care about most.
- Add `branch` column to `slots` table — requires schema migration for a check that disk scan handles natively.
**Rationale:** `.slot` is the source of truth for what branch a slot owns. Successfully created slots always have one; failed creations don't — so the guard catches real duplicates with zero false positives from debris. Disk scan is a handful of directories; performance is negligible.
**Trade-offs:** Disk scan doesn't see slots whose `.slot` was manually deleted. That's a corruption scenario already covered by `reconcile_slots.py`, not a guard concern.
**Sources:** `slot_manager.py:574-596` (write_slot_md), `slot_manager.py:987-1042` (parse_slot_md)
**Exploration:** quick
**Status:** captured
