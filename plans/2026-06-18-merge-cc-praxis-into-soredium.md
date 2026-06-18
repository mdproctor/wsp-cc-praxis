# Merge cc-praxis into soredium — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Merge cc-praxis's 30 skills, scripts, tests, and blog into Hortora/soredium as a single repo and plugin, then archive cc-praxis.

**Architecture:** Git history merge with `--allow-unrelated-histories` for code, semantic manual merge for ~10 conflicting root files, blog posts copied to hortora.github.io with global renumbering, workspace and global config updates to remove all cc-praxis references.

**Tech Stack:** Git, Python 3, Jekyll (hortora.github.io), YAML/JSON (marketplace, routing), Markdown (skills, docs)

**Spec:** `~/claude/public/cc-praxis/specs/2026-06-18-merge-cc-praxis-into-soredium-design.md`

---

## Task 1: Backup — Tag Both Repos

**Files:**
- No file changes — git tag operations only

- [ ] **Step 1: Tag cc-praxis**

```bash
git -C /Users/mdproctor/claude/cc-praxis tag pre-hortora-merge
git -C /Users/mdproctor/claude/cc-praxis push origin pre-hortora-merge
```

Expected: Tag created and pushed. Verify with `git -C /Users/mdproctor/claude/cc-praxis tag -l 'pre-*'`.

- [ ] **Step 2: Tag soredium**

```bash
git -C /Users/mdproctor/claude/hortora/soredium tag pre-merge
git -C /Users/mdproctor/claude/hortora/soredium push origin pre-merge
```

Expected: Tag created and pushed. Verify with `git -C /Users/mdproctor/claude/hortora/soredium tag -l 'pre-*'`.

- [ ] **Step 3: Verify both tags exist on remote**

```bash
git -C /Users/mdproctor/claude/cc-praxis ls-remote --tags origin | grep pre-hortora-merge
git -C /Users/mdproctor/claude/hortora/soredium ls-remote --tags origin | grep pre-merge
```

Expected: Both tags visible on remote. These are the rollback points.

---

## Task 2: Blog Migration — Copy Posts to hortora.github.io

This must happen BEFORE the git merge because the merge will later remove `docs/_posts/` from soredium.

**Files:**
- Copy: 17 files from `/Users/mdproctor/claude/cc-praxis/docs/_posts/*.md` (excluding INDEX.md) → `/Users/mdproctor/claude/hortora/hortora.github.io/_posts/`
- Modify: 1 post with hardcoded `/cc-praxis/` URL

- [ ] **Step 1: Copy the 17 cc-praxis posts to hortora.github.io**

Write a Python script to `/tmp/copy_posts.py`:

```python
#!/usr/bin/env python3
"""Copy cc-praxis blog posts to hortora.github.io, excluding INDEX.md."""
import shutil
from pathlib import Path

src = Path("/Users/mdproctor/claude/cc-praxis/docs/_posts")
dst = Path("/Users/mdproctor/claude/hortora/hortora.github.io/_posts")

copied = 0
for f in sorted(src.glob("*.md")):
    if f.name == "INDEX.md":
        continue
    shutil.copy2(f, dst / f.name)
    copied += 1
    print(f"  Copied: {f.name}")

print(f"\nTotal: {copied} posts copied")
```

Run: `python3 /tmp/copy_posts.py`

Expected: 17 posts copied. Verify with `ls /Users/mdproctor/claude/hortora/hortora.github.io/_posts/ | wc -l` — should be 41 (24 existing + 17 new).

- [ ] **Step 2: Fix the hardcoded baseurl reference**

The file `2026-04-09-mdp01-where-claude-lives-now.md` contains a `/cc-praxis/` URL reference. Find and fix it:

```bash
grep -n "/cc-praxis/" /Users/mdproctor/claude/hortora/hortora.github.io/_posts/2026-04-09-mdp01-where-claude-lives-now.md
```

Edit the file to remove the `/cc-praxis` prefix from the URL (the hortora site uses `baseurl: ""`).

- [ ] **Step 3: Global renumber all 41 posts**

Write a Python script to `/tmp/renumber_posts.py`:

