# Phase 4: Typed LLM Interface Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #451 — Phase 4: Typed LLM interface — JudgmentRequest/Response protocol
**Issue group:** #451

**Goal:** Replace the stringly-typed orchestrator↔LLM protocol with typed dataclasses, enums, and schema validation across all 3 orchestrators.

**Architecture:** New `project/judgment.py` module defines `StepType`, `Outcome`, `JudgmentRequest`, `JudgmentResponse`, and `DEFAULT_JUDGMENT_SCHEMA`. `run_loop` assembles `JudgmentRequest` from `StepDef` metadata + `action_context_fn` output. Response parsing constructs `JudgmentResponse` from CLI args and validates against the step's schema. `skip_step=X` unifies into `step_done=X outcome=skipped`.

**Tech Stack:** Python 3.14, dataclasses, JSON schema validation (jsonschema or manual), pytest

## Global Constraints

- All existing tests must continue passing — zero regressions
- The orchestrator stdout protocol stays KEY=VALUE (no JSON payloads on stdout)
- `action_context_fn` continues to return `dict` (not JudgmentRequest)
- Handler `.md` files are not modified
- Evidence CLI arg uses progress-file fallback for complex dicts

---

## Batch 1: Foundation — Types and StepType Enum

### Task 1: Create judgment.py with core types

**Files:**
- Create: `project/judgment.py`
- Test: `tests/test_judgment.py`

**Interfaces:**
- Produces: `StepType` enum (`MECHANICAL`, `JUDGMENT`, `LIFECYCLE`), `Outcome` enum (`COMPLETED`, `SKIPPED`, `BLOCKED`), `JudgmentRequest` frozen dataclass, `JudgmentResponse` frozen dataclass, `DEFAULT_JUDGMENT_SCHEMA` dict, `serialize_request(request) -> dict[str, str]`, `parse_response(args, step_def) -> JudgmentResponse`, `validate_response(response, schema) -> list[str]`

- [ ] **Step 1: Write failing tests for StepType and Outcome enums**

```python
# tests/test_judgment.py
import sys
from pathlib import Path

sys.path.insert(0, str(Path(__file__).parent.parent / "project"))

import pytest


class TestStepType:
    def test_mechanical_value(self):
        from judgment import StepType
        assert StepType.MECHANICAL.value == "mechanical"

    def test_judgment_value(self):
        from judgment import StepType
        assert StepType.JUDGMENT.value == "judgment"

    def test_lifecycle_value(self):
        from judgment import StepType
        assert StepType.LIFECYCLE.value == "lifecycle"

    def test_from_string(self):
        from judgment import StepType
        assert StepType("mechanical") == StepType.MECHANICAL

    def test_invalid_raises(self):
        from judgment import StepType
        with pytest.raises(ValueError):
            StepType("invalid")


class TestOutcome:
    def test_completed_value(self):
        from judgment import Outcome
        assert Outcome.COMPLETED.value == "completed"

    def test_skipped_value(self):
        from judgment import Outcome
        assert Outcome.SKIPPED.value == "skipped"

    def test_blocked_value(self):
        from judgment import Outcome
        assert Outcome.BLOCKED.value == "blocked"

    def test_from_string(self):
        from judgment import Outcome
        assert Outcome("skipped") == Outcome.SKIPPED
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest tests/test_judgment.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'judgment'`

- [ ] **Step 3: Write failing tests for JudgmentRequest and JudgmentResponse**

