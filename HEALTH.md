# Workspace & Slot Health Report

**Last full audit:** 2026-08-02
**Audited by:** session "Stability Audit and Slot Promotion Fix"

## Executive Summary

| Metric | Status |
|--------|--------|
| Active slots | 17 (casehub) |
| Archived slots | 40 (casehub), 1 (hortora) |
| Gitignore fix applied | All active slots patched |
| Promotion bugs fixed | 4 root-cause bugs (source-dir, scan_source, SameFileError) |
| Artifacts recovered | 29 files across 10 repos |
| Still unrecovered | 3 (1 unique file in examples/ workspace) |
| Published blogs (mdproctor.github.io) | 1,260 |
| Test coverage | 85 tests across promotion system |

## Fixes Applied This Audit

### #148 — Slot workspace gitignore bug (CLOSED)
- **Root cause:** Slot workspace clones inherit `.gitignore` from parent, hiding project subdirectories. All artifacts written there are invisible to git.
- **Fix:** `_unignore_subdir()` runs at slot creation. Retroactively patched 16 active slots (26 gitignore entries removed).

### Promotion data flow bugs (CLOSED)
- **BUG 1:** `to_workspace_main` — branch only exists in slot clone. Fix: `source-dir` filesystem copy.
- **BUG 2:** `to_project` — reads from original workspace (on main). Fix: pass `scan_source`.
- **BUG 3:** `archive_plans` — same as BUG 1. Fix: `source-dir`.
- **BUG 4:** `to_project` single-repo — SameFileError when workspace==project. Fix: resolve guard.

### #139, #145, #115 (CLOSED)
- archive_plans bare-pass fix, phase_a_complete.py worklog call, blog person-reference gate.

## Recovery Log

| Date | Items | Repos | Notes |
|------|-------|-------|-------|
| 2026-08-01 | 29 | clinical, claudony, blocks-ui, devtown, qhorus, platform, aml, pages, engine, eidos | First batch via `recover_slot_artifacts.py` |
| 2026-08-01 | 5 | workspace (3 specs), claudony (1 spec) | Slot 62 manual recovery |

## Known False Positives in Audit

These show as "LOST" in `audit_slot_artifacts.py` but are NOT data loss:

| Category | Count | Reason |
|----------|-------|--------|
| Inherited blogs (`virtual-threads-retire`, `trust-workbench`) | ~22 | Same file inherited from workspace main onto every slot clone. Already published. |
| `proj/docs/` symlink files | ~3 | Project files visible through `proj` symlink. Tracked by project git, not workspace git. |
| Already-recovered files | ~25 | Still exist in attic but copies already in project repos from recovery. |

**Genuinely unrecovered:** 3 items — all the same file (`2026-07-27-phase-2.5-autonomous-characters.md`) in the `examples/` workspace subdirectory across slots 41, 43, 44. This is a plan for the WackyManor examples project.

## Active Slot Status (casehub)

| Slot | Branch | Issue | State | Stamp |
|------|--------|-------|-------|-------|
| 30 | — | — | active | no |
| 46 | — | — | active | no |
| 51 | issue-29-service-lifecycle | #30 | active | no |
| 53 | issue-120-cbr-multi-scope-dsmb | #120 | active | no |
| 54 | issue-813-alternative-scheduler-spi | #813 | active | no |
| 56 | issue-47-llm-htn-heuristics | #47 | active | no |
| 57 | issue-48-dspy-prompt-optimisation | #48 | active | no |
| 58 | issue-821-lifecycle-scope-wiring | #821 | active | no |
| 59 | issue-2-staged-layer-comparison | #2 | active | no |
| 62 | — | — | active | no |
| 63 | epic-b-conversation-maturity | #177 | active | no |
| 65 | issue-84-milestone-goal-alignment | #84 | active | no |
| 66 | issue-237-structured-progress | #237 | active | no |
| 67 | issue-830-a2a-mcp-integration | #830 | active | no |
| 68 | issue-152-per-capability-duration-breakdown | #152 | active | no |
| 69 | issue-833-acl-engine-integration | #833 | active | no |
| 70 | issue-24-contributor-trust-routing | #24 | active | no |

Slots 30, 46, 62 have no `.slot` file — likely remnants or manually created. All active slots have had their gitignore entries patched.

## Archived Slot Status (casehub)

40 slots in `worktrees/attic/`. All have been scanned. Artifacts in gitignored subdirectories were recovered in the batch above. The remaining "LOST" items are inherited duplicates or already-recovered files.

## Outstanding Issues

- [ ] Image promotion: markdown files referencing images (`.png`, `.jpg`, `.svg`) — images are not promoted or published alongside the markdown. No test coverage.
- [ ] `audit_slot_artifacts.py` needs filters for inherited blogs and `proj/` false positives to reduce noise.
- [ ] Superpowers → standard `docs/` path migration needed across repos (technical debt).
- [ ] Recover `2026-07-27-phase-2.5-autonomous-characters.md` from examples/ workspace (3 copies in attic, 1 unique file).
- [ ] Blog hook (`blog_person_hook.sh`) not yet wired into settings.json.

## How to Re-Audit

```bash
# Full audit (includes false positives)
python3 ~/claude/hortora/soredium/scripts/audit_slot_artifacts.py

# Recovery (dry-run first)
python3 ~/claude/hortora/soredium/scripts/recover_slot_artifacts.py --dry-run
python3 ~/claude/hortora/soredium/scripts/recover_slot_artifacts.py --apply

# Fix active slot gitignore entries
python3 ~/claude/hortora/soredium/scripts/fix_active_slots.py ~/claude/casehub
```
