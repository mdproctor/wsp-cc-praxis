# Verify Image References Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #484 — Verify image references render before stamp and publication
**Issue group:** #484

**Goal:** Ensure images travel with their .md files during promotion and verify image references resolve before stamp and publication.

**Architecture:** Extract image ref parsing into a shared utility (`extract_image_refs`), use it to co-promote images alongside .md files in `artifact_promote.py`, verify at stamp time via `close_verify.py`, and pre-check before blog publication. Two-layer approach: promote fixes the root cause, verify catches stragglers.

**Tech Stack:** Python 3, pytest, pathlib, shutil, re

## Global Constraints

- `extract_image_refs` reuses existing `_IMAGE_RE`, `_HTML_IMG_RE`, `_SKIP_IMAGE_REFS` constants
- Image verification findings are WARN severity, not ERROR (non-blocking)
- All new `.py` scripts require tests committed together (protocol: externalised-scripts-require-tests)
- Follow existing `verification/` module patterns: pure functions, `list[Finding]` return types, no side effects in check functions

---

## Batch 1: Shared image extraction utility and tests

### Task 1: Add `extract_image_refs()` and refactor `check_broken_images()`

**Files:**
- Modify: `verification/repo_checks.py:280-333`
- Create: `tests/test_image_refs.py`

**Interfaces:**
- Produces: `extract_image_refs(md_file: Path) -> list[tuple[str, Path]]` — returns `(raw_ref, resolved_absolute_path)` for all local image references in a .md file. Skips URLs, placeholders, chrome:// refs.
- Produces: `check_broken_images(md_file: Path) -> list[Finding]` — unchanged signature, now calls `extract_image_refs` internally

- [ ] **Step 1: Write failing tests for `extract_image_refs`**

Create `tests/test_image_refs.py`:

