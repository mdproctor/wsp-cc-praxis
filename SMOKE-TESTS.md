# Slot & Promotion Smoke Tests

Checklist for verifying the slot lifecycle is working correctly. Run these
after any changes to slot creation, promotion, or archival code.

## History of Issues Found

These are the failure modes that have been discovered and fixed. Each smoke
test exists because one of these broke silently in production.

| Issue | Failure Mode | How It Failed | Fix |
|-------|-------------|---------------|-----|
| #148 | Gitignore hides workspace subdirs in slot clones | Files written to `<slot>/work/<project>/` invisible to git. 29 artifacts lost across 10 repos. | `_unignore_subdir()` at slot creation |
| #148 | `to_workspace_main` branch not available | `git checkout branch --` fails because branch only exists in slot clone, not original workspace | `source-dir` param — filesystem copy |
| #148 | `to_project` reads from wrong source | Reads from original workspace (on main) instead of scan source (slot workspace) | Pass `scan_source` not `workspace` |
| #148 | `archive_plans` branch not available | Same as to_workspace_main | `source-dir` param |
| #148 | `to_project` single-repo SameFileError | `shutil.copy2(src, dst)` where src==dst | Guard `src.resolve() == dst.resolve()` |
| #139 | `archive_plans` bare pass on checkout failure | Stale main-version files silently archived instead of branch version | Track skipped plans, exclude from archival |
| #139 | Push failure doesn't block stamp | `.artifacts-promoted` written even when push fails | Track push failure, withhold stamp |
| #143 | Double-archive when attic slot exists | `shutil.move` nests slot inside existing attic entry | Check `dest.exists()` before move |
| #143 | CWD inside slot during removal | `shutil.rmtree` fails — CWD can't be deleted | `_escape_slot_cwd()` before removal |
| #143 | Workspace clone name collision | Repo named `work` collides with workspace clone name | Deconflict with `work-<name>` prefix |
| #145 | `record_slot_phase_a` never called | Worklog shows slots as `active` after Phase A | `phase_a_complete.py` script |
| #145 | `record_slot_archive` no metadata | No audit trail for what was promoted/published | Enriched with promoted, published, paths |

## Smoke Test Checklist

### 1. Slot Creation

- [ ] **Gitignore cleared:** After `create-slot`, check `.gitignore` in workspace clone does NOT contain the project subdirectory name
  ```bash
  cat <slot>/work/.gitignore | grep '<project-name>'  # should return nothing
  ```
- [ ] **Workspace subdir writable:** Write a file, verify git can see it
  ```bash
  echo test > <slot>/work/<project>/test.md
  git -C <slot>/work status --short  # should show the file
  ```
- [ ] **Symlinks correct:** `wksp` → workspace subdir, `proj` → project clone
  ```bash
  readlink <slot>/<project>/wksp   # → ../work/<project>
  readlink <slot>/work/<project>/proj  # → ../<project>
  ```
- [ ] **Scaffold created:** `.meta` and `JOURNAL.md` in workspace design/
- [ ] **Worklog recorded:** `slot-create` event in worklog.db

### 2. During Work (artifact creation)

- [ ] **Spec committable:** Write to `$WORKSPACE/specs/`, git add, git commit — succeeds
- [ ] **Blog committable:** Write to `$WORKSPACE/blog/`, git add, git commit — succeeds
- [ ] **Plan committable:** Write to `$WORKSPACE/plans/`, git add, git commit — succeeds
- [ ] **Images alongside markdown:** Any `.png`/`.jpg`/`.svg` referenced in markdown files are in the same directory or a relative path

### 3. Phase A (pre-merge)

- [ ] **Marker written:** `.phase-a-complete` exists in slot root with branch, repos, timestamp
- [ ] **Worklog updated:** `slot-phase-a` event recorded, slot state = `ready`
- [ ] **Branch pushed:** Feature branch pushed to origin from slot clone

### 4. Artifact Promotion (Phase B or work-end)

- [ ] **Workspace-routed artifacts promoted:** `WORKSPACE_PROMOTED >= 1` when scan finds artifacts
- [ ] **Project-routed artifacts promoted:** `PROJECT_PROMOTED >= 1` when scan finds artifacts
- [ ] **Plans archived:** `PLANS_ARCHIVED >= 1` when scan finds plans
- [ ] **Stamp written:** `.artifacts-promoted` exists with counts and timestamp
- [ ] **Push verified:** `PUSH_VERIFIED=yes` after push (when remote exists)
- [ ] **No silent skips:** If `SKIPPED > 0`, `SKIPPED_PATHS` lists the files and stderr has `SKIP_DETAIL`

### 5. Blog Publication

- [ ] **Blog entries copied:** Files from workspace `blog/` reach the publication destination
- [ ] **Blog committed and pushed:** Publication repo has the commit
- [ ] **Images published:** Any images referenced in blog markdown are also at the destination (KNOWN GAP — not yet implemented)

