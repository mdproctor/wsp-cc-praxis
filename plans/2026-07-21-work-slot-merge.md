# work-slot merge Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #85 — feat: work-end Phase B from main repo — scan worktree slots for pending merges
**Issue group:** #85

**Goal:** Add `work-slot merge` command for full slot lifecycle management from the main repo — scan, merge, and archive worktree slots.

**Architecture:** Three new `slot_manager.py` subcommands (`scan-ready`, `merge-slot`, `archive-slot`) handle mechanical git/filesystem operations. The `work-slot` SKILL.md orchestrates user interaction, GitHub API calls, and cross-skill invocations. Four lifecycle states tracked by filesystem markers.

**Tech Stack:** Python 3.12+, git CLI, existing `artifact_promote.py` / `blog_dest.py` / `branch_cleanup.py` scripts

## Global Constraints

- All script output uses `KEY=VALUE` format on stdout, matching existing conventions
- `slot_manager.py` functions follow existing patterns: `run_cmd()` for git operations, `parse_args()` for CLI parsing
- Workspace worktree names are dynamic (`work`, `work-iot`, etc.) — never hardcode `work`
- Project repos are filtered as `not name.startswith("work") and name != ".m2"`, matching existing `list_slots()` filter
- `allocate_slot_number()` must scan both `worktrees/*/` and `worktrees/attic/*/`

---

### Task 1: Fix `allocate_slot_number()` to scan attic

**Files:**
- Modify: `work-slot/slot_manager.py` — `allocate_slot_number()` function
- Test: `tests/test_slot_manager.py` (create)

**Interfaces:**
- Consumes: existing `allocate_slot_number(worktrees_dir: Path) -> int`
- Produces: updated `allocate_slot_number(worktrees_dir: Path) -> int` that also scans `attic/`

- [ ] **Step 1: Write the failing test**

```python
# tests/test_slot_manager.py
import tempfile
from pathlib import Path

from work_slot.slot_manager import allocate_slot_number


def test_allocate_slot_number_considers_attic():
    with tempfile.TemporaryDirectory() as tmp:
        worktrees = Path(tmp)
        (worktrees / "1").mkdir()
        (worktrees / "2").mkdir()
        attic = worktrees / "attic"
        attic.mkdir()
        (attic / "3").mkdir()
        (attic / "5").mkdir()
        assert allocate_slot_number(worktrees) == 6


def test_allocate_slot_number_empty():
    with tempfile.TemporaryDirectory() as tmp:
        worktrees = Path(tmp)
        assert allocate_slot_number(worktrees) == 1


def test_allocate_slot_number_only_attic():
    with tempfile.TemporaryDirectory() as tmp:
        worktrees = Path(tmp)
        attic = worktrees / "attic"
        attic.mkdir()
        (attic / "4").mkdir()
        assert allocate_slot_number(worktrees) == 5
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_slot_manager.py::test_allocate_slot_number_considers_attic -v`
Expected: FAIL — returns 3 instead of 6 (attic not scanned)

- [ ] **Step 3: Write minimal implementation**

Replace `allocate_slot_number` in `work-slot/slot_manager.py`:

```python
def allocate_slot_number(worktrees_dir: Path) -> int:
    if not worktrees_dir.exists():
        return 1
    existing = [
        int(d.name) for d in worktrees_dir.iterdir()
        if d.is_dir() and d.name.isdigit()
    ]
    attic_dir = worktrees_dir / "attic"
    if attic_dir.exists():
        existing.extend(
            int(d.name) for d in attic_dir.iterdir()
            if d.is_dir() and d.name.isdigit()
        )
    return max(existing, default=0) + 1
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_slot_manager.py -v`
Expected: 3 PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-slot/slot_manager.py tests/test_slot_manager.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "fix(#85): allocate_slot_number scans attic to prevent collisions"
```

---

### Task 2: Add `scan-ready` subcommand

**Files:**
- Modify: `work-slot/slot_manager.py` — add `scan_ready()` function and CLI routing
- Test: `tests/test_slot_manager.py` — add scan-ready tests

**Interfaces:**
- Consumes: `SLOT.md` format (written by `write_slot_md()`), `.phase-a-complete` marker, `.landed` marker
- Produces: `scan_ready(family_root: Path) -> list[dict]` returning JSON-serialisable slot data with keys: `number`, `branch`, `repos` (list of `{"name", "commits", "diff"}`), `issue`, `issue_repo`, `covers`, `context`, `phase_a_timestamp`

- [ ] **Step 1: Write the failing test**

```python
# tests/test_slot_manager.py (append)
import json
from work_slot.slot_manager import scan_ready