```python
"""Tests for extract_image_refs() and check_broken_images() in verification/repo_checks.py."""
import pytest
from pathlib import Path

from verification.repo_checks import extract_image_refs, check_broken_images


class TestExtractImageRefs:
    def test_markdown_image(self, tmp_path):
        md = tmp_path / "doc.md"
        md.write_text("![alt](images/fig1.png)\n")
        refs = extract_image_refs(md)
        assert len(refs) == 1
        assert refs[0][0] == "images/fig1.png"
        assert refs[0][1] == (tmp_path / "images/fig1.png").resolve()

    def test_markdown_image_no_alt(self, tmp_path):
        md = tmp_path / "doc.md"
        md.write_text("![](screenshot.png)\n")
        refs = extract_image_refs(md)
        assert len(refs) == 1
        assert refs[0][0] == "screenshot.png"

    def test_html_img(self, tmp_path):
        md = tmp_path / "doc.md"
        md.write_text('<img src="assets/logo.svg">\n')
        refs = extract_image_refs(md)
        assert len(refs) == 1
        assert refs[0][0] == "assets/logo.svg"

    def test_html_img_single_quotes(self, tmp_path):
        md = tmp_path / "doc.md"
        md.write_text("<img src='assets/logo.svg'>\n")
        refs = extract_image_refs(md)
        assert len(refs) == 1
        assert refs[0][0] == "assets/logo.svg"

    def test_skips_http_url(self, tmp_path):
        md = tmp_path / "doc.md"
        md.write_text("![alt](https://example.com/img.png)\n")
        refs = extract_image_refs(md)
        assert refs == []

    def test_skips_https_url(self, tmp_path):
        md = tmp_path / "doc.md"
        md.write_text("![alt](https://cdn.example.com/img.png)\n")
        refs = extract_image_refs(md)
        assert refs == []

    def test_skips_data_url(self, tmp_path):
        md = tmp_path / "doc.md"
        md.write_text("![alt](data://base64stuff)\n")
        refs = extract_image_refs(md)
        assert refs == []

    def test_skips_protocol_relative(self, tmp_path):
        md = tmp_path / "doc.md"
        md.write_text("![alt](//cdn.example.com/img.png)\n")
        refs = extract_image_refs(md)
        assert refs == []

    def test_skips_placeholder(self, tmp_path):
        md = tmp_path / "doc.md"
        md.write_text("![alt](x)\n![alt]({thumb_src})\n![alt](path)\n")
        refs = extract_image_refs(md)
        assert refs == []

    def test_skips_chrome_url(self, tmp_path):
        md = tmp_path / "doc.md"
        md.write_text("![alt](chrome://something)\n")
        refs = extract_image_refs(md)
        assert refs == []

    def test_root_relative_path(self, tmp_path):
        (tmp_path / ".git").mkdir()
        subdir = tmp_path / "docs"
        subdir.mkdir()
        md = subdir / "doc.md"
        md.write_text("![alt](/assets/img/fig.png)\n")
        refs = extract_image_refs(md)
        assert len(refs) == 1
        assert refs[0][0] == "/assets/img/fig.png"
        assert refs[0][1] == tmp_path / "assets/img/fig.png"

    def test_multiple_refs_on_one_line(self, tmp_path):
        md = tmp_path / "doc.md"
        md.write_text("![a](one.png) text ![b](two.png)\n")
        refs = extract_image_refs(md)
        assert len(refs) == 2
        assert refs[0][0] == "one.png"
        assert refs[1][0] == "two.png"

    def test_nonexistent_image_still_returned(self, tmp_path):
        md = tmp_path / "doc.md"
        md.write_text("![alt](does-not-exist.png)\n")
        refs = extract_image_refs(md)
        assert len(refs) == 1

    def test_url_encoded_path(self, tmp_path):
        md = tmp_path / "doc.md"
        md.write_text("![alt](images/my%20image.png)\n")
        refs = extract_image_refs(md)
        assert len(refs) == 1
        assert refs[0][0] == "images/my image.png"

    def test_empty_file(self, tmp_path):
        md = tmp_path / "doc.md"
        md.write_text("")
        refs = extract_image_refs(md)
        assert refs == []

    def test_no_images(self, tmp_path):
        md = tmp_path / "doc.md"
        md.write_text("# Hello\n\nJust text.\n")
        refs = extract_image_refs(md)
        assert refs == []


class TestCheckBrokenImagesRefactored:
    def test_existing_image_no_findings(self, tmp_path):
        img = tmp_path / "fig.png"
        img.write_bytes(b"\x89PNG")
        md = tmp_path / "doc.md"
        md.write_text("![alt](fig.png)\n")
        findings = check_broken_images(md)
        assert findings == []

    def test_missing_image_returns_finding(self, tmp_path):
        md = tmp_path / "doc.md"
        md.write_text("![alt](missing.png)\n")
        findings = check_broken_images(md)
        assert len(findings) == 1
        assert findings[0].severity == "ERROR"
        assert findings[0].category == "broken-image"
        assert "missing.png" in findings[0].message

    def test_url_reference_no_findings(self, tmp_path):
        md = tmp_path / "doc.md"
        md.write_text("![alt](https://example.com/img.png)\n")
        findings = check_broken_images(md)
        assert findings == []

    def test_mixed_existing_and_missing(self, tmp_path):
        img = tmp_path / "exists.png"
        img.write_bytes(b"\x89PNG")
        md = tmp_path / "doc.md"
        md.write_text("![a](exists.png)\n![b](missing.png)\n")
        findings = check_broken_images(md)
        assert len(findings) == 1
        assert "missing.png" in findings[0].message
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_image_refs.py -v`
Expected: FAIL — `extract_image_refs` not defined

- [ ] **Step 3: Implement `extract_image_refs` and refactor `check_broken_images`**

In `verification/repo_checks.py`, add `extract_image_refs` before `check_broken_images` and refactor:

```python
def extract_image_refs(md_file: Path) -> list[tuple[str, Path]]:
    """Return (raw_ref, resolved_path) for all local image references in a .md file.

    Skips URLs, placeholders, chrome:// refs. Resolves relative paths
    against the .md file's parent; root-relative paths against the repo root.
    """
    from urllib.parse import unquote

    refs_out: list[tuple[str, Path]] = []
    try:
        content = md_file.read_text(errors="replace")
    except OSError:
        return refs_out

    repo_root = md_file.parent
    while repo_root.parent != repo_root:
        if (repo_root / ".git").exists():
            break
        repo_root = repo_root.parent

    for line in content.splitlines():
        raw_refs: list[str] = []
        for m in _IMAGE_RE.finditer(line):
            raw_refs.append(m.group(2))
        for m in _HTML_IMG_RE.finditer(line):
            raw_refs.append(m.group(1))

        for ref in raw_refs:
            ref = unquote(ref).strip()
            if ref.startswith(("http://", "https://", "data://", "//")):
                continue
            if ref in _SKIP_IMAGE_REFS:
                continue
            if ref.startswith("chrome://"):
                continue

            if ref.startswith("/"):
                resolved = repo_root / ref.lstrip("/")
            else:
                resolved = (md_file.parent / ref).resolve()

            refs_out.append((ref, resolved))

    return refs_out


def check_broken_images(md_file: Path) -> list[Finding]:
    """Check that all image references in a .md file resolve to real files."""
    findings: list[Finding] = []
    for ref, resolved in extract_image_refs(md_file):
        if not resolved.exists():
            findings.append(Finding(
                "ERROR", "broken-image",
                f"{md_file.name} — image not found: {ref}",
            ))
    return findings
```

Note: the refactored `check_broken_images` drops the `:line_num` from the finding message since `extract_image_refs` doesn't track line numbers. This is acceptable — the ref string itself identifies the problem.

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_image_refs.py -v`
Expected: all PASS

- [ ] **Step 5: Commit**

```bash
git add verification/repo_checks.py tests/test_image_refs.py
git commit -m "feat(#484): extract_image_refs utility and refactored check_broken_images

Add extract_image_refs() that returns (raw_ref, resolved_path) pairs
for local image references in .md files. Refactor check_broken_images()
to use it internally.  Refs #484"
```

---

## Batch 2: Image co-promotion in artifact_promote.py

### Task 2: Co-promote images alongside .md files in `to_project()`

**Files:**
- Modify: `work-end/artifact_promote.py:158-236`
- Create: `tests/test_image_promotion.py`

**Interfaces:**
- Consumes: `extract_image_refs(md_file: Path) -> list[tuple[str, Path]]` from `verification.repo_checks`
- Produces: Modified `to_project()` that copies referenced images alongside .md artifacts

- [ ] **Step 1: Write failing test for image co-promotion in to_project**

Create `tests/test_image_promotion.py`:

```python
"""Tests for image co-promotion in artifact_promote.py."""
import subprocess
from pathlib import Path
from unittest.mock import patch

import pytest


def _init_git(path: Path) -> None:
    subprocess.run(["git", "-C", str(path), "init"], capture_output=True)
    subprocess.run(["git", "-C", str(path), "config", "user.email", "t@t.com"], capture_output=True)
    subprocess.run(["git", "-C", str(path), "config", "user.name", "test"], capture_output=True)
    (path / ".gitkeep").touch()
    subprocess.run(["git", "-C", str(path), "add", "."], capture_output=True)
    subprocess.run(["git", "-C", str(path), "commit", "-m", "init"], capture_output=True)


