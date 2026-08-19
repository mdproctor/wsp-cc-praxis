# Slot Workspace Convergence

## Problem

Slots clone a single family workspace repo (`work-casehub/`) and nest
per-repo artifacts as subdirectories. Outside slots, workspace-init
creates separate git repos per project (`wsp-casehub-connectors/`,
`wsp-casehub-pages/`, etc.) and each project's `wksp` symlink points
directly to its own workspace repo.

This mismatch forces every skill that touches workspace paths to branch
on "am I in a slot?" — work-end, work-start, topology detection, symlink
resolution, promotion, squash, merge. The divergence is the root cause
of slot-specific bugs and unreliable behaviour.

### Symptoms

1. **Ambiguous artifact ownership.** `work-casehub/specs/` and
   `work-casehub/connectors/specs/` both exist. When connectors is
   primary, it's unclear which holds connectors' specs.
2. **12 divergence points** between slot and non-slot code paths in
   work-end alone (audit in decisions.md).
3. **Dead weight.** The family workspace clone carries subdirectories for
   every project in the family, including repos not in the slot.
4. **Detection fragility.** `is_project_repo()` uses name-based
   exclusion (`work-*` prefix) to filter workspace clones. Renaming
   breaks detection.

## Target State

A slot mirrors the same structure workspace-init creates outside slots.
Each project repo in the slot has a paired workspace clone, named by its
actual git repo name, flat in the slot directory.

### Layout

```
slots/N/
├── connectors/              ← git clone --shared of casehub/connectors
├── wsp-casehub-connectors/  ← git clone --shared of public/casehub/connectors
├── pages/                   ← git clone --shared of casehub/pages
├── wsp-casehub-pages/       ← git clone --shared of public/casehub/pages
├── .slot
├── .plan
└── .m2/
```

### Symlinks

Same shape as outside slots:

- `connectors/wksp` → `../wsp-casehub-connectors/`
- `wsp-casehub-connectors/proj` → `../connectors/`

A skill following `wksp` from any project clone — in or out of a slot —
sees an identical workspace directory. No detection needed.

### Artifact Placement

- **Repo-specific** artifacts go in that repo's workspace.
- **Cross-cutting** artifacts go in the primary repo's workspace, tagged
  if the file format supports tagging (e.g., frontmatter `repos:` field).
- No duplication. One file, one place.

### Work-end Flow

Same per-repo logical flow regardless of topology:

1. **Promote** artifacts within each workspace clone
2. **Squash** each repo (project + workspace) on its branch
3. **Merge** each workspace clone → its original repo
4. **Merge** each project clone → its original repo
5. **Push** each original to remote

The only topology-specific concern is **transport**: slots use a two-hop
push (clone → original → remote) while non-slots push directly. This is
encapsulated as a transport helper called from the shared flow, not a
separate orchestration path.

**Current gap:** `merge_slot` (`slot_manager.py:1163-1458`) processes ONLY
project repos via `get_slot_repos()` and explicitly skips workspace repos
(printed as `SKIPPED_WORKSPACE`). Workspace branch merging, pushing, and
stamping exists only in branch mode via `cmd_land`
(`work_end_execute.py:373-413`). The `work-end/SKILL.md` (line 402-403)
incorrectly claims `merge_slot` stamps workspace repos via
`get_all_slot_repos()` — the code uses `get_slot_repos()` (project only).

Converging concerns (shared code — requires adding workspace handling
to the slot path, not just refactoring what exists):
- Workspace merge, push, and stamp (currently branch-mode only)
- Artifact promotion
- Squash analysis and execution
- Branch stamping (project repos: both paths; workspace repos: branch-mode only)

Topology-specific transport (helper functions):
- Original-repo preflight sync (`slot_manager.py:1211-1277`)
- Two-hop push clone→original→remote (`slot_manager.py:1336-1386`)
- `updateInstead` configuration (`slot_manager.py:951`)
- SHA verification via `.landed` markers (`slot_manager.py:1402-1408`)

### Convergence Architecture

The converged flow adopts `merge_slot`'s batch orchestration model —
preflight all repos before touching any, then rebase all, then
merge+push per repo — extended to include workspace repos:

1. **Enumerate:** `get_all_slot_repos()` returns project + workspace repos.
   Partition into project repos and workspace repos via `is_workspace_clone()`.
2. **Preflight:** Check all originals (project AND workspace) for clean state,
   sync their `main` with remote. Fail fast if any repo is dirty or diverged.
3. **Rebase:** Rebase all feature branches (project AND workspace) onto
   updated `main`. Retry up to 3 times. Abort all on persistent conflict.
4. **Merge + push (per repo):** For each repo (workspace first, then project):
   merge feature branch into `main`, push via topology-appropriate transport.
   Collect results. Fail if any local push fails.