def test_scan_ready_finds_phase_a_complete_slots(tmp_path):
    worktrees = tmp_path / "worktrees"
    worktrees.mkdir()

    # Slot 1: phase-a-complete (should appear)
    slot1 = worktrees / "1"
    slot1.mkdir()
    (slot1 / ".phase-a-complete").write_text(
        "branch=issue-42-spi\nrepos=engine\ntimestamp=2026-07-18T14:32:00\n"
    )
    (slot1 / "SLOT.md").write_text(
        "# Slot 1 — issue-42-spi\n\n## Issue\ncasehubio/engine#42\n"
        "Covers: 42\n\n## What to do\nImplement SPI\n\n## Repos\n- engine (primary)\n"
    )
    # Fake repo worktree (no real git, scan_ready will skip git stats)
    engine = slot1 / "engine"
    engine.mkdir()

    # Slot 2: active (no marker — should NOT appear)
    slot2 = worktrees / "2"
    slot2.mkdir()

    # Slot 3: landed (should NOT appear)
    slot3 = worktrees / "3"
    slot3.mkdir()
    (slot3 / ".phase-a-complete").write_text("branch=issue-99\n")
    (slot3 / ".landed").write_text("landed\n")

    result = scan_ready(tmp_path)
    assert len(result) == 1
    assert result[0]["number"] == 1
    assert result[0]["branch"] == "issue-42-spi"
    assert result[0]["context"] == "Implement SPI"


def test_scan_ready_empty_when_no_ready_slots(tmp_path):
    worktrees = tmp_path / "worktrees"
    worktrees.mkdir()
    slot1 = worktrees / "1"
    slot1.mkdir()  # active, no marker
    result = scan_ready(tmp_path)
    assert result == []


def test_scan_ready_no_worktrees_dir(tmp_path):
    result = scan_ready(tmp_path)
    assert result == []
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_slot_manager.py::test_scan_ready_finds_phase_a_complete_slots -v`
Expected: FAIL — `scan_ready` not defined

- [ ] **Step 3: Write minimal implementation**

Add to `work-slot/slot_manager.py`:

```python
def parse_slot_md(slot_dir: Path) -> dict:
    """Extract branch, issue, covers, repos, context from SLOT.md."""
    slot_md = slot_dir / "SLOT.md"
    if not slot_md.exists():
        return {}
    content = slot_md.read_text()
    result: dict = {"repos": [], "context": "", "issue": "", "issue_repo": "", "covers": ""}

    for line in content.splitlines():
        if line.startswith("# Slot") and "—" in line:
            result["branch"] = line.split("—", 1)[1].strip()
        if line.startswith("Covers:"):
            result["covers"] = line.split(":", 1)[1].strip()

    # Parse issue line (format: owner/repo#N)
    in_issue = False
    in_what = False
    in_repos = False
    context_lines: list[str] = []
    for line in content.splitlines():
        if line.startswith("## Issue"):
            in_issue = True
            in_what = False
            in_repos = False
            continue
        if line.startswith("## What to do"):
            in_issue = False
            in_what = True
            in_repos = False
            continue
        if line.startswith("## Repos"):
            in_issue = False
            in_what = False
            in_repos = True
            continue
        if line.startswith("## "):
            in_issue = False
            in_what = False
            in_repos = False
            continue
        if in_issue and "#" in line and not line.startswith("Covers:"):
            parts = line.strip().split("#")
            if len(parts) == 2:
                result["issue_repo"] = parts[0]
                result["issue"] = parts[1]
        if in_what:
            context_lines.append(line.strip())
        if in_repos and line.strip().startswith("- "):
            repo_name = line.strip().lstrip("- ").split(" ")[0].strip()
            if repo_name:
                result["repos"].append(repo_name)

    result["context"] = " ".join(l for l in context_lines if l).strip()
    return result


def get_repo_stats(repo_path: Path) -> dict:
    """Get commit count and diff stats for a repo worktree vs origin/main."""
    rc, log_out, _ = run_cmd(
        ["git", "-C", str(repo_path), "log", "--oneline", "origin/main..HEAD"]
    )
    commits = len(log_out.strip().splitlines()) if rc == 0 and log_out.strip() else 0

    rc, stat_out, _ = run_cmd(
        ["git", "-C", str(repo_path), "diff", "--shortstat", "origin/main..HEAD"]
    )
    diff = stat_out.strip() if rc == 0 else ""

    return {"commits": commits, "diff": diff}