```python
class TestJudgmentRequest:
    def test_frozen_dataclass(self):
        from judgment import JudgmentRequest, DEFAULT_JUDGMENT_SCHEMA
        req = JudgmentRequest(
            step="code_review",
            handler="handlers/review.md",
            context={"DIFF_RANGE": "main..feature"},
            response_schema=DEFAULT_JUDGMENT_SCHEMA,
            skippable=True,
            timeout_hint="standard",
        )
        assert req.step == "code_review"
        assert req.handler == "handlers/review.md"
        with pytest.raises(AttributeError):
            req.step = "other"

    def test_none_handler(self):
        from judgment import JudgmentRequest, DEFAULT_JUDGMENT_SCHEMA
        req = JudgmentRequest(
            step="forage", handler=None,
            context={}, response_schema=DEFAULT_JUDGMENT_SCHEMA,
            skippable=True, timeout_hint="standard",
        )
        assert req.handler is None


class TestJudgmentResponse:
    def test_completed_response(self):
        from judgment import JudgmentResponse, Outcome
        resp = JudgmentResponse(
            step="code_review",
            outcome=Outcome.COMPLETED,
            produced=3,
            evidence={"findings": 3},
            block_reason=None,
        )
        assert resp.outcome == Outcome.COMPLETED
        assert resp.produced == 3

    def test_blocked_response(self):
        from judgment import JudgmentResponse, Outcome
        resp = JudgmentResponse(
            step="squash",
            outcome=Outcome.BLOCKED,
            produced=0,
            evidence={},
            block_reason="merge conflict",
        )
        assert resp.block_reason == "merge conflict"
```

- [ ] **Step 4: Write failing tests for serialization and parsing**

```python
class TestSerializeRequest:
    def test_round_trip(self):
        from judgment import JudgmentRequest, DEFAULT_JUDGMENT_SCHEMA, serialize_request
        req = JudgmentRequest(
            step="code_review",
            handler="handlers/review.md",
            context={"DIFF_RANGE": "main..feature"},
            response_schema=DEFAULT_JUDGMENT_SCHEMA,
            skippable=True,
            timeout_hint="standard",
        )
        result = serialize_request(req)
        assert result["ACTION"] == "code_review"
        assert result["HANDLER"] == "handlers/review.md"
        assert "DIFF_RANGE" in result.get("CONTEXT", result.get("DIFF_RANGE", ""))
        assert result["SKIPPABLE"] == "yes"
        assert result["TIMEOUT_HINT"] == "standard"
        assert "step_done=code_review" in result["COMPLETE_WITH"]
        assert "outcome=" in result["COMPLETE_WITH"]

    def test_none_handler_omitted(self):
        from judgment import JudgmentRequest, DEFAULT_JUDGMENT_SCHEMA, serialize_request
        req = JudgmentRequest(
            step="forage", handler=None,
            context={}, response_schema=DEFAULT_JUDGMENT_SCHEMA,
            skippable=True, timeout_hint="standard",
        )
        result = serialize_request(req)
        assert "HANDLER" not in result


class TestParseResponse:
    def test_parse_completed(self):
        from judgment import parse_response, Outcome
        args = {"step_done": "code_review", "produced": "3", "outcome": "completed"}
        resp = parse_response(args)
        assert resp.step == "code_review"
        assert resp.outcome == Outcome.COMPLETED
        assert resp.produced == 3

    def test_parse_defaults_outcome_completed(self):
        from judgment import parse_response, Outcome
        args = {"step_done": "code_review", "produced": "2"}
        resp = parse_response(args)
        assert resp.outcome == Outcome.COMPLETED

    def test_parse_skipped(self):
        from judgment import parse_response, Outcome
        args = {"step_done": "forage", "outcome": "skipped"}
        resp = parse_response(args)
        assert resp.outcome == Outcome.SKIPPED
        assert resp.produced == 0

    def test_parse_blocked(self):
        from judgment import parse_response, Outcome
        args = {"step_done": "squash", "outcome": "blocked",
                "block_reason": "merge conflict"}
        resp = parse_response(args)
        assert resp.outcome == Outcome.BLOCKED
        assert resp.block_reason == "merge conflict"

    def test_parse_evidence_json(self):
        from judgment import parse_response
        args = {"step_done": "review", "produced": "1",
                "evidence": '{"findings": 1}'}
        resp = parse_response(args)
        assert resp.evidence == {"findings": 1}

    def test_parse_invalid_outcome_raises(self):
        from judgment import parse_response
        args = {"step_done": "review", "outcome": "invalid"}
        with pytest.raises(ValueError):
            parse_response(args)


class TestValidateResponse:
    def test_valid_default_schema(self):
        from judgment import JudgmentResponse, Outcome, DEFAULT_JUDGMENT_SCHEMA, validate_response
        resp = JudgmentResponse("review", Outcome.COMPLETED, 0, {}, None)
        errors = validate_response(resp, DEFAULT_JUDGMENT_SCHEMA)
        assert errors == []

    def test_missing_step_fails(self):
        from judgment import validate_response, DEFAULT_JUDGMENT_SCHEMA
        resp_dict = {"outcome": "completed", "produced": 0}
        errors = validate_response_dict(resp_dict, DEFAULT_JUDGMENT_SCHEMA)
        assert len(errors) > 0
```

