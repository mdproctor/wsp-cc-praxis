# Final Review Mode Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #66 — Phase 4: Final code review — replace code-review skill and superpowers review
**Issue group:** #66

**Goal:** Add `--mode final-review` with depth-scaled adversarial review to the design-review engine, complementing (not replacing) the per-commit code-review skill.

**Architecture:** Final-review operates on branch diffs (not specs). It adds a `--depth light|standard|deep` flag with auto-detection from diff stats. Light depth is non-adversarial (reviewer-only, no implementor). Standard and deep run the full adversarial loop. The review.py loop gains mode-conditional commit/verify logic for code-modifying modes.

**Tech Stack:** Python 3.11+, pytest, existing design-review infrastructure (review.py, setup.py, prompts.py, tracker.py, parser.py)

## Global Constraints

- All new Python scripts in skill directories must ship with tests (protocol: externalised-scripts-require-tests)
- Generator functions in setup.py follow the existing zero-arg callable pattern — depth is passed via a module-level setter, not function arguments
- Constraint blocks use `_assemble_constraints()` for numbered assembly
- Prompt builders follow the existing signature conventions (reviewer: 7 params + depth, implementor: 5 params + depth)
- `MODE_DEFAULTS["final-review"]` already exists — do not modify it (it serves as the fallback when `--depth` is not used)
- Existing tests must continue passing after every task

---

### Task 1: Depth presets and auto-detection

**Files:**
- Modify: `design-review/review.py:1075-1115` (add DEPTH_PRESETS, --depth flag, auto-detection)
- Test: `tests/test_adr_review.py` (new test classes)

**Interfaces:**
- Produces: `DEPTH_PRESETS` dict, `_auto_detect_depth(source_dirs: list[str], diff_base: str | None) -> str`, `--depth` CLI flag
- Produces: `.depth` file written to workspace on setup, loaded on resume

- [ ] **Step 1: Write failing tests for depth presets**

```python
class TestDepthPresets:
    def test_depth_presets_exist(self) -> None:
        from design_review.review import DEPTH_PRESETS
        assert "light" in DEPTH_PRESETS
        assert "standard" in DEPTH_PRESETS
        assert "deep" in DEPTH_PRESETS

    def test_light_is_single_round(self) -> None:
        from design_review.review import DEPTH_PRESETS
        assert DEPTH_PRESETS["light"]["max_rounds"] == 1
        assert DEPTH_PRESETS["light"]["min_rounds"] == 1

    def test_standard_preset(self) -> None:
        from design_review.review import DEPTH_PRESETS
        assert DEPTH_PRESETS["standard"]["max_rounds"] == 3
        assert DEPTH_PRESETS["standard"]["min_rounds"] == 2
        assert DEPTH_PRESETS["standard"]["budget_per_session"] == 5.0

    def test_deep_preset(self) -> None:
        from design_review.review import DEPTH_PRESETS
        assert DEPTH_PRESETS["deep"]["max_rounds"] == 5
        assert DEPTH_PRESETS["deep"]["min_rounds"] == 3
        assert DEPTH_PRESETS["deep"]["budget_per_session"] == 8.0
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_adr_review.py::TestDepthPresets -v`
Expected: FAIL with `ImportError` — `DEPTH_PRESETS` does not exist

- [ ] **Step 3: Implement DEPTH_PRESETS in review.py**

Add after `MODE_DEFAULTS` (after line 1082):

```python
DEPTH_PRESETS: Final = {
    "light":    {"max_rounds": 1, "min_rounds": 1, "budget_per_session": 1.5},
    "standard": {"max_rounds": 3, "min_rounds": 2, "budget_per_session": 5.0},
    "deep":     {"max_rounds": 5, "min_rounds": 3, "budget_per_session": 8.0},
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_adr_review.py::TestDepthPresets -v`
Expected: PASS

- [ ] **Step 5: Write failing tests for auto-detection**

```python
class TestAutoDetectDepth:
    def test_small_diff_is_light(self, tmp_path: Path) -> None:
        from design_review.review import _auto_detect_depth
        stats = "2 files changed, 30 insertions(+), 5 deletions(-)"
        result = _auto_detect_depth(stats, new_file_count=0)
        assert result == "light"

    def test_medium_diff_is_standard(self, tmp_path: Path) -> None:
        from design_review.review import _auto_detect_depth
        stats = "6 files changed, 120 insertions(+), 30 deletions(-)"
        result = _auto_detect_depth(stats, new_file_count=0)
        assert result == "standard"

    def test_large_diff_is_deep(self, tmp_path: Path) -> None:
        from design_review.review import _auto_detect_depth
        stats = "15 files changed, 400 insertions(+), 100 deletions(-)"
        result = _auto_detect_depth(stats, new_file_count=0)
        assert result == "deep"

    def test_new_files_double_count(self, tmp_path: Path) -> None:
        from design_review.review import _auto_detect_depth
        # 40 lines + 2 new files = 80 effective lines → standard, not light
        stats = "3 files changed, 30 insertions(+), 10 deletions(-)"
        result = _auto_detect_depth(stats, new_file_count=2)
        assert result == "standard"

    def test_many_files_triggers_deep(self, tmp_path: Path) -> None:
        from design_review.review import _auto_detect_depth
        stats = "12 files changed, 100 insertions(+), 20 deletions(-)"
        result = _auto_detect_depth(stats, new_file_count=0)
        assert result == "deep"
```

