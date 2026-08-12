# ISX Isolation for Slots — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #223 — ISX isolation for slots — run work inside incus-spawn containers
**Issue group:** #223

**Goal:** Add optional ISX container isolation to the slot workflow — `--isx` on create, sync subcommand, destroy on remove, staleness pre-flight.

**Architecture:** Four new code paths in `slot_manager.py` (create ISX, sync, add-repo wiring, teardown), parser/writer extensions for `.slot` metadata, staleness check in `work_end_context.py`, and SKILL.md updates. All ISX operations shell out to the blessed `isx` CLI and use `isx://` git remotes for sync.

**Tech Stack:** Python 3, subprocess (isx CLI, git), pytest (mocked subprocess)

## Global Constraints

- Only blessed upstream ISX commands: `isx branch`, `isx destroy`, `isx shell`, `isx templates list`, `isx git-remote-helper`. NOT `isx sync`.
- Container-side repo path convention: `/home/agentuser/<repo-name>`
- Incus instance name limit: 63 characters
- All new `.py` functions ship with tests in the same commit (protocol: `externalised-scripts-require-tests`)
- Tests mock `subprocess.run` — no actual containers

---

### Task 1: Parser/Writer Extensions for `.slot` Isolation Metadata

All other tasks depend on reading/writing the `## Isolation` section.

**Files:**
- Modify: `work-slot/slot_manager.py` — `parse_slot_md()` (~line 773) and `write_slot_md()` (~line 413)
- Test: `tests/test_slot_manager.py` — new classes `TestParseSlotMdIsolation`, `TestWriteSlotMdIsolation`

**Interfaces:**
- Produces: `parse_slot_md()` returns `isolation_type: str`, `isx_instance: str`, `isx_template: str` in its dict
- Produces: `write_slot_md()` accepts optional `isolation_type: str = ""`, `isx_instance: str = ""`, `isx_template: str = ""` params

- [ ] **Step 1: Write failing tests for `parse_slot_md()` isolation parsing**

```python
class TestParseSlotMdIsolation:
    def test_parse_with_isolation_section(self, tmp_path):
        (tmp_path / ".slot").write_text(
            "# Slot 7 — issue-42-fix\n\n"
            "## Issue\nHortora/soredium#42\nCovers: 42\n\n"
            "## What to do\nFix scoring\n\n"
            "## Repos\n- soredium (primary)\n\n"
            "## Isolation\ntype: isx\ninstance: issue-42-fix\n"
            "template: tpl-java\n\n"
            "## Created\n2026-08-12, branch: issue-42-fix\n"
        )
        result = slot_manager.parse_slot_md(tmp_path)
        assert result["isolation_type"] == "isx"
        assert result["isx_instance"] == "issue-42-fix"
        assert result["isx_template"] == "tpl-java"

    def test_parse_without_isolation_section(self, tmp_path):
        (tmp_path / ".slot").write_text(
            "# Slot 7 — issue-42-fix\n\n"
            "## Issue\nHortora/soredium#42\nCovers: 42\n\n"
            "## Repos\n- soredium (primary)\n\n"
            "## Created\n2026-08-12, branch: issue-42-fix\n"
        )
        result = slot_manager.parse_slot_md(tmp_path)
        assert result["isolation_type"] == ""
        assert result["isx_instance"] == ""
        assert result["isx_template"] == ""

    def test_parse_isolation_preserves_existing_fields(self, tmp_path):
        (tmp_path / ".slot").write_text(
            "# Slot 7 — issue-42-fix\n\n"
            "## Issue\nHortora/soredium#42\nCovers: 42\n\n"
            "## What to do\nFix scoring\n\n"
            "## Repos\n- soredium (primary)\n\n"
            "## Isolation\ntype: isx\ninstance: issue-42-fix\n"
            "template: tpl-java\n\n"
            "## Created\n2026-08-12, branch: issue-42-fix\n"
        )
        result = slot_manager.parse_slot_md(tmp_path)
        assert result["repos"] == ["soredium"]
        assert result["issue"] == "42"
        assert result["covers"] == "42"
        assert result["context"] == "Fix scoring"
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_slot_manager.py::TestParseSlotMdIsolation -v`
Expected: FAIL — `isolation_type` key not in result dict

- [ ] **Step 3: Implement `parse_slot_md()` isolation parsing**

In `parse_slot_md()`, add `in_isolation = False` to the state tracking. Add to the result dict init:
```python
result: dict = {
    "repos": [], "context": "", "issue": "", "issue_repo": "",
    "covers": "", "is_epic": False,
    "isolation_type": "", "isx_instance": "", "isx_template": "",
}
```

Add section detection:
```python
if line.startswith("## Isolation"):
    in_issue, in_what, in_repos, in_isolation = False, False, False, True
    continue
```

Add field parsing inside `in_isolation`:
```python
if in_isolation:
    if line.strip().startswith("type:"):
        result["isolation_type"] = line.strip().split(":", 1)[1].strip()
    elif line.strip().startswith("instance:"):
        result["isx_instance"] = line.strip().split(":", 1)[1].strip()
    elif line.strip().startswith("template:"):
        result["isx_template"] = line.strip().split(":", 1)[1].strip()
```

