# Epic-Driven Slots Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** TBD — create before starting
**Issue group:** TBD

**Goal:** Add `work-slot epic`, `work-slot next`, and `work-slot status`
commands so a single slot can iterate through an epic's child issues
using batch planning, with progress tracked via GitHub and SLOT.md.
Fix `work` skill routing so "resume" on a feature branch reads the
handover instead of routing to work-resume (paused branch path).

**Architecture:** New `epic_manager.py` script handles batch plan
parsing, issue advancement, and progress queries — separated from
`slot_manager.py` which owns slot lifecycle. SLOT.md extends the
standard format with `Type: epic`, `## Batch Plan`, and
`## Session State` sections. `parse_slot_md()` gains an `is_epic`
flag. Skills (`work-start`, `handover`, `work-end`) detect epic context
via `$PROJECT/../SLOT.md` guarded by `/worktrees/` path check.

**Tech Stack:** Python 3.11+, pytest, GitHub CLI (`gh`), existing
slot_manager.py / scaffold.py / worklog.py infrastructure

## Global Constraints

- SLOT.md issue line: `owner/repo#N` only — no title suffix (parser
  splits on `#`, expects exactly 2 parts)
- `COVERS` accumulates strictly per-issue — never pre-load batches
- Issue closure deferred to `work-end` — `work-slot next` only updates
  checkboxes
- Branch naming: `issue-NNN-<slug>` convention, never `epic-*`
- All new scripts follow existing `KEY=VALUE` stdout + exit code pattern
- Tests use `tmp_path` and mock git commands where possible

---

### Task 1: `epic_manager.py` — Core batch plan operations

**Files:**
- Create: `work-slot/epic_manager.py`
- Test: `tests/test_epic_manager.py`

**Interfaces:**
- Produces:
  - `parse_batch_plan(slot_dir: Path) -> dict` — returns
    `{"is_epic": bool, "batches": [...], "current_batch": int, "current_issue": int, "completed": [int], "epic_number": str, "epic_repo": str}`
  - `advance(slot_dir: Path) -> dict` — returns
    `{"completed": int, "next_issue": int|None, "next_issue_title": str, "batch_complete": bool, "epic_complete": bool, "safe_exit": bool}`
  - `status(slot_dir: Path) -> dict` — returns progress summary
  - `write_epic_slot_md(slot_dir, slot_number, repos, branch, issue, issue_repo, batches, context) -> None`
  - CLI subcommands: `plan`, `advance`, `status`

- [ ] **Step 1: Write tests for `parse_batch_plan`**

```python
# tests/test_epic_manager.py

import sys
from pathlib import Path

import pytest

skill_dir = Path(__file__).parent.parent / "work-slot"
sys.path.insert(0, str(skill_dir))

import epic_manager


SAMPLE_EPIC_SLOT_MD = """\
# Slot 38 — issue-50-weighted-profiles

## Issue
casehubio/engine#50
Covers: 108,109
Type: epic
Safe exit: after any completed batch

## What to do
Epic #50 — Weighted Profiles. Working through batched child issues.
Current: Batch 1 — Vocabulary and docs (S+S)

## Batch Plan

### Batch 1 — Vocabulary and docs (S+S)
- [x] #108 — Rename disposition
- [ ] #109 — Update terminology ← active

### Batch 2 — Weighted profiles API (M+M) 
- [ ] #111 — Add weight parameter
- [ ] #112 — Dominant-auxiliary scoring

## Session State
Current batch: 1
Current issue: #109 — Update terminology
Last wrap: 2026-07-28, session started batch 1

## Repos
- engine (primary)

## Created
2026-07-28, branch: issue-50-weighted-profiles
"""


class TestParseBatchPlan:
    def test_parses_epic_slot(self, tmp_path):
        (tmp_path / "SLOT.md").write_text(SAMPLE_EPIC_SLOT_MD)
        result = epic_manager.parse_batch_plan(tmp_path)
        assert result["is_epic"] is True
        assert result["epic_number"] == "50"
        assert result["epic_repo"] == "casehubio/engine"
        assert len(result["batches"]) == 2
        assert result["current_batch"] == 1
        assert result["current_issue"] == 109
        assert result["completed"] == [108]

    def test_batch_structure(self, tmp_path):
        (tmp_path / "SLOT.md").write_text(SAMPLE_EPIC_SLOT_MD)
        result = epic_manager.parse_batch_plan(tmp_path)
        b1 = result["batches"][0]
        assert b1["name"] == "Vocabulary and docs (S+S)"
        assert b1["number"] == 1
        assert len(b1["issues"]) == 2
        assert b1["issues"][0] == {"number": 108, "title": "Rename disposition", "done": True}
        assert b1["issues"][1] == {"number": 109, "title": "Update terminology", "done": False}

    def test_non_epic_slot(self, tmp_path):
        (tmp_path / "SLOT.md").write_text(
            "# Slot 1 — issue-42-spi\n\n## Issue\nrepo#42\nCovers: 42\n"
        )
        result = epic_manager.parse_batch_plan(tmp_path)
        assert result["is_epic"] is False

    def test_missing_slot_md(self, tmp_path):
        result = epic_manager.parse_batch_plan(tmp_path)
        assert result["is_epic"] is False
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_epic_manager.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'epic_manager'`

- [ ] **Step 3: Implement `parse_batch_plan`**

Create `work-slot/epic_manager.py`:

```python
#!/usr/bin/env python3
"""
epic_manager.py — Epic batch plan operations for work-slot

Subcommands:
  plan <slot-dir>       Parse SLOT.md, return batch plan as JSON
  advance <slot-dir>    Advance to next issue, update SLOT.md + .meta
  status <slot-dir>     Return progress summary as JSON

Operates on SLOT.md's ## Batch Plan section. Separated from
slot_manager.py to enable future single-repo epic support.
"""

import json
import re
import sys
from pathlib import Path


def parse_batch_plan(slot_dir: Path) -> dict:
    """Parse SLOT.md and extract epic batch plan state."""
    slot_md = slot_dir / "SLOT.md"
    if not slot_md.exists():
        return {"is_epic": False}

    content = slot_md.read_text()
    is_epic = False
    epic_number = ""
    epic_repo = ""

    # Check for Type: epic in ## Issue section
    in_issue = False
    for line in content.splitlines():
        if line.startswith("## Issue"):
            in_issue = True
            continue
        if line.startswith("## ") and in_issue:
            in_issue = False
            continue
        if in_issue:
            if line.strip().startswith("Type: epic"):
                is_epic = True
            if "#" in line and not line.startswith("Covers:") and not line.startswith("Type:") and not line.startswith("Safe exit:"):
                parts = line.strip().split("#")
                if len(parts) == 2:
                    epic_repo = parts[0].strip()
                    epic_number = parts[1].strip()

    if not is_epic:
        return {"is_epic": False}

    # Parse batches from ## Batch Plan
    batches = []
    current_batch_obj = None
    in_batch_plan = False
    for line in content.splitlines():
        if line.strip() == "## Batch Plan":
            in_batch_plan = True
            continue
        if line.startswith("## ") and in_batch_plan:
            break
        if not in_batch_plan:
            continue
        # Batch header: ### Batch N — Name
        m = re.match(r"^### Batch (\d+) — (.+?)(?:\s*←.*)?$", line)
        if m:
            if current_batch_obj:
                batches.append(current_batch_obj)
            current_batch_obj = {
                "number": int(m.group(1)),
                "name": m.group(2).strip(),
                "issues": [],
            }
            continue
        # Issue line: - [x] #NNN — Title or - [ ] #NNN — Title
        im = re.match(r"^- \[([ x])\] #(\d+) — (.+?)(?:\s*←.*)?$", line)
        if im and current_batch_obj is not None:
            current_batch_obj["issues"].append({
                "number": int(im.group(2)),
                "title": im.group(3).strip(),
                "done": im.group(1) == "x",
            })
    if current_batch_obj:
        batches.append(current_batch_obj)

    # Derive current state
    completed = []
    current_batch = 0
    current_issue = 0
    for batch in batches:
        for issue in batch["issues"]:
            if issue["done"]:
                completed.append(issue["number"])
            elif current_issue == 0:
                current_batch = batch["number"]
                current_issue = issue["number"]

    return {
        "is_epic": True,
        "epic_number": epic_number,
        "epic_repo": epic_repo,
        "batches": batches,
        "current_batch": current_batch,
        "current_issue": current_issue,
        "completed": completed,
    }
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_epic_manager.py::TestParseBatchPlan -v`
Expected: PASS

- [ ] **Step 5: Write tests for `advance`**

```python
class TestAdvance:
    def test_advances_to_next_issue_in_batch(self, tmp_path):
        (tmp_path / "SLOT.md").write_text(SAMPLE_EPIC_SLOT_MD)
        design = tmp_path / "work" / "engine" / "design"
        design.mkdir(parents=True)
        (design / ".meta").write_text(
            "branch: issue-50-weighted-profiles\n"
            "issue: 50\n"
            "issue-repo: casehubio/engine\n"
            "covers: 108\n"
        )
        result = epic_manager.advance(tmp_path, meta_path=design / ".meta")
        assert result["completed"] == 109
        assert result["next_issue"] == 111
        assert result["batch_complete"] is True
        assert result["epic_complete"] is False
        assert result["safe_exit"] is True

    def test_advances_within_batch(self, tmp_path):
        md = SAMPLE_EPIC_SLOT_MD.replace(
            "- [x] #108 — Rename disposition\n- [ ] #109 — Update terminology ← active",
            "- [ ] #108 — Rename disposition ← active\n- [ ] #109 — Update terminology",
        ).replace("Covers: 108,109", "Covers:").replace(
            "Current batch: 1\nCurrent issue: #109 — Update terminology",
            "Current batch: 1\nCurrent issue: #108 — Rename disposition",
        )
        (tmp_path / "SLOT.md").write_text(md)
        design = tmp_path / "work" / "engine" / "design"
        design.mkdir(parents=True)
        (design / ".meta").write_text(
            "branch: issue-50-weighted-profiles\n"
            "issue: 50\n"
            "issue-repo: casehubio/engine\n"
            "covers:\n"
        )
        result = epic_manager.advance(tmp_path, meta_path=design / ".meta")
        assert result["completed"] == 108
        assert result["next_issue"] == 109
        assert result["batch_complete"] is False
        assert result["safe_exit"] is False

    def test_updates_slot_md_checkboxes(self, tmp_path):
        (tmp_path / "SLOT.md").write_text(SAMPLE_EPIC_SLOT_MD)
        design = tmp_path / "work" / "engine" / "design"
        design.mkdir(parents=True)
        (design / ".meta").write_text(
            "branch: issue-50-weighted-profiles\n"
            "issue: 50\n"
            "issue-repo: casehubio/engine\n"
            "covers: 108\n"
        )
        epic_manager.advance(tmp_path, meta_path=design / ".meta")
        updated = (tmp_path / "SLOT.md").read_text()
        assert "- [x] #109 — Update terminology" in updated
        assert "← active" not in updated or "#111" in updated.split("← active")[0]

    def test_updates_meta_covers(self, tmp_path):
        (tmp_path / "SLOT.md").write_text(SAMPLE_EPIC_SLOT_MD)
        design = tmp_path / "work" / "engine" / "design"
        design.mkdir(parents=True)
        (design / ".meta").write_text(
            "branch: issue-50-weighted-profiles\n"
            "issue: 50\n"
            "issue-repo: casehubio/engine\n"
            "covers: 108\n"
        )
        epic_manager.advance(tmp_path, meta_path=design / ".meta")
        meta = (design / ".meta").read_text()
        assert "covers: 108,109" in meta

    def test_epic_complete_on_last_issue(self, tmp_path):
        md = """\
# Slot 1 — issue-50-test

## Issue
repo#50
Covers: 108
Type: epic
Safe exit: after any completed batch

## What to do
Test

## Batch Plan

### Batch 1 — Final (S)
- [x] #108 — First
- [ ] #109 — Last ← active

## Session State
Current batch: 1
Current issue: #109 — Last

## Repos
- engine (primary)

## Created
2026-07-28, branch: issue-50-test
"""
        (tmp_path / "SLOT.md").write_text(md)
        design = tmp_path / "work" / "engine" / "design"
        design.mkdir(parents=True)
        (design / ".meta").write_text(
            "branch: issue-50-test\nissue: 50\nissue-repo: repo\ncovers: 108\n"
        )
        result = epic_manager.advance(tmp_path, meta_path=design / ".meta")
        assert result["epic_complete"] is True
        assert result["batch_complete"] is True
```