- [ ] **Step 6: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_adr_review.py::TestAutoDetectDepth -v`
Expected: FAIL with `ImportError`

- [ ] **Step 7: Implement _auto_detect_depth in review.py**

Add after `_detect_last_round` (after line 838):

```python
def _auto_detect_depth(diff_stat_summary: str, new_file_count: int = 0) -> str:
    """Detect review depth from diff stat summary line.

    Args:
        diff_stat_summary: The summary line from git diff --stat (e.g.,
            "6 files changed, 120 insertions(+), 30 deletions(-)").
        new_file_count: Number of new files in the diff. Each counts double
            toward the line threshold.
    """
    import re
    files_m = re.search(r"(\d+) files? changed", diff_stat_summary)
    ins_m = re.search(r"(\d+) insertions?", diff_stat_summary)
    del_m = re.search(r"(\d+) deletions?", diff_stat_summary)

    file_count = int(files_m.group(1)) if files_m else 0
    insertions = int(ins_m.group(1)) if ins_m else 0
    deletions = int(del_m.group(1)) if del_m else 0
    total_lines = insertions + deletions + (new_file_count * 20)

    if file_count > 10 or total_lines > 300:
        return "deep"
    if file_count > 3 or total_lines > 50:
        return "standard"
    return "light"
```

- [ ] **Step 8: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_adr_review.py::TestAutoDetectDepth -v`
Expected: PASS

- [ ] **Step 9: Write failing tests for --depth CLI flag**

```python
class TestDepthFlag:
    def test_depth_flag_parsed(self) -> None:
        from design_review.review import parse_args
        args = parse_args(["--spec", "s.md", "--title", "t",
                           "--source-dirs", "/src",
                           "--mode", "final-review",
                           "--depth", "deep"])
        assert args.depth == "deep"

    def test_depth_default_none(self) -> None:
        from design_review.review import parse_args
        args = parse_args(["--spec", "s.md", "--title", "t",
                           "--source-dirs", "/src"])
        assert args.depth is None

    def test_depth_choices(self) -> None:
        import pytest
        from design_review.review import parse_args
        with pytest.raises(SystemExit):
            parse_args(["--spec", "s.md", "--title", "t",
                         "--source-dirs", "/src",
                         "--mode", "final-review",
                         "--depth", "extreme"])
```

- [ ] **Step 10: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_adr_review.py::TestDepthFlag -v`
Expected: FAIL — `args` has no attribute `depth`

- [ ] **Step 11: Add --depth flag to parse_args in review.py**

Add after the `--diff-base` argument (after line 1106):

```python
parser.add_argument("--depth", choices=("light", "standard", "deep"),
                    default=None,
                    help="Review depth for final-review mode (default: auto-detect)")
```

- [ ] **Step 12: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_adr_review.py::TestDepthFlag -v`
Expected: PASS

- [ ] **Step 13: Write failing tests for depth file persistence**

```python
class TestDepthPersistence:
    def test_depth_file_written(self, tmp_path: Path) -> None:
        ws = tmp_path / "ws"
        ws.mkdir()
        depth_file = ws / ".depth"
        depth_file.write_text("standard")
        assert depth_file.read_text() == "standard"

    def test_depth_file_loaded_on_resume(self, tmp_path: Path) -> None:
        from design_review.review import _load_depth
        ws = tmp_path / "ws"
        ws.mkdir()
        (ws / ".depth").write_text("deep")
        assert _load_depth(ws) == "deep"

    def test_depth_file_missing_returns_none(self, tmp_path: Path) -> None:
        from design_review.review import _load_depth
        ws = tmp_path / "ws"
        ws.mkdir()
        assert _load_depth(ws) is None
```

- [ ] **Step 14: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_adr_review.py::TestDepthPersistence -v`
Expected: FAIL — `_load_depth` does not exist

- [ ] **Step 15: Implement _load_depth in review.py**

Add after `_auto_detect_depth`:

```python
def _load_depth(ws: Path) -> str | None:
    """Load persisted depth from workspace, or None if absent."""
    depth_file = ws / ".depth"
    if depth_file.exists():
        value = depth_file.read_text().strip()
        if value in DEPTH_PRESETS:
            return value
    return None
```

- [ ] **Step 16: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_adr_review.py::TestDepthPersistence -v`
Expected: PASS

- [ ] **Step 17: Run full test suite**

Run: `python3 -m pytest tests/test_adr_review.py -v`
Expected: All existing tests PASS plus new tests PASS

- [ ] **Step 18: Commit**

```bash
git add design-review/review.py tests/test_adr_review.py
git commit -m "feat: #66 add DEPTH_PRESETS, --depth flag, auto-detection, and persistence"
```

---

### Task 2: Final-review mode generators (setup.py)

**Files:**
- Modify: `design-review/setup.py:151-780` (add constraint constants, generator functions, mode registration)
- Modify: `design-review/setup.py:154-171` (update _generate_agent_claude_mds to pass depth)
- Test: `tests/test_adr_review.py` (new test class)