Reset `in_isolation` on new section header (existing `if line.startswith("## "):` block).

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_slot_manager.py::TestParseSlotMdIsolation -v`
Expected: PASS

- [ ] **Step 5: Write failing tests for `write_slot_md()` isolation output**

```python
class TestWriteSlotMdIsolation:
    def test_write_with_isolation(self, tmp_path):
        slot_manager.write_slot_md(
            tmp_path, 7, ["soredium"], "issue-42-fix", "42",
            "Hortora/soredium", "42", "Fix scoring",
            isolation_type="isx", isx_instance="issue-42-fix",
            isx_template="tpl-java",
        )
        content = (tmp_path / ".slot").read_text()
        assert "## Isolation" in content
        assert "type: isx" in content
        assert "instance: issue-42-fix" in content
        assert "template: tpl-java" in content

    def test_write_without_isolation(self, tmp_path):
        slot_manager.write_slot_md(
            tmp_path, 7, ["soredium"], "issue-42-fix", "42",
            "Hortora/soredium", "42", "Fix scoring",
        )
        content = (tmp_path / ".slot").read_text()
        assert "## Isolation" not in content

    def test_write_isolation_roundtrip(self, tmp_path):
        slot_manager.write_slot_md(
            tmp_path, 7, ["soredium"], "issue-42-fix", "42",
            "Hortora/soredium", "42", "Fix scoring",
            isolation_type="isx", isx_instance="issue-42-fix",
            isx_template="tpl-java",
        )
        result = slot_manager.parse_slot_md(tmp_path)
        assert result["isolation_type"] == "isx"
        assert result["isx_instance"] == "issue-42-fix"
        assert result["isx_template"] == "tpl-java"
        assert result["repos"] == ["soredium"]
```

- [ ] **Step 6: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_slot_manager.py::TestWriteSlotMdIsolation -v`
Expected: FAIL — `write_slot_md()` doesn't accept isolation params

- [ ] **Step 7: Implement `write_slot_md()` isolation support**

Add optional parameters to `write_slot_md()`:
```python
def write_slot_md(slot_dir: Path, slot_number: int, repos: list[str],
                  branch: str, issue: str, issue_repo: str,
                  covers: str, context: str,
                  isolation_type: str = "", isx_instance: str = "",
                  isx_template: str = "") -> None:
```

After the repos section and before the Created section, add:
```python
if isolation_type:
    content += f"\n## Isolation\ntype: {isolation_type}\n"
    content += f"instance: {isx_instance}\ntemplate: {isx_template}\n"
```

- [ ] **Step 8: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_slot_manager.py::TestParseSlotMdIsolation tests/test_slot_manager.py::TestWriteSlotMdIsolation -v`
Expected: PASS

- [ ] **Step 9: Run full slot_manager test suite for regressions**

Run: `python3 -m pytest tests/test_slot_manager.py -v`
Expected: All existing tests still PASS

- [ ] **Step 10: Commit**

```bash
git -C <PROJECT> add work-slot/slot_manager.py tests/test_slot_manager.py
git -C <PROJECT> commit -m "feat(#223): parse/write ## Isolation section in .slot metadata

Extends parse_slot_md() to return isolation_type, isx_instance,
isx_template fields. Extends write_slot_md() to accept and emit
the ## Isolation section.

Refs #223"
```

---

### Task 2: ISX Helpers — Pre-flight, Instance Naming, Teardown

Utility functions used by create, remove, and archive operations.

**Files:**
- Modify: `work-slot/slot_manager.py` — add `_check_isx_available()`, `_truncate_instance_name()`, `_teardown_isx()`, `_wire_isx_remotes()`
- Test: `tests/test_slot_manager.py` — new classes

**Interfaces:**
- Produces: `_check_isx_available() -> bool` — returns True if `isx` on PATH
- Produces: `_truncate_instance_name(name: str, max_len: int = 63) -> str` — truncates safely
- Produces: `_teardown_isx(slot_dir: Path) -> None` — reads `.slot`, runs `isx destroy`, warns on failure
- Produces: `_wire_isx_remotes(slot_dir: Path, repos: list[str], instance: str) -> None` — adds `isx://` remote per repo

- [ ] **Step 1: Write failing tests**

