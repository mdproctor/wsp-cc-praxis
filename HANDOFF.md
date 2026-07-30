# Handover — 2026-07-30

**Previous handover:** `git show HEAD~1:HANDOFF.md` | diff: `git diff HEAD~1 HEAD -- HANDOFF.md`

## What Changed This Session

- #119: `git clone --shared` replaces `git worktree add` for all slot repos. Eliminates the class of bugs where lifecycle operations fail because worktrees share refs and can't checkout main. Auto-migration for existing worktree slots via `ensure_clone_layout()`.
- #119: Fixed silent worklog event loss — path normalization bug caused zero work-end/pause/merge events to ever be recorded. macOS `/tmp` → `/private/tmp` symlink caused path mismatch in SQLite lookups.
- #119: Fixed ctx.py broken symlink resolution — `Path.exists()` returns False for broken symlinks, dropping workspace discovery. Walk-up to nearest git root handles partial symlink targets.
- #119: Per-project HANDOFF files — `HANDOFF-{project_name}.md` in shared workspaces. Router checks project-specific file first, falls back to `HANDOFF.md`. Fixes engine sessions finding eidos handovers.
- #116: Slot archival completion gate in work-end Phase B.
- #115: Filed — blog publish gate via pre-commit hook with fresh Claude context.
- #118: Filed — evaluate splitting HANDOFF.md roles (history, state, planning).
- Closed #75 (all children done) and #80 (replaced by #115).

## State Right Now

On branch `issue-119-slot-clone-shared`. 6 commits, not yet merged to main. Two open slots on hortora (slot 1: #117 work UI, slot 2: #120 trellis).

## Immediate Next Step

Run `work end` to merge #119 to main. The handover skill needs updating to write `HANDOFF-{project_name}.md` — the router and commit script support it but the skill still hardcodes `HANDOFF.md`. work-end's per-repo evidence checking and worklog slot-merge event recording are remaining #119 work.

## What's Left

- Handover skill: update to write `HANDOFF-{project_name}.md` using PROJECT_NAME from ctx.py · S · Low
- work-end: per-repo evidence gate (mechanical verification all repos swept) · M · Med
- work-end: record slot-merge event in worklog when closing via Phase B · S · Low
- work-end: unify as single entry point for slots (iterate all repos, not just primary) · M · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #119 | Remaining: evidence gate, worklog, work-end unification | M | Med | Branch open |
| #117 | UI for work lifecycle management | M | Med | Slot 1 open |
| #110 | Support nested epics | M | High | — |
| #95 | Mechanize LLM-executed state-changing operations | L | Med | — |
| #92 | Add restore-slot command | M | Med | — |

## References

*Unchanged — `git show HEAD~1:HANDOFF.md`*