**Interfaces:**
- Consumes: `DEPTH_PRESETS` from Task 1 (for depth awareness)
- Produces: `_MODE_GENERATORS["final-review"]` with `reviewer` and `implementor` generators
- Produces: `_FINAL_REVIEW_APPROACH_REVIEWER`, `_FINAL_REVIEW_APPROACH_IMPLEMENTOR`, `_FINAL_REVIEW_STARTING_POINTS`, `_FINAL_REVIEW_MAIN_CODE_FOCUS`, `_FINAL_REVIEW_TEST_CODE_FOCUS`, `_CROSS_MODULE_IMPACT`

- [ ] **Step 1: Write failing tests for mode generator registration**

```python
class TestFinalReviewTemplates:
    def test_mode_generators_registered(self) -> None:
        from design_review.setup import _MODE_GENERATORS
        assert "final-review" in _MODE_GENERATORS
        assert "reviewer" in _MODE_GENERATORS["final-review"]
        assert "implementor" in _MODE_GENERATORS["final-review"]

    def test_reviewer_template_has_final_review_constraints(self) -> None:
        from design_review.setup import _MODE_GENERATORS
        content = _MODE_GENERATORS["final-review"]["reviewer"]()
        assert "production" in content.lower() or "readiness" in content.lower()
        assert "branch diff" in content.lower() or "branch changes" in content.lower()

    def test_implementor_template_has_final_review_constraints(self) -> None:
        from design_review.setup import _MODE_GENERATORS
        content = _MODE_GENERATORS["final-review"]["implementor"]()
        assert "fix" in content.lower() or "code" in content.lower()

    def test_reviewer_template_has_main_code_focus(self) -> None:
        from design_review.setup import _MODE_GENERATORS
        content = _MODE_GENERATORS["final-review"]["reviewer"]()
        assert "correctness" in content.lower()
        assert "security" in content.lower()
        assert "error handling" in content.lower()

    def test_reviewer_template_has_test_code_focus(self) -> None:
        from design_review.setup import _MODE_GENERATORS
        content = _MODE_GENERATORS["final-review"]["reviewer"]()
        assert "coverage" in content.lower()
        assert "assertion" in content.lower()

    def test_reviewer_template_differs_from_spec_review(self) -> None:
        from design_review.setup import _default_reviewer_md, _MODE_GENERATORS
        spec_content = _default_reviewer_md()
        final_content = _MODE_GENERATORS["final-review"]["reviewer"]()
        assert spec_content != final_content

    def test_implementor_fixes_code_not_spec(self) -> None:
        from design_review.setup import _MODE_GENERATORS
        content = _MODE_GENERATORS["final-review"]["implementor"]()
        assert "source" in content.lower() or "code" in content.lower()
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_adr_review.py::TestFinalReviewTemplates -v`
Expected: FAIL — `"final-review"` not in `_MODE_GENERATORS`

- [ ] **Step 3: Add constraint constants to setup.py**

Add after `_CODE_REVIEW_APPROACH_IMPLEMENTOR` (after line 526):

```python
_FINAL_REVIEW_APPROACH_REVIEWER = """\
You are reviewing the **branch diff** for production readiness. This is NOT a \
spec compliance check — there may be no spec. Your job is to find issues in the \
actual code changes that would cause bugs, security holes, or maintenance problems \
in production.

Compute the diff: `git diff <base>..HEAD` in each source directory.
Read every changed file. Apply production readiness criteria to each change."""

_FINAL_REVIEW_APPROACH_IMPLEMENTOR = """\
You are defending the implementation. When the reviewer raises a valid issue, \
fix the code directly in the source directories. When the reviewer raises an \
invalid concern, defend with evidence — show the code, explain the reasoning, \
reference tests that cover the case.

Do NOT capitulate on correct code. Do NOT update a spec — fix the source code."""

_FINAL_REVIEW_STARTING_POINTS = """\
### Starting points — main code and test code

**Main code review areas:**
- Architecture — does the change fit the existing module structure?
- Correctness — edge cases, off-by-one, null/empty handling
- Error handling — are failures caught, logged, and recovered from?
- Performance — unnecessary allocations, O(n²) where O(n) suffices
- Concurrency — race conditions, shared mutable state, deadlocks
- Security — injection, auth bypass, data exposure
- Naming and structure — clear, consistent, follows project conventions
- Layer compliance — does the change respect module boundaries?

**Test code review areas:**
- Coverage completeness — are the important paths tested?
- Assertion quality — do assertions check the right things, not just "no exception"?
- Missing scenarios — what edge cases, error paths, or boundary conditions are untested?
- Test isolation — do tests depend on shared state or execution order?
- Fixture patterns — are test fixtures clear, minimal, and reusable?"""

_FINAL_REVIEW_MAIN_CODE_FOCUS = """\
Focus on the main (non-test) code first. These are the areas that affect \
production behavior and are hardest to fix after release."""

_FINAL_REVIEW_TEST_CODE_FOCUS = """\
After reviewing main code, shift attention to test code. Poor tests create \
false confidence — a passing test suite that misses the important cases is \
worse than no tests at all."""

_CROSS_MODULE_IMPACT = """\
### Cross-module impact (deep review only)

Check for cross-cutting effects of the branch changes:
- Changes to shared interfaces/types used by other modules
- Behavioral changes observable from callers in other packages
- Transaction boundary changes that affect coordinating services
- Configuration changes that alter deployment or runtime behavior
- Test isolation — whether changes could cause flaky tests in unrelated modules"""
```

