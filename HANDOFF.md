# HANDOFF — soredium

## Last Session

Two tracks. First: #195 (slot DB/disk integrity) — full implementation from brainstorm through work-end. DB-authoritative slot numbering with reserve-first pattern, inline drift detection on list_slots, three-phase reconciliation script. Second: discovered and fixed HANDOFF.md lifecycle bug — handoffs were committed to workspace main instead of the workspace branch, breaking pause/resume isolation. Fixed across 8 files (skills, scripts, tests). Audit confirmed branch-scoped model works. Also fixed: issue-matching gate silently dropping valid handoffs, brief not surfacing .notes, executing-plans routing to work-end instead of work-next when plan queue has remaining issues.

Then discovered #237: `create_slot` silently fails to wire `wksp/` symlinks when workspace subdirectories are gitignored. The workspace's `.gitignore` lists child repo dirs (e.g., `/work`), so `git clone --shared` doesn't create them. The `wksp/` symlink in the repo clone points to nothing. Sessions in affected slots run as `SINGLE_REPO=yes` — handoffs go to wrong place, workspace invisible. Fixed 5 slots manually. Branch created for the code fix.

## Immediate Next Step

Branch `issue-237-slot-symlink-validation` is open. Brainstorm then implement:

1. **`create_slot`**: after cloning workspace, create gitignored subdirectories (`mkdir`) and un-ignore them before creating `wksp/` symlinks
2. **`add_repo`**: same fix — create workspace subdir if missing
3. **Post-creation validation**: verify all repo clones have working `wksp/` symlinks. Fail fast with clear error if not.
4. **Naming collision**: workspace clone name must not collide with any repo name in the family (check all family repos, not just the current slot's repos list)

Verification command in issue #237 body.
