# Unified Review and Findings Persistence Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** TBD — create before starting work
**Issue group:** TBD

**Goal:** Build a unified findings persistence layer, branch-level code
audit, loose ends sweep, and forcing function that ensure nothing slips
through work-end or handover unaddressed.

**Architecture:** JSONL append-only file at `$WORKSPACE/.audit/findings.jsonl`
is the single persistence sink. Four producers (hygiene_scan, code-review,
branch-audit, loose-ends-sweep) write findings. Three consumers
(work_health.py at session entry, forcing function at work-end, loose
ends sweep) read them via a shared `read_findings()` function implementing
the reader contract. Two deprecated skills are removed, VBC preserved as
a protocol.

**Tech Stack:** Python 3.11+, pytest, JSONL, fcntl (advisory locking)

## Global Constraints

- All Python files require type hints on public functions
- All new scripts require tests (protocol: externalised-scripts-require-tests)
- JSONL format (one JSON object per line, append-only)
- Advisory locking via `fcntl.flock` on all writes
- Dedup key: `(check, location, branch)` — readers resolve, writers blind-append
- Backward compatible with existing `findings.json` (migration in Batch 1)

---

## Batch 1: Findings Persistence Foundation

After this batch: findings.jsonl exists as the persistence format with
a shared reader/writer library, existing hygiene_scan.py and work_health.py
migrated from JSON to JSONL, all existing tests pass.

### Task 1: Create findings library with reader contract and writer protocol

**Files:**
- Create: `project/findings.py`
- Test: `tests/test_findings.py`

**Interfaces:**
- Produces: `read_findings(path: Path) -> list[dict]` — canonical reader
  implementing the 5-step contract (group by dedup key, latest timestamp
  for status, highest severity, source from highest-severity entry,
  resolution updates are appended lines)
- Produces: `append_finding(path: Path, finding: dict) -> None` — writer
  with advisory flock
- Produces: `compact_findings(path: Path, archive_path: Path, max_age_days: int = 30) -> int` —
  compaction with flock + atomic rename, returns archived count
- Produces: `Finding` TypedDict with all extended format fields

- [ ] **Step 1: Write failing tests for read_findings**