class TestToProjectImageCopromotion:
    def test_md_with_image_promotes_both(self, tmp_path, capsys):
        ws = tmp_path / "workspace"
        ws.mkdir()
        _init_git(ws)
        proj = tmp_path / "project"
        proj.mkdir()
        _init_git(proj)

        blog_dir = ws / "docs" / "blog"
        blog_dir.mkdir(parents=True)
        img_dir = ws / "docs" / "blog" / "images"
        img_dir.mkdir()
        img = img_dir / "fig1.png"
        img.write_bytes(b"\x89PNG")
        md = blog_dir / "entry.md"
        md.write_text("![alt](images/fig1.png)\n")

        import sys
        sys.path.insert(0, str(Path(__file__).parent.parent / "work-end"))
        sys.path.insert(0, str(Path(__file__).parent.parent / "project"))
        from artifact_promote import to_project

        with patch("artifact_promote._push_or_report"):
            rc = to_project(str(proj), str(ws), {
                "artifacts": "docs/blog/entry.md",
                "covers": "",
            })

        assert rc == 0
        assert (proj / "docs/blog/entry.md").exists()
        assert (proj / "docs/blog/images/fig1.png").exists()

    def test_md_with_missing_image_promotes_md_only(self, tmp_path, capsys):
        ws = tmp_path / "workspace"
        ws.mkdir()
        _init_git(ws)
        proj = tmp_path / "project"
        proj.mkdir()
        _init_git(proj)

        blog_dir = ws / "docs" / "blog"
        blog_dir.mkdir(parents=True)
        md = blog_dir / "entry.md"
        md.write_text("![alt](images/missing.png)\n")

        import sys
        sys.path.insert(0, str(Path(__file__).parent.parent / "work-end"))
        sys.path.insert(0, str(Path(__file__).parent.parent / "project"))
        from artifact_promote import to_project

        with patch("artifact_promote._push_or_report"):
            rc = to_project(str(proj), str(ws), {
                "artifacts": "docs/blog/entry.md",
                "covers": "",
            })

        assert rc == 0
        assert (proj / "docs/blog/entry.md").exists()
        assert not (proj / "docs/blog/images/missing.png").exists()

    def test_nested_image_dir_preserved(self, tmp_path, capsys):
        ws = tmp_path / "workspace"
        ws.mkdir()
        _init_git(ws)
        proj = tmp_path / "project"
        proj.mkdir()
        _init_git(proj)

        spec_dir = ws / "specs" / "feature"
        spec_dir.mkdir(parents=True)
        img_dir = spec_dir / "images" / "screenshots"
        img_dir.mkdir(parents=True)
        img = img_dir / "screen1.png"
        img.write_bytes(b"\x89PNG")
        md = spec_dir / "design.md"
        md.write_text("![screen](images/screenshots/screen1.png)\n")

        import sys
        sys.path.insert(0, str(Path(__file__).parent.parent / "work-end"))
        sys.path.insert(0, str(Path(__file__).parent.parent / "project"))
        from artifact_promote import to_project

        with patch("artifact_promote._push_or_report"):
            rc = to_project(str(proj), str(ws), {
                "artifacts": "specs/feature/design.md",
                "covers": "",
            })

        assert rc == 0
        assert (proj / "specs/feature/images/screenshots/screen1.png").exists()
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_image_promotion.py -v`
Expected: FAIL — images not co-promoted

- [ ] **Step 3: Add image co-promotion to `to_project()`**

In `work-end/artifact_promote.py`, add the import at the top of the file (after the existing imports):

```python
_verification_dir = str(Path(__file__).resolve().parent.parent / "verification")
if _verification_dir not in sys.path:
    sys.path.insert(0, _verification_dir)
```

Add a helper function `_copromote_images` and call it from `to_project()` after each .md artifact is copied:

```python
def _copromote_images(md_source: Path, md_dest: Path, repo_cwd: str) -> int:
    """Copy images referenced by a .md file from source to destination.

    Returns the number of images copied.
    """
    from verification.repo_checks import extract_image_refs

    if not md_source.suffix == ".md":
        return 0

    copied = 0
    for raw_ref, resolved_src in extract_image_refs(md_source):
        if not resolved_src.exists():
            continue
        # Compute destination: same relative path from .md file
        rel = Path(raw_ref)
        if rel.is_absolute():
            continue
        img_dest = md_dest.parent / rel
        img_dest.parent.mkdir(parents=True, exist_ok=True)
        shutil.copy2(str(resolved_src), str(img_dest))
        img_rel = str(img_dest.relative_to(Path(repo_cwd)))
        git(repo_cwd, "add", img_rel)
        copied += 1
    return copied
```

In `to_project()`, after the `shutil.copy2(src, dst)` call and `git add` (around line 206-208), insert:

```python
        if artifact.endswith(".md"):
            _copromote_images(src, dst, project)
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_image_promotion.py -v`
Expected: all PASS

- [ ] **Step 5: Add same co-promotion to `to_workspace_main()`**

In `to_workspace_main()`, after `shutil.copy2(str(src), str(dst))` (line 128) and `git add` (line 129), insert:

```python
            if artifact.endswith(".md"):
                _copromote_images(src, dst, str(wt))
```

- [ ] **Step 6: Commit**

```bash
git add work-end/artifact_promote.py tests/test_image_promotion.py
git commit -m "feat(#484): co-promote images alongside .md files during artifact promotion