```python
#!/usr/bin/env python3
"""
Renumber all blog posts sequentially by date then filename.
Updates the entry: field in YAML frontmatter.
"""
import re
from pathlib import Path

posts_dir = Path("/Users/mdproctor/claude/hortora/hortora.github.io/_posts")
posts = sorted(posts_dir.glob("*.md"))  # sorted by filename = sorted by date then mdpNN

for i, post in enumerate(posts, start=1):
    content = post.read_text()

    # Extract frontmatter
    match = re.match(r'^---\n(.*?)\n---\n', content, re.DOTALL)
    if not match:
        print(f"  SKIP (no frontmatter): {post.name}")
        continue

    fm = match.group(1)
    body = content[match.end():]
    entry_num = f'"{i:02d}"'

    # Update or add entry: field
    if re.search(r'^entry:', fm, re.MULTILINE):
        fm = re.sub(r'^entry:.*$', f'entry: {entry_num}', fm, flags=re.MULTILINE)
    else:
        # Add entry: after date: line
        fm = re.sub(r'^(date:.*)$', rf'\1\nentry: {entry_num}', fm, flags=re.MULTILINE)

    post.write_text(f'---\n{fm}\n---\n{body}')
    print(f"  {post.name} → entry: {entry_num}")

print(f"\nTotal: {i} posts renumbered")
```

Run: `python3 /tmp/renumber_posts.py`

Expected: 41 posts renumbered sequentially. Verify a sample: `head -8 /Users/mdproctor/claude/hortora/hortora.github.io/_posts/2026-03-29-mdp01-day-zero.md` should show `entry: "01"`.

- [ ] **Step 4: Commit to hortora.github.io**

```bash
git -C /Users/mdproctor/claude/hortora/hortora.github.io add _posts/
git -C /Users/mdproctor/claude/hortora/hortora.github.io commit -m "feat: merge cc-praxis diary entries + global renumber

17 cc-praxis posts migrated. All 41 posts renumbered sequentially
by date then filename. Fixes entry gaps and 3 posts missing entry
field entirely."
```

- [ ] **Step 5: Push hortora.github.io**

```bash
git -C /Users/mdproctor/claude/hortora/hortora.github.io push
```

Verify the site builds: check https://hortora.github.io/blog/ after GitHub Pages rebuilds (~2 min).

---

## Task 3: Git History Merge

**Files:**
- All cc-praxis files merge into soredium
- ~10 files will conflict (handled in subsequent tasks)

- [ ] **Step 1: Add cc-praxis as remote and fetch**

```bash
git -C /Users/mdproctor/claude/hortora/soredium remote add cc-praxis https://github.com/mdproctor/cc-praxis.git
git -C /Users/mdproctor/claude/hortora/soredium fetch cc-praxis
```

Expected: Remote added, all branches and tags fetched.

- [ ] **Step 2: Merge with --no-commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium merge cc-praxis/main --allow-unrelated-histories --no-commit
```

Expected: Merge pauses with conflicts. Note which files conflict — compare against the spec's conflict list:
- `CLAUDE.md`
- `README.md`
- `.claude-plugin/marketplace.json`
- `scripts/claude-skill`
- `.gitignore`
- `docs/protocols/INDEX.md`
- `tests/test_base.py`
- `tests/test_claude_skill.py`
- `pytest.ini` (may not conflict if soredium has none)
- `requirements.txt` (may not conflict if soredium has none)

Any unexpected conflicts should be investigated before proceeding.

- [ ] **Step 3: Resolve trivial conflicts**

For files where one repo has the file and the other doesn't (no actual merge needed), accept the incoming version:

```bash
# pytest.ini — cc-praxis only, take it
git -C /Users/mdproctor/claude/hortora/soredium checkout cc-praxis/main -- pytest.ini

# .gitignore — union of both
# Read both, write combined version (see step 4)
```

- [ ] **Step 4: Write merged .gitignore**

Combine entries from both repos. Write this content to `/Users/mdproctor/claude/hortora/soredium/.gitignore`:

```
# General
.DS_Store
.worktrees/
__MACOSX

# Claude Code local settings
.claude/

# Python
__pycache__/
*.pyc
.venv