```python
# tests/test_findings.py
import json
import tempfile
from pathlib import Path
from project.findings import read_findings, append_finding, Finding

def test_read_empty_file():
    with tempfile.NamedTemporaryFile(suffix=".jsonl", delete=False) as f:
        path = Path(f.name)
    path.write_text("")
    assert read_findings(path) == []

def test_read_single_finding():
    with tempfile.NamedTemporaryFile(suffix=".jsonl", delete=False) as f:
        path = Path(f.name)
    finding = {"category": "hygiene", "check": "stale_branch",
               "location": "branch:issue-100", "detail": "stale",
               "status": "open", "timestamp": "2026-08-19T10:00:00Z"}
    path.write_text(json.dumps(finding) + "\n")
    result = read_findings(path)
    assert len(result) == 1
    assert result[0]["status"] == "open"

def test_dedup_by_check_location_branch():
    with tempfile.NamedTemporaryFile(suffix=".jsonl", delete=False) as f:
        path = Path(f.name)
    f1 = {"category": "audit", "check": "missing-req", "location": "spec:req-3",
          "branch": "issue-123", "detail": "v1", "severity": "warning",
          "status": "open", "timestamp": "2026-08-19T10:00:00Z"}
    f2 = {**f1, "detail": "v2", "status": "resolved",
          "resolution": "fixed in abc1234",
          "timestamp": "2026-08-19T11:00:00Z"}
    path.write_text(json.dumps(f1) + "\n" + json.dumps(f2) + "\n")
    result = read_findings(path)
    assert len(result) == 1
    assert result[0]["status"] == "resolved"

def test_highest_severity_wins():
    with tempfile.NamedTemporaryFile(suffix=".jsonl", delete=False) as f:
        path = Path(f.name)
    f1 = {"category": "review", "check": "unsafe-code", "location": "src/x.py:42",
          "branch": "issue-123", "detail": "bad", "severity": "critical",
          "status": "open", "timestamp": "2026-08-19T10:00:00Z"}
    f2 = {**f1, "severity": "warning", "timestamp": "2026-08-19T11:00:00Z"}
    path.write_text(json.dumps(f1) + "\n" + json.dumps(f2) + "\n")
    result = read_findings(path)
    assert result[0]["severity"] == "critical"

def test_default_severity_is_warning():
    with tempfile.NamedTemporaryFile(suffix=".jsonl", delete=False) as f:
        path = Path(f.name)
    finding = {"category": "hygiene", "check": "stale_branch",
               "location": "branch:issue-100", "detail": "stale",
               "status": "open", "timestamp": "2026-08-19T10:00:00Z"}
    path.write_text(json.dumps(finding) + "\n")
    result = read_findings(path)
    assert result[0]["severity"] == "warning"

def test_fallback_dedup_when_no_location():
    with tempfile.NamedTemporaryFile(suffix=".jsonl", delete=False) as f:
        path = Path(f.name)
    f1 = {"category": "hygiene", "check": "stale_branch",
          "detail": "issue-100 stale", "status": "open",
          "timestamp": "2026-08-19T10:00:00Z"}
    f2 = {**f1, "status": "dismissed", "resolution": "cleaned up",
          "timestamp": "2026-08-19T11:00:00Z"}
    path.write_text(json.dumps(f1) + "\n" + json.dumps(f2) + "\n")
    result = read_findings(path)
    assert len(result) == 1
    assert result[0]["status"] == "dismissed"

def test_file_not_found_returns_empty():
    result = read_findings(Path("/nonexistent/findings.jsonl"))
    assert result == []
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_findings.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'project.findings'`

- [ ] **Step 3: Implement findings.py**