- [ ] **Step 6: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_epic_manager.py::TestAdvance -v`
Expected: FAIL — `AttributeError: module 'epic_manager' has no attribute 'advance'`

- [ ] **Step 7: Implement `advance`**

Add to `epic_manager.py`:

```python
def advance(slot_dir: Path, meta_path: Path | None = None) -> dict:
    """Advance to the next issue. Updates SLOT.md and .meta COVERS."""
    plan = parse_batch_plan(slot_dir)
    if not plan["is_epic"]:
        return {"error": "not an epic slot"}

    current = plan["current_issue"]
    if current == 0:
        return {"error": "no active issue"}

    # Find the current issue and the next one
    found_current = False
    next_issue = None
    next_title = ""
    batch_complete = False
    epic_complete = False
    current_batch_num = plan["current_batch"]
    in_current_batch = False

    for batch in plan["batches"]:
        if batch["number"] == current_batch_num:
            in_current_batch = True
        for issue in batch["issues"]:
            if found_current and next_issue is None:
                next_issue = issue["number"]
                next_title = issue["title"]
                if not in_current_batch:
                    batch_complete = True
            if issue["number"] == current:
                found_current = True
                # Check if this is the last issue in its batch
                batch_issues = batch["issues"]
                if issue == batch_issues[-1]:
                    batch_complete = True
        if batch["number"] == current_batch_num:
            in_current_batch = False

    if next_issue is None:
        epic_complete = True
        batch_complete = True

    # Update SLOT.md
    _update_slot_md_advance(slot_dir, current, next_issue)

    # Update .meta COVERS
    if meta_path and meta_path.exists():
        _update_meta_covers(meta_path, current)

    # Update SLOT.md Covers line
    _update_slot_md_covers(slot_dir, current)

    safe_exit = batch_complete

    return {
        "completed": current,
        "next_issue": next_issue,
        "next_issue_title": next_title,
        "batch_complete": batch_complete,
        "epic_complete": epic_complete,
        "safe_exit": safe_exit,
    }


def _update_slot_md_advance(slot_dir: Path, completed: int, next_issue: int | None) -> None:
    """Check off completed issue, move ← active marker to next."""
    slot_md = slot_dir / "SLOT.md"
    content = slot_md.read_text()
    lines = content.splitlines()
    new_lines = []
    for line in lines:
        # Check off the completed issue
        if f"- [ ] #{completed} —" in line:
            line = line.replace("- [ ]", "- [x]")
            line = re.sub(r"\s*← active\s*$", "", line)
        # Move ← current marker if batch changed
        elif "← current" in line:
            line = re.sub(r"\s*← current\s*$", "", line)
        # Add ← active to next issue
        if next_issue and f"- [ ] #{next_issue} —" in line:
            line = re.sub(r"\s*$", " ← active", line)
        new_lines.append(line)

    # Add ← current to the batch containing the next issue
    if next_issue:
        final_lines = []
        for line in new_lines:
            m = re.match(r"^### Batch (\d+) — (.+)$", line)
            if m:
                # Check if next_issue is in this batch
                batch_num = int(m.group(1))
                # Look ahead to see if next_issue is in this batch
                idx = new_lines.index(line)
                batch_has_next = False
                for j in range(idx + 1, len(new_lines)):
                    if new_lines[j].startswith("### Batch"):
                        break
                    if f"#{next_issue} —" in new_lines[j]:
                        batch_has_next = True
                        break
                if batch_has_next:
                    line = f"{line} ← current"
            final_lines.append(line)
        new_lines = final_lines

    # Update Session State
    result_lines = []
    in_session = False
    for line in new_lines:
        if line.strip() == "## Session State":
            in_session = True
            result_lines.append(line)
            continue
        if line.startswith("## ") and in_session:
            in_session = False
        if in_session:
            if line.startswith("Current issue:") and next_issue:
                # Find next issue title
                for b in parse_batch_plan(slot_dir)["batches"]:
                    for iss in b["issues"]:
                        if iss["number"] == next_issue:
                            line = f"Current issue: #{next_issue} — {iss['title']}"
                            break
            elif line.startswith("Current batch:") and next_issue:
                plan = parse_batch_plan(slot_dir)
                for b in plan["batches"]:
                    for iss in b["issues"]:
                        if iss["number"] == next_issue:
                            line = f"Current batch: {b['number']}"
                            break
        result_lines.append(line)

    slot_md.write_text("\n".join(result_lines))


def _update_meta_covers(meta_path: Path, issue_number: int) -> None:
    """Append issue_number to covers in .meta."""
    content = meta_path.read_text()
    lines = content.splitlines()
    new_lines = []
    for line in lines:
        if line.startswith("covers:"):
            existing = line.split(":", 1)[1].strip()
            nums = [n.strip() for n in existing.split(",") if n.strip()]
            issue_str = str(issue_number)
            if issue_str not in nums:
                nums.append(issue_str)
            line = f"covers: {','.join(nums)}"
        new_lines.append(line)
    meta_path.write_text("\n".join(new_lines) + "\n")


def _update_slot_md_covers(slot_dir: Path, issue_number: int) -> None:
    """Update the Covers: line in SLOT.md ## Issue section."""
    slot_md = slot_dir / "SLOT.md"
    content = slot_md.read_text()
    lines = content.splitlines()
    new_lines = []
    for line in lines:
        if line.startswith("Covers:"):
            existing = line.split(":", 1)[1].strip()
            nums = [n.strip() for n in existing.split(",") if n.strip()]
            issue_str = str(issue_number)
            if issue_str not in nums:
                nums.append(issue_str)
            line = f"Covers: {','.join(nums)}"
        new_lines.append(line)
    slot_md.write_text("\n".join(new_lines))
