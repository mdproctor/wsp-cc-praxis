# Revert subtype log→diary Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Revert the mistaken `subtype: diary` → `subtype: log` rename by updating all ~108 affected markdown files and 4 skill/doc files back to `subtype: diary`.

**Architecture:** A permanent Python batch script (`scripts/revert_diary_subtype.py`) handles the frontmatter-aware revert across all repos under `~/claude/`. Four targeted text edits fix the skill/doc files that serve as the source of truth for future sessions. The script is kept for re-running during the eventual-consistency period while active Claude sessions pick up the corrected skills.

**Tech Stack:** Python 3, `re` module, `pathlib`. No external dependencies.

---

## File Map

| Action | Path | Purpose |
|--------|------|---------|
| Create | `cc-praxis/scripts/revert_diary_subtype.py` | Batch revert script — permanent, idempotent |
| Modify | `cc-praxis/write-blog/SKILL.md` | 3 lines: `subtype: log` → `subtype: diary` |
| Modify | `cc-praxis/publish-blog/SKILL.md` | 1 line: subtype enumeration |
| Modify | `cc-praxis/docs/skills-catalog.md` | 1 line: subtype field description |
| Modify | `cc-praxis/docs/content-taxonomy-article-notes.md` | 1 line: taxonomy table |

---

## Task 1: Write the revert script

**Files:**
- Create: `scripts/revert_diary_subtype.py`

- [ ] **Step 1: Write the script**

```python
#!/usr/bin/env python3
"""Revert subtype: log → subtype: diary in YAML frontmatter.

Idempotent — files already at subtype: diary are untouched.
Re-run during eventual consistency period as active sessions pick up the fix.

Usage:
    python3 scripts/revert_diary_subtype.py           # dry-run (default)
    python3 scripts/revert_diary_subtype.py --apply   # apply changes
"""

import argparse
import re
import sys
from pathlib import Path

ROOT = Path.home() / "claude"

SKIP_PARTS = {".git", "_site", "adr"}
SKIP_DIR_NAMES = {"superpowers"}   # skips specs/, plans/, snapshots/ under superpowers/
SKIP_FILENAMES = {"HANDOFF.md"}

FRONTMATTER_RE = re.compile(r'^---\n(.*?)\n---\n?(.*)', re.DOTALL)
SUBTYPE_LOG_RE = re.compile(r'^subtype: log$', re.MULTILINE)


def should_skip(path: Path) -> bool:
    parts = set(path.parts)
    if parts & SKIP_PARTS:
        return True
    if path.name in SKIP_FILENAMES:
        return True
    # Skip anything inside a 'superpowers' directory
    for part in path.parts:
        if part in SKIP_DIR_NAMES:
            return True
    return False


def process(path: Path, apply: bool) -> bool:
    """Return True if file was changed (or would change in dry-run)."""
    try:
        raw = path.read_bytes()
    except (PermissionError, OSError):
        return False

    content = raw.decode("utf-8", errors="replace").replace("\r\n", "\n")

    m = FRONTMATTER_RE.match(content)
    if not m:
        return False

    frontmatter, body = m.group(1), m.group(2)

    if "subtype: log" not in frontmatter:
        return False

    new_frontmatter = SUBTYPE_LOG_RE.sub("subtype: diary", frontmatter)
    new_content = f"---\n{new_frontmatter}\n---\n{body}"

    if apply:
        path.write_text(new_content, encoding="utf-8")

    return True


def main() -> None:
    parser = argparse.ArgumentParser(description=__doc__,
                                     formatter_class=argparse.RawDescriptionHelpFormatter)
    parser.add_argument("--apply", action="store_true",
                        help="Apply changes (default is dry-run)")
    args = parser.parse_args()

    by_repo: dict[str, list[Path]] = {}

    for md in ROOT.rglob("*.md"):
        if should_skip(md):
            continue
        if process(md, apply=args.apply):
            try:
                rel = md.relative_to(ROOT)
                repo_key = str(Path(*rel.parts[:2])) if len(rel.parts) > 2 else rel.parts[0]
            except ValueError:
                repo_key = "?"
            by_repo.setdefault(repo_key, []).append(md)

    action = "Changed" if args.apply else "Would change"
    total = 0
    for repo_key in sorted(by_repo):
        files = by_repo[repo_key]
        print(f"\n{repo_key}: {len(files)} file(s)")
        for f in sorted(files):
            print(f"  {f.relative_to(ROOT)}")
        total += len(files)

    print(f"\n{action}: {total} file(s) total")
    if not args.apply:
        print("Re-run with --apply to apply changes.")


if __name__ == "__main__":
    main()
```

