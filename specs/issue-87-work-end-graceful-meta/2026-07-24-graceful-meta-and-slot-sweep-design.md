# Design: work-end graceful .meta + slot-mode per-repo sweep

**Branch:** issue-87-work-end-graceful-meta
**Issues:** #87 (work-end graceful without .meta), #88 (slot-mode per-repo sweep)
**Date:** 2026-07-24

---

## Issue #87 — work-end graceful without .meta

### Problem

work-end hard-stops at pre-condition 2 when `$WORKSPACE/design/.meta` is absent.
This blocks closing branches that were created manually, created before the
scaffold system existed, or where `.meta` was accidentally deleted.

### Design

Replace the hard gate with a reconstruction block. No new scripts — the change
is entirely in work-end SKILL.md pre-conditions.

#### New pre-condition 2 flow

```
.meta exists?
  YES → read normally (no change to current behaviour)
  NO  → on a feature branch?
    NO (on main) → "Nothing to close — you're on main." Exit cleanly.
    YES → reconstruct context:
      1. Parse branch name for issue-NNN pattern
         - Found → gh issue view NNN --repo <project-github-repo>,
                    extract title and confirm with user
         - Not found → ask user for one-line work description,
                        invoke issue-workflow Phase 2 to create an issue
      2. Build in-memory context (not written to disk):
         - ISSUE_N = resolved issue number
         - ISSUE_REPO = project's GitHub repo (from CLAUDE.md)
         - COVERS = ISSUE_N (single issue only without .meta)
         - DESIGN_REPO = "project" (safe default)
         - FLYWAY_NEXT_V = "unknown"
         - design-section-hashes = "" (skip ARC42STORIES journal merge)
      3. Proceed with normal close flow using these values
```

#### What changes

- **work-end/SKILL.md** pre-condition 2 — replace hard gate with reconstruction block

#### What doesn't change

- Pre-condition 3 (orphaned `.meta` on main) stays as-is
- All downstream steps (close_artifacts.py, artifact promotion, stamping) unchanged
- `ctx.py` unchanged — it already handles missing `.meta` gracefully
- `close_artifacts.py` unchanged — receives values as CLI args regardless of source
- Slot-mode detection unchanged

---

## Issue #88 — slot-mode per-repo sweep and scan-workspace

### Problem

In multi-repo slots, the pre-close sweep (Step 3b) and artifact promotion
target only the primary repo. Secondary repos' CLAUDE.md, docs, protocols,
and workspace artifacts are invisible.

### Design

Two changes: a per-repo sweep loop in SKILL.md, and a `scan-workspace`
parameter on `close_artifacts.py`.

#### Per-repo sweep (work-end Step 3b in slot mode)

Slot mode is detected by `/worktrees/` in the `$PROJECT` path (existing logic).

**Normal mode (no change):** sweeps run against `$PROJECT` as today.

**Slot mode — new Step 3b-slot:**

```
1. Discover repos in the slot:
   - Primary repo = $PROJECT (the repo work-slot was invoked from)
   - Secondary repos = other repos checked out in the slot worktree

2. Per-repo loop (primary + secondaries):
   For each repo R:
     - update-claude-md against R's CLAUDE.md
     - implementation-doc-sync against R's docs/
     - protocol sweep against R's docs/protocols/
     - Batch commit all sweep changes for R (one commit per repo)

3. Session-bound (run once, not per-repo):
   - forage SWEEP → global (no change)
   - write-content (diary) → primary workspace blog/ only
```

#### close_artifacts.py scan-workspace parameter

New optional CLI parameter: `scan-workspace=<path>`

```bash
python3 close_artifacts.py <workspace> <project> <branch> \
  issue-repo=<repo> covers=<issues> \
  scan-workspace=<slot-workspace-path>
```

**When `scan-workspace` is provided:**
- Scan artifacts from the slot workspace at the given path
- Promote them to the original workspace (the `<workspace>` positional arg)
- Blog entries route to primary workspace only

**When omitted:** behaviour unchanged — scan `<workspace>` itself (current behaviour).

Called once per workspace in the slot. The loop lives in SKILL.md instructions,
not in the script — keeping `close_artifacts.py` composable and slot-agnostic.

#### What changes

- **work-end/SKILL.md** Step 3b — add slot-mode branch with per-repo sweep loop
- **work-end/close_artifacts.py** — add `scan-workspace` optional parameter;
  when present, scan artifacts from that path instead of the workspace arg
- **tests/** — new tests for `close_artifacts.py` covering `scan-workspace`

#### What doesn't change

- Normal-mode sweep (no slot) unchanged
- `close_artifacts.py` internal promotion logic unchanged — just a different source path
- Phase A/B split for slot mode unchanged
- `artifact_promote.py`, `blog_dest.py`, `branch_cleanup.py` unchanged

---

## Protocols

- **externalised-scripts-require-tests:** any changes to `close_artifacts.py` must
  ship with corresponding pytest tests in the same commit

## Testing

- `close_artifacts.py` with `scan-workspace` — happy path, missing path, empty workspace
- work-end SKILL.md changes are instruction-only (no script to unit test)