```python
class TestIsxHelpers:
    def test_check_isx_available_found(self):
        with patch("shutil.which", return_value="/opt/homebrew/bin/isx"):
            assert slot_manager._check_isx_available() is True

    def test_check_isx_available_missing(self):
        with patch("shutil.which", return_value=None):
            assert slot_manager._check_isx_available() is False

    def test_truncate_short_name(self):
        assert slot_manager._truncate_instance_name("issue-42-fix") == "issue-42-fix"

    def test_truncate_long_name(self):
        long_name = "issue-223-isx-isolation-for-slots-with-very-long-description-that-exceeds-limit"
        result = slot_manager._truncate_instance_name(long_name, max_len=63)
        assert len(result) <= 63
        assert result.startswith("issue-223-isx-isolation")

    def test_truncate_strips_trailing_hyphens(self):
        name = "a" * 60 + "---bcd"
        result = slot_manager._truncate_instance_name(name, max_len=63)
        assert not result.endswith("-")


class TestTeardownIsx:
    def test_teardown_isx_slot(self, tmp_path):
        (tmp_path / ".slot").write_text(
            "# Slot 1 — test\n\n## Isolation\ntype: isx\n"
            "instance: test-instance\ntemplate: tpl-java\n"
        )
        with patch("slot_manager.run_cmd", return_value=(0, "", "")) as mock:
            slot_manager._teardown_isx(tmp_path)
            mock.assert_called_once_with(["isx", "destroy", "test-instance"])

    def test_teardown_non_isx_slot(self, tmp_path):
        (tmp_path / ".slot").write_text(
            "# Slot 1 — test\n\n## Repos\n- soredium\n"
        )
        with patch("slot_manager.run_cmd") as mock:
            slot_manager._teardown_isx(tmp_path)
            mock.assert_not_called()

    def test_teardown_destroy_fails_warns(self, tmp_path, capsys):
        (tmp_path / ".slot").write_text(
            "# Slot 1 — test\n\n## Isolation\ntype: isx\n"
            "instance: gone-instance\ntemplate: tpl-java\n"
        )
        with patch("slot_manager.run_cmd", return_value=(1, "", "not found")):
            slot_manager._teardown_isx(tmp_path)
            out = capsys.readouterr().out
            assert "WARN" in out


class TestWireIsxRemotes:
    def test_wire_remotes_adds_per_repo(self, tmp_path):
        repos = ["engine", "iot"]
        for r in repos:
            repo_dir = tmp_path / r
            repo_dir.mkdir()
            (repo_dir / ".git").mkdir()
        with patch("slot_manager.run_cmd", return_value=(0, "", "")) as mock:
            slot_manager._wire_isx_remotes(tmp_path, repos, "test-instance")
            assert mock.call_count == 2
            mock.assert_any_call([
                "git", "-C", str(tmp_path / "engine"),
                "remote", "add", "isx",
                "isx://test-instance/home/agentuser/engine",
            ])
            mock.assert_any_call([
                "git", "-C", str(tmp_path / "iot"),
                "remote", "add", "isx",
                "isx://test-instance/home/agentuser/iot",
            ])
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_slot_manager.py::TestIsxHelpers tests/test_slot_manager.py::TestTeardownIsx tests/test_slot_manager.py::TestWireIsxRemotes -v`
Expected: FAIL — functions don't exist

- [ ] **Step 3: Implement helper functions**

Add to `slot_manager.py`:

```python
def _check_isx_available() -> bool:
    return shutil.which("isx") is not None


def _truncate_instance_name(name: str, max_len: int = 63) -> str:
    if len(name) <= max_len:
        return name
    truncated = name[:max_len].rstrip("-")
    return truncated


def _teardown_isx(slot_dir: Path) -> None:
    info = parse_slot_md(slot_dir)
    if info.get("isolation_type") != "isx":
        return
    instance = info.get("isx_instance", "")
    if not instance:
        return
    rc, _, stderr = run_cmd(["isx", "destroy", instance])
    if rc != 0:
        print(f"WARN=isx_destroy_failed instance={instance} err={stderr.strip()}")


def _wire_isx_remotes(slot_dir: Path, repos: list[str], instance: str) -> None:
    for repo_name in repos:
        clone_path = slot_dir / repo_name
        if not clone_path.is_dir():
            continue
        remote_url = f"isx://{instance}/home/agentuser/{repo_name}"
        run_cmd(["git", "-C", str(clone_path), "remote", "add", "isx", remote_url])
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_slot_manager.py::TestIsxHelpers tests/test_slot_manager.py::TestTeardownIsx tests/test_slot_manager.py::TestWireIsxRemotes -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C <PROJECT> add work-slot/slot_manager.py tests/test_slot_manager.py
git -C <PROJECT> commit -m "feat(#223): ISX helper functions — pre-flight, naming, teardown, remote wiring

Adds _check_isx_available(), _truncate_instance_name(),
_teardown_isx(), _wire_isx_remotes() for use by create/remove/archive.

Refs #223"
```

---

### Task 3: `create-slot` ISX Extension

Wire the ISX flow into `create_slot()`.

**Files:**
- Modify: `work-slot/slot_manager.py` — `create_slot()` function (~line 434) and `main()` CLI dispatch (~line 1700)
- Test: `tests/test_slot_manager.py` — new class `TestCreateSlotIsx`

**Interfaces:**
- Consumes: `_check_isx_available()`, `_truncate_instance_name()`, `_wire_isx_remotes()`, `write_slot_md()` (with isolation params)
- Produces: `create_slot()` accepts `isx: bool = False`, `isx_template: str = ""`, `isx_instance: str = ""` params

- [ ] **Step 1: Write failing tests**