- [ ] **Step 2: Verify script syntax**

```bash
python3 -c "import ast; ast.parse(open('scripts/revert_diary_subtype.py').read()); print('syntax ok')"
```

Expected: `syntax ok`

---

## Task 2: Dry-run — verify scope before touching anything

**Files:** none modified

- [ ] **Step 1: Run dry-run from cc-praxis root**

```bash
python3 scripts/revert_diary_subtype.py
```

Expected: output grouped by repo showing ~108 files. Check:
- `mdproctor.github.io/_notes`: ~54 files
- `public/casehub/...`: ~35 files across multiple subdirs
- `drools`: ~4 files
- `hortora/hortora.github.io`: ~2 files
- `public/quarkmind`: ~5 files
- `public/cc-praxis`: ~1 file
- `cc-praxis/docs/_posts`: ~2 files

- [ ] **Step 2: Spot-check one entry**

```bash
head -10 ~/claude/mdproctor.github.io/_notes/2026-05-21-mdp01-two-fixes-finally-together.md
```

Expected: `subtype: log` in frontmatter — confirming it's a genuine target, not a false positive.

---

## Task 3: Apply the revert to all markdown files

**Files:** ~108 markdown files across 7 git repos

- [ ] **Step 1: Apply**

```bash
python3 scripts/revert_diary_subtype.py --apply
```

Expected: same file list as dry-run, with "Changed: N file(s) total"

- [ ] **Step 2: Verify no subtype: log remains in posts**

```bash
grep -rn "subtype: log" ~/claude/ --include="*.md" \
  | grep -v ".git/" | grep -v "superpowers/" | grep -v "HANDOFF.md" | grep -v "adr/"
```

Expected: no output (zero matches outside excluded paths)

- [ ] **Step 3: Verify subtype: diary is present where expected**

```bash
grep -c "subtype: diary" ~/claude/mdproctor.github.io/_notes/*.md | grep -v ":0" | wc -l
```

Expected: count matches number of notes (all now have `subtype: diary`)

---

## Task 4: Targeted skill and doc edits

**Files:**
- Modify: `write-blog/SKILL.md`
- Modify: `publish-blog/SKILL.md`
- Modify: `docs/skills-catalog.md`
- Modify: `docs/content-taxonomy-article-notes.md`

- [ ] **Step 1: Fix write-blog/SKILL.md (3 occurrences)**

Find and replace — all three must change:

| Old | New |
|-----|-----|
| `` - `subtype: log` (if note — omit for articles) `` | `` - `subtype: diary` (if note — omit for articles) `` |
| `subtype: log                                     # omit for articles` | `subtype: diary                                     # omit for articles` |
| `` - ✅ `subtype: log` present for note entries; omitted for articles `` | `` - ✅ `subtype: diary` present for note entries; omitted for articles `` |

```bash
grep -n "subtype: log\|subtype.*log" write-blog/SKILL.md
```

Expected: 3 lines. Make the edits, then verify:

```bash
grep -n "subtype: log" write-blog/SKILL.md
```

Expected: no output.

- [ ] **Step 2: Fix publish-blog/SKILL.md**

Line 85: change:
```
- `subtype` — log | ... (notes only)
```
to:
```
- `subtype` — diary | ... (notes only)
```

Verify:
```bash
grep -n "subtype.*log" publish-blog/SKILL.md
```
Expected: no output.

- [ ] **Step 3: Fix docs/skills-catalog.md**

Line 485: change:
```
- `subtype`: `log` for note entries
```
to:
```
- `subtype`: `diary` for note entries
```

Verify:
```bash
grep -n "subtype.*log" docs/skills-catalog.md
```
Expected: no output.

- [ ] **Step 4: Fix docs/content-taxonomy-article-notes.md**

Line 1907: change:
```
| subtype | log / musing / idea / tutorial / how-to / explanation / commentary / essay / release / event / industry |
```
to:
```
| subtype | diary / musing / idea / tutorial / how-to / explanation / commentary / essay / release / event / industry |
```

