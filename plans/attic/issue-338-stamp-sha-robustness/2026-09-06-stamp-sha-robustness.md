# Stamp SHA Robustness Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** TBD — create before implementation
**Issue group:** single issue

**Goal:** Make the branch stamp's landed SHA always correct after squash
by validating existing stamps, capturing SHAs fresh from main, and
recording a stamp history trail.

**Architecture:** Three changes to the stamping and verification pipeline:
1. `_stamp_repo` (land_flow.py) and `cmd_stamp` (land_branch.py) validate
   existing stamps against main before skipping — re-stamp if stale.
2. Both stamp functions capture the landed SHA fresh from `rev-parse
   {base_branch}` rather than relying on a passed-in SHA that may be stale.
3. Stamp commits include a structured `stamp-history:` block in the commit
   body, and `check_landing_sha` (verify_slot_close.py) parses and reports
   the lineage.

**Tech Stack:** Python 3, git subprocess, pytest

## Global Constraints

- All stamp functions must remain idempotent for resume safety
- Stamp format must be backward-compatible (old stamps without history
  still pass verification)
- The stamp commit subject line format is unchanged — history goes in
  the body only
- `--force-with-lease` for all branch pushes (existing convention)

---

## Batch 1: Stamp history and validate-before-skip in land_flow.py

### Task 1: Add stamp history builder helper

**Files:**
- Modify: `work-end/land_flow.py:546-597` (`_stamp_repo`)
- Test: `tests/test_land_flow.py`

**Interfaces:**
- Produces: `_build_stamp_message(landed_sha, base_branch, issue_ref, history)` — returns
  subject + body string. `_parse_stamp_history(commit_body)` — returns list of
  `{"timestamp": str, "sha": str, "reason": str, "prev": str | None}`.

- [ ] **Step 1: Write failing tests for stamp history helpers**

```python
# In tests/test_land_flow.py

from datetime import datetime, timezone

class TestStampHistory:
    def test_build_stamp_message_initial(self):
        from land_flow import _build_stamp_message
        msg = _build_stamp_message("abc123", "main", "  Refs #42", [])
        subject, body = msg.split("\n\n", 1)
        assert subject == "chore: branch closed — landed as abc123 on main  Refs #42"
        assert "stamp-history:" in body
        assert "reason=initial" in body
        assert "sha=abc123" in body

    def test_build_stamp_message_restamp(self):
        from land_flow import _build_stamp_message
        prev_history = [
            {"timestamp": "2026-09-06T10:00:00Z", "sha": "old111", "reason": "initial", "prev": None},
        ]
        msg = _build_stamp_message("new222", "main", "", prev_history)
        subject, body = msg.split("\n\n", 1)
        assert "landed as new222" in subject
        assert "sha=old111" in body
        assert "sha=new222" in body
        assert "reason=stale_sha_not_on_base" in body
        assert "prev=old111" in body

    def test_parse_stamp_history_empty_body(self):
        from land_flow import _parse_stamp_history
        result = _parse_stamp_history("")
        assert result == []

    def test_parse_stamp_history_roundtrip(self):
        from land_flow import _build_stamp_message, _parse_stamp_history
        msg = _build_stamp_message("abc123", "main", "", [])
        _, body = msg.split("\n\n", 1)
        history = _parse_stamp_history(body)
        assert len(history) == 1
        assert history[0]["sha"] == "abc123"
        assert history[0]["reason"] == "initial"
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_land_flow.py::TestStampHistory -v`
Expected: FAIL — `_build_stamp_message` and `_parse_stamp_history` don't exist

- [ ] **Step 3: Implement the helpers in land_flow.py**

Add above `_stamp_repo` (around line 542):