```python
# project/findings.py
"""Unified findings persistence — JSONL append-only with reader contract.

Writers blind-append under advisory flock. Readers group by dedup key,
resolve status by latest timestamp, severity by highest across group.
"""
from __future__ import annotations

import fcntl
import json
import os
import sys
from datetime import datetime, timezone
from pathlib import Path
from typing import TypedDict

SEVERITY_ORDER = {"critical": 3, "warning": 2, "note": 1}


class Finding(TypedDict, total=False):
    category: str       # review|audit|loose-end|hygiene
    dimension: str      # conformance|coherence|structure|robustness|null
    severity: str       # critical|warning|note
    check: str          # missing-requirement|dead-code|deferred-plan-item|...
    location: str       # src/sync.py:42|spec:req-3|plan:add-tests
    detail: str         # human-readable description
    source: str         # branch-audit|code-review|loose-ends-sweep|hygiene-scan
    branch: str         # issue-123-feature-name
    status: str         # open|resolved|dismissed|filed
    resolution: str     # fixed in abc1234|filed as #456|dismissed: reason
    timestamp: str      # ISO-8601


def _dedup_key(f: dict) -> tuple:
    loc = f.get("location")
    if loc:
        return (f.get("check", ""), loc, f.get("branch"))
    return (f.get("check", ""), f.get("detail", ""), f.get("branch"))


def read_findings(path: Path) -> list[dict]:
    if not path.exists():
        return []
    groups: dict[tuple, list[dict]] = {}
    with open(path) as fh:
        for line in fh:
            line = line.strip()
            if not line:
                continue
            try:
                entry = json.loads(line)
            except json.JSONDecodeError:
                continue
            key = _dedup_key(entry)
            groups.setdefault(key, []).append(entry)

    result = []
    for key, entries in groups.items():
        latest = max(entries, key=lambda e: e.get("timestamp", ""))
        highest_sev = max(
            entries,
            key=lambda e: SEVERITY_ORDER.get(e.get("severity", "warning"), 2),
        )
        merged = {**latest}
        merged["severity"] = highest_sev.get("severity", "warning")
        if "severity" not in merged or not merged["severity"]:
            merged["severity"] = "warning"
        merged["source"] = highest_sev.get("source", latest.get("source"))
        result.append(merged)
    return result


def append_finding(path: Path, finding: dict) -> None:
    path.parent.mkdir(parents=True, exist_ok=True)
    with open(path, "a") as fh:
        fcntl.flock(fh, fcntl.LOCK_EX)
        try:
            fh.write(json.dumps(finding) + "\n")
        finally:
            fcntl.flock(fh, fcntl.LOCK_UN)


def compact_findings(
    path: Path, archive_path: Path, max_age_days: int = 30
) -> int:
    if not path.exists():
        return 0
    cutoff = datetime.now(timezone.utc).isoformat()
    with open(path) as fh:
        fcntl.flock(fh, fcntl.LOCK_EX)
        try:
            lines = fh.readlines()
            keep = []
            archive = []
            for line in lines:
                line = line.strip()
                if not line:
                    continue
                try:
                    entry = json.loads(line)
                except json.JSONDecodeError:
                    keep.append(line)
                    continue
                if entry.get("status") in ("resolved", "dismissed", "filed"):
                    ts = entry.get("timestamp", "")
                    if ts and ts < cutoff:
                        archive.append(line)
                    else:
                        keep.append(line)
                else:
                    keep.append(line)
            tmp = path.with_suffix(".tmp")
            tmp.write_text("\n".join(keep) + ("\n" if keep else ""))
            os.rename(str(tmp), str(path))
            if archive:
                archive_path.parent.mkdir(parents=True, exist_ok=True)
                with open(archive_path, "a") as af:
                    af.write("\n".join(archive) + "\n")
        finally:
            fcntl.flock(fh, fcntl.LOCK_UN)
    return len(archive)
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_findings.py -v`
Expected: All 7 tests PASS

- [ ] **Step 5: Write tests for append_finding and compact_findings**

```python
# append to tests/test_findings.py
from project.findings import compact_findings

def test_append_finding_creates_file():
    with tempfile.TemporaryDirectory() as d:
        path = Path(d) / ".audit" / "findings.jsonl"
        finding = {"category": "audit", "check": "test", "detail": "x",
                   "status": "open", "timestamp": "2026-08-19T10:00:00Z"}
        append_finding(path, finding)
        assert path.exists()
        lines = path.read_text().strip().split("\n")
        assert len(lines) == 1
        assert json.loads(lines[0])["check"] == "test"

def test_append_finding_appends_to_existing():
    with tempfile.TemporaryDirectory() as d:
        path = Path(d) / "findings.jsonl"
        f1 = {"category": "audit", "check": "a", "detail": "x",
              "status": "open", "timestamp": "2026-08-19T10:00:00Z"}
        f2 = {"category": "audit", "check": "b", "detail": "y",
              "status": "open", "timestamp": "2026-08-19T11:00:00Z"}
        append_finding(path, f1)
        append_finding(path, f2)
        lines = path.read_text().strip().split("\n")
        assert len(lines) == 2

def test_compact_archives_old_resolved():
    with tempfile.TemporaryDirectory() as d:
        path = Path(d) / "findings.jsonl"
        archive = Path(d) / "findings-archive.jsonl"
        old = {"category": "audit", "check": "a", "detail": "x",
               "status": "resolved", "resolution": "fixed",
               "timestamp": "2026-07-01T10:00:00Z"}
        current = {"category": "audit", "check": "b", "detail": "y",
                   "status": "open",
                   "timestamp": "2026-08-19T10:00:00Z"}
        path.write_text(json.dumps(old) + "\n" + json.dumps(current) + "\n")
        archived = compact_findings(path, archive)
        assert archived == 1
        remaining = path.read_text().strip().split("\n")
        assert len(remaining) == 1
        assert json.loads(remaining[0])["check"] == "b"
        assert archive.exists()
```

