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

## D3: Rebase failure strategy

**Choice:** Keep rebase, hard stop on failure, report to user for guidance, record in worklog DB
**Alternatives:**
- Auto-fallback to merge for long-lived branches — introduces merge commit complexity, non-linear history, complicates squash; solves an ergonomic problem not a safety problem
- Auto-retry with --onto — complex to wire, fragile heuristic for identifying already-landed commits
**Rationale:** D1+D2 prevent silent data loss. Rebase failure is now safely caught. The right response is to stop and let the user decide — they know the branch context and can choose merge, cherry-pick, or manual resolution. Recording in worklog DB enables fleet-wide monitoring of rebase failures across repos and slots.
**Trade-offs:** User must manually handle rebase conflicts on long-lived branches — acceptable given the rarity and D1+D2 safety nets
**Depends on:** D1 (land postcondition catches content gap after any landing strategy)
**Sources:** Slot 140 incident (849 commits, rebase doomed), worklog DB schema
**Exploration:** quick
**Status:** captured

## D4: Worklog DB rebase failure recording

**Choice:** Record rebase failures as a worklog event with repo, branch, commit counts, and error detail
**Alternatives:**
- Log only to stdout (current behaviour) — invisible to monitoring, lost after session ends
**Rationale:** Fleet-wide visibility. When multiple slots/repos have rebase failures, the DB lets you query what's stuck, how long it's been stuck, and what the error was — without manually checking each repo
**Trade-offs:** Adds a worklog schema event type; minimal cost
**Depends on:** D3 (rebase failure is now a first-class event, not a silent continue)
**Sources:** ~/.claude/lib/worklog.py (existing event recording pattern), work-end/land_flow.py (existing _record_worklog_end)
**Exploration:** quick
**Status:** captured
