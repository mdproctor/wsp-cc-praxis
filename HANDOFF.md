# HANDOFF — soredium

**Last session:** 2026-08-03 (worktree fix + slots rename + blog escape fix)
**Branches closed:** issue-156-ctx-worktree-symlink-fix, issue-162-rename-worktrees-to-slots, issue-163-blog-slot-escape
**On main:** yes

## What was done

### #156 — ctx.py worktree symlink fix (CLOSED)
- ctx.py resolves proj/wksp symlinks from the main working tree when running in a git worktree
- Uses `git worktree list --porcelain` to find the main root
- New output fields: IN_WORKTREE, MAIN_WORKTREE_ROOT

### #162 — Rename slot directory worktrees/ → slots/ (CLOSED)
- New slots create under `slots/`. Existing `worktrees/` slots continue working via dual-path reads
- `is_slot_path()` replaces `"/worktrees/" in path` — excludes `.claude/worktrees/` and `.worktrees/`
- Updated slot_manager, work_router, pause_exec, ctx.py, 5 SKILL.md files

### #163 — Blog entries escaping slot boundary (CLOSED)
- Root cause: absolute `Blog directory:` paths in CLAUDE.md escape the slot clone
- `resolve_blog_dir.py` detects slot escape, falls back to `$WORKSPACE/blog/`
- `safe_commit.py` provides reusable branch-guarded commit-to-main helper
- Branch guards wired into blog_publish.py and close_artifacts.py

## What is NOT done
- 9 failing tests in `test_design_review.py` from #159's `depth` → `degree` rename — import error
- 2 pre-existing failures in `test_close_report.py` — rendered output format mismatch
- #161 — ctx.py finds wrong .meta in Claude Code worktrees (S / Low) — unblocked by #162
- Fix 3 casehub CLAUDE.md absolute blog paths: `platform`, `worker`, `pages` — change to `blog/` (different repo, do in casehub sessions)
- Untracked file in mdproctor.github.io: `_articles/2026-08-03-mdp01-teaching-agents-who-they-are.md` — commit or discard