```python
class TestCreateSlotIsx:
    def test_create_isx_slot_preflight_fails(self, tmp_path):
        """ISX not on PATH — should abort before creating slot dir."""
        family = tmp_path / "family"
        family.mkdir()
        repo = init_repo(family / "engine")
        with patch("shutil.which", return_value=None):
            with pytest.raises(SystemExit):
                slot_manager.create_slot(
                    family_root=family, repos=["engine"],
                    branch="issue-42", issue="42",
                    issue_repo="Hortora/soredium", covers="42",
                    context="test", isx=True, isx_template="tpl-java",
                )
        assert not (family / "slots" / "1").exists()

    def test_create_isx_slot_writes_isolation(self, tmp_path):
        """ISX available — slot created with isolation metadata."""
        family = tmp_path / "family"
        family.mkdir()
        repo = init_repo(family / "engine")
        mock_returns = {
            ("isx", "branch"): (0, "", ""),
            ("git",): (0, "", ""),
        }
        def mock_run(args, **kw):
            if args[0] == "isx":
                return subprocess.CompletedProcess(args, 0, "", "")
            return subprocess.run(args, **kw)

        with patch("shutil.which", return_value="/opt/homebrew/bin/isx"), \
             patch("slot_manager.run_cmd") as mock_cmd:
            mock_cmd.side_effect = lambda args, cwd=None: (0, "", "")
            result = slot_manager.create_slot(
                family_root=family, repos=["engine"],
                branch="issue-42-fix", issue="42",
                issue_repo="Hortora/soredium", covers="42",
                context="test", isx=True, isx_template="tpl-java",
            )
        slot_dir = family / "slots" / str(result["slot_number"])
        info = slot_manager.parse_slot_md(slot_dir)
        assert info["isolation_type"] == "isx"
        assert info["isx_template"] == "tpl-java"

    def test_create_non_isx_unchanged(self, tmp_path):
        """Without isx=True, no isolation section."""
        family = tmp_path / "family"
        family.mkdir()
        repo = init_repo(family / "engine")
        with patch("slot_manager.run_cmd") as mock_cmd:
            mock_cmd.side_effect = lambda args, cwd=None: (0, "", "")
            result = slot_manager.create_slot(
                family_root=family, repos=["engine"],
                branch="issue-42-fix", issue="42",
                issue_repo="Hortora/soredium", covers="42",
                context="test",
            )
        slot_dir = family / "slots" / str(result["slot_number"])
        info = slot_manager.parse_slot_md(slot_dir)
        assert info["isolation_type"] == ""
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_slot_manager.py::TestCreateSlotIsx -v`
Expected: FAIL — `create_slot()` doesn't accept `isx` param

- [ ] **Step 3: Implement ISX extension in `create_slot()`**

Add params to `create_slot()` signature:
```python
def create_slot(family_root: Path, repos: list[str], branch: str,
                issue: str, issue_repo: str, covers: str,
                context: str,
                isx: bool = False, isx_template: str = "",
                isx_instance: str = "") -> dict:
```

At the top of the function, before `slots_dir.mkdir()`:
```python
if isx and not _check_isx_available():
    print("ERROR=isx_not_found")
    print("ERROR_DETAIL=isx is not on PATH. Install with: brew install sanne/tap/incus-spawn")
    sys.exit(1)
```

Before the existing `write_slot_md()` call, compute instance name and
create the ISX instance. This ordering ensures: if `isx branch` fails,
no isolation metadata is written and the slot is treated as non-ISX.
```python
instance_name = ""
if isx:
    instance_name = isx_instance or _truncate_instance_name(branch)
    rc, _, stderr = run_cmd(["isx", "branch", instance_name, "--from", isx_template])
    if rc != 0:
        print(f"ERROR=isx_branch_failed instance={instance_name} err={stderr.strip()}")
        sys.exit(1)
```

Update the `write_slot_md()` call to pass isolation params:
```python
write_slot_md(slot_dir, slot_num, repos, branch, issue,
              issue_repo, covers, context,
              isolation_type="isx" if isx else "",
              isx_instance=instance_name if isx else "",
              isx_template=isx_template if isx else "")
```

After `write_slot_md()`, wire remotes:
```python
if isx:
    _wire_isx_remotes(slot_dir, repos, instance_name)
```

Update `main()` CLI dispatch to pass `isx`, `template`, `instance` from args.

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_slot_manager.py::TestCreateSlotIsx -v`
Expected: PASS

- [ ] **Step 5: Run full suite for regressions**

Run: `python3 -m pytest tests/test_slot_manager.py -v`
Expected: All PASS

- [ ] **Step 6: Commit**

```bash
git -C <PROJECT> add work-slot/slot_manager.py tests/test_slot_manager.py
git -C <PROJECT> commit -m "feat(#223): ISX extension for create-slot

Pre-flight checks isx on PATH before mkdir. Creates ISX instance,
wires isx:// remotes, writes isolation metadata to .slot.