- [ ] **Step 4: Add generator functions and registration**

Add after the constraint constants (before the existing code-review generators):

```python
def _final_review_reviewer_md() -> str:
    items = [
        _CORE_APPROACH_REVIEWER,
        _FINAL_REVIEW_APPROACH_REVIEWER,
        _FINAL_REVIEW_STARTING_POINTS,
        _FINAL_REVIEW_MAIN_CODE_FOCUS,
        _FINAL_REVIEW_TEST_CODE_FOCUS,
        _INTELLIJ_OPEN,
        _NO_SKILL_TOOL,
    ]
    return _assemble_constraints(items)


def _final_review_implementor_md() -> str:
    items = [
        _CORE_APPROACH_IMPLEMENTOR,
        _FINAL_REVIEW_APPROACH_IMPLEMENTOR,
        _INTELLIJ_OPEN,
        _NO_SKILL_TOOL,
    ]
    return _assemble_constraints(items)


_MODE_GENERATORS["final-review"] = {
    "reviewer": _final_review_reviewer_md,
    "implementor": _final_review_implementor_md,
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_adr_review.py::TestFinalReviewTemplates -v`
Expected: PASS

- [ ] **Step 6: Run full test suite**

Run: `python3 -m pytest tests/test_adr_review.py -v`
Expected: All tests PASS

- [ ] **Step 7: Commit**

```bash
git add design-review/setup.py tests/test_adr_review.py
git commit -m "feat: #66 add final-review constraint constants and mode generators"
```

---

### Task 3: Final-review prompt builders (prompts.py)

**Files:**
- Modify: `design-review/prompts.py:6-151` (add dispatch branches and builder functions)
- Test: `tests/test_adr_review.py` (new test class)

**Interfaces:**
- Consumes: existing prompt builder signature pattern from code-review mode
- Produces: `_build_final_review_reviewer_prompt()`, `_build_final_review_implementor_prompt()`
- Produces: dispatch branches in `build_reviewer_prompt()` and `build_implementor_prompt()`

- [ ] **Step 1: Write failing tests for prompt dispatch**

```python
class TestFinalReviewPrompts:
    def test_reviewer_prompt_is_production_focused(self) -> None:
        from design_review.prompts import build_reviewer_prompt
        prompt = build_reviewer_prompt(
            round_num=1, focus_items=[], handover_path=None,
            source_dirs=["/src"], workspace_root="/ws",
            spec_path="", mode="final-review",
        )
        assert "production" in prompt.lower() or "branch diff" in prompt.lower()

    def test_reviewer_prompt_round2_has_evidence_requirement(self) -> None:
        from design_review.prompts import build_reviewer_prompt
        prompt = build_reviewer_prompt(
            round_num=2, focus_items=["R1-01: Bug found"],
            handover_path=None,
            source_dirs=["/src"], workspace_root="/ws",
            spec_path="", mode="final-review",
        )
        assert "R1-01" in prompt

    def test_implementor_prompt_is_code_focused(self) -> None:
        from design_review.prompts import build_implementor_prompt
        prompt = build_implementor_prompt(
            round_num=1, focus_items=["R1-01: Bug found"],
            source_dirs=["/src"], workspace_root="/ws",
            spec_path="", mode="final-review",
        )
        assert "fix" in prompt.lower() or "code" in prompt.lower()

    def test_reviewer_prompt_light_depth_is_focused(self) -> None:
        from design_review.prompts import build_reviewer_prompt
        prompt = build_reviewer_prompt(
            round_num=1, focus_items=[], handover_path=None,
            source_dirs=["/src"], workspace_root="/ws",
            spec_path="", mode="final-review", depth="light",
        )
        assert "quick" in prompt.lower() or "sanity" in prompt.lower()

    def test_reviewer_prompt_deep_depth_has_cross_module(self) -> None:
        from design_review.prompts import build_reviewer_prompt
        prompt = build_reviewer_prompt(
            round_num=1, focus_items=[], handover_path=None,
            source_dirs=["/src"], workspace_root="/ws",
            spec_path="", mode="final-review", depth="deep",
        )
        assert "cross-module" in prompt.lower() or "cross module" in prompt.lower()

    def test_reviewer_prompt_convergence_override(self) -> None:
        from design_review.prompts import build_reviewer_prompt
        prompt = build_reviewer_prompt(
            round_num=2, focus_items=["R1-01: Bug"],
            handover_path=None,
            convergence_override_ids=["R1-01"],
            source_dirs=["/src"], workspace_root="/ws",
            spec_path="", mode="final-review",
        )
        assert "R1-01" in prompt
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_adr_review.py::TestFinalReviewPrompts -v`
Expected: FAIL — prompt builders don't handle `final-review` mode or `depth` param

- [ ] **Step 3: Add depth parameter to build_reviewer_prompt and build_implementor_prompt**

In `prompts.py`, add `depth: str | None = None` to both function signatures:

`build_reviewer_prompt` (line 6) — add after `mode`:
```python
def build_reviewer_prompt(
    round_num: int,
    focus_items: list[str],
    handover_path: str | None,
    convergence_override_ids: list[str] | None = None,
    source_dirs: list[str] | None = None,
    workspace_root: str = "",
    spec_path: str = "",
    mode: str = "spec-review",
    depth: str | None = None,
) -> str:
```