```python
from datetime import datetime, timezone


def _build_stamp_message(
    landed_sha: str, base_branch: str, issue_ref: str,
    prev_history: list[dict[str, str | None]],
    reason: str = "",
) -> str:
    """Build a stamp commit message with structured history in the body."""
    subject = f"chore: branch closed — landed as {landed_sha} on {base_branch}{issue_ref}"
    ts = datetime.now(timezone.utc).strftime("%Y-%m-%dT%H:%M:%SZ")
    if not reason:
        reason = "stale_sha_not_on_base" if prev_history else "initial"
    prev_sha = prev_history[-1]["sha"] if prev_history else None
    new_entry = f"- {ts} sha={landed_sha} reason={reason}"
    if prev_sha:
        new_entry += f" prev={prev_sha}"
    lines = ["stamp-history:"]
    for h in prev_history:
        line = f"- {h['timestamp']} sha={h['sha']} reason={h['reason']}"
        if h.get("prev"):
            line += f" prev={h['prev']}"
        lines.append(line)
    lines.append(new_entry)
    return subject + "\n\n" + "\n".join(lines) + "\n"


def _parse_stamp_history(body: str) -> list[dict[str, str | None]]:
    """Parse stamp-history block from a commit body."""
    result: list[dict[str, str | None]] = []
    in_history = False
    for line in body.splitlines():
        if line.strip() == "stamp-history:":
            in_history = True
            continue
        if in_history and line.startswith("- "):
            entry: dict[str, str | None] = {"timestamp": "", "sha": "", "reason": "", "prev": None}
            parts = line[2:].split()
            if parts:
                entry["timestamp"] = parts[0]
            for part in parts[1:]:
                if "=" in part:
                    k, v = part.split("=", 1)
                    if k in entry:
                        entry[k] = v
            result.append(entry)
        elif in_history and not line.startswith("- "):
            break
    return result
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_land_flow.py::TestStampHistory -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add work-end/land_flow.py tests/test_land_flow.py
git commit -m "feat: stamp history builder and parser helpers"
```

### Task 2: Validate-and-re-stamp in _stamp_repo + fresh SHA capture

**Files:**
- Modify: `work-end/land_flow.py:546-597` (`_stamp_repo`)
- Test: `tests/test_land_flow.py`

**Interfaces:**
- Consumes: `_build_stamp_message`, `_parse_stamp_history` from Task 1
- Produces: Updated `_stamp_repo` that validates existing stamps and
  captures fresh SHA from main

- [ ] **Step 1: Write failing test — re-stamp when SHA is stale**

