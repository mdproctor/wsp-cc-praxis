# HANDOFF — soredium

**Last session:** 2026-08-03 (ctx.py worktree fix + slots rename)
**Branches closed:** issue-156-ctx-worktree-symlink-fix, issue-162-rename-worktrees-to-slots
**On main:** yes

## What was done

### #156 — ctx.py worktree symlink fix (CLOSED)
- ctx.py resolves proj/wksp symlinks from the main working tree when running in a git worktree
- Uses `git worktree list --porcelain` to find the main root
- New output fields: IN_WORKTREE, MAIN_WORKTREE_ROOT
- WORKSPACE_OK fast-path also checks main root
- 7 new tests, 95 total pass

### #162 — Rename slot directory worktrees/ → slots/ (CLOSED)
- New slots create under `slots/`. Existing `worktrees/` slots continue working via dual-path reads
- `is_slot_path()` replaces `"/worktrees/" in path` — excludes `.claude/worktrees/` and `.worktrees/`
- Helpers: `_resolve_slots_dir()`, `_resolve_slot_dir_for_number()`
- Updated slot_manager, work_router, pause_exec, ctx.py, 5 SKILL.md files
- 15 new tests, 355 total pass

## What is NOT done
- 9 failing tests in `test_design_review.py` from #159's `depth` → `degree` rename — import error
- 2 pre-existing failures in `test_close_report.py` — rendered output format mismatch (assertions expect old format)
- #161 — ctx.py finds wrong .meta in Claude Code worktrees (S / Low) — unblocked by #162