- [ ] **Step 6: Run tests**

Run: `python3 -m pytest tests/test_findings.py -v`
Expected: All 10 tests PASS

- [ ] **Step 7: Commit**

```bash
git add project/findings.py tests/test_findings.py
git commit -m "feat(#N): add findings persistence library with JSONL reader contract and writer protocol"
```

### Task 2: Migrate hygiene_scan.py from JSON to JSONL

**Files:**
- Modify: `work-end/hygiene_scan.py:246-287` (replace `persist_findings`)
- Modify: `tests/test_hygiene_scan.py`

**Interfaces:**
- Consumes: `append_finding(path: Path, finding: dict) -> None` from `project/findings.py`
- Produces: writes to `$WORKSPACE/.audit/findings.jsonl` (was `findings.json`)

- [ ] **Step 1: Write failing test for JSONL output**

```python
# add to tests/test_hygiene_scan.py
def test_persist_findings_writes_jsonl(tmp_path):
    """persist_findings writes JSONL (one object per line), not JSON array."""
    workspace = tmp_path / "workspace"
    workspace.mkdir()
    result = {
        "unrecovered_artifacts": [
            {"type": "blog", "file": "entry.md", "branch": "issue-100"}
        ],
        "unstamped_branches": [],
        "stale_branches": [],
    }
    from importlib import reload
    import work_end.hygiene_scan as hs
    reload(hs)
    hs.persist_findings(str(workspace), result)
    path = workspace / ".audit" / "findings.jsonl"
    assert path.exists()
    lines = path.read_text().strip().split("\n")
    assert len(lines) == 1
    entry = json.loads(lines[0])
    assert entry["category"] == "hygiene"
    assert entry["check"] == "unrecovered_artifact"
    assert "location" in entry
    assert "severity" in entry
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_hygiene_scan.py::test_persist_findings_writes_jsonl -v`
Expected: FAIL (still writes JSON array to `findings.json`)

- [ ] **Step 3: Rewrite persist_findings to use append_finding**

Replace the `persist_findings` function in `work-end/hygiene_scan.py:246-287`
with a version that uses `append_finding` from `project/findings.py`,
writes to `findings.jsonl`, adds `location`, `severity`, and `source`
fields, and uses the extended format.

- [ ] **Step 4: Run all hygiene_scan tests**

Run: `python3 -m pytest tests/test_hygiene_scan.py -v`
Expected: All pass (existing tests may need updates for JSONL path change)

- [ ] **Step 5: Commit**

```bash
git add work-end/hygiene_scan.py tests/test_hygiene_scan.py
git commit -m "feat(#N): migrate hygiene_scan.py from JSON to JSONL via findings library"
```

### Task 3: Migrate work_health.py from JSON to JSONL

**Files:**
- Modify: `project/work_health.py:427-441` (replace `check_prior_findings`)
- Modify: `tests/test_work_health.py`

**Interfaces:**
- Consumes: `read_findings(path: Path) -> list[dict]` from `project/findings.py`
- Produces: enhanced `CHECK=prior_findings` output with severity and category

- [ ] **Step 1: Write failing test for enhanced display**

```python
# add to tests/test_work_health.py
def test_check_prior_findings_reads_jsonl(tmp_path):
    """check_prior_findings reads findings.jsonl with extended format."""
    workspace = tmp_path / "workspace"
    audit = workspace / ".audit"
    audit.mkdir(parents=True)
    finding = json.dumps({
        "category": "audit", "dimension": "conformance",
        "severity": "warning", "check": "missing-req",
        "location": "spec:req-3",
        "detail": "Requirement 3 not implemented",
        "source": "branch-audit", "branch": "issue-123",
        "status": "open", "timestamp": "2026-08-19T10:00:00Z"
    })
    (audit / "findings.jsonl").write_text(finding + "\n")
    result = check_prior_findings(tmp_path, workspace)
    assert "STATUS=warn" in result
    assert "audit/conformance/WARNING" in result or "warning" in result.lower()
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_work_health.py::test_check_prior_findings_reads_jsonl -v`
Expected: FAIL (still reads `findings.json`)