```

Note: The `_update_slot_md_advance` function reads the file before
the Session State update to get the batch plan — this works because
the checkbox update hasn't been written yet at that point. The function
writes once at the end. This will need refinement during implementation
if the two-pass approach causes issues — split into write-checkbox then
re-read-for-session if needed.

- [ ] **Step 8: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_epic_manager.py::TestAdvance -v`
Expected: PASS

- [ ] **Step 9: Write tests for `status`**

```python
class TestStatus:
    def test_returns_progress(self, tmp_path):
        (tmp_path / "SLOT.md").write_text(SAMPLE_EPIC_SLOT_MD)
        result = epic_manager.status(tmp_path)
        assert result["total_issues"] == 4
        assert result["completed_count"] == 1
        assert result["total_batches"] == 2
        assert result["completed_batches"] == 0
        assert result["current_batch"] == 1
        assert result["current_issue"] == 109
        assert result["safe_exit"] is False

    def test_safe_exit_after_batch_complete(self, tmp_path):
        md = SAMPLE_EPIC_SLOT_MD.replace(
            "- [ ] #109 — Update terminology ← active",
            "- [x] #109 — Update terminology",
        ).replace(
            "Current issue: #109 — Update terminology",
            "Current issue: #111 — Add weight parameter",
        )
        (tmp_path / "SLOT.md").write_text(md)
        result = epic_manager.status(tmp_path)
        assert result["completed_batches"] == 1
        assert result["safe_exit"] is True

    def test_non_epic(self, tmp_path):
        (tmp_path / "SLOT.md").write_text("# Slot 1\n\n## Issue\nrepo#42\n")
        result = epic_manager.status(tmp_path)
        assert result.get("is_epic") is False
```

- [ ] **Step 10: Implement `status`**

```python
def status(slot_dir: Path) -> dict:
    """Return epic progress summary."""
    plan = parse_batch_plan(slot_dir)
    if not plan["is_epic"]:
        return {"is_epic": False}

    total_issues = sum(len(b["issues"]) for b in plan["batches"])
    completed_count = len(plan["completed"])
    total_batches = len(plan["batches"])

    completed_batches = 0
    for batch in plan["batches"]:
        if all(i["done"] for i in batch["issues"]):
            completed_batches += 1

    safe_exit = completed_batches > 0 and (
        plan["current_batch"] == 0 or
        any(
            all(i["done"] for i in b["issues"])
            for b in plan["batches"]
            if b["number"] == plan["current_batch"] - 1
        ) or completed_batches > 0
    )
    # Safe exit is true when at least one full batch is complete
    # and the current issue is the first in its batch (or between batches)
    safe_exit = completed_batches > 0

    return {
        "is_epic": True,
        "epic_number": plan["epic_number"],
        "epic_repo": plan["epic_repo"],
        "total_issues": total_issues,
        "completed_count": completed_count,
        "total_batches": total_batches,
        "completed_batches": completed_batches,
        "current_batch": plan["current_batch"],
        "current_issue": plan["current_issue"],
        "safe_exit": safe_exit,
        "batches": plan["batches"],
    }
```

- [ ] **Step 11: Run all tests**

Run: `python3 -m pytest tests/test_epic_manager.py -v`
Expected: PASS

- [ ] **Step 12: Add CLI entry point and `write_epic_slot_md`**

Add to `epic_manager.py`:

```python
def write_epic_slot_md(slot_dir: Path, slot_number: int, repos: list[str],
                       branch: str, issue: str, issue_repo: str,
                       batches: list[dict], context: str) -> None:
    """Write SLOT.md with epic batch plan structure."""
    lines = [f"# Slot {slot_number} — {branch}", ""]
    lines.append("## Issue")
    lines.append(f"{issue_repo}#{issue}")
    lines.append("Covers:")
    lines.append("Type: epic")
    lines.append("Safe exit: after any completed batch")
    lines.append("")
    lines.append("## What to do")
    lines.append(f"Epic #{issue} — {context}")
    if batches:
        lines.append(f"Current: Batch 1 — {batches[0]['name']}")
    lines.append("")
    lines.append("## Batch Plan")
    lines.append("")

    first_issue_set = False
    for batch in batches:
        lines.append(f"### Batch {batch['number']} — {batch['name']}" +
                     (" ← current" if batch["number"] == 1 else ""))
        for i, issue_item in enumerate(batch["issues"]):
            marker = " ← active" if not first_issue_set and batch["number"] == 1 and i == 0 else ""
            lines.append(f"- [ ] #{issue_item['number']} — {issue_item['title']}{marker}")
            if marker:
                first_issue_set = True
        lines.append("")

    first_issue_info = batches[0]["issues"][0] if batches and batches[0]["issues"] else None
    lines.append("## Session State")
    lines.append("Current batch: 1")
    if first_issue_info:
        lines.append(f"Current issue: #{first_issue_info['number']} — {first_issue_info['title']}")
    lines.append(f"Last wrap: slot created")
    lines.append("")
    lines.append("## Repos")
    for i, repo in enumerate(repos):
        primary = " (primary)" if i == 0 else ""
        lines.append(f"- {repo}{primary}")
    lines.append("")
    from datetime import date
    lines.append("## Created")
    lines.append(f"{date.today().isoformat()}, branch: {branch}")
    lines.append("")
    (slot_dir / "SLOT.md").write_text("\n".join(lines))


def main() -> None:
    if len(sys.argv) < 3:
        print(__doc__, file=sys.stderr)
        sys.exit(1)

    command = sys.argv[1]
    slot_dir = Path(sys.argv[2])

    if command == "plan":
        result = parse_batch_plan(slot_dir)
        print(json.dumps(result, indent=2))
    elif command == "advance":
        # Find .meta — look in workspace subdirs
        meta_path = None
        for sub in slot_dir.iterdir():
            if sub.is_dir():
                candidate = sub / "design" / ".meta"
                if candidate.exists():
                    meta_path = candidate
                    break
                # Check workspace subdirs
                for ws_sub in sub.iterdir():
                    if ws_sub.is_dir():
                        candidate = ws_sub / "design" / ".meta"
                        if candidate.exists():
                            meta_path = candidate
                            break
        result = advance(slot_dir, meta_path=meta_path)
        print(json.dumps(result, indent=2))
    elif command == "status":
        result = status(slot_dir)
        print(json.dumps(result, indent=2))
    else:
        print(f"ERROR=unknown_command command={command}", file=sys.stderr)
        sys.exit(1)


if __name__ == "__main__":
    main()
```