Refs #223"
```

---

### Task 4: `sync-isx` Subcommand

New subcommand for fetching committed work from ISX containers.

**Files:**
- Modify: `work-slot/slot_manager.py` — add `sync_isx()` function and `main()` dispatch
- Test: `tests/test_slot_manager.py` — new class `TestSyncIsx`

**Interfaces:**
- Consumes: `parse_slot_md()` (isolation fields), `get_slot_repos()`, `_resolve_slot_dir_for_number()`
- Produces: `sync_isx(slot_dir: Path) -> int` — returns 0 on success, 1 on failure

- [ ] **Step 1: Write failing tests**

```python
class TestSyncIsx:
    def test_sync_happy_path(self, tmp_path):
        slot_dir = tmp_path / "slots" / "1"
        slot_dir.mkdir(parents=True)
        repo = init_repo(slot_dir / "engine")
        (slot_dir / ".slot").write_text(
            "# Slot 1 — test\n\n## Issue\nOrg/repo#42\nCovers: 42\n\n"
            "## Repos\n- engine (primary)\n\n"
            "## Isolation\ntype: isx\ninstance: test-inst\n"
            "template: tpl-java\n\n## Created\n2026-08-12, branch: test\n"
        )
        with patch("slot_manager.run_cmd") as mock:
            mock.return_value = (0, "", "")
            result = slot_manager.sync_isx(slot_dir)
        assert result == 0
        calls = [c[0][0] for c in mock.call_args_list]
        assert any("fetch" in c for c in calls)
        assert any("merge" in c for c in calls)

    def test_sync_non_isx_slot(self, tmp_path):
        slot_dir = tmp_path / "slots" / "1"
        slot_dir.mkdir(parents=True)
        (slot_dir / ".slot").write_text(
            "# Slot 1 — test\n\n## Repos\n- engine\n"
        )
        result = slot_manager.sync_isx(slot_dir)
        assert result == 1

    def test_sync_diverged_stops(self, tmp_path):
        slot_dir = tmp_path / "slots" / "1"
        slot_dir.mkdir(parents=True)
        repo = init_repo(slot_dir / "engine")
        (slot_dir / ".slot").write_text(
            "# Slot 1 — test\n\n## Issue\nOrg/repo#42\nCovers: 42\n\n"
            "## Repos\n- engine (primary)\n\n"
            "## Isolation\ntype: isx\ninstance: test-inst\n"
            "template: tpl-java\n\n## Created\n2026-08-12, branch: test\n"
        )
        def side_effect(args, cwd=None):
            if "merge" in args and "--ff-only" in args:
                return (1, "", "fatal: Not possible to fast-forward")
            return (0, "", "")
        with patch("slot_manager.run_cmd", side_effect=side_effect):
            result = slot_manager.sync_isx(slot_dir)
        assert result == 1

    def test_sync_no_isx_remote_skips(self, tmp_path):
        slot_dir = tmp_path / "slots" / "1"
        slot_dir.mkdir(parents=True)
        repo = init_repo(slot_dir / "engine")
        (slot_dir / ".slot").write_text(
            "# Slot 1 — test\n\n## Issue\nOrg/repo#42\nCovers: 42\n\n"
            "## Repos\n- engine (primary)\n\n"
            "## Isolation\ntype: isx\ninstance: test-inst\n"
            "template: tpl-java\n\n## Created\n2026-08-12, branch: test\n"
        )
        def side_effect(args, cwd=None):
            if "fetch" in args:
                return (1, "", "fatal: 'isx' does not appear to be a git repository")
            return (0, "", "")
        with patch("slot_manager.run_cmd", side_effect=side_effect):
            result = slot_manager.sync_isx(slot_dir)
        assert result == 0  # skips with warning, doesn't fail
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_slot_manager.py::TestSyncIsx -v`
Expected: FAIL — `sync_isx` not defined

- [ ] **Step 3: Implement `sync_isx()`**

```python
def sync_isx(slot_dir: Path) -> int:
    info = parse_slot_md(slot_dir)
    if info.get("isolation_type") != "isx":
        print("ERROR=not_isx_slot")
        print("ERROR_DETAIL=This slot has no ISX isolation.")
        return 1

    instance = info.get("isx_instance", "")
    branch = info.get("branch", "")
    repos = get_slot_repos(slot_dir)

    for repo_name in repos:
        clone_path = slot_dir / repo_name
        if not clone_path.is_dir():
            continue
        rc, _, _ = run_cmd(["git", "-C", str(clone_path), "remote", "get-url", "isx"])
        if rc != 0:
            print(f"WARN=no_isx_remote repo={repo_name}")
            continue
        rc, _, stderr = run_cmd([
            "git", "-C", str(clone_path), "fetch", "isx", branch,
        ])
        if rc != 0:
            print(f"WARN=fetch_failed repo={repo_name} err={stderr.strip()}")
            continue
        rc, _, stderr = run_cmd([
            "git", "-C", str(clone_path), "merge", "--ff-only", f"isx/{branch}",
        ])
        if rc != 0:
            print(f"ERROR=merge_failed repo={repo_name} err={stderr.strip()}")
            print("ERROR_DETAIL=Histories have diverged. Resolve manually or reset.")
            return 1
        print(f"SYNCED={repo_name}")

    return 0
