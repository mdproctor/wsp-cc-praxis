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

Converging concerns (shared code):
- Workspace interaction, artifact promotion
- Squash analysis and execution
- Branch stamping

Topology-specific transport (helper functions):
- Original-repo preflight sync (`slot_manager.py:1211-1277`)
- Two-hop push clone→original→remote (`slot_manager.py:1336-1386`)
- `updateInstead` configuration (`slot_manager.py:951`)
- SHA verification via `.landed` markers (`slot_manager.py:1402-1408`)

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
- `workspace-init` during workspace repo creation

`is_workspace_clone()` checks `.workspace` marker only. Name-based
and `proj`-symlink detection removed after marker rollout.

`is_project_repo()` becomes: is a git repo AND NOT `is_workspace_clone()`.
No hardcoded name exclusions.

## Slot Creation Changes

### Current (`create_slot` at `slot_manager.py:547-711`)

1. For each project repo in the slot, clone with `--shared`
2. Find the family workspace source (single repo)
3. Clone the family workspace once as `work-<family>/`
4. For each project repo, `repoint_wksp` to `work-<family>/<repo>/`
5. For each project repo, `create_proj_symlink` in `work-<family>/<repo>/`

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
2. For each project repo subdirectory in the family workspace clone:
   a. Create new workspace clone directory at slot root
      (`wsp-casehub-connectors/`)
   b. Move content from `work-casehub/connectors/*` to
      `wsp-casehub-connectors/`
   c. Replace `work-casehub/connectors/` with symlink to
      `../wsp-casehub-connectors/`
   d. Repoint project's `wksp` symlink to `../wsp-casehub-connectors/`
   e. Place `.workspace` marker in new workspace clone
3. The family workspace clone (`work-casehub/`) remains as a shell with
   symlinks — old sessions following old paths still resolve correctly.

**Git handling:** The family workspace clone is one git repo. Moving
subdirectories out and replacing with symlinks changes the git working
tree. The migration must commit these changes on the feature branch
so git state stays clean.

### Phase 3: Code Path Convergence + Sunset

**Goal:** Single code path. Remove all slot-specific workspace logic.

1. Update work-end to use the shared per-repo flow. Remove
   `merge-slot` as a separate orchestration path — decompose into shared
   flow + transport helpers.
2. Remove `is_project_repo()` name-based exclusions. Detection is
   purely structural (`.workspace` marker).
3. Remove name-based fallback from `is_workspace_clone()`.
4. **14-day sunset:** any slot not opened since migration rollout gets
   auto-migrated with a warning (not silently). After sunset, remove
   the symlink bridge code and family-workspace-clone support entirely.

## Affected Code

| File | Function/Section | Change |
|------|-----------------|--------|
| `slot_manager.py:547-711` | `create_slot` | Per-repo workspace cloning, `.workspace` marker |
| `slot_manager.py:872-877` | `is_project_repo` | Remove name-based exclusion |
| `slot_manager.py:880-894` | `is_workspace_clone` | `.workspace` marker only |
| `slot_manager.py:897-902` | `get_slot_repos` | Unchanged (uses updated detection) |
| `slot_manager.py:431-444` | `repoint_wksp`, `create_proj_symlink` | Point to sibling, not subdirectory |
| `slot_manager.py:1163-1458` | `merge_slot` | Decompose into shared flow + transport helpers |
| `topology.py:76-90` | `_detect_slot` | Unchanged (`.slot` detection is correct) |
| `topology.py:140-147` | layout resolution | May simplify — fewer layout variants |
| `ctx.py:243-244` | `IN_SLOT` / `SLOT_PATH` | Unchanged |
| `work-end/SKILL.md` | Slot-specific steps | Converge to single flow |
| `work-end/work_end_execute.py:226-414` | `cmd_land` | Already topology-agnostic — verify |
| `work-end/work_end_execute.py:534-569` | `cmd_archive_slot` | Unchanged |

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

## References

- `slot_manager.py:547-711` — `create_slot` (workspace cloning)
- `slot_manager.py:872-894` — `is_project_repo`, `is_workspace_clone` (detection)
- `slot_manager.py:1163-1458` — `merge_slot` (two-hop push)
- `slot_manager.py:1564-1664` — `archive_slot`
- `topology.py:76-147` — layout detection and resolution
- `ctx.py:243-244` — `IN_SLOT` / `SLOT_PATH` output
- `work_end_execute.py:226-414` — `cmd_land`
- `workspace-init/SKILL.md` — per-repo workspace creation model
- Decisions: `specs/slot-workspace-convergence/decisions.md` (D1-D7)
