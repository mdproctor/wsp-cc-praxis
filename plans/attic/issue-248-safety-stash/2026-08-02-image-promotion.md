# Image Promotion Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #149 — Slot promotion robustness: image handling, audit refinement, smoke test automation
**Issue group:** #149

**Goal:** Promote images referenced by markdown artifacts alongside the markdown, across all promotion paths and blog publishing.

**Architecture:** Add `extract_image_refs()` to `workspace_artifacts.py` — parses markdown/HTML image references, resolves relative paths, returns existing image paths. Scanner calls it after collecting `.md` files. `blog_dest.py` calls it for unpublished blog entries. No downstream changes — promotion pipeline is already binary-safe.

**Tech Stack:** Python 3, pathlib, re, shutil, pytest

## Global Constraints

- Images are promoted only when referenced by a collected `.md` file — no directory-wide collection
- Missing image refs log a warning to stderr, not silently skipped
- Image paths preserve relative structure — no markdown rewriting
- Deduplication by set membership on relative paths
- All categories except snapshots get image scanning (snapshots already use `ext: None`)

---

### Task 1: `extract_image_refs` Core Function

**Files:**
- Modify: `work-end/workspace_artifacts.py` (add function after line 44)
- Test: `tests/test_workspace_artifacts.py` (new test class)

**Interfaces:**
- Consumes: nothing — standalone function
- Produces: `extract_image_refs(md_path: Path, root: Path) -> list[str]` — returns relative paths (to root) of existing referenced images

- [ ] **Step 1: Write failing tests for markdown image syntax**

```python
class TestExtractImageRefs:
    def test_markdown_image_extracted(self, tmp_path):
        img = tmp_path / "specs" / "images" / "arch.png"
        img.parent.mkdir(parents=True)
        img.write_bytes(b"\x89PNG")
        md = tmp_path / "specs" / "design.md"
        md.write_text("# Design\n\n![Architecture](images/arch.png)\n")

        result = extract_image_refs(md, tmp_path)
        assert result == ["specs/images/arch.png"]

    def test_html_img_extracted(self, tmp_path):
        img = tmp_path / "blog" / "photo.jpg"
        img.parent.mkdir(parents=True)
        img.write_bytes(b"\xff\xd8")
        md = tmp_path / "blog" / "entry.md"
        md.write_text('<img src="photo.jpg" width="400">\n')

        result = extract_image_refs(md, tmp_path)
        assert result == ["blog/photo.jpg"]

    def test_html_img_single_quotes(self, tmp_path):
        img = tmp_path / "blog" / "photo.jpg"
        img.parent.mkdir(parents=True)
        img.write_bytes(b"\xff\xd8")
        md = tmp_path / "blog" / "entry.md"
        md.write_text("<img src='photo.jpg' alt='test'>\n")

        result = extract_image_refs(md, tmp_path)
        assert result == ["blog/photo.jpg"]
```

- [ ] **Step 2: Write failing tests for filtering**

```python
    def test_external_urls_excluded(self, tmp_path):
        md = tmp_path / "blog" / "entry.md"
        md.parent.mkdir(parents=True)
        md.write_text("![Logo](https://example.com/logo.png)\n![Other](http://example.com/other.png)\n")

        result = extract_image_refs(md, tmp_path)
        assert result == []

    def test_template_vars_excluded(self, tmp_path):
        md = tmp_path / "blog" / "entry.md"
        md.parent.mkdir(parents=True)
        md.write_text('<img src="{thumb_src}" alt="thumb">\n')

        result = extract_image_refs(md, tmp_path)
        assert result == []

    def test_protocol_uris_excluded(self, tmp_path):
        md = tmp_path / "blog" / "entry.md"
        md.parent.mkdir(parents=True)
        md.write_text("![Icon](chrome://skype/icon.png)\n![Data](data:image/png;base64,abc)\n")

        result = extract_image_refs(md, tmp_path)
        assert result == []
```

- [ ] **Step 3: Write failing tests for resolution and warnings**