`build_implementor_prompt` (line 104) — add after `mode`:
```python
def build_implementor_prompt(
    round_num: int,
    focus_items: list[str],
    source_dirs: list[str] | None = None,
    workspace_root: str = "",
    spec_path: str = "",
    mode: str = "spec-review",
    depth: str | None = None,
) -> str:
```

- [ ] **Step 4: Add dispatch branches for final-review**

In `build_reviewer_prompt`, add after the code-review dispatch (after line 25):
```python
    if mode == "final-review":
        return _build_final_review_reviewer_prompt(
            round_num, focus_items, handover_path,
            convergence_override_ids, source_dirs,
            workspace_root, spec_path, depth)
```

In `build_implementor_prompt`, add after the code-review dispatch (after line 119):
```python
    if mode == "final-review":
        return _build_final_review_implementor_prompt(
            round_num, focus_items, source_dirs,
            workspace_root, spec_path, depth)
```

- [ ] **Step 5: Implement _build_final_review_reviewer_prompt**

Add after `_build_code_review_implementor_prompt` (after line 439):

```python
def _build_final_review_reviewer_prompt(
    round_num: int,
    focus_items: list[str],
    handover_path: str | None,
    convergence_override_ids: list[str] | None = None,
    source_dirs: list[str] | None = None,
    workspace_root: str = "",
    spec_path: str = "",
    depth: str | None = None,
) -> str:
    depth = depth or "standard"
    parts: list[str] = []

    if round_num == 1:
        if depth == "light":
            parts.append(
                "Quick review of the branch diff. Focus on correctness risks "
                "and security issues only. Flag items that would cause bugs or "
                "vulnerabilities in production. This is a single-pass sanity check."
            )
        else:
            parts.append(
                "Review the branch diff for production readiness. "
                "Compute `git diff <base>..HEAD` in each source directory. "
                "Read every changed file and apply production readiness criteria."
            )
            parts.append(
                "\n**Main code:** architecture, correctness, edge cases, error "
                "handling, performance, concurrency, security, naming, structure, "
                "layer compliance."
            )
            parts.append(
                "\n**Test code:** coverage completeness, assertion quality, "
                "missing scenarios, test isolation."
            )
            if depth == "deep":
                parts.append(
                    "\n**Cross-module impact:** shared interface changes, "
                    "behavioral changes visible to other modules, transaction "
                    "boundary changes, configuration changes, test isolation "
                    "across modules."
                )
        if source_dirs:
            parts.append(f"\nSource directories: {', '.join(source_dirs)}")
    else:
        parts.append(f"Round {round_num} review.")
        if focus_items:
            parts.append("\nOpen items from previous rounds:\n")
            for item in focus_items:
                parts.append(f"- {item}")
            parts.append(
                "\nFor each open item: verify the implementor's fix by reading "
                "the actual code. Mark as RESOLVED (fix confirmed), "
                "ACCEPTED (rejection is valid), or STILL OPEN (not fixed)."
            )
        parts.append(
            "\nAlso look for new issues not previously raised."
        )

    if convergence_override_ids:
        ids = ", ".join(convergence_override_ids)
        parts.append(
            f"\n**CONVERGENCE OVERRIDE:** Items {ids} were marked resolved "
            "without sufficient evidence. You MUST provide concrete evidence "
            "for each — quote the code or explain why the concern no longer applies."
        )

    if handover_path:
        parts.append(f"\nPrevious session handover: {handover_path}")

    return "\n".join(parts)
```

- [ ] **Step 6: Implement _build_final_review_implementor_prompt**

Add after the reviewer prompt:

```python
def _build_final_review_implementor_prompt(
    round_num: int,
    focus_items: list[str],
    source_dirs: list[str] | None = None,
    workspace_root: str = "",
    spec_path: str = "",
    depth: str | None = None,
) -> str:
    parts: list[str] = []

    parts.append(
        "The reviewer has raised issues with the branch code. For each item:"
    )
    parts.append(
        "\n- **FIXED:** Fix the code in the source directories. Show what you "
        "changed. Do NOT update a spec — fix the actual source code."
    )
    parts.append(
        "- **REJECTED:** Defend the implementation with evidence. Quote the "
        "code, reference tests, explain the design rationale."
    )

    if focus_items:
        parts.append("\n## Items to address\n")
        for item in focus_items:
            parts.append(f"- {item}")

    if source_dirs:
        parts.append(f"\nSource directories: {', '.join(source_dirs)}")

    return "\n".join(parts)
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_adr_review.py::TestFinalReviewPrompts -v`
Expected: PASS

- [ ] **Step 8: Run full test suite**

Run: `python3 -m pytest tests/test_adr_review.py -v`
Expected: All tests PASS

- [ ] **Step 9: Commit**

```bash
git add design-review/prompts.py tests/test_adr_review.py
git commit -m "feat: #66 add final-review prompt builders with depth-scaled content"
```

---

### Task 4: Review loop changes — light depth skip and code-mode commit/verify

**Files:**
- Modify: `design-review/review.py:29-73` (setup_review gains depth param)
- Modify: `design-review/review.py:93-193` (resume loads .depth)
- Modify: `design-review/review.py:263-558` (loop: light skip, mode-conditional commit/verify)
- Modify: `design-review/review.py:811-838` (_detect_last_round gains depth param)
- Modify: `design-review/review.py:942-947` (add _get_source_diff helper)
- Modify: `design-review/review.py:1108-1114` (depth preset overrides MODE_DEFAULTS)
- Modify: `design-review/setup.py:29-73` (setup_review gains depth, writes .depth)
- Test: `tests/test_adr_review.py` (new test classes)

