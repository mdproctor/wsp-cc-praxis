# Archive Slot Session Guard Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #314 — fix: archive-slot must check for active sessions before moving
**Issue group:** #314

**Goal:** Prevent archive-slot from moving a slot directory while a Claude session is actively running inside it.

**Architecture:** `find_active_sessions()` in `slot_claude.py` uses `lsof +D` to detect processes with CWD/open files inside the slot. `archive_slot()` in `slot_lifecycle.py` calls it before `shutil.move()` — blocks unless `--force`.

**Tech Stack:** Python 3, lsof (macOS/Linux)

## Global Constraints

- `lsof` timeout: 10 seconds max — fail open if unavailable
- `--force` overrides the guard (consistent with existing force behavior)

---

## Batch 1: Session guard

### Task 1: `find_active_sessions()` and gate in `archive_slot()`

**Files:**
- Modify: `work-slot/slot_claude.py` (add `find_active_sessions` after `relocate_claude_projects`)
- Modify: `work-slot/slot_lifecycle.py:588-605` (add guard in `archive_slot` after SHA check, before move)
- Test: `tests/test_slot_lifecycle.py`

**Interfaces:**
- Consumes: `subprocess.run` for lsof
- Produces: `find_active_sessions(slot_dir: Path) -> list[tuple[int, str, str]]` — returns `[(pid, command, path)]`

- [ ] **Step 1: Write failing test — `find_active_sessions` returns empty for inactive dir**

```python
class TestFindActiveSessions:
    def test_returns_empty_for_inactive_dir(self, tmp_path):
        from slot_claude import find_active_sessions
        result = find_active_sessions(tmp_path)
        assert result == []
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_slot_lifecycle.py::TestFindActiveSessions::test_returns_empty_for_inactive_dir -v`
Expected: ImportError — `find_active_sessions` does not exist

- [ ] **Step 3: Implement `find_active_sessions` in `slot_claude.py`**

Add after `relocate_claude_projects` (line 42):

```python
def find_active_sessions(slot_dir: Path) -> list[tuple[int, str, str]]:
    """Find processes with open file descriptors inside slot_dir.

    Uses lsof +D for recursive scan. Returns [(pid, command, path)].
    Fails open (returns []) if lsof is unavailable or times out.
    """
    import subprocess
    try:
        result = subprocess.run(
            ["lsof", "+D", str(slot_dir)],
            capture_output=True, text=True, timeout=10,
        )
    except (FileNotFoundError, subprocess.TimeoutExpired):
        return []
    if not result.stdout.strip():
        return []
    sessions = []
    seen_pids: set[int] = set()
    for line in result.stdout.strip().splitlines()[1:]:
        parts = line.split()
        if len(parts) < 2:
            continue
        try:
            pid = int(parts[1])
        except ValueError:
            continue
        if pid in seen_pids:
            continue
        seen_pids.add(pid)
        cmd = parts[0]
        path = parts[-1] if len(parts) >= 9 else ""
        sessions.append((pid, cmd, path))
    return sessions
```

- [ ] **Step 4: Run test to verify it passes**

Run: `python3 -m pytest tests/test_slot_lifecycle.py::TestFindActiveSessions::test_returns_empty_for_inactive_dir -v`
Expected: PASS

- [ ] **Step 5: Write failing test — `find_active_sessions` detects process in dir**

```python
    def test_detects_process_in_dir(self, tmp_path):
        import subprocess, time
        test_file = tmp_path / "hold.txt"
        test_file.write_text("hold")
        proc = subprocess.Popen(
            ["tail", "-f", str(test_file)],
            stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL,
        )
        try:
            time.sleep(0.5)
            from slot_claude import find_active_sessions
            result = find_active_sessions(tmp_path)
            pids = [r[0] for r in result]
            assert proc.pid in pids, f"Expected pid {proc.pid} in {pids}"
        finally:
            proc.terminate()
            proc.wait()
```

- [ ] **Step 6: Run test to verify it passes**

Run: `python3 -m pytest tests/test_slot_lifecycle.py::TestFindActiveSessions::test_detects_process_in_dir -v`
Expected: PASS (implementation already handles this)

- [ ] **Step 7: Write failing test — `archive_slot` blocks when active session detected**