```

Add CLI dispatch in `main()`:
```python
elif subcommand == "sync-isx":
    target = args.get("target", "")
    slot_num_str = args.get("slot", "")
    if slot_num_str:
        family_root = Path(target) if target else Path(".")
        slot_dir = _resolve_slot_dir_for_number(family_root, int(slot_num_str))
    elif target:
        slot_dir = Path(target)
    else:
        slot_dir = Path(".")
    if not slot_dir.exists():
        print(f"ERROR=slot_not_found")
        sys.exit(1)
    sys.exit(sync_isx(slot_dir))
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_slot_manager.py::TestSyncIsx -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C <PROJECT> add work-slot/slot_manager.py tests/test_slot_manager.py
git -C <PROJECT> commit -m "feat(#223): sync-isx subcommand

Fetches committed work from ISX container to local slot clones
via isx:// git remotes. Supports slot number and CWD invocation.

Refs #223"
```

---

### Task 5: `add-repo` ISX Wiring, Remove/Archive Cleanup, List Isolation

Three small extensions to existing functions.

**Files:**
- Modify: `work-slot/slot_manager.py` — `add_repo()` (~line 571), `remove_slot()` (~line 1523), `archive_slot()` (~line 1363), `list_slots()` (~line 1454)
- Test: `tests/test_slot_manager.py` — new classes

**Interfaces:**
- Consumes: `parse_slot_md()` (isolation fields), `_teardown_isx()`, `_wire_isx_remotes()`

- [ ] **Step 1: Write failing tests**

```python
class TestAddRepoIsx:
    def test_add_repo_wires_isx_remote(self, tmp_path):
        family = tmp_path / "family"
        family.mkdir()
        slot_dir = family / "slots" / "1"
        slot_dir.mkdir(parents=True)
        repo = init_repo(slot_dir / "engine")
        new_repo = init_repo(family / "iot")
        (slot_dir / ".slot").write_text(
            "# Slot 1 — test\n\n## Issue\nOrg/repo#42\nCovers: 42\n\n"
            "## Repos\n- engine (primary)\n\n"
            "## Isolation\ntype: isx\ninstance: test-inst\n"
            "template: tpl-java\n\n## Created\n2026-08-12, branch: test\n"
        )
        with patch("slot_manager.run_cmd") as mock:
            mock.return_value = (0, "", "")
            slot_manager.add_repo(family, 1, "iot", "test-branch")
        remote_calls = [c for c in mock.call_args_list
                       if "remote" in c[0][0] and "add" in c[0][0]
                       and "isx" in c[0][0]]
        assert len(remote_calls) >= 1

    def test_add_repo_non_isx_no_remote(self, tmp_path):
        family = tmp_path / "family"
        family.mkdir()
        slot_dir = family / "slots" / "1"
        slot_dir.mkdir(parents=True)
        repo = init_repo(slot_dir / "engine")
        new_repo = init_repo(family / "iot")
        (slot_dir / ".slot").write_text(
            "# Slot 1 — test\n\n## Repos\n- engine (primary)\n\n"
            "## Created\n2026-08-12, branch: test\n"
        )
        with patch("slot_manager.run_cmd") as mock:
            mock.return_value = (0, "", "")
            slot_manager.add_repo(family, 1, "iot", "test-branch")
        isx_calls = [c for c in mock.call_args_list
                    if any("isx://" in str(a) for a in c[0][0])]
        assert len(isx_calls) == 0


class TestRemoveSlotIsx:
    def test_remove_destroys_isx(self, tmp_path):
        family = tmp_path / "family"
        slot_dir = family / "slots" / "1"
        slot_dir.mkdir(parents=True)
        (slot_dir / ".slot").write_text(
            "# Slot 1 — test\n\n## Isolation\ntype: isx\n"
            "instance: test-inst\ntemplate: tpl-java\n"
        )
        (slot_dir / ".landed").write_text("landed")
        with patch("slot_manager.run_cmd", return_value=(0, "", "")):
            with patch("slot_manager._teardown_isx") as mock_teardown:
                slot_manager.remove_slot(family, 1)
                mock_teardown.assert_called_once()


class TestListSlotsIsolation:
    def test_list_shows_isx_isolation(self, tmp_path):
        slot_dir = tmp_path / "slots" / "1"
        slot_dir.mkdir(parents=True)
        (slot_dir / ".slot").write_text(
            "# Slot 1 — test-branch\n\n## Repos\n- engine\n\n"
            "## Isolation\ntype: isx\ninstance: test-inst\n"
            "template: tpl-java\n"
        )
        repo = init_repo(slot_dir / "engine")
        slots = slot_manager.list_slots(tmp_path)
        assert slots[0]["isolation"] == "isx"

    def test_list_shows_none_isolation(self, tmp_path):
        slot_dir = tmp_path / "slots" / "1"
        slot_dir.mkdir(parents=True)
        (slot_dir / ".slot").write_text(
            "# Slot 1 — test-branch\n\n## Repos\n- engine\n"
        )
        repo = init_repo(slot_dir / "engine")
        slots = slot_manager.list_slots(tmp_path)
        assert slots[0]["isolation"] == "none"
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_slot_manager.py::TestAddRepoIsx tests/test_slot_manager.py::TestRemoveSlotIsx tests/test_slot_manager.py::TestListSlotsIsolation -v`
Expected: FAIL

- [ ] **Step 3: Implement changes**

**`add_repo()`** — after the existing workspace/symlink wiring (before `_update_slot_repos()`), add:
```python
slot_info = parse_slot_md(slot_dir)
if slot_info.get("isolation_type") == "isx":
    instance = slot_info.get("isx_instance", "")
    if instance:
        _wire_isx_remotes(slot_dir, [repo_name], instance)