def scan_ready(family_root: Path) -> list[dict]:
    """Scan for slots with .phase-a-complete but no .landed."""
    worktrees_dir = family_root / "worktrees"
    if not worktrees_dir.exists():
        return []

    slots = []
    for d in sorted(worktrees_dir.iterdir()):
        if not d.is_dir() or not d.name.isdigit():
            continue
        if not (d / ".phase-a-complete").exists():
            continue
        if (d / ".landed").exists():
            continue

        phase_a = d / ".phase-a-complete"
        timestamp = ""
        for line in phase_a.read_text().splitlines():
            if line.startswith("timestamp="):
                timestamp = line.split("=", 1)[1]

        md = parse_slot_md(d)
        repo_names = md.get("repos", [])
        repo_data = []
        for repo_name in repo_names:
            repo_path = d / repo_name
            if repo_path.is_dir() and (repo_path / ".git").exists():
                stats = get_repo_stats(repo_path)
                repo_data.append({"name": repo_name, **stats})
            else:
                repo_data.append({"name": repo_name, "commits": 0, "diff": ""})

        slots.append({
            "number": int(d.name),
            "branch": md.get("branch", ""),
            "repos": repo_data,
            "issue": md.get("issue", ""),
            "issue_repo": md.get("issue_repo", ""),
            "covers": md.get("covers", ""),
            "context": md.get("context", ""),
            "phase_a_timestamp": timestamp,
        })

    return slots
```

Add CLI routing in `main()`:

```python
    elif subcommand == "scan-ready":
        family_root = Path(args.get("target", "."))
        slots = scan_ready(family_root)
        print(json.dumps({"slots": slots}, indent=2))