5. **Stamp:** Stamp all feature branches (project AND workspace) with
   branch-closed commit containing landed SHA.

`cmd_land` becomes a thin adapter: construct a single-repo batch (one
project + its workspace) and call the shared flow. The transport helpers
detect topology (slot two-hop vs direct push) and are called from the
shared flow — they don't define a separate orchestration path.

**Error handling:** The shared flow uses `merge_slot`'s coordinated model.
All push results are collected; if any local push fails, the full status
is reported. Per-repo rollback (reset to pre-merge SHA) applies to each
repo independently but the batch reports holistically.

## Detection

### Current (fragile)

`is_project_repo()` (`slot_manager.py:872-877`) uses name-based
exclusion — rejects anything matching `work-*`, `.m2`, or `attic`.
`is_workspace_clone()` (`slot_manager.py:880-894`) checks three
signals: `.workspace` marker, `proj` symlink, naming convention.

Name-based detection breaks when workspaces are renamed (the whole
point of this change).

### Target (structural)

`.workspace` marker file is the primary detection signal. Placed by:
- `create_slot()` during workspace clone creation
- Migration script during Phase 1 (existing slots)
- `workspace-init` during workspace repo creation — specifically, after
  `git init` or `git clone` of the workspace repo, before symlink setup.
  Change in `workspace-init/SKILL.md`: add `touch .workspace` to the
  repo creation step and commit it as part of the initial workspace setup.
  Change in `workspace-init/create_symlinks.py`: if the workspace repo
  already exists but lacks `.workspace`, place the marker (idempotent
  catch-up for repos created before this change).

`is_workspace_clone()` checks `.workspace` marker only. Name-based
and `proj`-symlink detection removed after marker rollout.

`is_project_repo()` becomes: is a git repo AND NOT `is_workspace_clone()`.
No hardcoded name exclusions.

## Slot Creation Changes

### Current (`create_slot` at `slot_manager.py:547-711`)

1. For each project repo in the slot:
   a. Clone the project repo with `--shared`
   b. Call `resolve_workspace_source(repo_path)` to find the workspace
      git repo from the project's `wksp` symlink
   c. Deduplicate via `ws_created` dict keyed by workspace source path —
      if multiple projects resolve to the same workspace repo, clone once
   d. Clone the workspace repo with `--shared`, naming it `work-{source.name}`
      (with collision disambiguation against family repo names)
   e. Resolve the project's `wksp` target relative to the workspace source
      to find the subdirectory within the workspace clone
   f. `repoint_wksp` to the subdirectory within the workspace clone
   g. `create_proj_symlink` in that subdirectory back to the project clone

In practice, for the casehub family, all projects resolve to the same
workspace source (`~/claude/public/casehub/`), producing a single family
workspace clone with per-project subdirectories. But the code already
handles per-source resolution — the deduplication is explicit.

### Target

1. For each project repo in the slot:
   a. Clone the project repo with `--shared`
   b. Resolve the project's `wksp` symlink to find its workspace repo
   c. Clone that workspace repo with `--shared`, using its actual directory
      name (derived from git remote URL or original directory name)
   d. Place `.workspace` marker in the workspace clone
   e. Create bidirectional symlinks:
      `<project>/wksp` → `../<workspace>/`
      `<workspace>/proj` → `../<project>/`
   f. Create feature branch in both clones

No family workspace clone. No subdirectory nesting. Each workspace
clone is independent — same as workspace-init creates them.

**1:1 mapping assumption:** workspace-init guarantees each project has
its own independent workspace repo. Two projects should never resolve
to the same workspace source. If they do (legacy configuration), the
target `create_slot` detects the collision (same `slot_workspace_name`
already cloned) and errors with a diagnostic — the workspace-init
configuration must be fixed before slot creation succeeds. The current
`ws_created` deduplication is removed; it masked a configuration problem
that the target makes explicit.

### Resolving Workspace Repo Identity

For each project repo being cloned into a slot:

```
original_wksp = readlink(original_project/wksp)   # e.g., ~/claude/public/casehub/connectors
workspace_repo = git_toplevel(original_wksp)       # the git repo root
workspace_name = basename(workspace_repo)           # e.g., "connectors"

# But the directory name may collide with the project name.
# Use the remote URL to derive a unique name:
remote_url = git_remote(workspace_repo, "origin")   # e.g., wsp-casehub-connectors.git
slot_workspace_name = stem(basename(remote_url))     # e.g., "wsp-casehub-connectors"
```

If no remote exists, fall back to the workspace directory's parent path
to construct a unique name: `wsp-<family>-<project>`.

## Migration

### Phase 1: Detection Bootstrap + Attic Migration