- [ ] **Step 5: Implement judgment.py**

```python
# project/judgment.py
import json
from dataclasses import dataclass, asdict
from enum import Enum


class StepType(Enum):
    MECHANICAL = "mechanical"
    JUDGMENT = "judgment"
    LIFECYCLE = "lifecycle"


class Outcome(Enum):
    COMPLETED = "completed"
    SKIPPED = "skipped"
    BLOCKED = "blocked"


@dataclass(frozen=True)
class JudgmentRequest:
    step: str
    handler: str | None
    context: dict
    response_schema: dict
    skippable: bool
    timeout_hint: str


@dataclass(frozen=True)
class JudgmentResponse:
    step: str
    outcome: Outcome
    produced: int
    evidence: dict
    block_reason: str | None


DEFAULT_JUDGMENT_SCHEMA = {
    "type": "object",
    "properties": {
        "step": {"type": "string"},
        "outcome": {"type": "string", "enum": ["completed", "skipped", "blocked"]},
        "produced": {"type": "integer", "minimum": 0},
        "evidence": {"type": "object"},
        "block_reason": {"type": ["string", "null"]},
    },
    "required": ["step", "outcome", "produced"],
}


def serialize_request(request: JudgmentRequest) -> dict[str, str]:
    result: dict[str, str] = {"ACTION": request.step}
    if request.handler:
        result["HANDLER"] = request.handler
    result.update(request.context)
    result["SCHEMA"] = json.dumps(request.response_schema)
    result["SKIPPABLE"] = "yes" if request.skippable else "no"
    result["TIMEOUT_HINT"] = request.timeout_hint
    result["COMPLETE_WITH"] = (
        f"step_done={request.step} outcome=completed produced=N"
    )
    return result


def parse_response(args: dict[str, str]) -> JudgmentResponse:
    step = args.get("step_done", "")
    outcome_str = args.get("outcome", "completed")
    outcome = Outcome(outcome_str)
    produced_raw = args.get("produced", "0")
    try:
        produced = int(produced_raw)
    except (ValueError, TypeError):
        produced = 0
    evidence_raw = args.get("evidence", "{}")
    try:
        evidence = json.loads(evidence_raw) if evidence_raw else {}
    except (json.JSONDecodeError, TypeError):
        evidence = {}
    block_reason = args.get("block_reason")
    return JudgmentResponse(step, outcome, produced, evidence, block_reason)


def validate_response(response: JudgmentResponse, schema: dict) -> list[str]:
    errors: list[str] = []
    resp_dict = {
        "step": response.step,
        "outcome": response.outcome.value,
        "produced": response.produced,
        "evidence": response.evidence,
        "block_reason": response.block_reason,
    }
    required = schema.get("required", [])
    for field in required:
        if field not in resp_dict or resp_dict[field] is None:
            errors.append(f"missing required field: {field}")
    props = schema.get("properties", {})
    if "outcome" in props and "enum" in props["outcome"]:
        if resp_dict.get("outcome") not in props["outcome"]["enum"]:
            errors.append(f"invalid outcome: {resp_dict.get('outcome')}")
    if "produced" in props:
        p = resp_dict.get("produced")
        if not isinstance(p, int):
            errors.append(f"produced must be integer, got {type(p).__name__}")
        elif "minimum" in props["produced"] and p < props["produced"]["minimum"]:
            errors.append(f"produced must be >= {props['produced']['minimum']}")
    return errors


def validate_response_dict(resp_dict: dict, schema: dict) -> list[str]:
    errors: list[str] = []
    required = schema.get("required", [])
    for field in required:
        if field not in resp_dict:
            errors.append(f"missing required field: {field}")
    return errors
```

- [ ] **Step 6: Run all tests to verify they pass**

Run: `python3 -m pytest tests/test_judgment.py -v`
Expected: all PASS

- [ ] **Step 7: Commit**