```python
class TestStampRevalidation:
    def test_restamps_when_sha_stale(self, tmp_path):
        """If branch has a stamp with a stale SHA, _stamp_repo re-stamps."""
        from land_flow import RepoDescriptor, Transport, _stamp_repo

        repo = _init_repo(tmp_path / "project")
        branch = "issue-42-test"
        _add_feature(repo, branch)

        # Merge to main and get SHA
        subprocess.run(["git", "-C", str(repo), "checkout", "main"], capture_output=True)
        subprocess.run(["git", "-C", str(repo), "merge", "--ff-only", branch], capture_output=True)
        subprocess.run(["git", "-C", str(repo), "push", "origin", "main"], capture_output=True)
        real_sha = subprocess.run(
            ["git", "-C", str(repo), "rev-parse", "main"],
            capture_output=True, text=True,
        ).stdout.strip()

        # Write a stamp with a FAKE stale SHA
        subprocess.run(["git", "-C", str(repo), "checkout", branch], capture_output=True)
        subprocess.run(
            ["git", "-C", str(repo), "commit", "--allow-empty",
             "-m", "chore: branch closed — landed as deadbeef00000000 on main"],
            capture_output=True,
        )
        subprocess.run(["git", "-C", str(repo), "checkout", "main"], capture_output=True)

        desc = RepoDescriptor(
            repo_path=repo, original_path=repo, push_target="origin",
            base_branch="main", is_workspace=False, transport=Transport.DIRECT,
        )
        progress = tmp_path / ".progress"
        result = _stamp_repo(desc, branch, real_sha, progress)

        assert result is True
        # Verify the stamp now has the correct SHA
        log = subprocess.run(
            ["git", "-C", str(repo), "log", "-1", "--format=%s", branch],
            capture_output=True, text=True,
        )
        assert f"landed as {real_sha}" in log.stdout.strip()

    def test_restamps_preserves_history(self, tmp_path):
        """Re-stamp includes history from previous stamp."""
        from land_flow import RepoDescriptor, Transport, _stamp_repo

        repo = _init_repo(tmp_path / "project")
        branch = "issue-42-test"
        _add_feature(repo, branch)

        subprocess.run(["git", "-C", str(repo), "checkout", "main"], capture_output=True)
        subprocess.run(["git", "-C", str(repo), "merge", "--ff-only", branch], capture_output=True)
        subprocess.run(["git", "-C", str(repo), "push", "origin", "main"], capture_output=True)
        real_sha = subprocess.run(
            ["git", "-C", str(repo), "rev-parse", "main"],
            capture_output=True, text=True,
        ).stdout.strip()

        # Write initial stamp with stale SHA
        subprocess.run(["git", "-C", str(repo), "checkout", branch], capture_output=True)
        subprocess.run(
            ["git", "-C", str(repo), "commit", "--allow-empty",
             "-m", "chore: branch closed — landed as deadbeef00000000 on main\n\nstamp-history:\n- 2026-09-01T00:00:00Z sha=deadbeef00000000 reason=initial\n"],
            capture_output=True,
        )
        subprocess.run(["git", "-C", str(repo), "checkout", "main"], capture_output=True)

        desc = RepoDescriptor(
            repo_path=repo, original_path=repo, push_target="origin",
            base_branch="main", is_workspace=False, transport=Transport.DIRECT,
        )
        progress = tmp_path / ".progress"
        _stamp_repo(desc, branch, real_sha, progress)

        body = subprocess.run(
            ["git", "-C", str(repo), "log", "-1", "--format=%B", branch],
            capture_output=True, text=True,
        ).stdout.strip()
        assert "deadbeef00000000" in body  # old SHA in history
        assert f"sha={real_sha}" in body  # new SHA in history
        assert "stale_sha_not_on_base" in body  # reason for re-stamp

    def test_skips_when_sha_valid(self, tmp_path):
        """If existing stamp has a valid SHA, skip (no re-stamp)."""
        from land_flow import RepoDescriptor, Transport, _stamp_repo

        repo = _init_repo(tmp_path / "project")
        branch = "issue-42-test"
        _add_feature(repo, branch)

        subprocess.run(["git", "-C", str(repo), "checkout", "main"], capture_output=True)
        subprocess.run(["git", "-C", str(repo), "merge", "--ff-only", branch], capture_output=True)
        subprocess.run(["git", "-C", str(repo), "push", "origin", "main"], capture_output=True)
        real_sha = subprocess.run(
            ["git", "-C", str(repo), "rev-parse", "main"],
            capture_output=True, text=True,
        ).stdout.strip()

        # Write stamp with the CORRECT SHA
        subprocess.run(["git", "-C", str(repo), "checkout", branch], capture_output=True)
        subprocess.run(
            ["git", "-C", str(repo), "commit", "--allow-empty",
             "-m", f"chore: branch closed — landed as {real_sha} on main"],
            capture_output=True,
        )
        subprocess.run(["git", "-C", str(repo), "checkout", "main"], capture_output=True)

        desc = RepoDescriptor(
            repo_path=repo, original_path=repo, push_target="origin",
            base_branch="main", is_workspace=False, transport=Transport.DIRECT,
        )
        progress = tmp_path / ".progress"

        commit_count_before = subprocess.run(
            ["git", "-C", str(repo), "rev-list", "--count", branch],
            capture_output=True, text=True,
        ).stdout.strip()

        _stamp_repo(desc, branch, real_sha, progress)

        commit_count_after = subprocess.run(
            ["git", "-C", str(repo), "rev-list", "--count", branch],
            capture_output=True, text=True,
        ).stdout.strip()
        assert commit_count_before == commit_count_after

    def test_fresh_sha_capture_overrides_passed_sha(self, tmp_path):
        """_stamp_repo captures fresh SHA from main, not the passed-in one."""
        from land_flow import RepoDescriptor, Transport, _stamp_repo

        repo = _init_repo(tmp_path / "project")
        branch = "issue-42-test"
        _add_feature(repo, branch)

        subprocess.run(["git", "-C", str(repo), "checkout", "main"], capture_output=True)
        subprocess.run(["git", "-C", str(repo), "merge", "--ff-only", branch], capture_output=True)
        subprocess.run(["git", "-C", str(repo), "push", "origin", "main"], capture_output=True)
        real_sha = subprocess.run(
            ["git", "-C", str(repo), "rev-parse", "main"],
            capture_output=True, text=True,
        ).stdout.strip()

        desc = RepoDescriptor(
            repo_path=repo, original_path=repo, push_target="origin",
            base_branch="main", is_workspace=False, transport=Transport.DIRECT,
        )
        progress = tmp_path / ".progress"
        # Pass a WRONG SHA — _stamp_repo should capture fresh from main
        _stamp_repo(desc, branch, "wrong_sha_passed_in", progress)

        log = subprocess.run(
            ["git", "-C", str(repo), "log", "-1", "--format=%s", branch],
            capture_output=True, text=True,
        )
        assert f"landed as {real_sha}" in log.stdout.strip()
        assert "wrong_sha_passed_in" not in log.stdout.strip()
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_land_flow.py::TestStampRevalidation -v`
Expected: FAIL — current `_stamp_repo` skips blindly or uses passed-in SHA