**Goal:** Make workspace detection name-independent before any renaming.

1. Place `.workspace` marker in every workspace clone across all slots
   (active and archived). Walk each slot directory, identify workspace
   clones using current detection (name-based), write marker.
2. Update `is_workspace_clone()` to check `.workspace` marker as primary
   signal. Keep name-based check as fallback for one release cycle.
3. Bulk-migrate archived slots (attic): for each archived slot with a
   family workspace clone, split into per-repo workspace clones. Archived
   slots have no active sessions — safe to restructure freely.
4. Update `create_slot()` to use the new per-repo workspace model and
   place `.workspace` markers.

### Phase 2: Active Slot Migration

**Goal:** Restructure active slots with backwards-compatible symlinks.

For each active slot with old structure (`work-<family>/`):

1. **Check lifecycle state.** Only migrate at session start (before work
   begins) or work-end (work being closed). Never mid-session.
2. For each project repo in the slot:
   a. Clone the project's workspace repo fresh from the original
      (e.g., `git clone --shared ~/claude/public/casehub/connectors
      wsp-casehub-connectors`). Resolve the original via the project
      repo's `wksp` symlink outside the slot.
   b. Check out the same feature branch. Replay workspace changes from
      the family clone into the new per-repo clone:
      ```
      # Extract connectors-only changes from family workspace commits
      git -C work-casehub/ format-patch --relative=connectors/ \
          main..feature-branch -- connectors/
      # Apply to the new per-repo workspace clone
      git -C wsp-casehub-connectors/ am *.patch
      ```
      `--relative=connectors/` strips the subdirectory prefix, producing
      patches with paths rooted at the workspace clone. Cross-subdirectory
      commits (touching both `connectors/` and `pages/`) produce patches
      containing only the `connectors/` changes — atomicity across repos
      is not preserved but each repo gets its complete file-level changes.
      If no commits touch the subdirectory, no patches are generated —
      the fresh clone on the feature branch is already correct.
   c. Place `.workspace` marker in the new clone.
   d. Repoint project's `wksp` symlink to `../wsp-casehub-connectors/`.
   e. Replace `work-casehub/connectors/` with symlink to
      `../wsp-casehub-connectors/` (backwards compat for loaded sessions).
3. Commit the symlink replacements to the family workspace clone's
   feature branch:
   ```
   git -C work-casehub/ add connectors/ pages/ ...
   git -C work-casehub/ commit -m "chore: migration — subdirs replaced with symlinks to per-repo clones"
   ```
   This keeps the working tree clean. `merge_slot` skips workspace repos
   (via `get_slot_repos` filtering), so this commit is never pushed — it
   exists only to prevent dirty-tree warnings.
4. The family workspace clone (`work-casehub/`) remains as a shell with
   symlinks — old sessions following old paths still resolve correctly.
   Writes through symlinks land in the per-repo workspace clone's git
   repo, which is the correct destination. The session's `wksp` symlink
   has been repointed (step 2d), so new git operations use the per-repo
   clone directly.