# Modular documentation cache
.doc-cache.json

# Health check reports (generated by project-health --save)
*-health-report.md

# Third-party skill (not maintained here)
skill-creator/

# Generated skill metadata (not committed)
**/skill.json

# Third-party tool cache (superpowers:brainstorm, etc.)
.superpowers/

# Workspace symlink (project-specific, not committed)
wksp

# Registry candidate report (generated)
registry/candidate_report.json
```

Stage it: `git -C /Users/mdproctor/claude/hortora/soredium add .gitignore`

- [ ] **Step 5: Write merged requirements.txt**

Write this content to `/Users/mdproctor/claude/hortora/soredium/requirements.txt`:

```
requests
pytest
PyYAML
sentence-transformers
numpy
mcp
```

Stage it: `git -C /Users/mdproctor/claude/hortora/soredium add requirements.txt`

- [ ] **Step 6: Resolve test_base.py — take cc-praxis version**

```bash
git -C /Users/mdproctor/claude/hortora/soredium checkout cc-praxis/main -- tests/test_base.py
git -C /Users/mdproctor/claude/hortora/soredium add tests/test_base.py
```

- [ ] **Step 7: Resolve docs/protocols/INDEX.md — write merged version**

Read both versions, then write a merged INDEX.md that includes all 5 protocol entries (4 from cc-praxis + 1 from soredium). Maintain the table format from whichever version has it. Stage with `git -C /Users/mdproctor/claude/hortora/soredium add docs/protocols/INDEX.md`.

- [ ] **Step 8: Mark remaining conflicts as "ours" temporarily**

The big semantic merges (CLAUDE.md, README.md, marketplace.json, scripts/claude-skill, tests/test_claude_skill.py) are handled in dedicated tasks. For now, accept soredium's version to unblock the merge commit:

```bash
git -C /Users/mdproctor/claude/hortora/soredium checkout --ours CLAUDE.md README.md .claude-plugin/marketplace.json scripts/claude-skill tests/test_claude_skill.py
git -C /Users/mdproctor/claude/hortora/soredium add CLAUDE.md README.md .claude-plugin/marketplace.json scripts/claude-skill tests/test_claude_skill.py
```

- [ ] **Step 9: Commit the merge**

```bash
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat: merge cc-praxis git history into soredium

Merges mdproctor/cc-praxis (30 workflow skills, scripts, tests, docs)
into Hortora/soredium. Root file conflicts resolved with soredium
versions as placeholders — semantic merges follow in subsequent commits.

Refs: spec at ~/claude/public/cc-praxis/specs/2026-06-18-merge-cc-praxis-into-soredium-design.md"
```

- [ ] **Step 10: Remove the cc-praxis remote**

```bash
git -C /Users/mdproctor/claude/hortora/soredium remote remove cc-praxis
```

---

## Task 4: Remove Jekyll Site Files

The cc-praxis Jekyll site is no longer needed — blog posts were already migrated to hortora.github.io in Task 2.

**Files:**
- Delete: `docs/_config.yml`, `docs/_layouts/`, `docs/_posts/`, `docs/Gemfile`, `docs/articles/`, `docs/blog/`, `docs/guide.html`, `docs/images/`, `docs/visuals/`, `docs/web-installer-mockup.html`, `.github/workflows/pages.yml`

- [ ] **Step 1: Remove all Jekyll site files**

```bash
git -C /Users/mdproctor/claude/hortora/soredium rm -r docs/_config.yml docs/_layouts docs/_posts docs/Gemfile docs/blog .github/workflows/pages.yml
git -C /Users/mdproctor/claude/hortora/soredium rm -r docs/articles docs/guide.html docs/web-installer-mockup.html
```

For directories that might not exist or might have been gitignored, check first:
```bash
git -C /Users/mdproctor/claude/hortora/soredium rm -r docs/images docs/visuals 2>/dev/null; true
```

- [ ] **Step 2: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add -A
git -C /Users/mdproctor/claude/hortora/soredium commit -m "chore: remove cc-praxis Jekyll site files

Blog posts already migrated to hortora.github.io (Task 2).
Jekyll config, layouts, Gemfile, and GitHub Pages workflow removed."
```