```

Add `import json` at the top of the file.

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_slot_manager.py -v -k scan_ready`
Expected: 3 PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-slot/slot_manager.py tests/test_slot_manager.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat(#85): add scan-ready subcommand to slot_manager.py"
```

---

### Task 3: Add `merge-slot` subcommand

**Files:**
- Modify: `work-slot/slot_manager.py` — add `merge_slot()` function and CLI routing
- Test: `tests/test_slot_manager.py` — add merge-slot tests

**Interfaces:**
- Consumes: slot directory structure, `.phase-a-complete`, `.merge-progress` (for partial recovery)
- Produces: `merge_slot(family_root: Path, slot_num: int) -> int` (exit code). Outputs `STAGE=rebase STATUS=pass|fail`, `STAGE=push STATUS=pass LANDED_SHAS=<repo:sha,...>`, or `ERROR=conflict|retry_exhausted|...`. Writes `.landed` marker on success.

- [ ] **Step 1: Write the failing tests**

```python
# tests/test_slot_manager.py (append)
import os
import subprocess
from work_slot.slot_manager import merge_slot, resolve_original_repo


def _create_test_repos(tmp_path, repo_names):
    """Create a family root with original repos and a slot with worktrees."""
    family = tmp_path / "family"
    family.mkdir()
    worktrees = family / "worktrees"
    worktrees.mkdir()

    originals = {}
    for name in repo_names:
        repo = family / name
        repo.mkdir()
        subprocess.run(["git", "init", str(repo)], capture_output=True)
        subprocess.run(["git", "-C", str(repo), "checkout", "-b", "main"], capture_output=True)
        (repo / "README.md").write_text(f"# {name}\n")
        subprocess.run(["git", "-C", str(repo), "add", "."], capture_output=True)
        subprocess.run(["git", "-C", str(repo), "commit", "-m", "initial"], capture_output=True)
        originals[name] = repo

    slot = worktrees / "1"
    slot.mkdir()

    branch = "issue-42-test"
    for name in repo_names:
        subprocess.run([
            "git", "-C", str(originals[name]),
            "worktree", "add", str(slot / name), "-b", branch,
        ], capture_output=True)
        # Make a commit on the branch
        (slot / name / "feature.py").write_text(f"# {name} feature\n")
        subprocess.run(["git", "-C", str(slot / name), "add", "."], capture_output=True)
        subprocess.run(["git", "-C", str(slot / name), "commit", "-m", f"feat: {name} feature"], capture_output=True)

    (slot / ".phase-a-complete").write_text(
        f"branch={branch}\nrepos={','.join(repo_names)}\ntimestamp=2026-07-18T14:32:00\n"
    )
    (slot / "SLOT.md").write_text(
        f"# Slot 1 — {branch}\n\n## Issue\ntest/repo#42\nCovers: 42\n\n"
        f"## What to do\nTest\n\n## Repos\n" +
        "\n".join(f"- {n}" for n in repo_names) + "\n"
    )

    return family, originals, slot, branch


def test_resolve_original_repo(tmp_path):
    family, originals, slot, _ = _create_test_repos(tmp_path, ["engine"])
    resolved = resolve_original_repo(slot / "engine")
    assert resolved == originals["engine"]


def test_merge_slot_clean_rebase_and_push(tmp_path):
    family, originals, slot, branch = _create_test_repos(tmp_path, ["engine"])
    exit_code = merge_slot(family, 1)
    assert exit_code == 0
    # Verify feature.py is on main
    assert (originals["engine"] / "feature.py").exists()
    # Verify .landed marker exists
    assert (slot / ".landed").exists()
    landed = (slot / ".landed").read_text()
    assert "branch=issue-42-test" in landed
    assert "engine:" in landed


def test_merge_slot_conflict(tmp_path):
    family, originals, slot, branch = _create_test_repos(tmp_path, ["engine"])
    # Create a conflicting commit on main
    (originals["engine"] / "feature.py").write_text("# conflict\n")
    subprocess.run(["git", "-C", str(originals["engine"]), "add", "."], capture_output=True)
    subprocess.run(["git", "-C", str(originals["engine"]), "commit", "-m", "conflict"], capture_output=True)
    exit_code = merge_slot(family, 1)
    assert exit_code != 0
    assert not (slot / ".landed").exists()
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_slot_manager.py::test_merge_slot_clean_rebase_and_push -v`
Expected: FAIL — `merge_slot` not defined

- [ ] **Step 3: Write minimal implementation**

Add to `work-slot/slot_manager.py`:

```python
def resolve_original_repo(worktree_path: Path) -> Path:
    """Resolve a worktree path back to its original repo."""
    rc, common_dir, _ = run_cmd(
        ["git", "-C", str(worktree_path), "rev-parse", "--git-common-dir"]
    )
    if rc != 0:
        return worktree_path
    common = Path(common_dir.strip())
    if not common.is_absolute():
        common = (worktree_path / common).resolve()
    # common_dir is the .git directory of the original repo
    return common.parent


def is_project_repo(name: str) -> bool:
    """Filter: project repos only (exclude workspace worktrees and .m2)."""
    return not name.startswith("work") and name != ".m2" and name != "attic"


def get_slot_repos(slot_dir: Path) -> list[str]:
    """List project repo names in a slot directory."""
    return [
        d.name for d in sorted(slot_dir.iterdir())
        if d.is_dir() and (d / ".git").exists() and is_project_repo(d.name)
    ]


def merge_slot(family_root: Path, slot_num: int) -> int:
    """Rebase, fast-forward merge, and push all repos in a slot."""
    slot_dir = family_root / "worktrees" / str(slot_num)
    if not slot_dir.exists():
        print(f"ERROR=slot_not_found slot={slot_num}")
        return 1

    if not (slot_dir / ".phase-a-complete").exists():
        print(f"ERROR=not_ready slot={slot_num}")
        return 1

    if (slot_dir / ".landed").exists():
        print(f"ERROR=already_landed slot={slot_num}")
        return 1

    # Read branch name from .phase-a-complete
    branch = ""
    for line in (slot_dir / ".phase-a-complete").read_text().splitlines():
        if line.startswith("branch="):
            branch = line.split("=", 1)[1]
    if not branch:
        print("ERROR=no_branch_in_marker")
        return 1

    repos = get_slot_repos(slot_dir)
    if not repos:
        print("ERROR=no_repos_in_slot")
        return 1

    # Read partial push progress if exists
    progress_file = slot_dir / ".merge-progress"
    pushed_repos: set[str] = set()
    if progress_file.exists():
        for line in progress_file.read_text().splitlines():
            if "=pushed:" in line:
                repo_name = line.split("=")[0]
                pushed_repos.add(repo_name)

    max_attempts = 3
    for attempt in range(1, max_attempts + 1):
        # Stage 1: Rebase all repo branches onto origin/main in slot worktrees
        print(f"STAGE=rebase ATTEMPT={attempt}")
        rebase_failed = False
        for repo_name in repos:
            slot_repo = slot_dir / repo_name
            run_cmd(["git", "-C", str(slot_repo), "fetch", "origin", "main"])
            rc, _, stderr = run_cmd(
                ["git", "-C", str(slot_repo), "rebase", "origin/main"]
            )
            if rc != 0:
                run_cmd(["git", "-C", str(slot_repo), "rebase", "--abort"])
                print(f"STAGE=rebase STATUS=fail")
                print(f"ERROR=conflict repo={repo_name}")
                return 1

        print("STAGE=rebase STATUS=pass")

        # Stage 2: Fast-forward and push in original repos
        print(f"STAGE=push ATTEMPT={attempt}")
        push_failed = False
        landed_shas: dict[str, str] = {}

        for repo_name in repos:
            if repo_name in pushed_repos:
                # Already pushed in a prior attempt — read SHA from progress
                for line in progress_file.read_text().splitlines():
                    if line.startswith(f"{repo_name}=pushed:"):
                        landed_shas[repo_name] = line.split(":")[1]
                continue

            slot_repo = slot_dir / repo_name
            original = resolve_original_repo(slot_repo)

            run_cmd(["git", "-C", str(original), "fetch", "origin", "main"])
            run_cmd(["git", "-C", str(original), "rebase", "origin/main"])

            rc, _, stderr = run_cmd(
                ["git", "-C", str(original), "merge", "--ff-only", branch]
            )
            if rc != 0:
                push_failed = True
                print(f"WARN=ff_only_failed repo={repo_name} attempt={attempt}")
                break

            rc, _, stderr = run_cmd(
                ["git", "-C", str(original), "push", "origin", "main"]
            )
            if rc != 0:
                push_failed = True
                print(f"WARN=push_failed repo={repo_name} attempt={attempt}")
                break

            rc, sha, _ = run_cmd(["git", "-C", str(original), "rev-parse", "HEAD"])
            sha = sha.strip() if rc == 0 else "unknown"
            landed_shas[repo_name] = sha

            # Record progress
            with open(progress_file, "a") as f:
                f.write(f"{repo_name}=pushed:{sha}\n")
            pushed_repos.add(repo_name)

        if push_failed:
            if attempt < max_attempts:
                continue
            print("STAGE=push STATUS=fail")
            print("ERROR=retry_exhausted")
            return 1

        # All repos pushed — write .landed marker
        if progress_file.exists():
            progress_file.unlink()

        shas_str = ",".join(f"{r}:{s}" for r, s in landed_shas.items())
        (slot_dir / ".landed").write_text(
            f"branch={branch}\n"
            f"repos={','.join(repos)}\n"
            f"landed_shas={shas_str}\n"
            f"timestamp={datetime.datetime.now(datetime.timezone.utc).isoformat()}\n"
        )

        print("STAGE=push STATUS=pass")
        print(f"LANDED_SHAS={shas_str}")
        return 0

    return 1
```

Add CLI routing in `main()`:

```python
    elif subcommand == "merge-slot":
        family_root = Path(args.get("target", "."))
        slot_num = int(args.get("slot", "0"))
        if slot_num == 0:
            print("ERROR=missing_slot_number")
            sys.exit(1)
        sys.exit(merge_slot(family_root, slot_num))
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_slot_manager.py -v -k merge_slot`
Expected: 3 PASS (resolve_original_repo, clean merge, conflict)

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-slot/slot_manager.py tests/test_slot_manager.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat(#85): add merge-slot subcommand with retry and partial recovery"
```

---

### Task 4: Add `archive-slot` subcommand

**Files:**
- Modify: `work-slot/slot_manager.py` — add `archive_slot()` function and CLI routing
- Test: `tests/test_slot_manager.py` — add archive-slot tests

**Interfaces:**
- Consumes: slot directory with worktrees
- Produces: `archive_slot(family_root: Path, slot_num: int) -> None`. Outputs `ARCHIVED=<N>`. Moves slot to `worktrees/attic/<N>/`.

- [ ] **Step 1: Write the failing test**

```python
# tests/test_slot_manager.py (append)
from work_slot.slot_manager import archive_slot


def test_archive_slot_moves_to_attic(tmp_path):
    family, originals, slot, branch = _create_test_repos(tmp_path, ["engine"])
    # Write markers so archive has metadata to preserve
    (slot / ".phase-a-complete").write_text("branch=issue-42-test\n")
    (slot / ".landed").write_text("landed\n")

    archive_slot(family, 1)

    # Slot dir should be gone from worktrees/
    assert not (family / "worktrees" / "1").exists()
    # Should be in attic
    attic_slot = family / "worktrees" / "attic" / "1"
    assert attic_slot.exists()
    assert (attic_slot / "SLOT.md").exists()
    assert (attic_slot / ".phase-a-complete").exists()
    assert (attic_slot / ".landed").exists()
    # Repo dirs should still exist but git worktrees removed
    # (worktree removal leaves the dir with only metadata)


def test_archive_slot_not_found(tmp_path):
    family = tmp_path / "family"
    family.mkdir()
    (family / "worktrees").mkdir()
    # Should exit non-zero
    import io
    import contextlib
    f = io.StringIO()
    with contextlib.redirect_stdout(f):
        try:
            archive_slot(family, 99)
        except SystemExit:
            pass
    assert "ERROR=slot_not_found" in f.getvalue()
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_slot_manager.py::test_archive_slot_moves_to_attic -v`
Expected: FAIL — `archive_slot` not defined (existing `remove_slot` deletes, doesn't archive)

- [ ] **Step 3: Write minimal implementation**

Add to `work-slot/slot_manager.py`:

```python
def archive_slot(family_root: Path, slot_num: int) -> None:
    """Remove worktrees and move slot to attic for auditing."""
    slot_dir = family_root / "worktrees" / str(slot_num)
    if not slot_dir.exists():
        print(f"ERROR=slot_not_found slot={slot_num}")
        sys.exit(1)

    # Remove git worktrees (repos and workspace worktrees)
    for sub in slot_dir.iterdir():
        if sub.is_dir() and (sub / ".git").exists():
            rc, _, stderr = run_cmd(["git", "worktree", "remove", "--force", str(sub)])
            if rc != 0:
                print(f"WARN=worktree_remove_failed dir={sub.name} stderr={stderr.strip()}")

    # Move to attic
    attic_dir = family_root / "worktrees" / "attic"
    attic_dir.mkdir(exist_ok=True)
    dest = attic_dir / str(slot_num)
    shutil.move(str(slot_dir), str(dest))
    print(f"ARCHIVED={slot_num}")
```

Add CLI routing in `main()`:

```python
    elif subcommand == "archive-slot":
        family_root = Path(args.get("target", "."))
        slot_num = int(args.get("slot", "0"))
        if slot_num == 0:
            print("ERROR=missing_slot_number")
            sys.exit(1)
        archive_slot(family_root, slot_num)
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_slot_manager.py -v -k archive`
Expected: 2 PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-slot/slot_manager.py tests/test_slot_manager.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat(#85): add archive-slot subcommand — worktree removal + attic preservation"
```

---

### Task 5: Update `list_slots()` for new states and `--all`

**Files:**
- Modify: `work-slot/slot_manager.py` — update `list_slots()` function
- Test: `tests/test_slot_manager.py` — add list-slots tests

**Interfaces:**
- Consumes: existing `list_slots(family_root: Path) -> list[dict]`
- Produces: updated `list_slots(family_root: Path, include_archived: bool = False) -> list[dict]` with state field including `landed`, and archived slots reading from SLOT.md

- [ ] **Step 1: Write the failing test**

```python
# tests/test_slot_manager.py (append)
from work_slot.slot_manager import list_slots


def test_list_slots_shows_landed_state(tmp_path):
    worktrees = tmp_path / "worktrees"
    worktrees.mkdir()
    slot = worktrees / "1"
    slot.mkdir()
    (slot / ".phase-a-complete").write_text("branch=issue-42\n")
    (slot / ".landed").write_text("landed\n")
    (slot / "SLOT.md").write_text("# Slot 1 — issue-42\n")

    result = list_slots(tmp_path, include_archived=False)
    assert len(result) == 1
    assert result[0]["state"] == "landed"


def test_list_slots_includes_archived(tmp_path):
    worktrees = tmp_path / "worktrees"
    worktrees.mkdir()
    attic = worktrees / "attic"
    attic.mkdir()
    archived = attic / "3"
    archived.mkdir()
    (archived / "SLOT.md").write_text(
        "# Slot 3 — issue-99-old\n\n## Repos\n- engine\n- iot\n"
    )
    (archived / ".phase-a-complete").write_text("branch=issue-99-old\n")
    (archived / ".landed").write_text("landed\n")

    # Without --all: no archived
    result = list_slots(tmp_path, include_archived=False)
    assert len(result) == 0

    # With --all: includes archived
    result = list_slots(tmp_path, include_archived=True)
    assert len(result) == 1
    assert result[0]["number"] == 3
    assert result[0]["state"] == "archived"
    assert result[0]["branch"] == "issue-99-old"
    assert "engine" in result[0]["repos"]
    assert "iot" in result[0]["repos"]
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_slot_manager.py::test_list_slots_shows_landed_state -v`
Expected: FAIL — `list_slots()` doesn't accept `include_archived` parameter

- [ ] **Step 3: Write minimal implementation**

Replace `list_slots` in `work-slot/slot_manager.py`:

```python
def list_slots(family_root: Path, include_archived: bool = False) -> list[dict]:
    worktrees_dir = family_root / "worktrees"
    if not worktrees_dir.exists():
        return []
    slots = []

    # Active/ready/landed slots
    for d in sorted(worktrees_dir.iterdir()):
        if not d.is_dir() or not d.name.isdigit():
            continue
        repos = [
            sub.name for sub in sorted(d.iterdir())
            if sub.is_dir() and (sub / ".git").exists() and is_project_repo(sub.name)
        ]
        branch = ""
        slot_md = d / "SLOT.md"
        if slot_md.exists():
            for line in slot_md.read_text().splitlines():
                if line.startswith("# Slot") and "—" in line:
                    branch = line.split("—", 1)[1].strip()
                    break

        if (d / ".landed").exists():
            state = "landed"
        elif (d / ".phase-a-complete").exists():
            state = "ready to land"
        else:
            state = "active"

        slots.append({
            "number": int(d.name),
            "branch": branch,
            "repos": repos,
            "state": state,
        })

    # Archived slots (from attic)
    if include_archived:
        attic_dir = worktrees_dir / "attic"
        if attic_dir.exists():
            for d in sorted(attic_dir.iterdir()):
                if not d.is_dir() or not d.name.isdigit():
                    continue
                # Read repos from SLOT.md (worktrees removed, .git files dead)
                md = parse_slot_md(d)
                branch = md.get("branch", "")
                repos = md.get("repos", [])

                slots.append({
                    "number": int(d.name),
                    "branch": branch,
                    "repos": repos,
                    "state": "archived",
                })

    return slots
```

Update CLI routing for `list-slots` to handle `--all`:

```python
    elif subcommand == "list-slots":
        family_root = Path(args.get("target", "."))
        include_archived = "--all" in sys.argv
        slots = list_slots(family_root, include_archived=include_archived)
        for s in slots:
            repos_str = ",".join(s["repos"]) if isinstance(s["repos"], list) else s["repos"]
            print(f"SLOT={s['number']} BRANCH={s['branch']} REPOS={repos_str} STATE={s['state']}")
        print(f"COUNT={len(slots)}")
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_slot_manager.py -v -k list_slots`
Expected: 2 PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-slot/slot_manager.py tests/test_slot_manager.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat(#85): update list_slots with landed state and --all for archived"
```

---

### Task 6: Add `work-slot merge` section to SKILL.md

**Files:**
- Modify: `work-slot/SKILL.md` — add merge section, update lifecycle docs

**Interfaces:**
- Consumes: `slot_manager.py` subcommands (`scan-ready`, `merge-slot`, `archive-slot`), `artifact_promote.py`, `blog_dest.py`, `branch_cleanup.py`
- Produces: user-facing `work-slot merge` workflow

- [ ] **Step 1: Read the current SKILL.md end section to find insertion point**

The new section goes after `## work-slot remove` and before `## How slots work`.

- [ ] **Step 2: Add the merge section**

Insert after the `work-slot remove` section in `work-slot/SKILL.md`:

```markdown
## `work-slot merge`

Merge ready-to-land slots from the main repo. Runs the full Phase B
sequence: rebase, push, close issues, promote artifacts, stamp, archive.

### Step 1 — Find family root

Walk up from CWD looking for a directory that is not itself a git repo
and contains child directories with `wksp` symlinks. For each candidate,
verify its child repos have `.git` directories (not files) — worktree
checkouts have `.git` files and must be skipped.

If the walk-up fails, ask:
> "Which directory is the family root? (e.g., ~/claude/casehub)"

### Step 2 — Scan and present

```bash
python3 ~/.claude/skills/work-slot/slot_manager.py scan-ready <family-root>
```

Parse the JSON output. For each slot, fetch the issue title:
```bash
gh issue view <issue-number> --repo <issue-repo> --json title --jq '.title'
```

Present the rich listing:
```
Slots ready to merge:

  [1] issue-42-spi
      Repos: engine (3 commits, +142/-38)
      Issue: casehubio/engine#42 — "Add expression SPI"
      Context: Implement SPI for pluggable expression evaluation
      Phase A completed: 2026-07-18 14:32

Merge which slot? (number, or "all")
```

If no slots are ready: "No slots ready to merge." Stop.

### Step 3 — Pre-check

For every original repo across all selected slots, verify:
1. Main checked out
2. Clean working tree (`git -C <repo> status --short` is empty)
3. No unpushed commits (`git -C <repo> log origin/main..main --oneline` is empty)
4. Fetch origin — warn if remote is ahead (non-blocking)

If any check fails, stop and report which repo failed and why.

### Step 4 — Merge each slot

For each selected slot, in order:

**4a. Rebase and push:**
```bash
python3 ~/.claude/skills/work-slot/slot_manager.py merge-slot <family-root> slot=<N>
```

Read output. If `ERROR=conflict`: stop, report which repo conflicted.
If `ERROR=retry_exhausted`: stop, provide manual instructions.
If `STAGE=push STATUS=pass`: continue to 4b.

**4b. Post-merge actions** (skill handles these):
- Close issues:
  ```bash
  python3 ~/.claude/skills/work-end/artifact_promote.py close-issues <issue-repo> covers=<covers>
  ```
- Promote artifacts from slot workspace to original workspace:
  ```bash
  python3 ~/.claude/skills/work-end/artifact_promote.py to-workspace-main <original-workspace> branch=<branch> artifacts=<paths>
  ```
- Clean up specs in slot workspace:
  ```bash
  python3 ~/.claude/skills/work-end/artifact_promote.py cleanup-specs <slot-workspace> branch=<branch>
  ```
- Publish blog:
  ```bash
  python3 ~/.claude/skills/work-end/blog_dest.py <original-workspace>/blog <branch>
  ```

**4c. Stamp branches** — empty commits in slot worktrees:
```bash
git -C <slot>/<repo> commit --allow-empty -m "chore: branch closed — landed as <SHA> on main"
```
Stamp ALL workspace worktrees too (discover dynamically: scan slot dir
for directories starting with `work` that contain `.git`).

**4d. Mark closed:**
```bash
python3 ~/.claude/skills/work-end/branch_cleanup.py create-epic-closed \
  <slot>/<primary-workspace> branch=<branch> date=$(date +%Y-%m-%d) \
  issues=<covers> single-repo=no
```

**4e. Archive:**
```bash
python3 ~/.claude/skills/work-slot/slot_manager.py archive-slot <family-root> slot=<N>
```

If archive fails: report error but do NOT roll back — code is on main.
Report manual cleanup commands.

### Step 5 — Report

```
✅ Slot <N> merged and archived
   Branch: <branch>
   Repos: <list>
   Issues closed: #<covers>
   Artifacts promoted: <count>
```

If "all" was selected, repeat Step 4 for next slot. If any slot fails at
4a, stop — report which slot failed and that prior slots landed.
```

- [ ] **Step 3: Update lifecycle documentation**

Add to the top of SKILL.md, after the current description:

```markdown
## Slot Lifecycle

| State | Marker | Meaning |
|-------|--------|---------|
| `active` | slot dir exists, no markers | Work in progress |
| `ready to land` | `.phase-a-complete` | Phase A done, awaiting merge |
| `landed` | `.landed` | Merged to main, awaiting archive |
| `archived` | in `worktrees/attic/<N>/` | Worktrees removed, metadata kept |
```

- [ ] **Step 4: Update Skill Chaining section**

Add to the existing Skill Chaining:
```markdown
- `work-end` — Phase A writes `.phase-a-complete`; work-slot merge reads it
  and runs Phase B externally
- `artifact_promote.py` / `blog_dest.py` / `branch_cleanup.py` — shared
  scripts used by both work-end Phase B and work-slot merge
```

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-slot/SKILL.md
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat(#85): add work-slot merge section to SKILL.md"
```

---

### Task 7: Sync and validate

**Files:**
- No new files — validation and sync only

- [ ] **Step 1: Run sync-local**

```bash
python3 /Users/mdproctor/claude/hortora/soredium/scripts/claude-skill sync-local --all -y
```

- [ ] **Step 2: Run commit-tier validation**

```bash
python3 /Users/mdproctor/claude/hortora/soredium/scripts/validate_all.py --tier commit
```

Fix any CRITICAL findings.

- [ ] **Step 3: Run all tests**

```bash
python3 -m pytest tests/test_slot_manager.py -v
```

Verify all tests pass.

- [ ] **Step 4: Generate slash command**

```bash
python3 /Users/mdproctor/claude/hortora/soredium/scripts/generate_commands.py
```

- [ ] **Step 5: Commit any fixes**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add -A
git -C /Users/mdproctor/claude/hortora/soredium commit -m "chore(#85): sync, validate, and generate commands"
```