artifact_promote.py now copies referenced images when promoting .md
artifacts to project or workspace-main destinations.  Refs #484"
```

---

## Batch 3: Verification gate and publish-blog update

### Task 3: Add `check_promoted_images()` to close-time verification

**Files:**
- Modify: `verification/close_checks.py`
- Modify: `verification/close_verify.py:36-62`
- Create: `tests/test_close_checks_images.py`

**Interfaces:**
- Consumes: `check_broken_images(md_file: Path) -> list[Finding]` from `verification.repo_checks`
- Produces: `check_promoted_images(project: Path, workspace: Path, branch: str, base: str) -> list[Finding]`

- [ ] **Step 1: Write failing test for `check_promoted_images`**

Create `tests/test_close_checks_images.py`:

```python
"""Tests for check_promoted_images in verification/close_checks.py."""
import subprocess
from pathlib import Path

import pytest

from verification import Finding


def _init_git(path: Path) -> None:
    subprocess.run(["git", "-C", str(path), "init"], capture_output=True)
    subprocess.run(["git", "-C", str(path), "config", "user.email", "t@t.com"], capture_output=True)
    subprocess.run(["git", "-C", str(path), "config", "user.name", "test"], capture_output=True)
    (path / ".gitkeep").touch()
    subprocess.run(["git", "-C", str(path), "add", "."], capture_output=True)
    subprocess.run(["git", "-C", str(path), "commit", "-m", "init"], capture_output=True)


class TestCheckPromotedImages:
    def test_all_images_present_no_findings(self, tmp_path):
        proj = tmp_path / "project"
        proj.mkdir()
        _init_git(proj)
        docs = proj / "docs" / "blog"
        docs.mkdir(parents=True)
        img = docs / "images" / "fig.png"
        img.parent.mkdir()
        img.write_bytes(b"\x89PNG")
        md = docs / "entry.md"
        md.write_text("![alt](images/fig.png)\n")

        from verification.close_checks import check_promoted_images
        findings = check_promoted_images(proj)
        assert findings == []

    def test_missing_image_returns_warn(self, tmp_path):
        proj = tmp_path / "project"
        proj.mkdir()
        _init_git(proj)
        docs = proj / "docs" / "blog"
        docs.mkdir(parents=True)
        md = docs / "entry.md"
        md.write_text("![alt](images/missing.png)\n")

        from verification.close_checks import check_promoted_images
        findings = check_promoted_images(proj)
        assert len(findings) >= 1
        assert all(f.severity == "WARN" for f in findings)
        assert any("missing.png" in f.message for f in findings)

    def test_url_refs_not_flagged(self, tmp_path):
        proj = tmp_path / "project"
        proj.mkdir()
        _init_git(proj)
        docs = proj / "docs" / "blog"
        docs.mkdir(parents=True)
        md = docs / "entry.md"
        md.write_text("![alt](https://example.com/img.png)\n")

        from verification.close_checks import check_promoted_images
        findings = check_promoted_images(proj)
        assert findings == []
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_close_checks_images.py -v`
Expected: FAIL — `check_promoted_images` not defined

- [ ] **Step 3: Implement `check_promoted_images` in `close_checks.py`**

Add to `verification/close_checks.py`:

```python
def check_promoted_images(project: Path) -> list[Finding]:
    """Check that promoted .md files have all referenced images present.

    Scans docs/blog/ and docs/specs/ in the project for broken image refs.
    Returns WARN findings (not blocking).
    """
    from verification.repo_checks import check_broken_images

    findings: list[Finding] = []
    scan_dirs = [project / "docs" / "blog", project / "docs" / "specs"]
    for scan_dir in scan_dirs:
        if not scan_dir.is_dir():
            continue
        for md_file in scan_dir.rglob("*.md"):
            for f in check_broken_images(md_file):
                findings.append(Finding(
                    "WARN", "promoted-broken-image", f.message,
                ))
    return findings
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_close_checks_images.py -v`
Expected: all PASS

- [ ] **Step 5: Wire `check_promoted_images` into `close_verify.py`**

In `verification/close_verify.py`, add the import:

```python
from verification.close_checks import (
    check_canonical_sync,
    check_landed_completeness,
    check_no_open_findings,
    check_no_stale_scaffold,
    check_slot_archive_status,
    check_promoted_images,      # NEW
    resolve_canonical_repos,
)
```

In `_run_branch_checks()`, add after the `artifacts_promoted` check (before `no_open_findings`):

```python
        checks.append(("promoted_images", check_promoted_images(project)))
