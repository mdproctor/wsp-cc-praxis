# HANDOFF — soredium

**Last session:** 2026-08-03 (branch close: workspace branch guard + OWNER_REPO fix)
**Branch closed:** issue-168-workspace-branch-guard
**Issues closed:** #168, #169
**On main:** yes

## What was done

- #169 — ctx.py strips markdown bold markers before regex matching; OWNER_REPO now extracts correctly from `**GitHub repo:**` format
- #168 — work_router detects workspace on stale non-main branch (project on main, no .meta) and routes to `workspace_dirty` warning instead of silent resume
- Filed #170 — pre-merge hook to block slot-to-original merges without artifact promotion stamp

## What's Left

- Run `recover_orphaned_specs.py` to recover 19 specs stranded on closed workspace branches · M · Low
- Fix 3 casehub CLAUDE.md absolute blog paths: `platform`, `worker`, `pages` — change to `blog/` · S · Low (different repo)
- Untracked file in mdproctor.github.io: `_articles/2026-08-03-mdp01-teaching-agents-who-they-are.md` — commit or discard · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #170 | Pre-merge hook for slot artifact promotion stamp | M | Med | Prevents silent skip of close_artifacts.py |
| — | Recover orphaned specs from closed branches | M | Low | Script exists, needs execution |
| — | Commit or discard untracked article in mdproctor.github.io | XS | Low | Different repo |
