# HANDOFF — soredium

**Last session:** 2026-08-03 (ctx.py worktree fix)
**Branches closed:** issue-156-ctx-worktree-symlink-fix
**On main:** yes

## What was done

### #156 — ctx.py worktree symlink fix (CLOSED)
- ctx.py now resolves `proj/`/`wksp/` symlinks from the main working tree when running in a git worktree
- Uses `git worktree list --porcelain` (chosen over `--git-common-dir` to avoid `--separate-git-dir` assumption)
- New output fields: `IN_WORKTREE`, `MAIN_WORKTREE_ROOT`
- `WORKSPACE_OK` fast-path also checks main root in worktree context
- 7 new tests using real `git worktree add` in pytest, 95 total pass
- Also split a prior session's bundled commit (garden-retriever + ctx.py) into two clean commits via force-push-with-lease

### Prior session cleanup
- Garden-retriever subagent changes (8 skill files) separated from #156 fix into own commit
- 9 failing `test_design_review.py` tests noted — import error from #159 `depth` → `degree` rename (not this branch's scope)

## What is NOT done
- 9 failing tests in `test_design_review.py` from #159's `depth` → `degree` rename — import error
- #161 — ctx.py finds wrong .meta in Claude Code worktrees — workspace epic .meta bleeds into worktree context (S / Low)
</content>
</invoke>