- [ ] **Step 13: Write test for `write_epic_slot_md` and verify roundtrip**

```python
class TestWriteEpicSlotMd:
    def test_writes_and_parses_roundtrip(self, tmp_path):
        batches = [
            {"number": 1, "name": "Vocabulary (S+S)", "issues": [
                {"number": 108, "title": "Rename X"},
                {"number": 109, "title": "Update docs"},
            ]},
            {"number": 2, "name": "API change (M)", "issues": [
                {"number": 111, "title": "Add weights"},
            ]},
        ]
        epic_manager.write_epic_slot_md(
            tmp_path, 1, ["engine"], "issue-50-profiles",
            "50", "casehubio/engine", batches, "Weighted profiles",
        )
        assert (tmp_path / "SLOT.md").exists()
        plan = epic_manager.parse_batch_plan(tmp_path)
        assert plan["is_epic"] is True
        assert plan["epic_number"] == "50"
        assert len(plan["batches"]) == 2
        assert plan["current_issue"] == 108
        assert plan["completed"] == []
```

- [ ] **Step 14: Run full test suite**

Run: `python3 -m pytest tests/test_epic_manager.py -v`
Expected: PASS

- [ ] **Step 15: Commit**

```bash
git add work-slot/epic_manager.py tests/test_epic_manager.py
git commit -m "feat(#TBD): epic_manager.py — batch plan parsing, advancement, and status

Refs #TBD"
```

---

### Task 2: Extend `parse_slot_md()` with `is_epic` flag

**Files:**
- Modify: `work-slot/slot_manager.py` (`parse_slot_md` function)
- Modify: `tests/test_slot_manager.py` (`TestParseSlotMd` class)

**Interfaces:**
- Consumes: existing `parse_slot_md()` return dict
- Produces: `parse_slot_md()` now also returns `"is_epic": bool`
  and `"type": str` in its result dict

- [ ] **Step 1: Write test for `is_epic` flag**

```python
# Add to tests/test_slot_manager.py TestParseSlotMd class

def test_detects_epic_type(self, tmp_path):
    (tmp_path / "SLOT.md").write_text(
        "# Slot 1 — issue-50-profiles\n\n## Issue\n"
        "casehubio/engine#50\nCovers: 108\nType: epic\n"
        "Safe exit: after any completed batch\n\n"
        "## What to do\nEpic work\n\n## Repos\n- engine\n"
    )
    md = slot_manager.parse_slot_md(tmp_path)
    assert md["is_epic"] is True
    assert md["issue"] == "50"
    assert md["issue_repo"] == "casehubio/engine"

def test_non_epic_defaults_false(self, tmp_path):
    (tmp_path / "SLOT.md").write_text(
        "# Slot 1 — issue-42-spi\n\n## Issue\n"
        "casehubio/engine#42\nCovers: 42\n\n"
        "## What to do\nSPI work\n\n## Repos\n- engine\n"
    )
    md = slot_manager.parse_slot_md(tmp_path)
    assert md.get("is_epic") is False
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_slot_manager.py::TestParseSlotMd::test_detects_epic_type -v`
Expected: FAIL — `KeyError: 'is_epic'`

- [ ] **Step 3: Implement `is_epic` detection in `parse_slot_md()`**

Modify the `parse_slot_md` function in `slot_manager.py`. Add `is_epic`
detection within the `## Issue` section parsing:

In the `in_issue` block, add detection for `Type: epic`:

```python
# Add to result initialization:
result["is_epic"] = False

# Inside the in_issue block, add:
if in_issue and line.strip().startswith("Type:"):
    result["is_epic"] = line.strip().split(":", 1)[1].strip() == "epic"
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_slot_manager.py::TestParseSlotMd -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add work-slot/slot_manager.py tests/test_slot_manager.py
git commit -m "feat(#TBD): parse_slot_md detects Type: epic flag

Refs #TBD"
```

---

### Task 3: Update `work-slot/SKILL.md` — epic, next, status commands

**Files:**
- Modify: `work-slot/SKILL.md`

**Interfaces:**
- Consumes: spec sections on Epic Slot Creation, In-Slot Workflow,
  work-slot status, Safe Exit
- Produces: updated SKILL.md with new command sections

- [ ] **Step 1: Add `work-slot epic` section to SKILL.md**

Add after the `## work-slot create` section. Content from the spec's
"Epic Slot Creation" section, adapted to skill instruction format:
- Step 0 (guard), Steps 1-9 (fetch, estimate, batch, approve, write
  GitHub, create slot, write SLOT.md, worklog)
- Include the batch plan table format example
- Reference `epic_manager.py write_epic_slot_md` for SLOT.md creation

- [ ] **Step 2: Add `work-slot next` section**

Add after the epic section. Content from the spec's "In-Slot Workflow,
Advancing through issues":
- Call `epic_manager.py advance <slot-dir>`
- Parse JSON result
- Announce batch complete / safe exit / epic complete based on flags
- Note: does NOT close issues — deferred to work-end

- [ ] **Step 3: Add `work-slot status` section**

Content from the spec's "work-slot status" section:
- Call `epic_manager.py status <slot-dir>`
- Format output as the progress table
- Include divergence detection note

- [ ] **Step 4: Update Skill Chaining section**

Add:
- **Complements: `work-start`** — reads SLOT.md `Type: epic` for
  resume context