- [ ] **Step 3: Rewrite check_prior_findings to use read_findings**

Replace the function in `project/work_health.py:427-441` to use
`read_findings` from `project/findings.py`, read from `findings.jsonl`,
and format output as `[category/dimension/severity] detail (source)`.

- [ ] **Step 4: Run all work_health tests**

Run: `python3 -m pytest tests/test_work_health.py -v`
Expected: All pass

- [ ] **Step 5: Commit**

```bash
git add project/work_health.py tests/test_work_health.py
git commit -m "feat(#N): migrate work_health.py to JSONL reader with enhanced display"
```

---

## Batch 2: Loose Ends Sweep

After this batch: loose ends sweep captures deferred/skipped/missing
items at handover and work-end, writing to findings.jsonl. Handover
checklist updated.

### Task 4: Create loose ends sweep script

**Files:**
- Create: `work-end/loose_ends_sweep.py`
- Test: `tests/test_loose_ends_sweep.py`

**Interfaces:**
- Consumes: `read_findings(path)` and `append_finding(path, finding)` from `project/findings.py`
- Produces: `main(workspace, project, branch, plan_path=None, cycle_start=None) -> dict` —
  returns `{"new_findings": N, "prior_open": M, "total_open": K}` as JSON to stdout

The script handles the mechanical checks (deferred plan items, TODOs in
code, open findings from prior sessions). LLM-dependent checks
("I'll come back to this" from conversation) are handled by the
skill SKILL.md, not this script.

- [ ] **Step 1: Write failing tests**

