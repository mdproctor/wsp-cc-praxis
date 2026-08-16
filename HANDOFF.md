# HANDOFF — 2026-08-16

## Session Summary

Lifecycle hardening session. Started with merge conflict resolution across 6 casehub repos, expanded into a full audit and overhaul of the work lifecycle system (work-start, continue, next, end, pause, resume). Ended with workspace structural integrity fixes.

## What Was Done

### 1. Merge conflict resolution (6 casehub repos)

Resolved 245+ merge conflicts from the work-end landing bug (#239). Repos: platform, qhorus, workers, iot, neocortex, work. All compiled, tested, pushed.

### 2. Work-end issue closure verification (#239)

**Problem:** Claude said "done" but never ran `gh issue close`. GitHub issues stayed open, confusing the next session's work-end.

**Fix:** Mechanical gate — `close-issues` subcommand in `work_end_execute.py`, `check_issues_closed()` in `verify_slot_close.py`, Step 4b in work-end SKILL.md. Issues are now closed by script and verified before work-end completes.

### 3. .plan root migration

**Problem:** `.plan` lived at `design/.plan` inside the workspace. The `design/` indirection caused path errors — scaffold commits landing on wrong branches, scripts looking in wrong locations, multiple implementations of path resolution diverging.

**Fix:** `.plan`, `JOURNAL.md`, `.pause-stack`, and all scaffold files moved to workspace root. `migrate_to_root()` in `plan_migrate.py` runs once at session entry via `ctx.py`. All 43+ consumers updated. Old `design/` paths handled as fallback for pre-migration branches.

### 4. Multi-agent lifecycle audit (38 findings fixed)

Three audit passes (7 agents, 5 agents, 1 targeted agent) found and fixed:
- `pre_push_hook.py` only searched for `.meta` — lifecycle gates completely bypassed post-migration
- `branch_recon.py` read JOURNAL.md from wrong path
- `sync_main` left repos in mid-rebase state on failure
- Workspace merge/push/stamp failures in `cmd_land` were completely silent
- `.execute-progress`, `.artifacts-promoted`, `.land-ledger.jsonl` not cleaned up (stale state leak)
- `promote_deferred` issue number collision
- `advance()` crashed with ValueError on empty queue
- SKILL.md `checkout-main` argument order reversed vs code
- `branch_alignment` precondition missing from work-end
- Commit-after-write discipline added to brainstorming, writing-plans, write-content
- Deferred items: per-item selection with reason field and CLI

### 5. Workspace structural integrity

**Problem:** Engine workspace symlink pointed to `casehub/work/engine/` — a directory INSIDE the work project's git tree. Every workspace commit went into the wrong repo. This kept recurring because nothing validated workspace location.

**Root cause:** `create_symlinks.py` had zero validation. SKILL.md argument order was swapped. No marker file to verify workspace identity.

**Fix (four layers):**
- `.workspace` marker file — written by `workspace_create.py`, contains project path. Created automatically at session entry if missing.
- `validate_workspace_location()` — confirms workspace is an independent git root (not nested inside another repo's tree)
- `validate_workspace_marker()` — confirms `.workspace` points to the correct project
- `resolve_workspace()` / `ensure_workspace()` — derives canonical workspace path mechanically from project path. No LLM path computation needed.
- `create_symlinks.py` — hard stop if workspace is nested
- `ctx.py` — auto-repairs: detects nested workspace, finds correct one via `resolve_workspace`, switches to it

### 6. Engine + Work project cleanup

- Engine `wksp` symlink fixed: now points to `public/casehub/engine/` (was `casehub/work/engine/`)
- Stray `engine/` directory removed from work project repo (41 files, 15K lines of misplaced workspace artifacts)
- Artifacts migrated to correct workspace before removal: 10 blog entries, 8 spec dirs, 6 plans, HANDOFF.md
- `.workspace` markers added to both engine and work workspaces
- Stale scaffold files cleaned from both workspaces (`design/` dirs removed)
- Work project returned to main and pushed

### 7. Design document references requirement

All specs, plans, and decisions now require a `## References` section listing sources consulted (code files, garden entries, ADRs, protocols, external docs, GitHub issues).

---

## What's Left

### Slot DB reconciliation (ready to execute)

`reconcile_slots.py` audit found 40 divergences between the worklog DB and disk state:

| Category | Count | Risk | Action |
|----------|-------|------|--------|
| DB says active, disk is archived | 12 | low | Update DB state to archived |
| Disk-only (in attic, no DB record) | 16 | low | Backfill DB from disk |
| DB says archived, disk says active/landed | 3 | low | Update DB to match disk |
| Ghost (dir with no .slot, only .m2) | 1 | low | Quarantine |
| DB-only (no directory on disk) | 1 | low | Remove stale DB record |
| Active but ready to land | 1 | low | Update DB to ready |

**To execute:**
```bash
# Review first
python3 scripts/reconcile_slots.py /Users/mdproctor/claude/casehub --strategy

# Then apply
python3 scripts/reconcile_slots.py /Users/mdproctor/claude/casehub --execute
```

All actions are low risk — DB state updates and backfills, no destructive disk operations.

### Broader workspace audit

The engine/work workspace fix was specific to those two repos. Other casehub repos may have similar issues (workspace symlinks pointing inside other project repos). To check:

```bash
for repo in /Users/mdproctor/claude/casehub/*/; do
  name=$(basename "$repo")
  wksp="$repo/wksp"
  if [ -L "$wksp" ]; then
    target=$(readlink "$wksp")
    toplevel=$(git -C "$target" rev-parse --show-toplevel 2>/dev/null)
    resolved=$(cd "$target" 2>/dev/null && pwd)
    if [ "$toplevel" != "$resolved" ]; then
      echo "NESTED: $name wksp -> $target (inside $toplevel)"
    fi
  fi
done
```

Any `NESTED` output needs the same fix as engine: re-point `wksp` to `public/casehub/<repo>/`.

The new `ctx.py` auto-repair should catch and fix these at session entry, but a proactive sweep confirms no data is at risk.

### Artifact promotion audit

Verify all closed branches had artifacts correctly promoted. Approach:

1. List stamped branches: `git branch --list | xargs -I{} git log -1 --format="%s" {} | grep "chore: branch closed"`
2. For each, check `docs/blog/` and `docs/specs/` for promoted content
3. Cross-reference workspace blog/specs for unpromoted content

### Remaining open bugs

- **#236** — Lifecycle `validate_state`: add `.claude/` to `_DEFAULT_EXCLUDES` and fix symlink matching. XS/Low, directly related to this session's work.

---

## Key Design Decisions

1. **Migrate-at-entry, not per-file fallback.** `migrate_to_root()` runs once in `ctx.py`. All downstream code reads from root only.

2. **`.workspace` marker for identity.** Machine-readable file: "I am a workspace for project X." Auto-created on first session entry. Prevents wrong-workspace-for-project mismatches.

3. **`resolve_workspace()` derives paths mechanically.** Checks `wksp/` symlink validity → `~/claude/public/<parent>/<project>/` → `~/claude/private/<parent>/<project>/`. No LLM path computation.

4. **Commit is the save.** Brainstorming, writing-plans, write-content all require immediate git commit after writing.

5. **Close-issues is mechanical.** `work_end_execute.py close-issues` + `verify_slot_close.py check_issues_closed`. Not LLM-dependent.

## Preventive Measures Added

| What | Where | Prevents |
|------|-------|----------|
| `.workspace` marker | `workspace_create.py`, `ctx.py` | Wrong workspace for project |
| `validate_workspace_location()` | `workspace_create.py`, `ctx.py`, `commit_scaffold`, `create_symlinks.py` | Nested workspace (commits to wrong repo) |
| `resolve_workspace()` | `workspace_create.py`, `ctx.py` | LLM computing wrong workspace path |
| `migrate_to_root()` | `plan_migrate.py`, called by `ctx.py` | Stale `design/` scaffold files |
| `check_issues_closed()` | `verify_slot_close.py` | Issues reported done but not closed |
| `check_branch_alignment()` | `work_end_context.py` | Workspace/project branch mismatch |
| `check_workspace_integrity()` | `work_health.py` | Nested or unmarked workspace at session entry |
| `check_stale_scaffold_on_main()` | `work_health.py` | Scaffold files polluting workspace main |
| Branch assertion in `commit_scaffold` | `branch_create.py` | Scaffold committed to wrong branch |
| Rebase `--abort` on failure | `branch_create.py sync_main` | Repo left in mid-rebase state |
| Workspace error reporting | `work_end_execute.py cmd_land` | Silent workspace merge/push/stamp failures |

## Branch State

- **soredium:** main, pushed. 509 tests pass. #239 closed.
- **casehub/engine:** `issue-510-case-level-sla-timer` branch, 7 implementation commits. Workspace correct (`public/casehub/engine/`), on main, clean.
- **casehub/work:** main, pushed. Stray engine dir removed.