---

## Task 5: Semantic Merge — marketplace.json

**Files:**
- Modify: `/Users/mdproctor/claude/hortora/soredium/.claude-plugin/marketplace.json`

- [ ] **Step 1: Write the merged marketplace.json**

Read both existing marketplace.json files (already in context from spec exploration). Write the merged version combining:
- Top-level metadata from soredium (owner: Hortora)
- Updated name and description
- All 28 cc-praxis plugins (copy their entries verbatim — name, source, description, path)
- soredium's 2 existing plugins (forage, harvest)
- protocol (newly added — copy its SKILL.md description)
- Drop the `bundles` array

The merged file should have 31 entries in the `plugins` array. No `bundles` key.

Each cc-praxis plugin entry needs its `source` field (if present) updated. But cc-praxis plugins use local `"source": "./skillname"` paths, not URLs — these are correct as-is since the skill directories now live in soredium.

- [ ] **Step 2: Validate the JSON**

```bash
python3 -c "import json; d=json.load(open('/Users/mdproctor/claude/hortora/soredium/.claude-plugin/marketplace.json')); print(f'{len(d[\"plugins\"])} plugins'); assert len(d['plugins']) == 31, f'Expected 31, got {len(d[\"plugins\"])}'"
```

Expected: `31 plugins` with no assertion error.

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add .claude-plugin/marketplace.json
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat: merge marketplace.json — 31 skills in one plugin

28 cc-praxis + forage + harvest + protocol (newly published).
Bundles removed — single flat plugin list."
```

---

## Task 6: Semantic Merge — scripts/claude-skill

**Files:**
- Modify: `/Users/mdproctor/claude/hortora/soredium/scripts/claude-skill`

- [ ] **Step 1: Take cc-praxis version as baseline**

```bash
git -C /Users/mdproctor/claude/hortora/soredium show cc-praxis/main:scripts/claude-skill > /Users/mdproctor/claude/hortora/soredium/scripts/claude-skill
```

Wait — the remote was already removed. Instead, copy from the cc-praxis checkout:

```bash
cp /Users/mdproctor/claude/cc-praxis/scripts/claude-skill /Users/mdproctor/claude/hortora/soredium/scripts/claude-skill
```

- [ ] **Step 2: Update URLs and description**

Edit `/Users/mdproctor/claude/hortora/soredium/scripts/claude-skill`:

Replace line 3:
```
Skill installer and local development sync tool for mdproctor/cc-praxis.
```
with:
```
Skill installer and local development sync tool for Hortora/soredium.
```

Replace line 20:
```python
MARKETPLACE_URL = "https://raw.githubusercontent.com/mdproctor/cc-praxis/main/.claude-plugin/marketplace.json"
```
with:
```python
MARKETPLACE_URL = "https://raw.githubusercontent.com/Hortora/soredium/main/.claude-plugin/marketplace.json"
```

- [ ] **Step 3: Verify the script runs**

```bash
python3 /Users/mdproctor/claude/hortora/soredium/scripts/claude-skill list 2>&1 | head -5
```

Expected: Lists installed skills without errors.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add scripts/claude-skill
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat: merge scripts/claude-skill — cc-praxis version with Hortora URLs

Takes cc-praxis version (superset with hook management), updates
marketplace URL and description to Hortora/soredium."
```

---

## Task 7: Semantic Merge — tests/test_claude_skill.py

**Files:**
- Modify: `/Users/mdproctor/claude/hortora/soredium/tests/test_claude_skill.py`

- [ ] **Step 1: Take cc-praxis version as baseline**

The cc-praxis version (400 lines, 31 test methods) is a superset of soredium's (329 lines, 20 test methods). cc-praxis has the same core tests plus 3 additional test classes for hook management (`TestIsHookRegistered`, `TestRegisterHook`, `TestSyncHook`).

The class names differ slightly between repos. cc-praxis has `TestSyncLocalEdgeCases` where soredium has `TestSyncLocalExclusions` (with an extra test `test_supporting_files_are_synced`).

Copy cc-praxis version:
```bash
cp /Users/mdproctor/claude/cc-praxis/tests/test_claude_skill.py /Users/mdproctor/claude/hortora/soredium/tests/test_claude_skill.py
```