- **Complements: `handover`** — includes epic progress in HANDOFF.md
- **Complements: `work-end`** — offers slot closure after Phase A

- [ ] **Step 5: Update CSO description**

Ensure the frontmatter `description` includes epic-related triggers:
"work-slot epic", "drive through the epic", "batch issues".

- [ ] **Step 6: Commit**

```bash
git add work-slot/SKILL.md
git commit -m "feat(#TBD): work-slot epic/next/status skill instructions

Refs #TBD"
```

---

### Task 4: Fix `work` skill routing — deterministic Python router

**Files:**
- Create: `work/work_router.py`
- Test: `tests/test_work_router.py`
- Modify: `work/SKILL.md`

**Interfaces:**
- Consumes: `ctx.py` output (via subprocess), filesystem checks
- Produces: `work_router.py` CLI with KEY=VALUE output:
  - `ROUTE=start|resume_branch|resume_stack|end|pause`
  - `ON_MAIN=yes|no`
  - `CURRENT_BRANCH=<name>`
  - `IN_SLOT=yes|no`
  - `IS_EPIC=yes|no`
  - `EPIC_BATCH=<N of M>` (if epic)
  - `EPIC_ACTIVE_ISSUE=<N>` (if epic)
  - `STACK_DEPTH=<N>`
  - `HAS_HANDOFF=yes|no`
  - `SLOT_MD_PATH=<path>` (if in slot)
  - `HANDOFF_PATH=<path>` (if exists)

**Problem:** Today `work resume` routes to `work-resume` immediately.
But `work-resume` is for restoring paused branches from the pause
stack on main. When a user says "resume handover" after `/clear`
while on a feature branch, they want to read the handover — not
restore from a pause stack. The LLM has demonstrated it gets this
routing wrong. A Python script makes it deterministic.

- [ ] **Step 1: Write tests for `work_router.py`**

```python
# tests/test_work_router.py

import sys
from pathlib import Path
from unittest.mock import patch

import pytest

skill_dir = Path(__file__).parent.parent / "work"
sys.path.insert(0, str(skill_dir))

import work_router


class TestDetectState:
    def test_on_main_no_stack(self, tmp_path):
        workspace = tmp_path / "workspace"
        workspace.mkdir()
        result = work_router.detect_state(
            current_branch="main",
            project_path=str(tmp_path / "project"),
            workspace_path=str(workspace),
        )
        assert result["ON_MAIN"] == "yes"
        assert result["ROUTE"] == "start"
        assert result["STACK_DEPTH"] == "0"

    def test_on_main_with_stack(self, tmp_path):
        workspace = tmp_path / "workspace"
        design = workspace / "design"
        design.mkdir(parents=True)
        (design / ".pause-stack").write_text(
            "- branch: issue-42-spi\n  issue: 42\n"
            "- branch: issue-55-ledger\n  issue: 55\n"
        )
        result = work_router.detect_state(
            current_branch="main",
            project_path=str(tmp_path / "project"),
            workspace_path=str(workspace),
        )
        assert result["ROUTE"] == "resume_stack"
        assert result["STACK_DEPTH"] == "2"

    def test_on_branch_no_slot(self, tmp_path):
        workspace = tmp_path / "workspace"
        workspace.mkdir()
        project = tmp_path / "project"
        project.mkdir()
        result = work_router.detect_state(
            current_branch="issue-42-spi",
            project_path=str(project),
            workspace_path=str(workspace),
        )
        assert result["ON_MAIN"] == "no"
        assert result["ROUTE"] == "resume_branch"
        assert result["IN_SLOT"] == "no"

    def test_on_branch_in_slot(self, tmp_path):
        workspace = tmp_path / "workspace"
        workspace.mkdir()
        family = tmp_path / "family"
        slot = family / "worktrees" / "1"
        project = slot / "engine"
        project.mkdir(parents=True)
        (slot / "SLOT.md").write_text(
            "# Slot 1 — issue-42-spi\n\n## Issue\n"
            "repo#42\nCovers: 42\n\n## What to do\nTest\n"
        )
        result = work_router.detect_state(
            current_branch="issue-42-spi",
            project_path=str(project),
            workspace_path=str(workspace),
        )
        assert result["IN_SLOT"] == "yes"
        assert result["IS_EPIC"] == "no"
        assert result["ROUTE"] == "resume_branch"

    def test_on_branch_in_epic_slot(self, tmp_path):
        workspace = tmp_path / "workspace"
        workspace.mkdir()
        family = tmp_path / "family"
        slot = family / "worktrees" / "1"
        project = slot / "engine"
        project.mkdir(parents=True)
        (slot / "SLOT.md").write_text(
            "# Slot 1 — issue-50-profiles\n\n## Issue\n"
            "casehubio/engine#50\nCovers: 108\nType: epic\n\n"
            "## What to do\nEpic work\n\n"
            "## Batch Plan\n\n"
            "### Batch 1 — Vocab (S+S)\n"
            "- [x] #108 — Done\n"
            "- [ ] #109 — Active ← active\n\n"
            "### Batch 2 — API (M)\n"
            "- [ ] #111 — Weights\n\n"
            "## Session State\nCurrent batch: 1\n"
            "Current issue: #109 — Active\n\n"
            "## Repos\n- engine\n"
        )
        result = work_router.detect_state(
            current_branch="issue-50-profiles",
            project_path=str(project),
            workspace_path=str(workspace),
        )
        assert result["IN_SLOT"] == "yes"
        assert result["IS_EPIC"] == "yes"
        assert result["EPIC_BATCH"] == "1 of 2"
        assert result["EPIC_ACTIVE_ISSUE"] == "109"
        assert result["ROUTE"] == "resume_branch"

    def test_on_branch_with_handoff(self, tmp_path):
        workspace = tmp_path / "workspace"
        workspace.mkdir()
        (workspace / "HANDOFF.md").write_text("# Handoff\nLast session did X.")
        project = tmp_path / "project"
        project.mkdir()
        result = work_router.detect_state(
            current_branch="issue-42-spi",
            project_path=str(project),
            workspace_path=str(workspace),
        )
        assert result["HAS_HANDOFF"] == "yes"
        assert result["HANDOFF_PATH"] == str(workspace / "HANDOFF.md")

    def test_on_branch_with_stack(self, tmp_path):
        workspace = tmp_path / "workspace"
        design = workspace / "design"
        design.mkdir(parents=True)
        (design / ".pause-stack").write_text(
            "- branch: issue-55-ledger\n  issue: 55\n"
        )
        project = tmp_path / "project"
        project.mkdir()
        result = work_router.detect_state(
            current_branch="issue-42-spi",
            project_path=str(project),
            workspace_path=str(workspace),
        )
        assert result["ROUTE"] == "resume_branch"
        assert result["STACK_DEPTH"] == "1"
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_work_router.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'work_router'`