```

This goes in the `else` branch (non-main), after line 55 and before line 56.

- [ ] **Step 6: Commit**

```bash
git add verification/close_checks.py verification/close_verify.py tests/test_close_checks_images.py
git commit -m "feat(#484): add image verification gate to close-time checks

check_promoted_images() scans promoted .md files for broken image refs
and reports WARN findings. Wired into close_verify for stamp-time
verification.  Refs #484"
```

### Task 4: Update `publish-blog/SKILL.md` with image verification step

**Files:**
- Modify: `publish-blog/SKILL.md:118-153`

**Interfaces:**
- Consumes: `check_broken_images(md_file: Path) -> list[Finding]` from `verification.repo_checks`
- Consumes: `extract_image_refs(md_file: Path) -> list[tuple[str, Path]]` from `verification.repo_checks`

- [ ] **Step 1: Add image verification between Steps 5 and 6 in SKILL.md**

In `publish-blog/SKILL.md`, insert a new Step 5b after Step 5 (validate destinations) and before Step 6 (copy entries). Renumber Steps 6-9 to 7-10:

```markdown
### Step 5b — Verify image references

Before copying entries, verify all image references resolve:

```bash
python3 -c "
from pathlib import Path
from verification.repo_checks import check_broken_images
md = Path('<entry_path>')
findings = check_broken_images(md)
for f in findings:
    print(f'{f.severity}: {f.message}')
"
```

If any broken refs found, present them to the user:
```
⚠️ Broken image references found:
  entry.md — image not found: images/missing.png

Proceed anyway? (y/n)
```

If user says `n`, stop. If `y`, continue with broken refs noted.

### Step 6 — Copy entries and images to destinations

For each (entry, destination) pair approved by the user:

1. Copy the entry:
```bash
python3 ~/.claude/skills/publish-blog/blog_publish.py copy-entry \
  "$BLOG_DIR/<filename>" "<target_dir>"
```

2. Copy referenced images alongside the entry:
```python
from verification.repo_checks import extract_image_refs
for raw_ref, resolved in extract_image_refs(Path(entry_path)):
    if resolved.exists():
        dest = Path(target_dir) / raw_ref
        dest.parent.mkdir(parents=True, exist_ok=True)
        shutil.copy2(str(resolved), str(dest))
```

**Git staging is handled separately in Step 7 — do not manually `git add` here.**
```

- [ ] **Step 2: Verify the SKILL.md updates are coherent**

Read through the full SKILL.md to ensure step numbering is consistent and the new step integrates cleanly with the existing flow.

- [ ] **Step 3: Commit**

```bash
git add publish-blog/SKILL.md
git commit -m "feat(#484): add image verification and co-copy to publish-blog

Add Step 5b to verify image references before copying entries, and
update Step 6 to copy referenced images alongside .md entries.  Refs #484"
```

- [ ] **Step 4: Run full test suite to verify no regressions**

Run: `python3 -m pytest tests/test_image_refs.py tests/test_image_promotion.py tests/test_close_checks_images.py -v`
Expected: all PASS

- [ ] **Step 5: Run commit-tier validators**

Run: `python3 scripts/validate_all.py --tier commit`
Expected: clean (exit 0)

## References

- [2026-11-03-verify-image-refs-design.md] — design spec this plan implements
- [verification/repo_checks.py:280-333] — existing `check_broken_images()` and regex constants
- [work-end/artifact_promote.py:158-236] — `to_project()` promotion logic
- [work-end/artifact_promote.py:75-155] — `to_workspace_main()` promotion logic
- [verification/close_checks.py] — close-time check functions
- [verification/close_verify.py:36-62] — `_run_branch_checks()` check composition
- [publish-blog/SKILL.md:118-153] — Steps 5-6 destination validation and file copy
- [publish-blog/blog_publish.py:13-39] — `copy_entry()` function
- [GitHub #484] — focal issue
- [GitHub #483] — image promotion gap (original report)
- [protocol: externalised-scripts-require-tests] — scripts require tests
- [protocol: evidence-before-claims] — verify before claiming done