- [ ] **Step 2: Check for soredium-only tests to preserve**

Compare the test method lists. soredium has `test_supporting_files_are_synced` in `TestSyncLocalUpdatesInstalled` and `test_specific_skill_synced` + `test_unknown_skill_exits` in `TestSyncLocalSkillsFlag` — verify these exist in the cc-praxis version. If not, add them.

Read both files and identify any soredium tests not present in the cc-praxis version. Add any missing tests to the merged file.

- [ ] **Step 3: Run the merged test suite**

```bash
python3 -m pytest /Users/mdproctor/claude/hortora/soredium/tests/test_claude_skill.py -v 2>&1 | tail -20
```

Expected: All tests pass. If any fail due to the merged claude-skill (e.g. default args changed), fix the test assertions.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add tests/test_claude_skill.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat: merge test_claude_skill.py — combined test suites

cc-praxis version with hook management tests, plus any soredium-only
tests preserved."
```

---

## Task 8: Run Full Test Suite

**Files:**
- No file changes — verification only

- [ ] **Step 1: Run all tests**

```bash
python3 -m pytest /Users/mdproctor/claude/hortora/soredium/tests/ -v --tb=short 2>&1 | tail -40
```

Expected: All tests pass. Note the total count — should be close to ~1988 (1152 cc-praxis + 836 soredium).

- [ ] **Step 2: Fix any failures**

If tests fail due to:
- Import paths — fix the import
- Hardcoded repo names — update to soredium
- Missing dependencies — add to requirements.txt

Fix each failure, then re-run until green.

- [ ] **Step 3: Commit any fixes**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add -A
git -C /Users/mdproctor/claude/hortora/soredium commit -m "fix: resolve test failures from merge

[describe what was fixed]"
```

Only commit if there were fixes needed. Skip if all tests passed clean.

---

## Task 9: Semantic Merge — CLAUDE.md

This is the largest semantic merge. The merged CLAUDE.md combines cc-praxis's detailed skills architecture, checklists, and quality framework with soredium's garden tooling, engine subproject, and ecosystem pipeline.

**Files:**
- Modify: `/Users/mdproctor/claude/hortora/soredium/CLAUDE.md`

- [ ] **Step 1: Read both CLAUDE.md files**

Read `/Users/mdproctor/claude/cc-praxis/CLAUDE.md` (the cc-praxis version — this is a symlink to the workspace, but the content is the project's CLAUDE.md) and `/Users/mdproctor/claude/hortora/soredium/CLAUDE.md` (currently soredium's pre-merge version since we took "ours" in Task 3).

Also read the soredium CLAUDE.md from git history to get the original: `git -C /Users/mdproctor/claude/hortora/soredium show pre-merge:CLAUDE.md`

- [ ] **Step 2: Write the merged CLAUDE.md**

The merged CLAUDE.md should follow this structure (see spec §3 for detailed requirements):

1. **Project Identity** — name: soredium, GitHub: Hortora/soredium, type: skills
2. **Repository Purpose** — workflow skills + garden skills + garden tooling + engine
3. **AI Attribution ban** — keep cc-praxis's strong wording
4. **Skills Architecture** — take cc-praxis's detailed version (frontmatter, CSO, naming, chaining, supporting files, flowcharts, success criteria, common mistakes tables). This is the superset.
5. **Developer Workflow** — merge both:
   - cc-praxis skill commands (sync-local, validate_all, generate_commands, etc.)
   - soredium garden commands (validate_pr, integrate_entry, run_pipeline, etc.)
   - Registry management (project_registry.py, etc.)
   - `python3 -m pytest tests/ -v` with combined test count