- [ ] **Step 3: Rewrite _stamp_repo with validation and fresh capture**

Replace `_stamp_repo` in `work-end/land_flow.py` (lines 546-597):

```python
def _stamp_repo(
    desc: RepoDescriptor, branch: str, landed_sha: str, progress_file: Path,
) -> bool:
    """Stamp a branch as closed. Validates existing stamps. Returns True on success."""
    key = _progress_key(desc, branch)
    repo_name = desc.repo_path.name

    # Capture fresh SHA from base branch (overrides passed-in SHA)
    fresh = _git(desc.repo_path, "rev-parse", desc.base_branch)
    if fresh.returncode == 0 and fresh.stdout.strip():
        landed_sha = fresh.stdout.strip()

    tip = _git(desc.repo_path, "log", "-1", "--format=%s", branch)
    if tip.returncode != 0:
        print(f"STAMP_WARN={repo_name} reason=branch_tip_unreadable branch={branch}")
        return False

    if tip.stdout.strip().startswith("chore: branch closed"):
        # Existing stamp — validate the SHA
        sha_match = re.search(r"landed as ([0-9a-f]+)", tip.stdout.strip())
        if sha_match:
            existing_sha = sha_match.group(1)
            check = _git(desc.repo_path, "merge-base", "--is-ancestor",
                         existing_sha, desc.base_branch)
            if check.returncode == 0:
                # SHA is valid — skip
                _write_progress(progress_file, key, "stamped")
                return True
            # SHA is stale — read history from existing stamp body
            body = _git(desc.repo_path, "log", "-1", "--format=%b", branch)
            prev_history = _parse_stamp_history(
                body.stdout if body.returncode == 0 else "")
            print(f"RESTAMP={repo_name} reason=stale_sha prev={existing_sha} new={landed_sha}")
        else:
            # Old format stamp without SHA — skip
            _write_progress(progress_file, key, "stamped")
            return True
    else:
        prev_history = []

    issue_match = re.match(r"issue-(\d+)", branch)
    issue_ref = f"  Refs #{issue_match.group(1)}" if issue_match else ""

    reason = "stale_sha_not_on_base" if prev_history else "initial"
    message = _build_stamp_message(landed_sha, desc.base_branch, issue_ref,
                                    prev_history, reason)

    co = _git(desc.repo_path, "checkout", branch)
    if co.returncode != 0:
        _git(desc.repo_path, "stash", "push", "-u", "-m", "work-end: stash before stamp")
        co2 = _git(desc.repo_path, "checkout", branch)
        if co2.returncode != 0:
            print(f"STAMP_FAIL={repo_name} reason=checkout_failed branch={branch}")
            return False

    if prev_history:
        # Amend existing stamp commit with corrected SHA and history
        commit = _git(desc.repo_path, "commit", "--allow-empty", "--amend", "-m", message)
    else:
        commit = _git(desc.repo_path, "commit", "--allow-empty", "-m", message)

    if commit.returncode != 0:
        print(f"STAMP_FAIL={repo_name} reason=commit_failed detail={commit.stderr.strip()}")
        return False

    if desc.transport == Transport.TWO_HOP:
        push = _git(desc.repo_path, "push", "origin", branch, "--force-with-lease", "--no-verify")
    else:
        push_remote = desc.push_target
        has_upstream = _git(desc.repo_path, "remote", "get-url", "upstream")
        if has_upstream.returncode == 0:
            push_remote = "origin"
        push = None
        if push_remote:
            push = _git(desc.repo_path, "push", push_remote, branch, "--force-with-lease", "--no-verify")

    if push and push.returncode != 0:
        print(f"STAMP_WARN={repo_name} reason=stamp_push_failed detail={push.stderr.strip()}")

    _write_progress(progress_file, key, "stamped")
    _record_worklog_end(branch, str(desc.repo_path), landed_sha)
    return True
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_land_flow.py::TestStampRevalidation tests/test_land_flow.py::TestStampHistory -v`
Expected: PASS