```

**`remove_slot()`** — before the attic move, add:
```python
_teardown_isx(slot_dir)
```

**`archive_slot()`** — before the attic move (after the promotion stamp check), add:
```python
_teardown_isx(slot_dir)
```

**`list_slots()`** — in the dict construction, add isolation field:
```python
slot_md_path = d / ".slot"
isolation = "none"
if slot_md_path.exists():
    md = parse_slot_md(d)
    isolation = md.get("isolation_type", "") or "none"

slots.append({
    "number": num,
    "branch": branch,
    "repos": repos,
    "state": state,
    "isolation": isolation,
})
```

Update the `list-slots` CLI output to include `ISOLATION=`:
```python
print(f"SLOT={s['number']} BRANCH={s['branch']} REPOS={repos_str} STATE={s['state']} ISOLATION={s.get('isolation', 'none')}")
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_slot_manager.py::TestAddRepoIsx tests/test_slot_manager.py::TestRemoveSlotIsx tests/test_slot_manager.py::TestListSlotsIsolation -v`
Expected: PASS

- [ ] **Step 5: Run full suite for regressions**

Run: `python3 -m pytest tests/test_slot_manager.py -v`
Expected: All PASS

- [ ] **Step 6: Commit**

```bash
git -C <PROJECT> add work-slot/slot_manager.py tests/test_slot_manager.py
git -C <PROJECT> commit -m "feat(#223): ISX wiring in add-repo, destroy in remove/archive, isolation in list

add-repo wires isx:// remote for ISX slots. remove/archive runs
isx destroy before archiving. list-slots shows ISOLATION column.

Refs #223"
```

---

### Task 6: Staleness Pre-flight in `work_end_context.py`

Extends work-end's precondition output to detect unsynced ISX commits.

**Files:**
- Modify: `work-end/work_end_context.py` — add `check_isx_staleness()`, call from `gather_context()`
- Test: `tests/test_work_end_context.py` — new class `TestIsxStaleness`

**Interfaces:**
- Consumes: `parse_slot_md()` from `work-slot/slot_manager.py` (via sys.path)
- Produces: `preconditions["isx_staleness"]` in the context JSON output with `status: pass|needs_input` and per-repo details

- [ ] **Step 1: Write failing tests**

```python
class TestIsxStaleness:
    def test_non_isx_slot_skipped(self, tmp_path):
        ws = tmp_path / "ws"
        ws.mkdir()
        proj = tmp_path / "proj"
        proj.mkdir()
        # No .slot file — not in a slot
        result = work_end_context.check_isx_staleness(str(ws), str(proj))
        assert result["status"] == "skip"

    def test_isx_stale_detected(self, tmp_path):
        ws = tmp_path / "ws"
        ws.mkdir()
        proj = tmp_path / "proj"
        proj.mkdir()
        slot_file = proj.parent / ".slot"  # slot root
        # This test needs to mock the slot detection and git ls-remote
        # Exact implementation depends on how we detect slot context
        pass  # Placeholder — refine during implementation

    def test_isx_in_sync(self, tmp_path):
        pass  # Placeholder — refine during implementation
```

Note: The exact test structure depends on how `work_end_context.py` discovers it's running inside an ISX slot. The workspace/project paths passed to it need to resolve to a slot directory where `.slot` lives. This will be refined during implementation — the key contract is:
- If not in a slot or not an ISX slot: `{"status": "skip"}`
- If ISX and in sync: `{"status": "pass"}`
- If ISX and stale: `{"status": "needs_input", "detail": "isx-stale", "repos": [...]}`

- [ ] **Step 2: Implement `check_isx_staleness()`**

Add to `work_end_context.py`:
```python
def check_isx_staleness(workspace: str, project: str) -> dict:
    # Walk up from project to find .slot
    slot_dir = Path(project).parent
    slot_file = slot_dir / ".slot"
    if not slot_file.exists():
        return {"status": "skip"}

    # Import slot parser (source-local, not installed skill path)
    slot_skill = Path(__file__).parent.parent / "work-slot"
    sys.path.insert(0, str(slot_skill))
    try:
        from slot_manager import parse_slot_md, get_slot_repos
    except ImportError:
        return {"status": "skip"}

    info = parse_slot_md(slot_dir)
    if info.get("isolation_type") != "isx":
        return {"status": "skip"}

    instance = info.get("isx_instance", "")
    stale_repos = []
    for repo_name in get_slot_repos(slot_dir):
        clone_path = slot_dir / repo_name
        local_head = git(str(clone_path), "rev-parse", "HEAD")
        remote = git(str(clone_path), "ls-remote", "isx", "HEAD")
        if local_head.returncode != 0 or remote.returncode != 0:
            continue
        local_sha = local_head.stdout.strip()
        remote_sha = remote.stdout.split()[0] if remote.stdout.strip() else ""
        if remote_sha and local_sha != remote_sha:
            stale_repos.append(repo_name)

    if stale_repos:
        return {
            "status": "needs_input",
            "detail": "isx-stale",
            "repos": stale_repos,
        }
    return {"status": "pass"}