```python
class TestArchiveSlotSessionGuard:
    def test_blocks_when_active_session(self, tmp_path, monkeypatch):
        """archive_slot refuses when find_active_sessions returns results."""
        monkeypatch.setattr(
            "slot_claude.find_active_sessions",
            lambda d: [(12345, "claude", str(d / "blocks"))],
        )
        # Set up minimal slot structure that passes other checks
        slot_dir = tmp_path / "slots" / "99"
        slot_dir.mkdir(parents=True)
        (slot_dir / ".landed").write_text("landed_shas=abc:123\n")
        (slot_dir / ".slot").write_text("## Repos\n- blocks (primary)\n")
        repo = slot_dir / "blocks"
        repo.mkdir()
        (repo / ".git").mkdir()

        from slot_lifecycle import archive_slot
        with pytest.raises(SystemExit) as exc_info:
            archive_slot(tmp_path, 99, force=False)
        assert exc_info.value.code == 1
```

- [ ] **Step 8: Run test to verify it fails**

Run: `python3 -m pytest tests/test_slot_lifecycle.py::TestArchiveSlotSessionGuard::test_blocks_when_active_session -v`
Expected: FAIL — archive_slot does not check for active sessions (proceeds to move)

- [ ] **Step 9: Wire guard into `archive_slot()` in `slot_lifecycle.py`**

Add after the SHA verification block (after line 613) and before the promotion stamp check (line 614):

```python
    from slot_claude import find_active_sessions
    active = find_active_sessions(slot_dir)
    if active:
        if not force:
            for pid, cmd, path in active:
                print(f"ACTIVE_SESSION={pid}:{cmd}:{path}")
            print(f"ERROR=active_sessions slot={slot_num}")
            print(f"ERROR_DETAIL={len(active)} active process(es) in slot directory")
            print("HINT=close the session first, or pass --force to override")
            sys.exit(1)
        else:
            for pid, cmd, path in active:
                print(f"WARN_ACTIVE_SESSION={pid}:{cmd}:{path}")
            print(f"WARN=active_sessions_overridden slot={slot_num}")
```

- [ ] **Step 10: Run test to verify it passes**

Run: `python3 -m pytest tests/test_slot_lifecycle.py::TestArchiveSlotSessionGuard::test_blocks_when_active_session -v`
Expected: PASS

- [ ] **Step 11: Write test — `--force` overrides active session guard**

```python
    def test_force_overrides_active_session(self, tmp_path, monkeypatch, capsys):
        """archive_slot proceeds with --force even when sessions active."""
        monkeypatch.setattr(
            "slot_claude.find_active_sessions",
            lambda d: [(12345, "claude", str(d / "blocks"))],
        )
        slot_dir = tmp_path / "slots" / "99"
        slot_dir.mkdir(parents=True)
        (slot_dir / ".landed").write_text("landed_shas=abc:123\n")
        (slot_dir / ".slot").write_text("## Repos\n- blocks (primary)\n")
        repo = slot_dir / "blocks"
        repo.mkdir()
        (repo / ".git").mkdir()
        attic = tmp_path / "slots" / "attic"
        attic.mkdir(parents=True)

        # Mock the remaining operations that would fail in test
        monkeypatch.setattr("slot_lifecycle._has_unmerged_content", lambda d: [])
        monkeypatch.setattr("slot_lifecycle.verify_landed_shas", lambda d, f: (True, []))
        monkeypatch.setattr("slot_lifecycle._repack_broken_alternates", lambda d, f: 0)
        monkeypatch.setattr("slot_lifecycle._teardown_isx", lambda d: None)
        monkeypatch.setattr("slot_lifecycle.sweep_orphaned_claude_projects", lambda f: 0)
        monkeypatch.setattr("slot_lifecycle._escape_slot_cwd", lambda d, f: (False, None))
        monkeypatch.setattr("slot_lifecycle.relocate_claude_projects", lambda s, d: 0)

        from slot_lifecycle import archive_slot
        archive_slot(tmp_path, 99, force=True)

        captured = capsys.readouterr()
        assert "WARN=active_sessions_overridden" in captured.out
        assert not slot_dir.exists(), "slot should have been moved to attic"
```

- [ ] **Step 12: Run test to verify it passes**

Run: `python3 -m pytest tests/test_slot_lifecycle.py::TestArchiveSlotSessionGuard::test_force_overrides_active_session -v`
Expected: PASS

- [ ] **Step 13: Run full slot_lifecycle test suite for regressions**

Run: `python3 -m pytest tests/test_slot_lifecycle.py -v --tb=short`
Expected: all pass

- [ ] **Step 14: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/soredium add work-slot/slot_claude.py work-slot/slot_lifecycle.py tests/test_slot_lifecycle.py
git -C /Users/mdproctor/claude/hortora/soredium commit -m "feat: archive-slot checks for active sessions before moving Refs #314"
```

## References

- [2026-08-31-archive-slot-session-guard-design.md] — design spec
- [work-slot/slot_lifecycle.py:588-677] — archive_slot function
- [work-slot/slot_claude.py:30-101] — existing Claude project management
- [GitHub #314] — incident description