- [ ] **Step 5: Run full test_land_flow.py to check for regressions**

Run: `python3 -m pytest tests/test_land_flow.py -v`
Expected: All tests PASS (existing tests for idempotency may need minor
updates since the skip behavior changed — if `test_skips_already_stamped`
relies on progress-file-based skip, it should still work since that skip
happens before `_stamp_repo` is called)

- [ ] **Step 6: Commit**

```bash
git add work-end/land_flow.py tests/test_land_flow.py
git commit -m "feat: validate-and-re-stamp with fresh SHA capture in _stamp_repo"
```

---

## Batch 2: Same fix in land_branch.py + cmd_stamp

### Task 3: Validate-and-re-stamp in cmd_stamp

**Files:**
- Modify: `work-end/land_branch.py:147-238` (`cmd_stamp`)
- Test: `tests/test_land_branch.py`

**Interfaces:**
- Consumes: `_build_stamp_message`, `_parse_stamp_history` from
  `land_flow.py` (import them)

- [ ] **Step 1: Write failing test — cmd_stamp re-stamps stale SHA**

```python
class TestCmdStampRevalidation:
    def test_restamps_when_sha_stale(self, tmp_path):
        project = tmp_path / "project"
        _init_git(project)

        subprocess.run(["git", "-C", str(project), "checkout", "-b", "issue-42-test"], capture_output=True)
        (project / "code.txt").write_text("code")
        subprocess.run(["git", "-C", str(project), "add", "."], capture_output=True)
        subprocess.run(["git", "-C", str(project), "commit", "-m", "feat: code"], capture_output=True)

        subprocess.run(["git", "-C", str(project), "checkout", "main"], capture_output=True)
        subprocess.run(["git", "-C", str(project), "rebase", "issue-42-test"], capture_output=True)
        real_sha = subprocess.run(
            ["git", "-C", str(project), "rev-parse", "main"],
            capture_output=True, text=True,
        ).stdout.strip()

        # Write stale stamp
        subprocess.run(["git", "-C", str(project), "checkout", "issue-42-test"], capture_output=True)
        subprocess.run(
            ["git", "-C", str(project), "commit", "--allow-empty",
             "-m", "chore: branch closed — landed as deadbeef00000000 on main"],
            capture_output=True,
        )
        subprocess.run(["git", "-C", str(project), "checkout", "main"], capture_output=True)

        result = cmd_stamp(str(project), {"branch": "issue-42-test", "base_branch": "main"})
        assert result == 0

        log = subprocess.run(
            ["git", "-C", str(project), "log", "-1", "--format=%s", "issue-42-test"],
            capture_output=True, text=True,
        )
        assert f"landed as {real_sha}" in log.stdout.strip()

    def test_restamps_includes_history(self, tmp_path):
        project = tmp_path / "project"
        _init_git(project)

        subprocess.run(["git", "-C", str(project), "checkout", "-b", "feature"], capture_output=True)
        (project / "code.txt").write_text("code")
        subprocess.run(["git", "-C", str(project), "add", "."], capture_output=True)
        subprocess.run(["git", "-C", str(project), "commit", "-m", "feat: code"], capture_output=True)

        subprocess.run(["git", "-C", str(project), "checkout", "main"], capture_output=True)
        subprocess.run(["git", "-C", str(project), "rebase", "feature"], capture_output=True)

        # Write stale stamp
        subprocess.run(["git", "-C", str(project), "checkout", "feature"], capture_output=True)
        subprocess.run(
            ["git", "-C", str(project), "commit", "--allow-empty",
             "-m", "chore: branch closed — landed as deadbeef00000000 on main"],
            capture_output=True,
        )
        subprocess.run(["git", "-C", str(project), "checkout", "main"], capture_output=True)

        cmd_stamp(str(project), {"branch": "feature", "base_branch": "main"})

        body = subprocess.run(
            ["git", "-C", str(project), "log", "-1", "--format=%B", "feature"],
            capture_output=True, text=True,
        ).stdout
        assert "stamp-history:" in body
        assert "deadbeef00000000" in body
        assert "stale_sha_not_on_base" in body
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_land_branch.py::TestCmdStampRevalidation -v`
Expected: FAIL