```bash
git add project/judgment.py tests/test_judgment.py
git commit -m "feat(#451): add judgment.py — StepType, Outcome, JudgmentRequest/Response, serialization Refs #451"
```

### Task 2: Migrate StepDef to StepType enum

**Files:**
- Modify: `project/shared_steps.py` — change `step_type: str` to `step_type: StepType`, add handler/schema/skippable/timeout_hint fields
- Modify: `project/orchestrator_engine.py` — update `step.step_type == "mechanical"` comparisons to use enum
- Modify: `work-end/work_end_orchestrator.py` — update all StepDef definitions and string comparisons
- Modify: `handover/wrap_orchestrator.py` — update StepDef definitions and string comparisons
- Modify: `project/work.py` — update StepDef definitions and string comparisons
- Test: `tests/test_judgment.py` (add StepType coverage test)

**Interfaces:**
- Consumes: `StepType` from `judgment.py` (Task 1)
- Produces: Updated `StepDef` with `step_type: StepType` and new optional fields

- [ ] **Step 1: Write failing test — all StepDefs use valid StepType**

```python
# Add to tests/test_judgment.py
class TestStepTypeConverted:
    def test_all_work_end_steps_use_enum(self):
        sys.path.insert(0, str(Path(__file__).parent.parent / "work-end"))
        from work_end_orchestrator import STEPS
        from judgment import StepType
        for step in STEPS:
            assert isinstance(step.step_type, StepType), (
                f"Step '{step.name}' uses {type(step.step_type).__name__}, not StepType"
            )

    def test_all_wrap_steps_use_enum(self):
        sys.path.insert(0, str(Path(__file__).parent.parent / "handover"))
        from wrap_orchestrator import WRAP_STEPS
        from judgment import StepType
        for step in WRAP_STEPS:
            assert isinstance(step.step_type, StepType), (
                f"Step '{step.name}' uses {type(step.step_type).__name__}, not StepType"
            )

    def test_all_work_pipeline_steps_use_enum(self):
        from work import PIPELINES
        from judgment import StepType
        for cmd, steps in PIPELINES.items():
            for step in steps:
                assert isinstance(step.step_type, StepType), (
                    f"Step '{step.name}' in pipeline '{cmd}' uses {type(step.step_type).__name__}"
                )
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_judgment.py::TestStepTypeConverted -v`
Expected: FAIL — steps still use string

- [ ] **Step 3: Update shared_steps.py — StepDef fields**

Change `step_type: str` to `step_type: StepType` and add new fields. Import `StepType` from `judgment`. Update the `make_*` factory functions.

```python
# In shared_steps.py, add import:
from judgment import StepType

# Change StepDef:
@dataclass
class StepDef:
    name: str
    phase: str
    step_type: StepType              # was: str
    # ... existing fields ...
    handler: str | None = None
    response_schema: dict | None = None
    skippable: bool = True
    timeout_hint: str = "standard"
    # ... rest unchanged ...
```

- [ ] **Step 4: Update orchestrator_engine.py — string comparisons to enum**

Replace all `step.step_type == "mechanical"` with `step.step_type == StepType.MECHANICAL`, etc. There are 3 comparisons in `run_loop`:
- Line ~309: `if step.step_type == "mechanical":`
- Line ~393: `if step.step_type == "judgment":`  
- Line ~410: `if step.step_type == "lifecycle":`

- [ ] **Step 5: Convert all work-end step definitions**

In `work_end_orchestrator.py`, update every `StepDef(...)` call — change the third positional arg from `"mechanical"` to `StepType.MECHANICAL`, `"judgment"` to `StepType.JUDGMENT`, `"lifecycle"` to `StepType.LIFECYCLE`. Also update `MECHANICAL_STEPS` set comprehension and any other string comparisons.

Import: `from judgment import StepType`

- [ ] **Step 6: Convert all wrap step definitions**

In `wrap_orchestrator.py`, update every `StepDef(...)` and the `WRAP_MECHANICAL_STEPS` set comprehension. Import `StepType`.

- [ ] **Step 7: Convert all work.py step definitions**

In `work.py`, update every `StepDef(...)` and the mechanical steps set comprehension. Import `StepType`.

- [ ] **Step 8: Run all tests**

