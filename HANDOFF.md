# HANDOFF — soredium

**Last session:** 2026-08-02
**Branch closed:** issue-149-image-promotion (landed as 406520b on main)
**Issue:** #149 — Slot promotion robustness: image handling, audit refinement, smoke test automation

## What was done

### Image promotion (main feature)
- Added `extract_image_refs()` to `workspace_artifacts.py` — parses `![](path)` and `<img src>` references
- Wired into `scan()` — all markdown categories (specs, adr, blog, plans) now include referenced images
- Wired into `blog_dest.py` — blog publishing copies images alongside entries
- 17 new tests across 3 test files, 60/60 passing

### Audit script refinement
- Three false-positive filters: `filter_proj_symlinks`, `filter_inherited` (3+ slots), `filter_already_recovered`
- Added `--verbose`, `--summary`, `--no-filter` modes
- 80 raw findings → 27 after filtering (31 proj/, 22 inherited)
- 7 new tests

### Blog image fixes (mdproctor.github.io)
- Audited both blog sites (1,876 + 207 markdown files)
- Fixed 18 broken image references: 5 sparge PNGs, 3 casehub SVGs, 2 path fixes, 1 YouTube thumb
- Pushed directly to main on mdproctor.github.io

### Hook wiring
- `blog_person_hook.sh` added to `~/.claude/settings.json` PreToolUse[Bash]

## Known issue
- `extract_image_refs` fires on example image syntax inside code blocks in specs/plans — produces harmless warnings (file not found, skipped). Could be fixed by skipping content inside fenced code blocks.

## What is NOT done from #149
- [ ] Recover WackyManor plan from examples/ workspace
- [ ] Superpowers → standard docs/ path migration (separate epic)
- 27 genuine audit findings remain — these are real lost artifacts on archived slots worth investigating