**Interfaces:**
- Consumes: `DEPTH_PRESETS`, `_auto_detect_depth()`, `_load_depth()` from Task 1
- Consumes: prompt builders with `depth` param from Task 3
- Produces: modified review loop with light-skip and code-mode commit/verify

- [ ] **Step 1: Write failing tests for _detect_last_round with depth**

```python
class TestDetectLastRoundWithDepth:
    def test_light_depth_reviewer_only_is_complete(self, tmp_path: Path) -> None:
        from design_review.review import _detect_last_round
        resp = tmp_path / "responses"
        resp.mkdir()
        (resp / "reviewer-1.md").write_text("review findings")
        # No implementor-1.md — for light depth this is complete
        last, reviewer_only = _detect_last_round(tmp_path, depth="light")
        assert last == 1
        assert reviewer_only is False  # complete, not partial

    def test_standard_depth_reviewer_only_is_partial(self, tmp_path: Path) -> None:
        from design_review.review import _detect_last_round
        resp = tmp_path / "responses"
        resp.mkdir()
        (resp / "reviewer-1.md").write_text("review findings")
        last, reviewer_only = _detect_last_round(tmp_path, depth="standard")
        assert last == 1
        assert reviewer_only is True  # partial — implementor needed

    def test_none_depth_preserves_existing_behavior(self, tmp_path: Path) -> None:
        from design_review.review import _detect_last_round
        resp = tmp_path / "responses"
        resp.mkdir()
        (resp / "reviewer-1.md").write_text("review findings")
        last, reviewer_only = _detect_last_round(tmp_path, depth=None)
        assert last == 1
        assert reviewer_only is True  # backward compatible
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_adr_review.py::TestDetectLastRoundWithDepth -v`
Expected: FAIL — `_detect_last_round` doesn't accept `depth`

- [ ] **Step 3: Update _detect_last_round to accept depth parameter**

Modify the function signature at line 811:

```python
def _detect_last_round(ws: Path, depth: str | None = None) -> tuple[int, bool]:
```

Modify the return logic at lines 833-838:

```python
    if not reviewer_rounds:
        return 0, False
    max_reviewer = max(reviewer_rounds)
    if max_reviewer in implementor_rounds:
        return max_reviewer, False
    if depth == "light":
        return max_reviewer, False  # complete — light is reviewer-only
    return max_reviewer, True  # partial — implementor needed
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_adr_review.py::TestDetectLastRoundWithDepth -v`
Expected: PASS

- [ ] **Step 5: Write failing tests for code-mode helpers**

```python
class TestCodeModeHelpers:
    def test_get_source_diff_combines_repos(self, tmp_path: Path) -> None:
        from design_review.review import _get_source_diff
        # Set up two git repos with changes
        for name in ("repo1", "repo2"):
            d = tmp_path / name
            d.mkdir()
            subprocess.run(["git", "init"], cwd=d, capture_output=True)
            subprocess.run(["git", "config", "user.email", "test@test.com"], cwd=d, capture_output=True)
            subprocess.run(["git", "config", "user.name", "Test"], cwd=d, capture_output=True)
            (d / "file.py").write_text("initial")
            subprocess.run(["git", "add", "."], cwd=d, capture_output=True)
            subprocess.run(["git", "commit", "-m", "init"], cwd=d, capture_output=True)
            (d / "file.py").write_text("modified")
            subprocess.run(["git", "add", "."], cwd=d, capture_output=True)
            subprocess.run(["git", "commit", "-m", "change"], cwd=d, capture_output=True)

        diff = _get_source_diff([str(tmp_path / "repo1"), str(tmp_path / "repo2")])
        assert "file.py" in diff
        assert diff.count("diff --git") == 2  # one per repo

    def test_verify_code_changed_with_diff(self) -> None:
        from design_review.review import verify_code_changed
        result = verify_code_changed("diff --git a/file.py b/file.py\n+new line")
        assert result.section_changed is True

    def test_verify_code_changed_empty(self) -> None:
        from design_review.review import verify_code_changed
        result = verify_code_changed("")
        assert result.section_changed is False
```

- [ ] **Step 6: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_adr_review.py::TestCodeModeHelpers -v`
Expected: FAIL — functions don't exist

- [ ] **Step 7: Implement _get_source_diff and verify_code_changed**

Add after `_get_git_diff` (after line 947):

```python
def _get_source_diff(source_dirs: list[str]) -> str:
    """Get combined diff across all source directories."""
    diffs: list[str] = []
    for sd in source_dirs:
        result = subprocess.run(
            ["git", "diff", "HEAD~1"], cwd=sd, capture_output=True, text=True,
        )
        if result.stdout.strip():
            diffs.append(result.stdout)
    return "\n".join(diffs)


class _VerifyResult:
    def __init__(self, section_changed: bool) -> None:
        self.section_changed = section_changed


def verify_code_changed(diff: str) -> _VerifyResult:
    """Check whether any code was modified."""
    return _VerifyResult(section_changed=bool(diff.strip()))