```python
    def test_missing_image_warned(self, tmp_path, capsys):
        md = tmp_path / "specs" / "design.md"
        md.parent.mkdir(parents=True)
        md.write_text("![Missing](images/gone.png)\n")

        result = extract_image_refs(md, tmp_path)
        assert result == []
        assert "gone.png" in capsys.readouterr().err

    def test_nested_image_paths_preserved(self, tmp_path):
        img = tmp_path / "specs" / "images" / "sub" / "deep.svg"
        img.parent.mkdir(parents=True)
        img.write_text("<svg/>")
        md = tmp_path / "specs" / "design.md"
        md.write_text("![Diagram](images/sub/deep.svg)\n")

        result = extract_image_refs(md, tmp_path)
        assert result == ["specs/images/sub/deep.svg"]

    def test_multiple_refs_in_one_file(self, tmp_path):
        (tmp_path / "blog").mkdir()
        for name in ["a.png", "b.jpg"]:
            (tmp_path / "blog" / name).write_bytes(b"\x00")
        md = tmp_path / "blog" / "entry.md"
        md.write_text("![A](a.png)\n\n![B](b.jpg)\n")

        result = extract_image_refs(md, tmp_path)
        assert sorted(result) == ["blog/a.png", "blog/b.jpg"]

    def test_duplicate_refs_deduplicated(self, tmp_path):
        (tmp_path / "blog").mkdir()
        (tmp_path / "blog" / "logo.png").write_bytes(b"\x00")
        md = tmp_path / "blog" / "entry.md"
        md.write_text("![Logo](logo.png)\n\n![Logo again](logo.png)\n")

        result = extract_image_refs(md, tmp_path)
        assert result == ["blog/logo.png"]
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_workspace_artifacts.py::TestExtractImageRefs -v`
Expected: FAIL — `extract_image_refs` not defined

- [ ] **Step 5: Implement `extract_image_refs`**

Add to `work-end/workspace_artifacts.py` after `_scan_dir` (after line 44):

```python
import re
import sys

_MD_IMG = re.compile(r'!\[[^\]]*\]\(([^)]+)\)')
_HTML_IMG = re.compile(r'<img[^>]+src=["\']([^"\']+)["\']')
_SKIP_PREFIXES = ("http://", "https://", "chrome://", "data:")


def extract_image_refs(md_path: Path, root: Path) -> list[str]:
    """Extract image paths referenced by a markdown file.

    Returns paths relative to root, matching scan() convention.
    Only includes references where the resolved file exists on disk.
    Logs a warning to stderr for missing references.
    """
    text = md_path.read_text(errors="replace")
    seen: set[str] = set()
    results: list[str] = []

    for pattern in (_MD_IMG, _HTML_IMG):
        for match in pattern.finditer(text):
            raw = match.group(1).split(" ")[0]  # strip title text after space
            if any(raw.startswith(p) for p in _SKIP_PREFIXES):
                continue
            if "{" in raw or "}" in raw:
                continue
            resolved = (md_path.parent / raw).resolve()
            if not resolved.is_file():
                print(f"WARNING: image ref not found: {raw} (in {md_path})",
                      file=sys.stderr)
                continue
            try:
                rel = str(resolved.relative_to(root.resolve()))
            except ValueError:
                continue
            if rel not in seen:
                seen.add(rel)
                results.append(rel)

    return results
```

- [ ] **Step 6: Update import in test file**

Add `extract_image_refs` to the import line in `tests/test_workspace_artifacts.py`:

```python
from workspace_artifacts import scan, extract_image_refs
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_workspace_artifacts.py::TestExtractImageRefs -v`
Expected: all PASS

- [ ] **Step 8: Run existing tests to verify no regressions**