Run: `python3 -m pytest tests/test_judgment.py tests/test_stale_attempt_reset.py tests/test_work_end_orchestrator.py tests/test_wrap_orchestrator.py -v`
Expected: All pass (including the StepType coverage tests)

- [ ] **Step 9: Commit**

```bash
git add project/shared_steps.py project/orchestrator_engine.py work-end/work_end_orchestrator.py handover/wrap_orchestrator.py project/work.py tests/test_judgment.py
git commit -m "feat(#451): migrate StepDef.step_type to StepType enum across all orchestrators Refs #451"
```

---

## Batch 2: Protocol — Request Emission and Response Parsing

### Task 3: Update run_loop to emit JudgmentRequest

**Files:**
- Modify: `project/orchestrator_engine.py` — `_yield_judgment` takes `JudgmentRequest`, `run_loop` assembles request
- Modify: `project/shared_steps.py` — `yield_judgment` takes `JudgmentRequest`
- Test: `tests/test_judgment.py` (add integration test)

**Interfaces:**
- Consumes: `JudgmentRequest`, `serialize_request` from `judgment.py` (Task 1); `StepDef` with new fields (Task 2)
- Produces: `_yield_judgment(request, workspace, progress) -> dict[str, str]` — returns serialized request with attempt tracking

- [ ] **Step 1: Write failing test — _yield_judgment accepts JudgmentRequest**

```python
class TestYieldJudgmentRequest:
    def test_emits_handler_in_output(self, tmp_path):
        from orchestrator_engine import _yield_judgment
        from judgment import JudgmentRequest, DEFAULT_JUDGMENT_SCHEMA
        from progress import Progress

        req = JudgmentRequest(
            step="code_review",
            handler="handlers/review.md",
            context={"DIFF_RANGE": "main..feature"},
            response_schema=DEFAULT_JUDGMENT_SCHEMA,
            skippable=True,
            timeout_hint="standard",
        )
        progress = Progress()
        result = _yield_judgment(req, tmp_path, progress)
        assert result["ACTION"] == "code_review"
        assert result["HANDLER"] == "handlers/review.md"
        assert result["SKIPPABLE"] == "yes"
        assert "outcome=" in result["COMPLETE_WITH"]

    def test_omits_handler_when_none(self, tmp_path):
        from orchestrator_engine import _yield_judgment
        from judgment import JudgmentRequest, DEFAULT_JUDGMENT_SCHEMA
        from progress import Progress

        req = JudgmentRequest(
            step="forage", handler=None,
            context={}, response_schema=DEFAULT_JUDGMENT_SCHEMA,
            skippable=True, timeout_hint="standard",
        )
        progress = Progress()
        result = _yield_judgment(req, tmp_path, progress)
        assert "HANDLER" not in result
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_judgment.py::TestYieldJudgmentRequest -v`
Expected: FAIL — `_yield_judgment` still takes `(step_name, workspace, progress, context)`

- [ ] **Step 3: Update _yield_judgment in orchestrator_engine.py**