```

- [ ] **Step 8: Run tests to verify they pass**

Run: `python3 -m pytest tests/test_adr_review.py::TestCodeModeHelpers -v`
Expected: PASS

- [ ] **Step 9: Update setup_review to accept and persist depth**

Modify `setup_review` signature in setup.py (line 29) to add `depth`:

```python
def setup_review(
    spec_path: Path,
    title: str,
    source_dirs: list[str],
    adr_root: Path | None = None,
    issue: str | None = None,
    mode: str = "spec-review",
    arch_files: list[str] | None = None,
    diff_base: str | None = None,
    depth: str | None = None,
) -> Path:
```

Add after the `.diff-base` write (after line 62):

```python
    if depth:
        (ws / ".depth").write_text(depth)
```

- [ ] **Step 10: Update main() to wire depth through the review loop**

This is the integration step. In `main()` in review.py:

1. After mode defaults resolution (line 1114), add depth resolution:
```python
    # Depth resolution for final-review
    if args.mode == "final-review":
        if args.depth:
            resolved_depth = args.depth
        elif args.workspace:
            resolved_depth = _load_depth(ws) or _auto_detect_depth(...)
        else:
            # Auto-detect from diff stats — requires source dirs and diff base
            resolved_depth = "standard"  # fallback until workspace exists
        if resolved_depth in DEPTH_PRESETS:
            preset = DEPTH_PRESETS[resolved_depth]
            args.max_rounds = preset["max_rounds"]
            args.min_rounds = preset["min_rounds"]
            args.budget_per_session = preset["budget_per_session"]
    else:
        resolved_depth = None
        if args.depth:
            _log(f"WARNING: --depth is only supported for final-review mode, ignored for {args.mode}")
```

2. Pass `depth=resolved_depth` to `setup_review()` call (line 161-169).

3. In resume section (after line 127), load depth:
```python
    depth_from_file = _load_depth(ws)
```

4. Pass `depth=resolved_depth` to `_detect_last_round()` call (line 129).

5. In the review loop, pass `depth=resolved_depth` to prompt builders (lines 280-290 and 400-410).

6. In the review loop, add light depth skip before implementor spawn (before line 416):
```python
        if resolved_depth == "light":
            _log("  Light depth — skipping implementor (reviewer-only mode)")
            # Record reviewer findings as OPEN and terminate
            _git_commit(ws, [f"responses/reviewer-{round_num}.md"],
                        f"round {round_num}: reviewer findings (light)")
            break
```

7. In the post-implementor commit section (lines 484-493), add mode-conditional logic:
```python
        if mode in ("final-review", "code-review"):
            for sd in args.source_dirs:
                subprocess.run(["git", "add", "-A"], cwd=sd, capture_output=True)
                subprocess.run(["git", "commit", "-m",
                    f"review: code fixes — {mode} round {round_num}",
                    "--allow-empty"], cwd=sd, capture_output=True)
        else:
            # Existing spec-commit logic
            spec_dir = Path(spec_path).parent
            spec_name = Path(spec_path).name
            subprocess.run(["git", "add", spec_name], cwd=spec_dir, capture_output=True)
            subprocess.run(["git", "commit", "-m",
                f"docs: spec revised — review round {round_num}",
                "--allow-empty"], cwd=spec_dir, capture_output=True)
```

8. In the verification section (lines 496-525), add mode-conditional logic:
```python
        if mode in ("final-review", "code-review"):
            diff = _get_source_diff(args.source_dirs)
            vr = verify_code_changed(diff)
        else:
            diff = _get_git_diff(spec_path)
            vr = verify_against_diff(diff, resp.section_ref)
```

- [ ] **Step 11: Handle --spec ignored for final-review**

In `main()`, after workspace setup but before the review loop, add spec-path handling for final-review:

```python
    if args.mode == "final-review":
        if spec_path:
            _log(f"WARNING: --spec is ignored for final-review mode")
        spec_path = ""  # final-review operates on branch diff, not a spec
```

This ensures the loop's spec-dependent code paths (commit, verify) correctly skip spec logic when `spec_path` is empty.

- [ ] **Step 12: Run full test suite**

Run: `python3 -m pytest tests/test_adr_review.py -v`
Expected: All tests PASS (existing + new)

- [ ] **Step 13: Commit**

```bash
git add design-review/review.py design-review/setup.py tests/test_adr_review.py
git commit -m "feat: #66 wire depth through review loop — light skip, code-mode commit/verify"
```

---

### Task 5: SKILL.md and workflow integration updates

**Files:**
- Modify: `design-review/SKILL.md` (Phase 4 active, --depth flag)
- Modify: `work-end/SKILL.md` (Step 3c conditional gate)
- Modify: `subagent-driven-development/SKILL.md` (switch to final-review)
- Modify: `requesting-code-review/SKILL.md` (deprecation notice)
- Modify: `code-review/SKILL.md` (Complements section)

**Interfaces:**
- Consumes: all Python changes from Tasks 1-4
- Produces: updated skill documentation across 5 SKILL.md files

- [ ] **Step 1: Update design-review/SKILL.md — Phase 4 active**

In the phase checklist (Step 0.5), change Phase 4 from "(coming soon)" to active:

```markdown
[ ] 4  Final review   Production-readiness check (1-5 rounds, depth-scaled)
```

Add `--depth` to the optional flags table:

```markdown
| "light review" / "quick check" | `--depth light` |
| "deep review" / "thorough" | `--depth deep` |
```

Update the "What this skill does NOT do" section to remove the Phase 4 "coming soon" reference.

- [ ] **Step 2: Update work-end/SKILL.md — Step 3c conditional gate**

Replace the existing Step 3c code-review invocation with:

```markdown
#### Step 3c — Code review (mandatory — HARD GATE)