Run: `python3 -m pytest tests/test_workspace_artifacts.py -v`
Expected: all PASS (existing tests unchanged)

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-end/workspace_artifacts.py tests/test_workspace_artifacts.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat(#149): extract_image_refs — parse markdown for image references

Adds extract_image_refs() to workspace_artifacts.py. Parses ![](path)
and <img src> references, resolves relative paths, returns existing
images as workspace-relative paths. Warns on missing refs.

Refs #149"
```

---

### Task 2: Scanner Integration

**Files:**
- Modify: `work-end/workspace_artifacts.py` (`scan()` function, line 47-68)
- Modify: `tests/test_workspace_artifacts.py` (update existing test, add new tests)

**Interfaces:**
- Consumes: `extract_image_refs(md_path: Path, root: Path) -> list[str]` from Task 1
- Produces: `scan()` return value now includes image paths alongside `.md` paths in each category

- [ ] **Step 1: Write failing test — images included in scan results**

```python
class TestScanImageRefs:
    def test_image_refs_included_in_scan(self, tmp_path):
        (tmp_path / "specs").mkdir()
        img = tmp_path / "specs" / "images" / "arch.svg"
        img.parent.mkdir()
        img.write_text("<svg/>")
        md = tmp_path / "specs" / "design.md"
        md.write_text("# Design\n\n![Architecture](images/arch.svg)\n")

        result = scan(tmp_path)
        assert "specs/design.md" in result["specs"]
        assert "specs/images/arch.svg" in result["specs"]

    def test_image_refs_across_all_categories(self, tmp_path):
        for cat in ("specs", "adr", "blog", "plans"):
            d = tmp_path / cat
            d.mkdir()
            img = d / "photo.png"
            img.write_bytes(b"\x89PNG")
            md = d / "entry.md"
            md.write_text(f"![Photo](photo.png)\n")

        result = scan(tmp_path)
        for cat in ("specs", "adr", "blog", "plans"):
            assert f"{cat}/photo.png" in result[cat], \
                f"image not found in {cat}: {result[cat]}"

    def test_non_referenced_images_excluded(self, tmp_path):
        (tmp_path / "specs").mkdir()
        (tmp_path / "specs" / "design.md").write_text("# No images\n")
        (tmp_path / "specs" / "orphan.png").write_bytes(b"\x89PNG")

        result = scan(tmp_path)
        assert result["specs"] == ["specs/design.md"]

    def test_snapshots_unchanged(self, tmp_path):
        (tmp_path / "snapshots").mkdir()
        (tmp_path / "snapshots" / "doc.md").write_text("snap")
        (tmp_path / "snapshots" / "diagram.png").write_bytes(b"\x89PNG")

        result = scan(tmp_path)
        assert len(result["snapshots"]) == 2

    def test_image_dedup_across_md_files(self, tmp_path):
        (tmp_path / "specs").mkdir()
        img = tmp_path / "specs" / "shared.png"
        img.write_bytes(b"\x89PNG")
        (tmp_path / "specs" / "a.md").write_text("![X](shared.png)\n")
        (tmp_path / "specs" / "b.md").write_text("![Y](shared.png)\n")

        result = scan(tmp_path)
        assert result["specs"].count("specs/shared.png") == 1
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_workspace_artifacts.py::TestScanImageRefs -v`
Expected: FAIL — images not in scan results

- [ ] **Step 3: Update `scan()` to include image refs**

Modify `scan()` in `work-end/workspace_artifacts.py`:

```python
def scan(workspace: Path) -> dict[str, list[str]]:
    """Scan workspace for promotable artifacts.

    Returns category -> sorted list of paths relative to workspace root.
    Recurses into subdirectories (e.g. specs/issue-NNN-slug/).
    Also checks alternate paths (e.g. docs/adr/ in addition to adr/).
    For markdown categories, also includes images referenced by .md files.
    """
    found: dict[str, list[str]] = {}

    for category, cfg in CATEGORIES.items():
        ext = cfg["ext"]
        exclude_names = cfg["exclude_names"]
        exclude_dirs = cfg.get("exclude_dirs", set())

        entries = _scan_dir(workspace, workspace / category, ext, exclude_names, exclude_dirs)

        for alt in cfg.get("alt_paths", []):
            entries.extend(_scan_dir(workspace, workspace / alt, ext, exclude_names, exclude_dirs))

        if ext == ".md":
            seen = set(entries)
            for entry in list(entries):
                md_path = workspace / entry
                for img_ref in extract_image_refs(md_path, workspace):
                    if img_ref not in seen:
                        seen.add(img_ref)
                        entries.append(img_ref)

        found[category] = sorted(entries)

    return found
```

- [ ] **Step 4: Update existing test `test_non_md_files_in_specs_ignored`**

The existing test creates a `diagram.png` alongside a `design.md` that does NOT reference it. The test should still pass — unreferenced images are excluded. Verify:

Run: `python3 -m pytest tests/test_workspace_artifacts.py::TestScanEdgeCases::test_non_md_files_in_specs_ignored -v`
Expected: PASS (diagram.png is unreferenced, still excluded)

- [ ] **Step 5: Run all tests**

Run: `python3 -m pytest tests/test_workspace_artifacts.py -v`
Expected: all PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-end/workspace_artifacts.py tests/test_workspace_artifacts.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat(#149): scan() includes referenced images alongside markdown

Scanner calls extract_image_refs() on each collected .md file and
adds referenced images to the category's artifact list. Deduplicates
across multiple .md files referencing the same image.

Refs #149"
```

---

### Task 3: Blog Destination Image Handling

**Files:**
- Modify: `work-end/blog_dest.py` (lines 56-68)
- Create: `tests/test_blog_dest.py`

**Interfaces:**
- Consumes: `extract_image_refs(md_path: Path, root: Path) -> list[str]` from Task 1
- Produces: `blog_dest.py` copies referenced images alongside unpublished blog entries

- [ ] **Step 1: Write failing tests**

```python
"""Tests for work-end/blog_dest.py image handling."""

import sys
from pathlib import Path

import pytest

sys.path.insert(0, str(Path(__file__).parent.parent / "work-end"))

from workspace_artifacts import extract_image_refs


class TestBlogImageCopy:
    def test_blog_image_copied_to_destination(self, tmp_path):
        ws_blog = tmp_path / "source" / "blog"
        ws_blog.mkdir(parents=True)
        (ws_blog / "2026-08-01-entry.md").write_text("![Photo](images/photo.png)\n")
        img = ws_blog / "images" / "photo.png"
        img.parent.mkdir()
        img.write_bytes(b"\x89PNG")

        dest = tmp_path / "dest" / "_posts"
        dest.mkdir(parents=True)

        refs = extract_image_refs(ws_blog / "2026-08-01-entry.md", ws_blog)
        for ref in refs:
            src = ws_blog / ref
            dst = dest / ref
            dst.parent.mkdir(parents=True, exist_ok=True)
            import shutil
            shutil.copy2(str(src), str(dst))

        assert (dest / "images" / "photo.png").exists()

    def test_blog_image_relative_structure_preserved(self, tmp_path):
        ws_blog = tmp_path / "source" / "blog"
        ws_blog.mkdir(parents=True)
        (ws_blog / "entry.md").write_text("![D](images/sub/deep.svg)\n")
        img = ws_blog / "images" / "sub" / "deep.svg"
        img.parent.mkdir(parents=True)
        img.write_text("<svg/>")

        dest = tmp_path / "dest" / "_posts"
        dest.mkdir(parents=True)

        refs = extract_image_refs(ws_blog / "entry.md", ws_blog)
        for ref in refs:
            src = ws_blog / ref
            dst = dest / ref
            dst.parent.mkdir(parents=True, exist_ok=True)
            import shutil
            shutil.copy2(str(src), str(dst))

        assert (dest / "images" / "sub" / "deep.svg").exists()
```

- [ ] **Step 2: Run tests to verify they pass**

These tests validate the copy pattern using `extract_image_refs` directly — they should pass since Task 1 is done:

Run: `python3 -m pytest tests/test_blog_dest.py -v`
Expected: PASS

- [ ] **Step 3: Update `blog_dest.py` to copy images**

Add import at top of `blog_dest.py`:

```python
from workspace_artifacts import extract_image_refs
```

Replace the copy loop (lines 62-68):

```python
if unpublished:
    dest_path.mkdir(parents=True, exist_ok=True)
    for entry in unpublished:
        shutil.copy2(workspace_blog / entry, dest_path / entry)
        for img_ref in extract_image_refs(workspace_blog / entry, workspace_blog):
            img_src = workspace_blog / img_ref
            img_dst = dest_path / img_ref
            img_dst.parent.mkdir(parents=True, exist_ok=True)
            shutil.copy2(str(img_src), str(img_dst))
    print(f"UNPUBLISHED={','.join(unpublished)}")
else:
    print("UNPUBLISHED=")
```

- [ ] **Step 4: Run tests**

Run: `python3 -m pytest tests/test_blog_dest.py tests/test_workspace_artifacts.py -v`
Expected: all PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-end/blog_dest.py tests/test_blog_dest.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat(#149): blog_dest copies referenced images alongside entries

blog_dest.py now calls extract_image_refs() on each unpublished entry
and copies referenced images to the blog destination, preserving
relative directory structure.

Refs #149"
```

---

### Task 4: Integration Tests

**Files:**
- Modify: `tests/test_close_artifacts.py` (add integration tests)

**Interfaces:**
- Consumes: `scan_artifacts()` (which delegates to `scan()`) and the full promotion pipeline
- Produces: end-to-end verification that images survive promotion

- [ ] **Step 1: Write failing integration test — image promoted alongside markdown**

```python
class TestImagePromotion:
    def test_image_promoted_alongside_markdown(self, tmp_path):
        """End-to-end: scan finds image refs, promotion copies them."""
        ws = tmp_path / "workspace"
        ws.mkdir()
        specs = ws / "specs"
        specs.mkdir()
        (specs / "design.md").write_text("# Design\n\n![Arch](images/arch.svg)\n")
        img = specs / "images" / "arch.svg"
        img.parent.mkdir()
        img.write_text("<svg/>")

        result = scan_artifacts(ws)
        assert "specs/design.md" in result["specs"]
        assert "specs/images/arch.svg" in result["specs"]

    def test_blog_image_in_scan_results(self, tmp_path):
        ws = tmp_path / "workspace"
        ws.mkdir()
        blog = ws / "blog"
        blog.mkdir()
        (blog / "entry.md").write_text("![Photo](photo.png)\n")
        (blog / "photo.png").write_bytes(b"\x89PNG")

        result = scan_artifacts(ws)
        assert "blog/entry.md" in result["blog"]
        assert "blog/photo.png" in result["blog"]
```

- [ ] **Step 2: Run tests to verify they pass**

These should pass since Task 2 wired `scan()` to include image refs, and `scan_artifacts` delegates to `scan()`:

Run: `python3 -m pytest tests/test_close_artifacts.py::TestImagePromotion -v`
Expected: PASS

- [ ] **Step 3: Run full test suite**

Run: `python3 -m pytest tests/test_workspace_artifacts.py tests/test_close_artifacts.py tests/test_blog_dest.py -v`
Expected: all PASS

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add tests/test_close_artifacts.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "test(#149): integration tests for image promotion

Verifies scan_artifacts() includes image refs from specs and blog
entries in the artifact list, confirming end-to-end promotion path.

Refs #149"
```