6. **Working in engine/** — NEW section. Maven commands, Java 21, Quarkus 3.x, note java-dev applies here. Document that this is a Quarkus/Java subproject with its own pom.xml.
7. **Migration Status** — from soredium (forage/harvest migration)
8. **Skills in This Repository** — merged table of all 33 skills
9. **Pre-Commit Checklist** — cc-praxis version (more comprehensive)
10. **Quality Assurance Framework** — cc-praxis version. Note: QUALITY.md references need scope audit for garden tooling.
11. **Protocols** — from soredium, expanded with cc-praxis protocols
12. **Work Tracking** — Hortora/soredium, enabled
13. **Meta-rules** — cc-praxis version (mandatory checklists, universality)
14. **Project Artifacts** — merged paths table

Drop: all `mdproctor/cc-praxis` references, cc-praxis blog section, cc-praxis GitHub Pages, the `CLAUDE.md -> workspace symlink` note, workspace model section (that's workspace CLAUDE.md, not project CLAUDE.md).

- [ ] **Step 3: Verify no cc-praxis GitHub URLs remain**

```bash
grep "mdproctor/cc-praxis" /Users/mdproctor/claude/hortora/soredium/CLAUDE.md
```

Expected: No output.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add CLAUDE.md
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat: unified CLAUDE.md — merged project conventions

Combines cc-praxis skills architecture, checklists, and quality
framework with soredium garden tooling, engine subproject, and
ecosystem pipeline. Documents engine/ for the first time."
```

---

## Task 10: Semantic Merge — README.md

**Files:**
- Modify: `/Users/mdproctor/claude/hortora/soredium/README.md`

- [ ] **Step 1: Write a functional merged README**

This is NOT the polished landing page (that's out of scope). This is a functional README for developers that describes what soredium now is. Structure:

1. **Title and one-line description** — Soredium: development workflow + knowledge garden skills for Claude Code
2. **What's in the box** — 31 skills across workflow, garden, and dev categories
3. **Install** — `plugin marketplace add github.com/Hortora/soredium`
4. **Skill categories** — table grouping skills by function (workflow, language dev, garden, health, integrators)
5. **Developer workflow** — brief commands for contributors
6. **Links** — hortora.github.io, Hortora org

Keep it concise — the landing page redesign is a separate issue.

- [ ] **Step 2: Verify no cc-praxis references**

```bash
grep "cc-praxis" /Users/mdproctor/claude/hortora/soredium/README.md
```

Expected: No output (or only in a "History" note if you choose to include one).

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add README.md
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat: unified README.md for merged soredium

Functional README covering 31 skills, install instructions, and
developer workflow. Landing page redesign is a separate issue."
```

---

## Task 11: Reference Sweep

**Files:**
- Modify: multiple files across the merged repo

- [ ] **Step 1: Priority fix — config-architecture.md fetch URL**

These three files have a `raw.githubusercontent.com/mdproctor/cc-praxis` URL that powers a daily silent fetch. Fix them first:

1. `update-claude-md/SKILL.md` line 52 — update `GENERIC_URL` to `https://raw.githubusercontent.com/Hortora/soredium/main/docs/config-architecture.md`
2. `tests/test_config_architecture_refresh.py` line 18 — update `GITHUB_URL` constant
3. `docs/config-architecture.md` line 3 — update source self-reference URL

- [ ] **Step 2: Sweep Category 1 — GitHub URLs**

```bash
grep -rn "mdproctor/cc-praxis" /Users/mdproctor/claude/hortora/soredium/ --include="*.md" --include="*.py" --include="*.json" --include="*.yaml" --include="*.yml" --include="*.sh" | grep -v ".git/"
```

For each file found, replace `mdproctor/cc-praxis` with `Hortora/soredium`. These are install instructions, marketplace references, and link resolution.

- [ ] **Step 3: Sweep Category 2 — project identity references**

```bash
grep -rn "cc-praxis" /Users/mdproctor/claude/hortora/soredium/ --include="*.md" --include="*.py" --include="*.json" --include="*.yaml" --include="*.yml" --include="*.sh" | grep -v ".git/" | grep -v "docs/specs/" | grep -v "docs/adr/" | grep -v "docs/superpowers/" | grep -v "docs/archive/"
```

This excludes Category 3 (historical) files. For each remaining file, update "cc-praxis" references to "soredium" where they refer to the project name, repo identity, or install path. Leave references that are historical context within the file (e.g. a comment explaining migration from cc-praxis).

- [ ] **Step 4: Verify Category 3 files are untouched**

```bash
grep -rn "cc-praxis" /Users/mdproctor/claude/hortora/soredium/docs/specs/ /Users/mdproctor/claude/hortora/soredium/docs/adr/ /Users/mdproctor/claude/hortora/soredium/docs/superpowers/ /Users/mdproctor/claude/hortora/soredium/docs/archive/ --include="*.md" | wc -l
```

Expected: Non-zero count. These are historical references and should NOT be changed.

- [ ] **Step 5: Commit the sweep**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add -A
git -C /Users/mdproctor/claude/hortora/soredium commit -m "chore: reference sweep — cc-praxis → Hortora/soredium

Category 1: 14 GitHub URL references updated.
Category 2: project identity references in SKILL.md files updated.
Category 3: historical references in specs/adr/archive left as-is.
Priority: config-architecture.md daily fetch URL fixed first."
```

---

## Task 12: Workspace Transition

**Files:**
- Modify: `~/claude/public/cc-praxis/CLAUDE.md`
- Modify: `~/claude/public/hortora/CLAUDE.md`
- Create: `~/claude/public/cc-praxis/blog-routing.yaml`
- Modify: symlinks

- [ ] **Step 1: Update cc-praxis workspace CLAUDE.md**

Edit `~/claude/public/cc-praxis/CLAUDE.md`:
- Change `**Name:** cc-praxis` → `**Name:** soredium`
- Change `**Project repo:** /Users/mdproctor/claude/cc-praxis` → `**Project repo:** /Users/mdproctor/claude/hortora/soredium`
- Update `add-dir /Users/mdproctor/claude/cc-praxis` → `add-dir /Users/mdproctor/claude/hortora/soredium`
- Update Git Discipline section paths
- Update any other cc-praxis references to soredium

- [ ] **Step 2: Update proj/ symlink**

```bash
rm /Users/mdproctor/claude/public/cc-praxis/proj
ln -s /Users/mdproctor/claude/hortora/soredium /Users/mdproctor/claude/public/cc-praxis/proj
```

Verify: `readlink /Users/mdproctor/claude/public/cc-praxis/proj` should show `/Users/mdproctor/claude/hortora/soredium`.

- [ ] **Step 3: Update wksp/ symlink in soredium**

```bash
rm /Users/mdproctor/claude/hortora/soredium/wksp 2>/dev/null
ln -s /Users/mdproctor/claude/public/cc-praxis /Users/mdproctor/claude/hortora/soredium/wksp
```

Verify: `readlink /Users/mdproctor/claude/hortora/soredium/wksp` should show `/Users/mdproctor/claude/public/cc-praxis`.

- [ ] **Step 4: Update hortora family workspace CLAUDE.md**

Edit `~/claude/public/hortora/CLAUDE.md`: update the member table description for soredium to reflect its merged scope (33 skills + engine, not just 3 garden skills).

- [ ] **Step 5: Create blog-routing.yaml**

Write to `~/claude/public/cc-praxis/blog-routing.yaml`:

```yaml
version: 1

destinations:
  hortora-diary:
    type: git
    path: ~/claude/hortora/hortora.github.io/
    subdir: _posts/

defaults:
  destinations: [hortora-diary]

rules:
  - match:
      entry_type: note
      subtype: diary
    destinations: [hortora-diary]
```

- [ ] **Step 6: Check pause stack**

```bash
cat ~/claude/public/cc-praxis/.pause-stack 2>/dev/null || echo "No pause stack"
```

If entries exist referencing cc-praxis paths, update them. If empty or missing, skip.

- [ ] **Step 7: Commit workspace changes**

```bash
git -C /Users/mdproctor/claude/public/cc-praxis add -A
git -C /Users/mdproctor/claude/public/cc-praxis commit -m "chore: workspace transition — cc-praxis → soredium

Updated project path, add-dir, symlinks, and blog routing.
Created blog-routing.yaml for hortora.github.io diary routing."
git -C /Users/mdproctor/claude/public/cc-praxis push

git -C /Users/mdproctor/claude/public/hortora add -A
git -C /Users/mdproctor/claude/public/hortora commit -m "chore: update soredium member description after merge"
git -C /Users/mdproctor/claude/public/hortora push
```

---

## Task 13: Global Config Updates

**Files:**
- Modify: `~/.claude/working-style.md`
- Modify: memory files in `~/.claude/projects/-Users-mdproctor-claude-cc-praxis/memory/`

- [ ] **Step 1: Update working-style.md**

Edit `~/.claude/working-style.md` lines 106–112. Change:

```
## Skill Edits — cc-praxis is the source of truth

Always edit skills in `~/claude/cc-praxis/<skill>/SKILL.md` first, then run `sync-local`...

# 1. Edit ~/claude/cc-praxis/<skill>/SKILL.md
# 2. Commit and push cc-praxis
```

To:

```
## Skill Edits — soredium is the source of truth

Always edit skills in `~/claude/hortora/soredium/<skill>/SKILL.md` first, then run `sync-local`...

# 1. Edit ~/claude/hortora/soredium/<skill>/SKILL.md
# 2. Commit and push soredium
```

- [ ] **Step 2: Check for other @-included files with cc-praxis references**

```bash
grep -rn "cc-praxis" ~/.claude/engagement.md ~/.claude/working-style.md ~/.claude/document-boundaries.md ~/.claude/design-implementation.md 2>/dev/null
```

Fix any references found.

- [ ] **Step 3: Update memory files**

Check memory files for cc-praxis path references:

```bash
grep -rn "cc-praxis" ~/.claude/projects/-Users-mdproctor-claude-cc-praxis/memory/ 2>/dev/null
```

Update any that reference cc-praxis paths to soredium paths. These are auto-memory entries that inform future sessions.

- [ ] **Step 4: Run sync-local from merged soredium**

```bash
python3 /Users/mdproctor/claude/hortora/soredium/scripts/claude-skill sync-local --all -y
```

Expected: All 33 skills synced to `~/.claude/skills/`. Verify with:
```bash
python3 /Users/mdproctor/claude/hortora/soredium/scripts/claude-skill list
```

---

## Task 14: Push Soredium and Verify

**Files:**
- No file changes — push and verify

- [ ] **Step 1: Push soredium**

```bash
git -C /Users/mdproctor/claude/hortora/soredium push
```

- [ ] **Step 2: Verify marketplace.json is accessible**

```bash
curl -s https://raw.githubusercontent.com/Hortora/soredium/main/.claude-plugin/marketplace.json | python3 -c "import json,sys; d=json.load(sys.stdin); print(f'{len(d[\"plugins\"])} plugins')"
```

Expected: `31 plugins`

- [ ] **Step 3: Verify blog site**

Check https://hortora.github.io/blog/ — all 41 posts should be listed with sequential entry numbers.

- [ ] **Step 4: Tag post-merge**

```bash
git -C /Users/mdproctor/claude/hortora/soredium tag post-merge
git -C /Users/mdproctor/claude/hortora/soredium push origin post-merge
```

---

## Task 15: Archive cc-praxis

**Files:**
- No local file changes — GitHub operation

- [ ] **Step 1: Archive the repository**

```bash
gh repo archive mdproctor/cc-praxis --yes
```

Expected: Repository archived. Verify with `gh repo view mdproctor/cc-praxis --json isArchived`.

- [ ] **Step 2: Verify archived state**

```bash
gh repo view mdproctor/cc-praxis --json isArchived --jq '.isArchived'
```

Expected: `true`

---

## Task Dependency Summary

```
Task 1 (backup) → Task 2 (blog migration) → Task 3 (git merge)
Task 3 → Task 4 (remove Jekyll files)
Task 3 → Task 5 (marketplace.json)
Task 3 → Task 6 (scripts/claude-skill) → Task 7 (test_claude_skill.py)
Task 7 → Task 8 (full test suite)
Task 3 → Task 9 (CLAUDE.md)
Task 3 → Task 10 (README.md)
Task 8 → Task 11 (reference sweep)
Task 11 → Task 12 (workspace transition)
Task 12 → Task 13 (global config)
Task 13 → Task 14 (push and verify)
Task 14 → Task 15 (archive)
```

Tasks 4–7 and 9–10 can run in parallel after Task 3. Task 8 gates the reference sweep. Everything after Task 11 is sequential.