Change signature from `_yield_judgment(step_name, workspace, progress, context)` to `_yield_judgment(request, workspace, progress)`. Use `serialize_request` for output. Keep attempt counter logic (including the #454 stale counter reset).

```python
def _yield_judgment(request: JudgmentRequest, workspace: Path,
                    progress) -> dict[str, str]:
    stored = progress.get_attempt(request.step)
    if stored >= MAX_JUDGMENT_RETRIES:
        progress.reset_attempt(request.step)
        update_close_progress(workspace, f"{request.step}_attempt", "0")
        stored = 0
    attempt = stored + 1

    if attempt > MAX_JUDGMENT_RETRIES:
        update_close_progress(workspace, "last_yielded", request.step)
        return {
            "ACTION": "user_input",
            "CONTEXT": "step_failed",
            "STEP": request.step,
            "ATTEMPTS": str(MAX_JUDGMENT_RETRIES),
            "REASON": f"Validation failed after {MAX_JUDGMENT_RETRIES} attempts",
        }

    update_close_progress(workspace, f"{request.step}_attempt", str(attempt))
    update_close_progress(workspace, "last_yielded", request.step)
    return serialize_request(request)
```

- [ ] **Step 4: Update run_loop judgment dispatch to assemble JudgmentRequest**

In `run_loop`, replace the judgment step section:

```python
if step.step_type == StepType.JUDGMENT:
    # ... per_repo_judgment handling unchanged ...
    context = step.action_context_fn(ctx) if step.action_context_fn else {}
    request = JudgmentRequest(
        step=step.name,
        handler=step.handler,
        context=context,
        response_schema=step.response_schema or DEFAULT_JUDGMENT_SCHEMA,
        skippable=step.skippable,
        timeout_hint=step.timeout_hint,
    )
    if step.name in user_input_steps:
        request = JudgmentRequest(
            step=request.step, handler=request.handler,
            context={**request.context, "USER_INPUT": "yes"},
            response_schema=request.response_schema,
            skippable=request.skippable, timeout_hint=request.timeout_hint,
        )
    return _yield_judgment(request, ctx.workspace, ctx.progress)
```

- [ ] **Step 5: Update shared_steps.py yield_judgment similarly**

- [ ] **Step 6: Remove _yield_user_input (merge into _yield_judgment)**

Delete `_yield_user_input` from both `orchestrator_engine.py` and `shared_steps.py`. The `user_input_steps` set still controls the `USER_INPUT=yes` context flag.

- [ ] **Step 7: Run all tests**

Run: `python3 -m pytest tests/ -k "judgment or orchestrator or work_end or wrap" -v`
Expected: All pass

- [ ] **Step 8: Commit**

```bash
git add project/orchestrator_engine.py project/shared_steps.py tests/test_judgment.py
git commit -m "feat(#451): _yield_judgment takes JudgmentRequest, merge _yield_user_input Refs #451"
```

### Task 4: Response parsing and schema validation

**Files:**
- Modify: `work-end/work_end_orchestrator.py` — parse JudgmentResponse from args, validate
- Modify: `project/work.py` — same response parsing
- Modify: `handover/wrap_orchestrator.py` — same response parsing
- Modify: `project/orchestrator_engine.py` — update `apply_step_done` to accept `JudgmentResponse`
- Test: `tests/test_judgment.py` (add response parsing integration tests)

**Interfaces:**
- Consumes: `parse_response`, `validate_response` from `judgment.py` (Task 1)
- Produces: Response parsing in all 3 orchestrators validates against step schemas before applying step_done

- [ ] **Step 1: Write failing test — schema violation returns specific error**

```python
class TestResponseValidationIntegration:
    def test_schema_violation_returns_error(self, tmp_path):
        from work_end_orchestrator import run_orchestrator
        from close_progress import update_close_progress
        update_close_progress(tmp_path, "review", "done")
        update_close_progress(tmp_path, "sweep_config", "done")
        update_close_progress(tmp_path, "sweep_selected", "forage")
        (tmp_path / "project").mkdir(exist_ok=True)
        result = run_orchestrator({
            "workspace": str(tmp_path),
            "project": str(tmp_path / "project"),
            "branch": "issue-test",
            "base_branch": "main",
            "meta_state": "closing:review",
            "step_done": "forage",
            "outcome": "invalid_value",
        })
        assert result.get("ERROR") == "schema_violation"
```

- [ ] **Step 2: Run test to verify it fails**

Expected: FAIL — no schema_violation error handling yet

- [ ] **Step 3: Update work_end_orchestrator.py step_done handling**

In `run_orchestrator`, after the existing `step_done` check, add response parsing:

```python
if args.get("step_done"):
    step_name = args["step_done"]
    # ... existing sweep_config rejection ...
    # ... existing verify_fn check ...

    # NEW: parse and validate response
    from judgment import parse_response, validate_response, Outcome
    try:
        response = parse_response(args)
    except ValueError as e:
        return {"ACTION": "error", "ERROR": "schema_violation",
                "STEP": step_name, "DETAIL": str(e)}

    step_def = next((s for s in STEPS if s.name == step_name), None)
    if step_def:
        schema = step_def.response_schema or DEFAULT_JUDGMENT_SCHEMA
        errors = validate_response(response, schema)
        if errors:
            return {"ACTION": "error", "ERROR": "schema_violation",
                    "STEP": step_name, "DETAIL": "; ".join(errors)}

    # Handle outcome
    if response.outcome == Outcome.SKIPPED:
        err = validate_skip(workspace, step_name)
        if err:
            return err
    elif response.outcome == Outcome.BLOCKED:
        return {"ACTION": "error", "ERROR": "step_blocked",
                "STEP": step_name, "REASON": response.block_reason or "no reason"}
    else:
        err = apply_step_done(workspace, step_name, str(response.produced),
                              mechanical_steps=MECHANICAL_STEPS)
        if err:
            return err
```

- [ ] **Step 4: Apply same pattern to work.py and wrap_orchestrator.py**

- [ ] **Step 5: Remove separate skip_step handling**

Remove `args.get("skip_step")` blocks from all 3 orchestrators. `skip_step=X` is now handled as `step_done=X outcome=skipped` in the response parsing path. Keep `validate_skip` logic (last_yielded check) inside the outcome=skipped branch.

- [ ] **Step 6: Run all tests**

Run: `python3 -m pytest tests/ -k "judgment or orchestrator or work_end or wrap or stale" -v`
Expected: All pass

- [ ] **Step 7: Commit**

```bash
git add work-end/work_end_orchestrator.py project/work.py handover/wrap_orchestrator.py project/orchestrator_engine.py tests/test_judgment.py
git commit -m "feat(#451): parse JudgmentResponse from args, validate against schema, unify skip_step Refs #451"
```

---

## Batch 3: Wiring — Step Metadata and SKILL.md Updates

### Task 5: Add handler and schema metadata to step definitions

**Files:**
- Modify: `work-end/work_end_orchestrator.py` — add `handler=`, `response_schema=`, `skippable=`, `timeout_hint=` to judgment step definitions
- Modify: `handover/wrap_orchestrator.py` — same
- Modify: `project/work.py` — same
- Test: `tests/test_judgment.py` (add metadata coverage tests)

**Interfaces:**
- Consumes: `StepDef` with new fields (Task 2)
- Produces: All 39 judgment steps have appropriate handler, schema, skippable, and timeout_hint values

- [ ] **Step 1: Write failing test — handler paths match existing files**

```python
class TestHandlerMetadata:
    def test_work_end_handlers_exist(self):
        from work_end_orchestrator import STEPS
        from judgment import StepType
        handler_root = Path(__file__).parent.parent / "work-end"
        for step in STEPS:
            if step.step_type == StepType.JUDGMENT and step.handler:
                path = handler_root / step.handler
                assert path.exists(), f"Handler for '{step.name}' not found: {path}"

    def test_work_handlers_exist(self):
        from work import PIPELINES
        from judgment import StepType
        handler_root = Path(__file__).parent.parent / "work"
        for cmd, steps in PIPELINES.items():
            for step in steps:
                if step.step_type == StepType.JUDGMENT and step.handler:
                    path = handler_root / step.handler
                    assert path.exists(), f"Handler for '{step.name}' not found: {path}"
```

- [ ] **Step 2: Run test to verify it fails**

Expected: FAIL — no handler metadata set on steps yet

- [ ] **Step 3: Add handler metadata to work-end judgment steps**

For each judgment step in `STEPS` that has a handler file, add `handler=`:
- `code_review` → `handler="handlers/review.md"`
- `branch_audit_*` → `handler="handlers/review.md"` (same handler)
- `loose_ends` → `handler="handlers/review.md"`
- `forcing_function` → `handler="handlers/review.md"`
- `sweep_config` → `handler="handlers/sweep_config.md"`
- `queue_decision` → `handler="handlers/queue-decision.md"`
- `trajectory` → no handler (skill-based)
- `squash` → no handler (skill-based)
- `forage`, `protocol`, `update_claude_md`, etc. → no handler (skill-based)
- `archive_slot` → `handler="handlers/archive_slot.md"`
- `arc42_scan`, `session_rename`, `garden_feedback`, `notes` → `handler="handlers/user_input.md"`

Add custom `response_schema=` for steps with verify_fn:
- `code_review`, `branch_audit_*`, `loose_ends` → produced-required schema
- `forcing_function` → findings-zero schema
- `squash` → verified-required schema

- [ ] **Step 4: Add handler metadata to work.py judgment steps**

- `resolve_issue` → `handler="handlers/resolve-issue.md"`
- `branch_name` → `handler="handlers/branch-name.md"`
- `platform_coherence` → `handler="handlers/platform-coherence.md"`
- `check_protocols` → `handler="handlers/check-protocols.md"`
- `brainstorm_offer` → `handler="handlers/brainstorm-offer.md"`
- `load_context` → `handler="handlers/load-context.md"`
- `stack_pick` → `handler="handlers/stack-pick.md"`
- `present_candidates` → `handler="handlers/present-candidates.md"`
- `deferred_check` → `handler="handlers/deferred-check.md"`
- `context_resume` → no handler
- `recover_plan_offer` → `handler="handlers/recover-plan-offer.md"`

- [ ] **Step 5: Add handler metadata to wrap_orchestrator.py judgment steps**

- `epic_hygiene` → no handler
- `wrap_sweep_config` → no handler (uses sweep config logic)
- `journal_entry` → no handler
- `arc42_scan_wrap` → no handler
- `handoff_write` → no handler

- [ ] **Step 6: Run tests**

Run: `python3 -m pytest tests/test_judgment.py -v`
Expected: All pass

- [ ] **Step 7: Commit**

```bash
git add work-end/work_end_orchestrator.py handover/wrap_orchestrator.py project/work.py tests/test_judgment.py
git commit -m "feat(#451): add handler/schema/skippable/timeout metadata to all judgment steps Refs #451"
```

### Task 6: Update SKILL.md dispatch tables and response protocol

**Files:**
- Modify: `work/SKILL.md` — update orchestrator loop protocol
- Modify: `work-end/SKILL.md` — update orchestrator loop protocol (if exists as separate file)

**Interfaces:**
- Consumes: New request format with HANDLER=, SKIPPABLE=, COMPLETE_WITH including outcome=
- Produces: Updated SKILL.md documentation matching the new protocol

- [ ] **Step 1: Update work/SKILL.md — add response protocol section**

The existing dispatch table stays as a fallback reference. Add a note that `HANDLER=` in the orchestrator output takes precedence. Update the response section:

```markdown
After handling a judgment step, call work.py again with:
  step_done=<step_name> outcome=completed produced=N [evidence=<json>]
  # or: step_done=<step_name> outcome=skipped
  # or: step_done=<step_name> outcome=blocked block_reason="..."
```

- [ ] **Step 2: Update work-end handler dispatch documentation similarly**

- [ ] **Step 3: Commit**

```bash
git add work/SKILL.md
git commit -m "docs(#451): update SKILL.md dispatch tables for JudgmentRequest protocol Refs #451"
```

---

## Batch 4: Cleanup and Validation

### Task 7: Final validation and cleanup

**Files:**
- Modify: `project/orchestrator_engine.py` — remove dead code (old _yield_user_input if not already removed)
- Test: Run full test suite

**Interfaces:**
- Consumes: All previous tasks
- Produces: Clean codebase with no dead code and all tests passing

- [ ] **Step 1: Search for dead code**

Grep for any remaining references to the old protocol:
- `_yield_user_input` (should be removed)
- `skip_step` arg handling (should be removed)
- `step_type == "mechanical"` string comparisons (should all be enum)

- [ ] **Step 2: Remove any remaining dead code**

- [ ] **Step 3: Run full test suite**

Run: `python3 -m pytest tests/ -v`
Expected: All pass, zero regressions

- [ ] **Step 4: Run commit-tier validators**

Run: `python3 scripts/validate_all.py --tier commit`
Expected: Pass

- [ ] **Step 5: Final commit**

```bash
git add -A
git commit -m "chore(#451): cleanup dead code from pre-typed protocol Refs #451"
```

## References

- specs/issue-451-typed-llm-interface/2026-10-14-typed-llm-interface-design.md — design spec
- project/orchestrator_engine.py:241-434 — run_loop implementation
- project/shared_steps.py:33-51 — StepDef dataclass
- project/judgment.py — new module (created in Task 1)
- work-end/work_end_orchestrator.py:1132-1379 — STEPS list
- handover/wrap_orchestrator.py:97-120 — WRAP_STEPS list
- project/work.py:143-355 — work pipeline steps
- docs/protocols/externalised-scripts-require-tests.md — test requirement
- Issue #451, parent #443