### 6. Issue Close

- [ ] **All COVERS closed:** Every issue number in the `.meta` covers field is CLOSED on GitHub
- [ ] **Partial failure reported:** If some issues fail to close, `ERRORS` field lists them

### 7. Branch Stamp

- [ ] **Content verified:** `verify_stamp.py` confirms branch content landed on main before stamping
- [ ] **Stamp format correct:** `chore: branch closed — landed as <SHA> on <base-branch>`
- [ ] **Both repos stamped:** Project branch AND workspace branch have closure stamps

### 8. Slot Archive

- [ ] **Landed marker exists:** `.landed` file with branch, repos, landed_shas, timestamp
- [ ] **SHA verified:** Each landed SHA is reachable from origin/main
- [ ] **Promotion stamp exists:** `.artifacts-promoted` in design/ subdirectory
- [ ] **Attic destination clean:** No pre-existing directory at `attic/<N>/`
- [ ] **Worklog recorded:** `slot-archive` event with promoted, published, archived_from, archived_to metadata
- [ ] **Claude projects relocated:** `.claude/projects/` dirs moved to match attic path
- [ ] **No remnant directories:** Slot directory fully removed after move to attic

### 9. Post-Close Hygiene

- [ ] **No unpublished blogs:** Hygiene scan finds zero unpublished entries
- [ ] **No unrecovered artifacts:** Closed branches with `EPIC-CLOSED.md` have no stranded specs/blogs
- [ ] **No unstamped branches:** Every `EPIC-CLOSED.md` branch has a corresponding project stamp

## Automated Test Coverage

| Test File | Class | Tests | Covers |
|-----------|-------|-------|--------|
| `test_work_end_scripts.py` | `TestToWorkspaceMain` | 7 | Normal mode promotion |
| `test_work_end_scripts.py` | `TestToWorkspaceMainPush` | 2 | Push skip/fail |
| `test_work_end_scripts.py` | `TestToWorkspaceMainSlotMode` | 4 | source-dir, multiple, missing, regression |
| `test_work_end_scripts.py` | `TestToProject` | 5 | Normal mode, directory, missing, errors |
| `test_work_end_scripts.py` | `TestToProjectPush` | 2 | Push skip/fail |
| `test_work_end_scripts.py` | `TestToProjectSingleRepo` | 2 | SameFileError guard |
| `test_close_artifacts.py` | `TestPromotionFailureDetection` | 4 | Scan source promotion, stamp, push failure |
| `test_close_artifacts.py` | `TestPostPushVerification` | 2 | Push verification |
| `test_close_artifacts.py` | `TestArchivePlans` | 2 | Normal archive, checkout failure skip |
| `test_close_artifacts.py` | `TestSlotModeProjectPromotion` | 1 | scan_source to_project |
| `test_close_artifacts.py` | `TestSlotModeArchivePlans` | 1 | source-dir archive |
| `test_close_artifacts.py` | `TestSlotModeEndToEnd` | 2 | Both routing paths, plan archival |
| `test_slot_manager.py` | `TestUnignoreSubdir` | 7 | Gitignore removal + integration |
| `test_slot_manager.py` | `TestReadPromotionStamp` | 3 | Stamp reading |
| `test_phase_a_complete.py` | `TestPhaseAComplete` | 4 | Marker writing |
| `test_phase_a_complete.py` | `TestWorklogRecording` | 2 | Phase A + enriched archive |
| `test_blog_person_check.py` | All | 18 | Person-reference detection |

**Total: 85 tests across promotion, slot lifecycle, and blog gate.**

## Known Gaps

- [ ] **Image promotion:** Markdown files with `![](image.png)` references — images not promoted alongside markdown. No tests.
- [ ] **Audit script noise:** `audit_slot_artifacts.py` reports inherited blogs and proj/ symlink files as "LOST". Needs filters.
- [ ] **Superpowers path migration:** Some repos use `docs/superpowers/specs/` instead of `docs/specs/`. Recovery script handles both; migration needed.
- [ ] **WackyManor plan:** `2026-07-27-phase-2.5-autonomous-characters.md` in examples/ workspace — 1 unique file, 3 copies in attic. Not yet recovered.

## How to Run

```bash
# Full automated test suite
python3 -m pytest tests/test_work_end_scripts.py tests/test_close_artifacts.py tests/test_slot_manager.py tests/test_phase_a_complete.py tests/test_blog_person_check.py -v

# Audit all slots
python3 ~/claude/hortora/soredium/scripts/audit_slot_artifacts.py

# Recovery (dry-run first)
python3 ~/claude/hortora/soredium/scripts/recover_slot_artifacts.py --dry-run

# Fix active slot gitignore
python3 ~/claude/hortora/soredium/scripts/fix_active_slots.py ~/claude/casehub
```