- [ ] **Step 3: Update cmd_stamp in land_branch.py**

Import helpers from land_flow, then update the stamp logic in
`cmd_stamp` (lines 194-222):

```python
# At top of land_branch.py, add import:
from land_flow import _build_stamp_message, _parse_stamp_history

# Replace lines 194-222 (the stamp section of cmd_stamp):
    tip_msg = git(project, "log", "-1", "--format=%s", branch)
    already_stamped = tip_msg.returncode == 0 and tip_msg.stdout.strip().startswith("chore: branch closed")

    if already_stamped:
        # Validate existing stamp SHA
        sha_match = re.search(r"landed as ([0-9a-f]+)", tip_msg.stdout.strip())
        if sha_match:
            existing_sha = sha_match.group(1)
            check = git(project, "merge-base", "--is-ancestor", existing_sha, base_branch)
            if check.returncode == 0:
                print("STAMP=ok")
                print("STAMP_SKIPPED=already_stamped")
                print(f"LANDED_SHA={landed_sha}")
                # Push stamp to remote (even if skipped — ensure remote has it)
                _push_stamp(project, branch)
                _record_worklog(branch, project, landed_sha)
                return 0
            # Stale — re-stamp
            body_result = git(project, "log", "-1", "--format=%b", branch)
            prev_history = _parse_stamp_history(
                body_result.stdout if body_result.returncode == 0 else "")
            print(f"RESTAMP=yes prev={existing_sha} new={landed_sha}")
        else:
            # Old format, no SHA to validate
            print("STAMP=ok")
            print("STAMP_SKIPPED=already_stamped")
            print(f"LANDED_SHA={landed_sha}")
            _push_stamp(project, branch)
            _record_worklog(branch, project, landed_sha)
            return 0
    else:
        prev_history = []

    result = git(project, "checkout", branch)
    if result.returncode != 0:
        print("ERROR=CHECKOUT_FAILED")
        print(f"ERROR_DETAIL=cannot checkout {branch}: {result.stderr.strip()}")
        return 1

    issue_match = re.match(r"issue-(\d+)", branch)
    issue_ref = f"  Refs #{issue_match.group(1)}" if issue_match else ""
    reason = "stale_sha_not_on_base" if prev_history else "initial"
    message = _build_stamp_message(landed_sha, base_branch, issue_ref,
                                    prev_history, reason)

    if prev_history:
        result = git(project, "commit", "--allow-empty", "--amend", "-m", message)
    else:
        result = git(project, "commit", "--allow-empty", "-m", message)
    if result.returncode != 0:
        print("ERROR=STAMP_FAILED")
        print(f"ERROR_DETAIL={result.stderr.strip()}")
        return 1

    result = git(project, "checkout", base_branch)
    if result.returncode != 0:
        print(f"CHECKOUT_WARNING=could not return to {base_branch}", file=sys.stderr)

    print("STAMP=ok")
    print(f"LANDED_SHA={landed_sha}")
```