```python
# tests/test_loose_ends_sweep.py
import json
import subprocess
import tempfile
from pathlib import Path

def test_sweep_finds_deferred_plan_items(tmp_path):
    """Deferred items in .plan are captured as findings."""
    workspace = tmp_path / "workspace"
    workspace.mkdir()
    plan = workspace / ".plan"
    plan.write_text(
        "## Queue\n"
        "- [x] #100 — done\n"
        "- [ ] #101 — pending\n"
        "\n## Deferred\n"
        "- [ ] #102 — deferred: out of scope\n"
    )
    result = subprocess.run(
        ["python3", "work-end/loose_ends_sweep.py",
         f"workspace={workspace}", f"project={tmp_path}",
         "branch=issue-100"],
        capture_output=True, text=True, cwd="/Users/mdproctor/claude/hortora/soredium"
    )
    output = json.loads(result.stdout)
    assert output["new_findings"] >= 1

def test_sweep_reads_prior_findings(tmp_path):
    """Prior session findings are surfaced."""
    workspace = tmp_path / "workspace"
    audit = workspace / ".audit"
    audit.mkdir(parents=True)
    finding = json.dumps({
        "category": "audit", "check": "missing-req",
        "location": "spec:req-3", "detail": "old finding",
        "status": "open", "branch": "issue-100",
        "timestamp": "2026-08-18T10:00:00Z"
    })
    (audit / "findings.jsonl").write_text(finding + "\n")
    result = subprocess.run(
        ["python3", "work-end/loose_ends_sweep.py",
         f"workspace={workspace}", f"project={tmp_path}",
         "branch=issue-100"],
        capture_output=True, text=True, cwd="/Users/mdproctor/claude/hortora/soredium"
    )
    output = json.loads(result.stdout)
    assert output["prior_open"] >= 1

def test_sweep_temporal_filter(tmp_path):
    """Findings from current cycle are excluded from prior count."""
    workspace = tmp_path / "workspace"
    audit = workspace / ".audit"
    audit.mkdir(parents=True)
    now_finding = json.dumps({
        "category": "review", "check": "unsafe-code",
        "location": "src/x.py:42", "detail": "current session",
        "status": "open", "branch": "issue-100",
        "timestamp": "2026-08-19T15:00:00Z"
    })
    (audit / "findings.jsonl").write_text(now_finding + "\n")
    result = subprocess.run(
        ["python3", "work-end/loose_ends_sweep.py",
         f"workspace={workspace}", f"project={tmp_path}",
         "branch=issue-100",
         "cycle_start=2026-08-19T14:00:00Z"],
        capture_output=True, text=True, cwd="/Users/mdproctor/claude/hortora/soredium"
    )
    output = json.loads(result.stdout)
    assert output["prior_open"] == 0
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_loose_ends_sweep.py -v`
Expected: FAIL with FileNotFoundError (script doesn't exist)

- [ ] **Step 3: Implement loose_ends_sweep.py**

Create `work-end/loose_ends_sweep.py` that:
1. Reads `.plan` for deferred items (if `plan_path` provided)
2. Searches changed files for TODOs referencing the branch/issue
3. Reads `findings.jsonl` for prior open findings (filtered by `cycle_start`)
4. Appends new findings to `findings.jsonl` via `append_finding`
5. Outputs JSON summary to stdout

- [ ] **Step 4: Run tests**

Run: `python3 -m pytest tests/test_loose_ends_sweep.py -v`
Expected: All 3 tests PASS

- [ ] **Step 5: Commit**

```bash
git add work-end/loose_ends_sweep.py tests/test_loose_ends_sweep.py
git commit -m "feat(#N): add loose ends sweep script with temporal filtering"
```

### Task 5: Update handover SKILL.md with loose ends sweep

**Files:**
- Modify: `handover/SKILL.md` (add loose ends sweep to Step 0 checklist)

**Interfaces:**
- Consumes: `work-end/loose_ends_sweep.py` (script call)

- [ ] **Step 1: Add loose ends sweep as item 1 in handover Step 0 checklist**

Insert `[x] 1  Loose ends sweep   capture deferred/skipped/missing items`
as the first item. Renumber existing items 1-8 to 2-9. Add the sweep
to the run order (first, before forage sweep). Note that the sweep is
session-bound (depends on conversation context for LLM checks).

- [ ] **Step 2: Add run instructions**

After the run order list, add instructions for invoking the sweep:

```bash
python3 work-end/loose_ends_sweep.py workspace=<WS> project=<PROJ> branch=<BRANCH>
```

The LLM supplements the script output with conversation-context items
("I'll come back to this") and appends those to `findings.jsonl` via
the findings library.

- [ ] **Step 3: Commit**

```bash
git add handover/SKILL.md
git commit -m "feat(#N): add loose ends sweep to handover wrap checklist"
```

---

## Batch 3: Branch Audit Skill

After this batch: branch-audit skill exists with four dimensions,
produces findings in the extended format.

### Task 6: Create branch-audit SKILL.md

**Files:**
- Create: `branch-audit/SKILL.md`
- Create: `branch-audit/commands/branch-audit.md`

**Interfaces:**
- Consumes: `append_finding(path, finding)` from `project/findings.py`
- Produces: findings written to `findings.jsonl` after each dimension

- [ ] **Step 1: Write SKILL.md with four dimensions**

Create `branch-audit/SKILL.md` with:
- CSO description: "Use when reviewing a full branch diff at lifecycle
  gates (work-end, handover) — holistic review across conformance,
  coherence, structure, and robustness dimensions. NOT for per-commit
  review (use code-review). NOT for spec review (use design-review)."
- Four dimension sections with "includes but not limited to" sub-concerns
  per the spec
- Conformance context detection (spec → issue → commit messages)
- Execution model: inline, single pass per dimension
- Security-audit escalation from Robustness
- Findings persistence: append to `findings.jsonl` after each dimension
- Skill Chaining: invoked by work-end Step 2.2; complements code-review,
  design-review, security-audit

- [ ] **Step 2: Generate slash command**

```bash
python3 scripts/generate_commands.py
```

Verify `branch-audit/commands/branch-audit.md` was created.

- [ ] **Step 3: Commit**

```bash
git add branch-audit/
git commit -m "feat(#N): add branch-audit skill with four dimensions"
```

---

## Batch 4: Work-End Restructure and Forcing Function

After this batch: work-end runs review at Step 2 (before sweep), forcing
function drains all findings before Execute.

### Task 7: Restructure work-end flow

**Files:**
- Modify: `work-end/SKILL.md` (major restructure — Steps 2 and 3 change)

**Interfaces:**
- Consumes: `branch-audit` skill (Step 2.2)
- Consumes: `work-end/loose_ends_sweep.py` (Step 2.3)
- Consumes: `read_findings()` and `append_finding()` from `project/findings.py`

- [ ] **Step 1: Insert new Step 2 — Review**

After Step 1 (Context), insert:

```
## Step 2 — Review

2.1  code-review         per-line checklist on branch diff
2.2  branch-audit        four dimensions on branch diff
2.3  loose ends sweep    session + prior findings scan
2.4  forcing function    present all findings, require resolution
```

Mark all four sub-steps as hard gates. Include the security-audit
suppression instruction for Step 2.1. Include the duration estimate
(10-30 minutes). Include the temporal filtering note for Step 2.3.

- [ ] **Step 2: Document the forcing function (Step 2.4)**

Add the forcing function specification:
- Presentation format (grouped by category)
- Resolution options (Fix/File/Dismiss) with severity constraints
  (CRITICAL: no Dismiss)
- Triage filtering (only surviving findings, not reviewer-rejected ones)
- Re-review loop after Fix resolutions
- Batch operations (File all as single issue, File each, Dismiss all NOTEs)
- Checkpoint behavior (each resolution appended immediately)

- [ ] **Step 3: Renumber old Step 2 (Sweep) to Step 3**

Old Step 2 (Sweep) becomes Step 3. Old Step 3 (Execute) becomes Step 4.
Remove code-review from old Step 3.1 (now Step 4.1) — it's in Step 2.1.
Remove the `design-review --mode final-review` conditional — branch-audit
replaces it at Step 2.2.

Update Execute sequence:
```
4.1  Promote artifacts
4.2  Phase A: Rebase
4.3  Phase B: Squash
4.4  Phase C: Land
```

- [ ] **Step 4: Add post-rebase mini-gate**

After Step 4.2 (Rebase), if non-fast-forward: re-run code-review on
conflict resolution diff. Mini-gate with Fix/File/Dismiss. Persist
findings to `findings.jsonl`.

- [ ] **Step 5: Update lifecycle transitions**

Renumber lifecycle transition milestones to match new step numbers.

- [ ] **Step 6: Update Skill Chaining section**

- Remove `design-review --mode final-review` reference
- Remove `verification-before-completion` reference
- Add `branch-audit` — Step 2.2, mandatory gate
- Update `code-review` — now Step 2.1 (was Step 3.1)

- [ ] **Step 7: Commit**

```bash
git add work-end/SKILL.md
git commit -m "feat(#N): restructure work-end — review before sweep, forcing function at Step 2.4"
```

---

## Batch 5: Skill Cleanup

After this batch: deprecated skills removed, VBC preserved as protocol,
all cross-references updated.

### Task 8: Remove requesting-code-review and retire VBC

**Files:**
- Delete: `requesting-code-review/` (entire directory)
- Delete: `verification-before-completion/` (entire directory)
- Create: `docs/protocols/evidence-before-claims.md`
- Modify: 18 SKILL.md files (cross-reference updates)
- Modify: `CLAUDE.md`, `README.md`
- Modify: `design-review/review-tiers.md` (activate post-implementation)
- Modify: `code-review/SKILL.md` (remove final-review, requesting-code-review refs)
- Modify: `design-review/SKILL.md` (remove final-review refs)
- Modify: `receiving-code-review/SKILL.md` (update invocation chain)

- [ ] **Step 1: Create evidence-before-claims protocol**

```markdown
# Evidence Before Claims

**Core principle:** Run the command. Read the output. THEN claim the result.

No completion claim without fresh verification evidence. If you haven't
run the verification command in this message, you cannot claim it passes.

## The Gate

1. IDENTIFY: What command proves this claim?
2. RUN: Execute the FULL command (fresh, complete)
3. READ: Full output, check exit code, count failures
4. VERIFY: Does output confirm the claim?
5. ONLY THEN: Make the claim

## Applies At

Every completion boundary: commits, PRs, task completion, agent
delegation. This is not optional and not situational.
```

- [ ] **Step 2: Delete deprecated skill directories**

Remove `requesting-code-review/` and `verification-before-completion/`
directories.

- [ ] **Step 3: Update cross-references in all 18 SKILL.md files**

For each file referencing `verification-before-completion`:
- Replace with: "Follow the evidence-before-claims protocol
  (`docs/protocols/evidence-before-claims.md`)"

For each file referencing `requesting-code-review`:
- Remove the reference or replace with `branch-audit` where appropriate

- [ ] **Step 4: Update design-review/review-tiers.md**

Change post-implementation row from `*(future — conformance, robustness,
structure, coherence)*` to `branch-audit (conformance, robustness,
structure, coherence)`.

- [ ] **Step 5: Update code-review/SKILL.md Skill Chaining**

Remove `requesting-code-review` and `design-review --mode final-review`
references. Add `branch-audit` as complement.

- [ ] **Step 6: Update receiving-code-review/SKILL.md**

Remove `requesting-code-review` from "Invoked by". Add `design-review`
as the dispatching counterpart.

- [ ] **Step 7: Update design-review/SKILL.md**

Remove `--mode final-review` references. Remove VBC reference.

- [ ] **Step 8: Update CLAUDE.md and README.md**

Remove `requesting-code-review` and `verification-before-completion` from
Key Skills. Add `branch-audit` to Key Skills. Add `evidence-before-claims`
to Protocols table. Update Skill Chaining Reference table if present.

- [ ] **Step 9: Update protocols INDEX.md**

Add `evidence-before-claims.md` to `docs/protocols/INDEX.md`.

- [ ] **Step 10: Run validation**

```bash
python3 scripts/validate_all.py --tier commit
```

Expected: 0 CRITICAL findings

- [ ] **Step 11: Sync skills**

```bash
python3 scripts/claude-skill sync-local --all -y
```

- [ ] **Step 12: Commit**

```bash
git add -A
git commit -m "feat(#N): remove deprecated skills, preserve VBC as protocol, activate post-implementation in review-tiers"
```

---

## References

- [2026-08-19-unified-review-and-findings-persistence-design.md](/Users/mdproctor/claude/public/cc-praxis/specs/branch-audit-design/2026-08-19-unified-review-and-findings-persistence-design.md) — design spec
- [decisions.md](/Users/mdproctor/claude/public/cc-praxis/specs/branch-audit-design/decisions.md) — 10 design decisions
- work-end/hygiene_scan.py:246 — existing persist_findings (JSON, to be migrated)
- project/work_health.py:427 — existing check_prior_findings (JSON, to be migrated)
- design-review/review-tiers.md:14 — post-implementation lifecycle point
- Blog: 2026-08-17-mdp01 "The Feedback Loop That Wasn't"
- Blog: 2026-08-17-mdp03 "Four Fixes, One Already Done"