Classify the branch diff:

1. Get diff stats: `git diff <base>..HEAD --stat`
2. Check for structural changes:
   - New files added → structural
   - Files deleted → structural
   - Files renamed → structural
   - New classes/interfaces → structural
   - Method signature changes → structural
   - Config files changed (pom.xml, application.properties) → structural
   - Only method body / import changes → body-only

**If structural:**
> "Branch diff is structural (N files, M lines, K new files).
>  Recommended: final-review --depth {auto} (~${cost}).
>  Run final-review? [y]es / [c]ode-review instead / [s]kip review"
- y → invoke `design-review --mode final-review`
- c → invoke `code-review` (checklist)
- s → skip (user takes responsibility)

**If body-only:**
> "Branch diff is body-only (N files, M lines). Running code-review checklist.
>  Override with final-review? (y/n)"
- y → invoke `design-review --mode final-review --depth light`
- n → invoke `code-review` (checklist, as today)
```

Update the Skill Chaining section to reference `design-review --mode final-review`.

- [ ] **Step 3: Update subagent-driven-development/SKILL.md**

Replace the `requesting-code-review` reference (around lines 315-316) with:

```markdown
**Final whole-branch review:** invoke `design-review --mode final-review --depth standard`
as a background subprocess. Read `tracker.md` from the review workspace for results.
```

Update the Skill Chaining section (around line 366) to reference `design-review` instead of `requesting-code-review`.

- [ ] **Step 4: Add deprecation notice to requesting-code-review/SKILL.md**

Add at the top of the SKILL.md, after the frontmatter:

```markdown
> **Deprecated:** This skill is superseded by `design-review --mode final-review`.
> Use `--mode final-review` for branch-level adversarial code review.
> This skill remains for backward compatibility but will not receive updates.
```

- [ ] **Step 5: Update code-review/SKILL.md Complements section**

Add to the Complements section:

```markdown
- `design-review --mode final-review` — branch-level adversarial review.
  Use code-review for per-commit checklist review; use final-review for
  pre-merge production readiness checks on structural changes.
```

- [ ] **Step 6: Run skill validation**

```bash
python3 scripts/validate_all.py --tier commit
```
Expected: All validators pass

- [ ] **Step 7: Commit**

```bash
git add design-review/SKILL.md work-end/SKILL.md subagent-driven-development/SKILL.md requesting-code-review/SKILL.md code-review/SKILL.md
git commit -m "docs: #66 update SKILL.md files — Phase 4 active, work-end gate, deprecation notices"
```

- [ ] **Step 8: Sync skills**

```bash
python3 scripts/claude-skill sync-local --all -y
```

---

### Task 6: Issue #66 text corrections and final verification

**Files:**
- No code changes — GitHub issue update and verification only

**Interfaces:**
- Consumes: all changes from Tasks 1-5
- Produces: updated issue #66 body, passing test suite

- [ ] **Step 1: Run full test suite**

```bash
python3 -m pytest tests/test_adr_review.py -v
```
Expected: All tests PASS (existing 98 + new ~30)

- [ ] **Step 2: Run commit-tier validators**

```bash
python3 scripts/validate_all.py --tier commit
```
Expected: All validators pass

- [ ] **Step 3: Update issue #66 body**

Per the spec's "Issue #66 text corrections needed before close" section, update the issue body:

1. Remove "split into main code (4a) and test code (4b) sub-phases" — resolved as prompt-level structure only
2. Remove "Replaces: `code-review` skill" — code-review stays; only `requesting-code-review` is replaced
3. Remove "Update `git-commit` workflow to reference final-review" — git-commit is unchanged

```bash
gh issue edit 66 --repo Hortora/soredium --body "$(cat <<'EOF'
**Parent:** #63

Add `--mode final-review` to the ADR tool with depth-scaled review (light/standard/deep). Reviewer brief structures both main code and test code concerns at the prompt level.

**Changes:**
- `review.py`: `--depth` flag, `DEPTH_PRESETS`, auto-detection, light-skip, code-mode commit/verify
- `setup.py`: `_MODE_GENERATORS["final-review"]`, constraint constants, generator functions
- `prompts.py`: `_build_final_review_*_prompt()`, dispatch branches, depth-scaled content
- `SKILL.md`: Phase 4 active, `--depth` flag documented
- `work-end/SKILL.md`: Step 3c conditional gate (structural → final-review, body-only → code-review)

**Deprecates:** `requesting-code-review` (branch-level role absorbed by final-review)
**Complements:** `code-review` (stays for per-commit checklist review)
EOF
)"
```

- [ ] **Step 4: Final smoke test — verify --mode final-review is accepted**

```bash
python3 design-review/review.py --help | grep -A2 "depth"
```
Expected: `--depth {light,standard,deep}` appears in help output

- [ ] **Step 5: Commit any remaining changes**

If the issue update or validation surfaced anything:

```bash
git status
# Stage and commit any fixes
```