Verify:
```bash
grep -n "^| subtype |" docs/content-taxonomy-article-notes.md
```
Expected: line shows `diary / musing / ...`

---

## Task 5: Sync skills

- [ ] **Step 1: Sync all skills to ~/.claude/skills/**

```bash
python3 scripts/claude-skill sync-local --all -y
```

Expected: sync completes, skills updated. Future sessions now generate `subtype: diary`.

---

## Task 6: Run tests

**Files:** none modified (tests already expect `diary`)

- [ ] **Step 1: Run blog frontmatter and Jekyll page tests**

```bash
python3 -m pytest tests/test_jekyll_pages.py tests/test_blog_frontmatter.py -v
```

Expected: all tests pass. No test changes needed — they were already written expecting `diary`.

---

## Task 7: Commit cc-praxis

- [ ] **Step 1: Stage and commit**

```bash
git -C ~/claude/cc-praxis add scripts/revert_diary_subtype.py write-blog/SKILL.md publish-blog/SKILL.md docs/skills-catalog.md docs/content-taxonomy-article-notes.md
git -C ~/claude/cc-praxis add $(git -C ~/claude/cc-praxis diff --name-only docs/_posts/)

git -C ~/claude/cc-praxis commit -m "chore: revert subtype log→diary (Closes #96)"
```

Expected: commit succeeds on branch `issue-96-diary-to-log-sweep`

---

## Task 8: Commit mdproctor.github.io

- [ ] **Step 1: Stage and commit**

```bash
git -C ~/claude/mdproctor.github.io add _notes/
git -C ~/claude/mdproctor.github.io commit -m "chore: revert subtype log→diary (refs #96)"
git -C ~/claude/mdproctor.github.io push
```

---

## Task 9: Commit drools

- [ ] **Step 1: Stage and commit**

```bash
git -C ~/claude/drools add blog/
git -C ~/claude/drools commit -m "chore: revert subtype log→diary (refs #96)"
git -C ~/claude/drools push
```

---

## Task 10: Commit hortora.github.io

- [ ] **Step 1: Stage and commit**

```bash
git -C ~/claude/hortora/hortora.github.io add _posts/
git -C ~/claude/hortora/hortora.github.io commit -m "chore: revert subtype log→diary (refs #96)"
git -C ~/claude/hortora/hortora.github.io push
```

---

## Task 11: Commit public workspaces (cc-praxis, quarkmind, casehub)

- [ ] **Step 1: Commit public/cc-praxis workspace**

```bash
git -C ~/claude/public/cc-praxis add blog/
git -C ~/claude/public/cc-praxis commit -m "chore: revert subtype log→diary (refs #96)"
git -C ~/claude/public/cc-praxis push
```

- [ ] **Step 2: Commit public/quarkmind workspace**

```bash
git -C ~/claude/public/quarkmind add blog/
git -C ~/claude/public/quarkmind commit -m "chore: revert subtype log→diary (refs #96)"
git -C ~/claude/public/quarkmind push
```

- [ ] **Step 3: Commit public/casehub workspace**

```bash
git -C ~/claude/public/casehub add .
git -C ~/claude/public/casehub status  # review staged files before committing
git -C ~/claude/public/casehub commit -m "chore: revert subtype log→diary (refs #96)"
git -C ~/claude/public/casehub push
```

---

## Task 12: Final verification

- [ ] **Step 1: Confirm zero stray subtype: log in posts**

```bash
grep -rn "subtype: log" ~/claude/ --include="*.md" \
  | grep -v ".git/" | grep -v "superpowers/" | grep -v "HANDOFF.md" | grep -v "adr/"
```

Expected: no output.

- [ ] **Step 2: Re-run script to confirm idempotency**

```bash
python3 ~/claude/cc-praxis/scripts/revert_diary_subtype.py
```

Expected: `Would change: 0 file(s) total`

---

## Ongoing: Re-run during eventual consistency

Active Claude sessions that loaded the old skills will continue generating `subtype: log`
until they reload. Run the script periodically until drift stops appearing:

```bash
python3 ~/claude/cc-praxis/scripts/revert_diary_subtype.py          # check
python3 ~/claude/cc-praxis/scripts/revert_diary_subtype.py --apply  # fix
# commit affected repos as in Tasks 8-11
```

Once `subtype: log` stops appearing in new posts, eventual consistency is reached.
The script can then be retired (or kept as a safety check).