- [ ] **Step 3: Implement `work_router.py`**

```python
#!/usr/bin/env python3
"""
work_router.py — Deterministic work lifecycle routing

Usage:
    python3 work_router.py <current_branch> <project_path> <workspace_path>

Output (KEY=VALUE lines):
    ROUTE=start|resume_branch|resume_stack|end|pause
    ON_MAIN=yes|no
    CURRENT_BRANCH=<name>
    IN_SLOT=yes|no
    IS_EPIC=yes|no
    EPIC_BATCH=<N of M>       (only if IS_EPIC=yes)
    EPIC_ACTIVE_ISSUE=<N>     (only if IS_EPIC=yes)
    STACK_DEPTH=<N>
    HAS_HANDOFF=yes|no
    HANDOFF_PATH=<path>       (only if HAS_HANDOFF=yes)
    SLOT_MD_PATH=<path>       (only if IN_SLOT=yes)
"""

import re
import sys
from pathlib import Path


def detect_state(current_branch: str, project_path: str,
                 workspace_path: str) -> dict[str, str]:
    project = Path(project_path)
    workspace = Path(workspace_path)

    on_main = current_branch == "main"

    # Pause stack
    stack_file = workspace / "design" / ".pause-stack"
    stack_depth = 0
    if stack_file.exists():
        stack_depth = sum(
            1 for line in stack_file.read_text().splitlines()
            if line.strip().startswith("- branch:")
        )

    # Slot detection: project path contains /worktrees/ and
    # SLOT.md exists one level up
    in_slot = False
    is_epic = False
    slot_md_path = ""
    epic_batch = ""
    epic_active_issue = ""

    if "/worktrees/" in str(project):
        candidate = project.parent / "SLOT.md"
        if candidate.exists():
            in_slot = True
            slot_md_path = str(candidate)
            content = candidate.read_text()

            # Check for Type: epic
            in_issue_section = False
            for line in content.splitlines():
                if line.startswith("## Issue"):
                    in_issue_section = True
                    continue
                if line.startswith("## ") and in_issue_section:
                    in_issue_section = False
                    continue
                if in_issue_section and line.strip() == "Type: epic":
                    is_epic = True

            if is_epic:
                # Count batches
                batch_numbers = re.findall(
                    r"^### Batch (\d+)", content, re.MULTILINE
                )
                total_batches = len(batch_numbers)

                # Find current batch from Session State
                m = re.search(
                    r"^Current batch:\s*(\d+)", content, re.MULTILINE
                )
                current_batch = m.group(1) if m else "0"
                epic_batch = f"{current_batch} of {total_batches}"

                # Find active issue
                m = re.search(
                    r"^Current issue:\s*#(\d+)", content, re.MULTILINE
                )
                epic_active_issue = m.group(1) if m else ""

    # Handoff detection
    has_handoff = False
    handoff_path = ""
    handoff_candidate = workspace / "HANDOFF.md"
    if handoff_candidate.exists():
        has_handoff = True
        handoff_path = str(handoff_candidate)

    # Route determination
    if on_main:
        route = "resume_stack" if stack_depth > 0 else "start"
    else:
        route = "resume_branch"

    result = {
        "ROUTE": route,
        "ON_MAIN": "yes" if on_main else "no",
        "CURRENT_BRANCH": current_branch,
        "IN_SLOT": "yes" if in_slot else "no",
        "IS_EPIC": "yes" if is_epic else "no",
        "STACK_DEPTH": str(stack_depth),
        "HAS_HANDOFF": "yes" if has_handoff else "no",
    }
    if epic_batch:
        result["EPIC_BATCH"] = epic_batch
    if epic_active_issue:
        result["EPIC_ACTIVE_ISSUE"] = epic_active_issue
    if handoff_path:
        result["HANDOFF_PATH"] = handoff_path
    if slot_md_path:
        result["SLOT_MD_PATH"] = slot_md_path

    return result


def main() -> int:
    if len(sys.argv) < 4:
        print(__doc__, file=sys.stderr)
        return 1

    current_branch = sys.argv[1]
    project_path = sys.argv[2]
    workspace_path = sys.argv[3]

    result = detect_state(current_branch, project_path, workspace_path)
    for key, value in result.items():
        print(f"{key}={value}")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_work_router.py -v`
Expected: PASS

- [ ] **Step 5: Update `work/SKILL.md` to use `work_router.py`**

Replace the current Step 1 routing table and Step 2 state detection
with a single script call:

```markdown
**Step 1 — Parse invocation and detect state**

| Invocation | Intent |
|------------|--------|
| `work` or `work start` | `intent=work` |
| `work end` | → **work-end** immediately (no router needed) |
| `work pause` | → **work-pause** immediately (no router needed) |
| `work resume` or `resume handover` or `continue` | `intent=resume` |

For `work end` and `work pause`, route immediately — no state
detection needed.

For all other invocations, run the router:

```bash
python3 ~/.claude/skills/project/ctx.py
# Read PROJECT, WORKSPACE, CURRENT_BRANCH from output

python3 ~/.claude/skills/work/work_router.py \
  $CURRENT_BRANCH $PROJECT $WORKSPACE