**Why clone fresh instead of splitting:** The family workspace clone is
one git repo. Its subdirectories are not independent git repos — you
can't extract `connectors/` as a standalone repo without `git
filter-branch` or `git subtree split`, both heavyweight and fragile.
Cloning fresh from the original workspace repo and replaying branch
commits is simpler and preserves clean git history.

### Phase 3: Code Path Convergence + Sunset

**Goal:** Single code path. Remove all slot-specific workspace logic.

1. Update work-end to use the shared per-repo flow. Remove
   `merge-slot` as a separate orchestration path — decompose into shared
   flow + transport helpers.
2. Remove `is_project_repo()` name-based exclusions. Detection is
   purely structural (`.workspace` marker).
3. Remove name-based fallback from `is_workspace_clone()`.
4. **14-day sunset:** any slot not opened since migration rollout gets
   auto-migrated with a warning (not silently). After sunset:
   - Remove the symlink bridge code and family-workspace-clone support.
   - Physical cleanup: delete the family workspace clone directory
     (`work-casehub/`) from each migrated slot. The directory is a shell
     with symlinks and the migration commit — no unique content remains.
   - Remove `_unignore_subdir()` (dead code — no subdirectory nesting).
   - Remove `ensure_clone_layout()` worktree→clone migration (legacy
     migration from pre-clone era; any surviving worktree-era slots
     would have been processed during Phase 1-2 lifecycle events).
   - Remove name-based fallback from `is_workspace_clone()`.
   - Remove `is_project_repo()` name-based exclusions.

## Affected Code

| File | Function/Section | Change |
|------|-----------------|--------|
| `slot_manager.py:547-711` | `create_slot` | Per-repo workspace cloning, `.workspace` marker, remove `ws_created` dedup |
| `slot_manager.py:309-320` | `resolve_workspace_source` | Keep detection (wksp→git repo resolution), replace naming with remote-URL derivation |
| `slot_manager.py:872-877` | `is_project_repo` | Remove name-based exclusion |
| `slot_manager.py:880-894` | `is_workspace_clone` | `.workspace` marker only |
| `slot_manager.py:897-902` | `get_slot_repos` | Unchanged (uses updated detection) |
| `slot_manager.py:905-912` | `get_all_slot_repos` | Verify filtering still correct with `.workspace` markers |
| `slot_manager.py:431-444` | `repoint_wksp`, `create_proj_symlink` | Point to sibling, not subdirectory |
| `slot_manager.py:419-429` | `_unignore_subdir` | Remove in Phase 3 (dead code — no subdirectory nesting) |
| `slot_manager.py:485-505` | `replicate_claude_md` | Update for flat layout (no subdirectory path) |
| `slot_manager.py:93-110` | `validate_slot_wksp` | Update validation for flat layout |
| `slot_manager.py:1152-1161` | `ensure_clone_layout` | Preserve through Phase 2, remove in Phase 3 |
| `slot_manager.py:1163-1458` | `merge_slot` | Decompose into shared flow + transport helpers; add workspace merge/push/stamp |
| `slot_manager.py:1564-1664` | `archive_slot` | Handle new per-repo workspace layout during archival |
| `slot_manager.py:1670-1700` | `restore_slot` | Handle new layout during restoration |
| `topology.py:76-90` | `_detect_slot` | Unchanged (`.slot` detection is correct) |
| `topology.py:140-147` | layout resolution | May simplify — fewer layout variants |
| `ctx.py:243-244` | `IN_SLOT` / `SLOT_PATH` | Unchanged |
| `work-end/SKILL.md` | Slot-specific steps (4.4 Phase C, 6.1) | Converge to single flow; fix `get_all_slot_repos()` claim at line 403 |
| `work-end/work_end_execute.py:226-414` | `cmd_land` | Refactor to call shared flow as thin adapter |
| `work-end/work_end_execute.py:534-569` | `cmd_archive_slot` | Unchanged |
| `work-end/verify_slot_close.py` | Slot-specific verification | Update checks for per-repo workspace layout |
| `workspace-init/SKILL.md` | Workspace repo creation | Add `.workspace` marker placement |
| `workspace-init/create_symlinks.py` | Symlink setup | Add `.workspace` marker catch-up for existing repos |

## Risks

1. **Git state during migration.** Moving subdirectories out of a git
   repo and replacing with symlinks changes the working tree. Must commit
   cleanly on the feature branch.
2. **Loaded sessions.** A Claude instance may hold old paths in context.
   The symlink bridge handles this, but if a session writes to a path
   that's now a symlink, the write goes to the new location — correct
   behaviour but worth validating.
3. **merge-slot decomposition.** The current `merge_slot` function is
   ~300 lines and handles both orchestration and transport. Decomposing
   it risks introducing bugs if the shared flow misses a step.

## Out of Scope

- Changing the workspace model outside slots (workspace-init is already
  correct — separate repos per project).
- Changing `.slot` file format or slot lifecycle states.
- Changing `.plan` location or format.

## Companion Tasks

- **ARC42STORIES.MD update:** After implementation, update S4 (workspace-as-CWD
  model) to describe converged slot workspace layout, and L2 (Lifecycle layer)
  to reflect the single-flow work-end model. File as a GitHub issue during
  implementation.
- **work-end/SKILL.md correction:** Line 402-403 claims `merge_slot` stamps via
  `get_all_slot_repos()` — the code uses `get_slot_repos()` (project only).
  Fix as part of Phase 3 convergence.

## References

- **Tracking issue:** To be filed in `Hortora/soredium` before implementation
  begins. Out-of-scope items will be captured as separate GitHub issues at
  that point.
- `slot_manager.py:547-711` — `create_slot` (workspace cloning)
- `slot_manager.py:309-320` — `resolve_workspace_source` (workspace detection + naming)
- `slot_manager.py:872-894` — `is_project_repo`, `is_workspace_clone` (detection)
- `slot_manager.py:1163-1458` — `merge_slot` (two-hop push)
- `slot_manager.py:1564-1664` — `archive_slot`
- `topology.py:76-147` — layout detection and resolution
- `ctx.py:243-244` — `IN_SLOT` / `SLOT_PATH` output
- `work_end_execute.py:226-414` — `cmd_land` (branch-mode merge + workspace handling)
- `workspace-init/SKILL.md` — per-repo workspace creation model
- Decisions: `specs/slot-workspace-convergence/decisions.md` (D1-D7)