Also extract a `_push_stamp` and `_record_worklog` helper to avoid
duplication with the existing push/worklog code at the end of `cmd_stamp`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_land_branch.py::TestCmdStampRevalidation tests/test_land_branch.py::TestCmdStamp -v`
Expected: All PASS

- [ ] **Step 5: Run full test_land_branch.py for regressions**

Run: `python3 -m pytest tests/test_land_branch.py -v`
Expected: All PASS

- [ ] **Step 6: Commit**

```bash
git add work-end/land_branch.py tests/test_land_branch.py
git commit -m "feat: validate-and-re-stamp with history in cmd_stamp"
```

---

## Batch 3: History-aware verification in verify_slot_close.py

### Task 4: Make check_landing_sha history-aware

**Files:**
- Modify: `work-end/verify_slot_close.py:92-100` (`check_landing_sha`)
- Test: `tests/test_verify_slot_close.py`

**Interfaces:**
- Consumes: `_parse_stamp_history` from `land_flow.py`
- Produces: `check_landing_sha` returns enriched detail including stamp
  lineage when history is present

- [ ] **Step 1: Write failing tests for history-aware verification**

```python
class TestCheckLandingShaHistory:
    def test_reports_stamp_lineage(self, tmp_path: Path) -> None:
        """When stamp has history, verify reports the lineage."""
        project = _init_repo(tmp_path / "project")
        _git(project, "checkout", "-b", "feature")
        (project / "f.txt").write_text("work\n")
        _git(project, "add", "f.txt")
        _git(project, "commit", "-m", "feat: work")
        _git(project, "checkout", "main")
        _git(project, "merge", "--ff-only", "feature")
        sha = _git(project, "rev-parse", "main")

        _git(project, "checkout", "feature")
        msg = (f"chore: branch closed — landed as {sha} on main\n\n"
               f"stamp-history:\n"
               f"- 2026-09-01T00:00:00Z sha=deadbeef00000000 reason=initial\n"
               f"- 2026-09-06T00:00:00Z sha={sha} reason=stale_sha_not_on_base prev=deadbeef00000000\n")
        _git(project, "commit", "--allow-empty", "-m", msg)
        _git(project, "checkout", "main")

        result = verify_slot_close.check_landing_sha(str(project), "feature", "main")
        assert result["status"] == "pass"
        assert "re-stamped" in result.get("detail", "") or "lineage" in result.get("detail", "")

    def test_backward_compatible_no_history(self, tmp_path: Path) -> None:
        """Old stamps without history still work."""
        project = _init_repo(tmp_path / "project")
        _git(project, "checkout", "-b", "feature")
        (project / "f.txt").write_text("work\n")
        _git(project, "add", "f.txt")
        _git(project, "commit", "-m", "feat: work")
        _git(project, "checkout", "main")
        _git(project, "merge", "--ff-only", "feature")
        sha = _git(project, "rev-parse", "main")

        _git(project, "checkout", "feature")
        _git(project, "commit", "--allow-empty", "-m",
             f"chore: branch closed — landed as {sha} on main")
        _git(project, "checkout", "main")

        result = verify_slot_close.check_landing_sha(str(project), "feature", "main")
        assert result["status"] == "pass"
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_verify_slot_close.py::TestCheckLandingShaHistory -v`
Expected: First test FAILs (no lineage reporting yet), second should PASS
(backward compat already works)

- [ ] **Step 3: Update check_landing_sha in verify_slot_close.py**

```python
def check_landing_sha(project: str, branch: str, base: str = "main") -> dict:
    result = git(project, "log", "-1", "--format=%s", branch)
    if result.returncode != 0:
        return {"status": "fail", "detail": "branch not found"}
    msg = result.stdout.strip()
    sha_match = re.search(r"landed as ([0-9a-f]+)", msg)
    if not sha_match:
        return {"status": "warn", "detail": "no landing SHA in stamp (old format)"}
    sha = sha_match.group(1)
    verify = _verify_sha_on_ref(project, sha, base)

    # Enrich with stamp history if present
    body_result = git(project, "log", "-1", "--format=%b", branch)
    if body_result.returncode == 0 and "stamp-history:" in body_result.stdout:
        from land_flow import _parse_stamp_history
        history = _parse_stamp_history(body_result.stdout)
        if len(history) > 1:
            stamps = len(history)
            verify["detail"] = (verify.get("detail", "") +
                f" (re-stamped {stamps - 1}x, lineage: " +
                " → ".join(h["sha"][:8] for h in history) + ")")
    return verify
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_verify_slot_close.py::TestCheckLandingShaHistory tests/test_verify_slot_close.py::TestCheckLandingShaTreeFallback -v`
Expected: All PASS

- [ ] **Step 5: Run full verify test suite for regressions**

Run: `python3 -m pytest tests/test_verify_slot_close.py -v`
Expected: All PASS

- [ ] **Step 6: Commit**

```bash
git add work-end/verify_slot_close.py tests/test_verify_slot_close.py
git commit -m "feat: history-aware stamp verification with lineage reporting"
```

---

## Batch 4: Integration test — full squash-after-stamp scenario

### Task 5: End-to-end test proving the fix

**Files:**
- Test: `tests/test_land_flow.py`

**Interfaces:**
- Consumes: Everything from Tasks 1-4

- [ ] **Step 1: Write the integration test**

```python
class TestSquashAfterStampRecovery:
    def test_full_squash_invalidation_and_recovery(self, tmp_path):
        """Simulate the exact scenario: stamp, then squash rewrites main."""
        from land_flow import RepoDescriptor, Transport, land_batch

        repo = _init_repo(tmp_path / "project")
        branch = "issue-42-test"

        # Create feature branch with 2 commits
        subprocess.run(["git", "-C", str(repo), "checkout", "-b", branch], capture_output=True)
        (repo / "a.py").write_text("# a\n")
        subprocess.run(["git", "-C", str(repo), "add", "."], capture_output=True)
        subprocess.run(["git", "-C", str(repo), "commit", "-m", "feat: add a"], capture_output=True)
        (repo / "b.py").write_text("# b\n")
        subprocess.run(["git", "-C", str(repo), "add", "."], capture_output=True)
        subprocess.run(["git", "-C", str(repo), "commit", "-m", "feat: add b"], capture_output=True)

        # First land — stamps with SHA-A
        desc = RepoDescriptor(
            repo_path=repo, original_path=repo, push_target="origin",
            base_branch="main", is_workspace=False, transport=Transport.DIRECT,
        )
        progress1 = tmp_path / ".progress1"
        result1 = land_batch([desc], branch, progress1)
        assert result1.success
        sha_a = result1.repos[0].landed_sha

        # Simulate squash on main: reset, squash commits, force push
        subprocess.run(["git", "-C", str(repo), "checkout", "main"], capture_output=True)
        subprocess.run(["git", "-C", str(repo), "reset", "--soft", "HEAD~2"], capture_output=True)
        subprocess.run(["git", "-C", str(repo), "commit", "-m", "feat: add a+b (squashed)"], capture_output=True)
        subprocess.run(["git", "-C", str(repo), "push", "origin", "main", "--force-with-lease"], capture_output=True)
        sha_b = subprocess.run(
            ["git", "-C", str(repo), "rev-parse", "main"],
            capture_output=True, text=True,
        ).stdout.strip()

        assert sha_a != sha_b  # Squash created a new SHA

        # Second land attempt with fresh progress
        progress2 = tmp_path / ".progress2"
        result2 = land_batch([desc], branch, progress2)
        assert result2.success

        # Stamp should now have SHA-B (the post-squash SHA)
        log = subprocess.run(
            ["git", "-C", str(repo), "log", "-1", "--format=%s", branch],
            capture_output=True, text=True,
        )
        assert f"landed as {sha_b}" in log.stdout.strip()

        # History should show the re-stamp
        body = subprocess.run(
            ["git", "-C", str(repo), "log", "-1", "--format=%B", branch],
            capture_output=True, text=True,
        ).stdout
        assert "stamp-history:" in body
        assert sha_a in body  # old SHA in history

        # Verify should now pass
        sys.path.insert(0, str(Path(__file__).parent.parent / "work-end"))
        import verify_slot_close
        check = verify_slot_close.check_landing_sha(str(repo), branch, "main")
        assert check["status"] == "pass"
```

- [ ] **Step 2: Run the integration test**

Run: `python3 -m pytest tests/test_land_flow.py::TestSquashAfterStampRecovery -v`
Expected: PASS

- [ ] **Step 3: Run the full test suite to confirm no regressions**

Run: `python3 -m pytest tests/test_land_flow.py tests/test_land_branch.py tests/test_verify_slot_close.py -v`
Expected: All PASS

- [ ] **Step 4: Commit**

```bash
git add tests/test_land_flow.py
git commit -m "test: integration test for squash-after-stamp recovery"
```

---

## References

- `work-end/land_flow.py:546-597` — `_stamp_repo` (primary stamp function)
- `work-end/land_branch.py:147-238` — `cmd_stamp` (alternative stamp entry point)
- `work-end/verify_slot_close.py:77-100` — `_verify_sha_on_ref`, `check_landing_sha`
- `work-end/work_end_orchestrator.py:864-879` — step ordering (squash → land → stamp_pass)
- `tests/test_land_flow.py` — existing land flow tests
- `tests/test_land_branch.py:144-222` — existing stamp tests
- `tests/test_verify_slot_close.py:290-311` — existing tree-SHA fallback tests
- GitHub #335 — inescapable final-gate verify with tree-SHA fallback (related prior work)