```

**Step 2 — Route based on output**

| `ROUTE` | `ON_MAIN` | Action |
|---------|-----------|--------|
| `start` | yes | → **work-start** — begin new work |
| `resume_stack` | yes | → show stack picker (Step 3), then **work-resume** |
| `resume_branch` | no | → contextual options (Step 4) |

**Step 3 — Stack picker** (unchanged from current)

**Step 4 — On feature branch: contextual options**

Present options based on the router output. The router has already
determined slot context, epic state, pause stack depth, and handoff
existence — do NOT re-derive these with additional tool calls.

Always present:
> 1. **resume** — read the last handover and continue where I left off

If `STACK_DEPTH > 0`:
> 2. **switch** — you have <N> paused branch(es) — resume one instead

Always present:
> 3. **end** — close this branch
> 4. **pause** — save WIP, switch to main

**On resume (option 1):**

If `HAS_HANDOFF=yes`: read `$HANDOFF_PATH`
If `IS_EPIC=yes`: also read SLOT.md at `$SLOT_MD_PATH` for batch
progress and active issue. Display:
```
Epic — Batch $EPIC_BATCH
Active issue: #$EPIC_ACTIVE_ISSUE
```
Set active issue for commit linkage (`Refs #$EPIC_ACTIVE_ISSUE`).

If `IN_SLOT=yes` but `IS_EPIC=no`: read SLOT.md for issue context.

Summarise what the last session accomplished and continue working.
Do NOT invoke work-start — the branch and scaffold already exist.

**On switch (option 2):**
Route to **work-pause**, then **work-resume** (stack picker).
```

- [ ] **Step 6: Commit**

```bash
git add work/work_router.py tests/test_work_router.py work/SKILL.md
git commit -m "fix(#TBD): deterministic work routing via work_router.py

Replaces LLM-interpreted routing with a Python script. Fixes
resume on a feature branch routing to work-resume (pause stack)
instead of reading the handover.

Refs #TBD"
```

---

### Task 5: Update `work-start/SKILL.md` — epic overlay on resume

**Files:**
- Modify: `work-start/SKILL.md`

**Interfaces:**
- Consumes: `parse_batch_plan()` from `epic_manager.py` (called by
  the LLM reading SLOT.md, not as a script call)
- Produces: updated skill with epic overlay after state 2 resume

- [ ] **Step 1: Add epic overlay section**

Add after the existing state 2 resume steps (after Step 11). Content
from the spec's "Integration with work-start":

1. Guard: `$PROJECT` path must contain `/worktrees/`
2. Check `$PROJECT/../SLOT.md` for `Type: epic`
3. If epic: read Session State, show batch context, set active issue
   for commit linkage

- [ ] **Step 2: Commit**

```bash
git add work-start/SKILL.md
git commit -m "feat(#TBD): work-start epic overlay on slot resume

Refs #TBD"
```

---

### Task 6: Update `handover/SKILL.md` — epic progress section

**Files:**
- Modify: `handover/SKILL.md`

**Interfaces:**
- Consumes: SLOT.md `Type: epic` detection, `## Batch Plan` and
  `## Session State` sections
- Produces: "Epic Progress" section in HANDOFF.md when in epic slot

- [ ] **Step 1: Add epic progress detection**

Add to the handover skill's output generation. Content from the spec's
"Integration with handover":

1. Guard: `$PROJECT` path must contain `/worktrees/`
2. Check `$PROJECT/../SLOT.md` for `Type: epic`
3. If epic: add "## Epic Progress" section to HANDOFF.md with batch
   summary, done/active/next issues
4. Placement: after "## Last Session", before "## What's Left"

- [ ] **Step 2: Also update SLOT.md Session State on wrap**

Add instruction: when handover runs and epic context is detected,
update `## Session State` in SLOT.md with current position and
last wrap timestamp.

- [ ] **Step 3: Commit**

```bash
git add handover/SKILL.md
git commit -m "feat(#TBD): handover includes epic progress in HANDOFF.md

Refs #TBD"
```

---

### Task 7: Update `work-end/SKILL.md` — slot closure prompt

**Files:**
- Modify: `work-end/SKILL.md`

**Interfaces:**
- Consumes: slot mode detection (`/worktrees/` in `$PROJECT`)
- Produces: post-Phase-A prompt to stamp/close/archive slot

- [ ] **Step 1: Add slot closure prompt after Phase A**

After the existing Phase A completion (`.phase-a-complete` marker
written), add:

> After Phase A completes in a slot, offer:
>
> "Phase A complete. Would you like to stamp, close, and archive
> this slot now? (y/n)"
>
> If yes: run `work-slot merge` flow (Phase B).
> If no: slot stays as "ready to land" for later merge.

- [ ] **Step 2: Add epic issue closure note**

In the Phase B / issue closure section, add note: when the slot is
an epic slot and `epic_complete` was flagged by `work-slot next`,
the epic issue itself is included in COVERS and closed alongside
child issues.

- [ ] **Step 3: Commit**

```bash
git add work-end/SKILL.md
git commit -m "feat(#TBD): work-end offers slot closure after Phase A

Refs #TBD"
```

---

### Task 8: Sync and validate

**Files:**
- Run: `python3 scripts/claude-skill sync-local --all -y`
- Run: `python3 scripts/validate_all.py --tier commit`
- Run: `python3 -m pytest tests/test_epic_manager.py tests/test_slot_manager.py -v`

- [ ] **Step 1: Sync skills**

```bash
python3 scripts/claude-skill sync-local --all -y
```

- [ ] **Step 2: Run commit-tier validation**

```bash
python3 scripts/validate_all.py --tier commit
```

Expected: PASS. Fix any validation errors.

- [ ] **Step 3: Run all relevant tests**

```bash
python3 -m pytest tests/test_epic_manager.py tests/test_slot_manager.py tests/test_worklog.py -v
```

Expected: PASS

- [ ] **Step 4: Follow readme-sync workflow**

Since SKILL.md files were modified, follow `docs/development/readme-sync.md`
to check if README.md needs updates (new commands, updated chaining).

- [ ] **Step 5: Commit any sync/validation fixes**

```bash
git add -A
git commit -m "chore(#TBD): sync, validate, and readme updates

Refs #TBD"
```