```

Add to `gather_context()`:
```python
preconditions["isx_staleness"] = check_isx_staleness(workspace, project)
```

- [ ] **Step 3: Write concrete tests against implementation**

After implementing, write tests that mock `git` subprocess calls and verify the output JSON structure.

- [ ] **Step 4: Run tests**

Run: `python3 -m pytest tests/test_work_end_context.py -v`
Expected: All PASS (new + existing)

- [ ] **Step 5: Commit**

```bash
git -C <PROJECT> add work-end/work_end_context.py tests/test_work_end_context.py
git -C <PROJECT> commit -m "feat(#223): ISX staleness pre-flight in work-end context

Detects unsynced ISX commits by comparing local HEAD against
isx:// remote HEAD. Surfaces as needs_input precondition.

Refs #223"
```

---

### Task 7: SKILL.md Updates

Update skill documentation for the new ISX capabilities.

**Files:**
- Modify: `work-slot/SKILL.md`

**Interfaces:**
- None — documentation only, no code dependencies

- [ ] **Step 1: Update `work-slot/SKILL.md` Step 1 (Gather input)**

Add `--isx` to input list:
```markdown
- `--isx` — create with ISX container isolation
- If `--isx`: run `isx templates list`, present available templates for
  selection. Optional instance name override (defaults to branch name,
  truncated to 63 chars).
```

- [ ] **Step 2: Update Step 4 (Create the slot)**

Add after the `create-slot` script call:
```markdown
If `--isx` was specified, pass `isx=yes template=<name>` to `create-slot`.
The script handles pre-flight (`isx` on PATH), instance creation
(`isx branch`), remote wiring (`isx://` per repo), and `.slot`
isolation metadata.
```

- [ ] **Step 3: Update Step 8 (Shell offering)**

Add ISX-aware offering:
```markdown
For ISX slots (`.slot` has `type: isx`), offer both:
1. iTerm tab for host-side ops (sync, work-end)
2. `isx shell <instance>` for container work

> "Open iTerm tab for host ops? (y/n)"
> "Open ISX shell for container work? (y/n)"
```

- [ ] **Step 4: Add `work-slot sync` section**

Add new section after `work-slot status`:
```markdown
## `work-slot sync`

Fetch committed work from an ISX container into local slot clones.

### Usage

`work-slot sync <N>` (by slot number) or
`work-slot sync` (from inside an ISX slot clone, auto-detected).

### What it does

For each project repo in the slot:
1. `git fetch isx <branch>` — fetch from ISX container
2. `git merge --ff-only isx/<branch>` — fast-forward local clone

### Error handling

- Non-ISX slot → error: "This slot has no ISX isolation."
- Diverged histories → stops, reports which repo diverged
- No `isx` remote on a repo → skips with warning

### When to run

After finishing work inside the ISX container (`isx shell`), before
running `work end` on the host. `work end` includes a staleness
pre-flight that warns if you forget.
```

- [ ] **Step 5: Update `work-slot remove` section**

Add note:
```markdown
For ISX slots, `isx destroy <instance>` runs before archiving to
clean up the container. Warns and continues if destroy fails
(instance may already be gone).
```

- [ ] **Step 6: Update Skill Chaining**

Add to Complements:
```markdown
- `work-start` — on resume for ISX slots, offers `isx shell` for
  container access (same offering as creation Step 8)
```

- [ ] **Step 7: Commit**

```bash
git -C <PROJECT> add work-slot/SKILL.md
git -C <PROJECT> commit -m "docs(#223): update work-slot SKILL.md for ISX isolation

Adds --isx flag docs, work-slot sync section, ISX shell offering,
ISX destroy on remove, resume path shell offering note.

Refs #223"
```

---

## Execution Order

Tasks 1–6 are sequential (each builds on prior). Task 7 (SKILL.md) can
run in parallel with any code task but is placed last for clean history.

```
Task 1: Parser/Writer      ← foundation
Task 2: Helpers             ← depends on Task 1 (parse_slot_md)
Task 3: create-slot ISX     ← depends on Tasks 1, 2
Task 4: sync-isx            ← depends on Task 1
Task 5: add/remove/list     ← depends on Tasks 1, 2
Task 6: Staleness           ← depends on Task 1
Task 7: SKILL.md            ← independent (docs)
```
