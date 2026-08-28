## D1: Land postcondition location

**Choice:** Inside `land_flow.py:land_batch()`, after merge+push, before stamp
**Alternatives:**
- verify_fn on orchestrator's land step — lifecycle transitions fire before verify catches the problem
**Rationale:** Catches failure at the earliest point, before stamp, before lifecycle transitions, before the orchestrator can fall through
**Trade-offs:** land_flow.py gets slightly more complex (~10 lines)
**Sources:** work-end/land_flow.py:548-683, work-end/verify_stamp.py, work-end/work_end_orchestrator.py:831-840
**Exploration:** quick
**Status:** captured

## D2: Pre-archive gate location

**Choice:** Content diff check in `archive_slot()` itself, as a hard gate — no `--force` override
**Alternatives:**
- verify_slot_close.py only — bypassed by direct CLI calls, and already failed to catch slot 140
- Both archive_slot and verify_slot_close — verify already failed; fix is separate concern
**Rationale:** Last line of defence — every path to archiving goes through archive_slot(). Cheap check (one git diff --stat per clone), can't be bypassed
**Trade-offs:** Adds ~15 lines to archive_slot which is already 108 lines (but this is a safety gate, not complexity)
**Sources:** work-slot/slot_manager.py:1636-1741 (archive_slot), slot 140 incident (verify=done but 377 files unmerged)
**Exploration:** quick
**Status:** captured
