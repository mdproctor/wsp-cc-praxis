# Decision Review + Ordered Dimensions Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #198 — feat: ordered dimensional reviews + decision validation phase
**Issue group:** #198

**Goal:** Build a unified design pipeline with decision capture, adversarial
decision review, enhanced approach exploration (steelman/DA/debate), ordered
dimensional reviews with cascading findings, and per-project documentation
maps (SOURCES.md).

**Architecture:** Extends the existing design-review Python framework
(review.py, prompts.py, setup.py, tracker.py) with a new `decision` review
mode. Brainstorming SKILL.md gains a pipeline state machine, decision capture
loop, and multi-agent debate orchestration. design-review SKILL.md gains
ordered dimension sequencing. SOURCES.md is a per-project documentation map
inlined by CLAUDE.md.

**Tech Stack:** Python 3 (design-review scripts), Markdown (SKILL.md files),
Claude Code Agent tool (multi-agent debate)

## Global Constraints

- All Python changes in `design-review/` use the existing bootstrap pattern
  (`adversarial_design_review` module registration in review.py)
- SKILL.md changes are markdown — no Python tests, but must follow soredium
  skill conventions (frontmatter, CSO, section naming)
- Tests use pytest, located in `design-review/tests/`
- Issue IDs in decision-review use standard `R{n}-{nn}` format (no parser changes)
- Pipeline state file uses key-value format with `format_version: 1`
- `@SOURCES.md` include directive in CLAUDE.md must tolerate missing file

---

### Task 1: SOURCES.md + ctx.py Detection

**Files:**
- Create: `SOURCES.md` (soredium project root — example/template)
- Modify: `project/ctx.py:236-300` — add HAS_SOURCES and SOURCES_PATH detection
- Test: `tests/test_ctx_sources.py`

**Interfaces:**
- Consumes: nothing
- Produces: `HAS_SOURCES=yes|no` and `SOURCES_PATH=<path>` in ctx.py output.
  All downstream tasks read these values.

- [ ] **Step 1: Write failing test for ctx.py SOURCES.md detection**

```python
# tests/test_ctx_sources.py
import subprocess
import tempfile
from pathlib import Path

def test_ctx_detects_sources_md(tmp_path):
    (tmp_path / "SOURCES.md").write_text("# Documentation Sources\n| Type | Path |\n")
    (tmp_path / "CLAUDE.md").write_text("## Project Type\n**Type:** generic\n**Stage:** pre-release\n")
    (tmp_path / ".git").mkdir()
    result = subprocess.run(
        ["python3", str(Path(__file__).parent.parent / "project" / "ctx.py")],
        cwd=tmp_path, capture_output=True, text=True,
    )
    assert "HAS_SOURCES=yes" in result.stdout
    assert f"SOURCES_PATH={tmp_path}/SOURCES.md" in result.stdout

def test_ctx_no_sources_md(tmp_path):
    (tmp_path / "CLAUDE.md").write_text("## Project Type\n**Type:** generic\n**Stage:** pre-release\n")
    (tmp_path / ".git").mkdir()
    result = subprocess.run(
        ["python3", str(Path(__file__).parent.parent / "project" / "ctx.py")],
        cwd=tmp_path, capture_output=True, text=True,
    )
    assert "HAS_SOURCES=no" in result.stdout
    assert "SOURCES_PATH=" in result.stdout
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest tests/test_ctx_sources.py -v`
Expected: FAIL — HAS_SOURCES not in ctx.py output

- [ ] **Step 3: Add SOURCES.md detection to ctx.py**

Add after the `has_protocols_dir` block (around line 252):

```python
sources_path = ""
if (Path(project) / "SOURCES.md").exists():
    has_sources = "yes"
    sources_path = str(Path(project) / "SOURCES.md")
else:
    has_sources = "no"
```

Add to the print block (after HAS_PROTOCOLS_DIR):

```python
print(f"HAS_SOURCES={has_sources}")
print(f"SOURCES_PATH={sources_path}")
```

- [ ] **Step 4: Run test to verify it passes**

Run: `python3 -m pytest tests/test_ctx_sources.py -v`
Expected: PASS

- [ ] **Step 5: Create SOURCES.md for soredium**

```markdown
# Documentation Sources

| Type | Path | Notes |
|------|------|-------|
| Specs | docs/specs/ | Design specifications |
| ADRs | docs/adr/ | Architecture decision records |
| Protocols | docs/protocols/INDEX.md | Standing conventions |
| Project types | docs/PROJECT-TYPES.md | Type definitions and routing |
| Quality | QUALITY.md | Validation framework |
| Blog | hortora.github.io/_posts/ | Published diary entries |
| Garden | ~/.hortora/garden/ | Knowledge garden |
```

- [ ] **Step 6: Add @SOURCES.md include to soredium CLAUDE.md**

Add after the `## Project Identity` section:

```markdown
@SOURCES.md
```

Verify the `@` directive tolerates the file's presence. If `@` include
does not work as expected (syntax error, not loaded), use inline content
with a path reference instead.

- [ ] **Step 7: Commit**

```bash
git add SOURCES.md project/ctx.py tests/test_ctx_sources.py CLAUDE.md
git commit -m "feat(#198): SOURCES.md documentation map and ctx.py detection"
```

---

### Task 2: Decision Capture in Brainstorming

**Files:**
- Modify: `brainstorming/SKILL.md` — add decision capture loop, pipeline
  state management, decisions.md writing

**Interfaces:**
- Consumes: `HAS_SOURCES` from ctx.py (Task 1)
- Produces: `decisions.md` format specification and pipeline.state management
  that Tasks 3, 4, 5 rely on

This is a SKILL.md change — no Python tests. The changes are instructions
to Claude, not executable code.

- [ ] **Step 1: Add pipeline state management to brainstorming**

Add a new section after "## Branch Context" in brainstorming/SKILL.md:

```markdown
## Pipeline State

Brainstorming manages a pipeline state file at
`$WORKSPACE/specs/<branch>/pipeline.state`. This file tracks progress
through the design pipeline and enables crash recovery and external
tool observation.

Write the state file at each transition using this format:

\```
format_version: 1
state: <STATE_NAME>
entered: <ISO-8601 timestamp>
decision_count: <N>
dimensions_completed: 0
dimensions_total: 0
ordered: false
dimensions_done:
current_dimension:
workspace_structure:
workspace_coherence:
workspace_robustness:
workspace_crosscutting:
workspace_decision:
\```

States: CONTEXT_GATHERING, CLARIFYING_QUESTIONS, APPROACH_EXPLORATION,
DECISION_CAPTURE, DECISION_REVIEW, DECISION_REVISION, SPEC_WRITING,
SPEC_SELF_REVIEW, POST_SPEC_REVIEW, PLANNING.

Update `decision_count` each time a decision is captured. Overwrite the
file at each transition — git history preserves the trail.
```

- [ ] **Step 2: Add decision capture to the approach exploration step**

Modify the "Exploring Approaches" section. After the user selects from
2-3 approaches, add:

```markdown
### Decision Capture

After the user selects an approach (or any sub-decision where 2+ options
were presented), write the decision to
`$WORKSPACE/specs/<branch>/decisions.md`:

\```markdown
## D<N>: <short title>

**Choice:** <what was selected>
**Alternatives:**
- <option B> — <one-line trade-off>
- <option C> — <one-line trade-off>
**Rationale:** <why this choice>
**Trade-offs:** <what we're giving up>
**Exploration:** <quick | deep-analysis | multi-agent-debate>
**Status:** captured
\```

If a later decision depends on an earlier one, add:
`**Depends on:** D<N> (<short description>)`

Write each decision incrementally — append to the file as decisions are
made. Update pipeline.state with incremented `decision_count` and
transition to `DECISION_CAPTURE` state.

After writing the decision, check if more sub-decisions are needed:
- Yes → transition back to `APPROACH_EXPLORATION`
- No (user approves overall design direction) → transition to
  `DECISION_REVIEW`
```

- [ ] **Step 3: Add SOURCES.md coherence reminders**

In the "Gathering Context" section, add after the protocol SEARCH:

```markdown
- If SOURCES.md exists in the project root (or is inlined via CLAUDE.md),
  reference it before proposing approaches: "Does the platform already
  have this? Where does this belong?" Check capability docs, boundary
  rules, and architecture docs listed in SOURCES.md.
```

In the "Spec Self-Review" section, add:

```markdown
5. **SOURCES.md coherence check:** Re-read SOURCES.md. Does the spec
   duplicate existing capability? Violate boundary rules? Fit the
   module hierarchy?
6. **Trade-off verification:** For each decision in decisions.md with
   non-empty Trade-offs, verify the spec acknowledges the limitation.
   For multi-agent-debate decisions, verify the spec references the
   strongest counter-argument from the losing positions.
```

- [ ] **Step 4: Update the flowchart**

Update the mermaid flowchart to include DECISION_CAPTURE and
DECISION_REVIEW nodes between DESIGN approval and WRITE.

- [ ] **Step 5: Commit**

```bash
git add brainstorming/SKILL.md
git commit -m "feat(#198): decision capture loop and pipeline state in brainstorming"
```

---

### Task 3: Approach Exploration Enhancement

**Files:**
- Modify: `brainstorming/SKILL.md` — add three exploration depth tiers

**Interfaces:**
- Consumes: Decision capture format from Task 2
- Produces: `Exploration:` field values (`quick`, `deep-analysis`,
  `multi-agent-debate`) used by decision-review calibration (Task 4)

- [ ] **Step 1: Add exploration depth tiers to brainstorming**

Replace the current "Exploring Approaches" content with an enhanced
version that includes three tiers. Add after the existing approach
presentation instructions:

```markdown
### Approach Exploration Depth

When presenting 2-3 approaches, assess architectural impact and offer
the appropriate exploration depth:

**Level 1 — Quick pick:** Low impact (config, naming, wiring). Present
options with recommendation. If user selects immediately, capture with
`Exploration: quick`.

**Level 2 — Deep analysis:** Moderate impact (new module, API surface).
When user is uncertain, perform:
1. Steelman each option — strongest possible case
2. Devil's advocate each option — why it might fail
3. Internet search — prior art, current best practices
4. First-principles analysis — improve on proposals, surface new options

Present strengthened recommendation. Capture with
`Exploration: deep-analysis`. Write analysis to
`$WORKSPACE/specs/<branch>/explorations/D<N>-exploration.md`.

**Level 3 — Multi-agent debate:** High impact (novel architecture,
cross-repo boundary, data model). When user signals high stakes or
requests debate:

1. Spawn N parallel agents (one per approach):
   Brief: "Make the strongest case for approach X. Explain why it is
   better than Y and Z. Address weaknesses honestly. Search the internet
   for supporting evidence."
2. Collect position papers
3. Spawn mediator agent: "Read these N position papers. Which approach
   wins? What strengths from losers should be incorporated? Propose a
   hybrid if optimal."
4. Present mediator synthesis to user

Subsequent rounds: re-spawn mediator only (not advocates) with user's
specific points. Max 3 debate rounds.

Capture with `Exploration: multi-agent-debate`. Write artifacts to
`$WORKSPACE/specs/<branch>/explorations/D<N>-debate/`:
- `advocate-A.md`, `advocate-B.md`, `advocate-C.md`
- `mediator-synthesis-1.md` (per round)

**Failure handling:**
- Advocate failure with ≥2 surviving: proceed with survivors
- Only 1 advocate: fall back to deep analysis (Level 2)
- Mediator failure: present raw position papers to user

**Proactive recommendation:** Don't wait for user to ask. Recommend
deep analysis for moderate-impact decisions, debate for high-impact.
User can always escalate or de-escalate.
```

- [ ] **Step 2: Commit**

```bash
git add brainstorming/SKILL.md
git commit -m "feat(#198): three-tier approach exploration (quick/deep/debate)"
```

---

### Task 4: Decision Review Type in Design-Review Framework

**Files:**
- Modify: `design-review/review.py:1611-1665` — add decision mode/type constants
- Modify: `design-review/prompts.py` — add decision review prompt builders
- Modify: `design-review/setup.py` — add decision constraints and mode generators
- Modify: `design-review/tracker.py:397-424` — extend section parsing for D<N>
- Modify: `design-review/review-tiers.md` — add decision lifecycle point
- Test: `design-review/tests/test_decision_review.py`

**Interfaces:**
- Consumes: `decisions.md` format from Task 2
- Produces: `"decision"` mode in review.py that Tasks 5 and 6 reference.
  Pipeline state writes (`DECISION_REVISION` or `SPEC_WRITING`) on completion.

- [ ] **Step 1: Write failing tests for decision review constants**

```python
# design-review/tests/test_decision_review.py
import sys
from pathlib import Path

sys.path.insert(0, str(Path(__file__).parent.parent.parent))

def test_decision_in_review_modes():
    from adversarial_design_review.review import REVIEW_MODES, REVIEW_TYPES
    assert "decision" in REVIEW_MODES
    assert "decision" in REVIEW_TYPES

def test_decision_type_defaults():
    from adversarial_design_review.review import TYPE_DEFAULTS, TYPE_TO_MODE, MODE_TO_TYPE
    assert TYPE_DEFAULTS["decision"] == "standard"
    assert TYPE_TO_MODE["decision"] == "decision"
    assert MODE_TO_TYPE["decision"] == ("decision", "standard")

def test_decision_mode_defaults():
    from adversarial_design_review.review import MODE_DEFAULTS
    assert "decision" in MODE_DEFAULTS
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest design-review/tests/test_decision_review.py::test_decision_in_review_modes -v`
Expected: FAIL — "decision" not in REVIEW_MODES

- [ ] **Step 3: Add decision constants to review.py**

Add `"decision"` to `REVIEW_MODES` tuple:
```python
REVIEW_MODES: Final = (
    "pre-review", "spec-review", "code-review", "final-review",
    "coherence", "structure", "robustness", "crosscutting",
    "decision",
)
```

Add to `REVIEW_TYPES`:
```python
REVIEW_TYPES: Final = ("coherence", "structure", "robustness", "conformance", "readiness", "crosscutting", "decision")
```

Add to mapping dicts:
```python
TYPE_DEFAULTS["decision"] = "standard"  # add to the Final dict
TYPE_TO_MODE["decision"] = "decision"
MODE_TO_TYPE["decision"] = ("decision", "standard")
MODE_DEFAULTS["decision"] = {"max_rounds": 3, "min_rounds": 2, "budget_per_session": 5.0}
```

Note: These are `Final` dicts declared at module level. Either change them
to mutable dicts, or add the entries directly in the existing dict literals.

- [ ] **Step 4: Run test to verify constants pass**

Run: `python3 -m pytest design-review/tests/test_decision_review.py -v`
Expected: PASS for all constant tests

- [ ] **Step 5: Write failing test for decision prompt dispatch**

```python
def test_decision_reviewer_prompt_dispatch():
    from adversarial_design_review.prompts import build_reviewer_prompt
    prompt = build_reviewer_prompt(
        round_num=1, focus_items=[], handover_path=None,
        mode="decision", workspace_root="/tmp/test",
        spec_path="/tmp/decisions.md",
    )
    assert "decision" in prompt.lower()
    assert "rationale" in prompt.lower()

def test_decision_implementor_prompt_dispatch():
    from adversarial_design_review.prompts import build_implementor_prompt
    prompt = build_implementor_prompt(
        round_num=1, focus_items=[], mode="decision",
        workspace_root="/tmp/test", spec_path="/tmp/decisions.md",
    )
    assert "decision" in prompt.lower()
```

- [ ] **Step 6: Run test to verify it fails**

Run: `python3 -m pytest design-review/tests/test_decision_review.py::test_decision_reviewer_prompt_dispatch -v`
Expected: FAIL — no dispatch branch for "decision" mode

- [ ] **Step 7: Add decision review prompts to prompts.py**

Add `_build_decision_review_reviewer_prompt()` and
`_build_decision_review_implementor_prompt()` functions. Add dispatch
branches in `build_reviewer_prompt()` and `build_implementor_prompt()`:

```python
if mode == "decision":
    return _build_decision_review_reviewer_prompt(
        round_num, focus_items, handover_path, convergence_override_ids,
        source_dirs, workspace_root, spec_path, maturity_stage,
    )
```

The reviewer prompt must include:
- Read decisions.md and SOURCES.md
- Challenge rationale, propose alternatives, check platform coherence
- Surface implicit decisions
- Calibrate by Exploration field (quick → max scrutiny, debate → light)
- Starting points from spec's `_DECISION_REVIEW_STARTING_POINTS`

The implementor prompt must include:
- Defend well-explored decisions, be open on quick-pick decisions
- Update decisions.md when revising (mark status as revised)
- Add implicit decisions as new entries when surfaced

- [ ] **Step 8: Run prompt dispatch tests**

Run: `python3 -m pytest design-review/tests/test_decision_review.py -v`
Expected: PASS

- [ ] **Step 9: Add decision mode generators to setup.py**

Add `_decision_review_reviewer_md()` and `_decision_review_implementor_md()`
functions using the composable constraint pattern. Register:

```python
_MODE_GENERATORS["decision"] = {
    "reviewer": _decision_review_reviewer_md,
    "implementor": _decision_review_implementor_md,
}
```

Add guard to `annotate_spec_headings()` — the brainstorming SKILL.md
must pass mode to setup_review, and setup.py must skip annotation when
mode is `decision` (decisions.md uses `## D1:` headings).

- [ ] **Step 10: Write failing test for tracker section parsing**

```python
def test_extract_section_number_decision_format():
    from adversarial_design_review.tracker import _extract_section_number
    assert _extract_section_number("D3") == "D3"
    assert _extract_section_number("EVIDENCE: D3 | commit:abc") == "D3"
    assert _extract_section_number("§4.1") == "4.1"  # existing behavior preserved

def test_find_section_range_decision_heading():
    from adversarial_design_review.tracker import _find_section_range
    content = "# Decisions\n\n## D1: First\n\nContent\n\n## D2: Second\n\nMore"
    result = _find_section_range(content, "D1")
    assert result is not None
    assert result[0] == 3  # line of ## D1
```

- [ ] **Step 11: Run test to verify it fails**

Run: `python3 -m pytest design-review/tests/test_decision_review.py::test_extract_section_number_decision_format -v`
Expected: FAIL — _extract_section_number doesn't match D<N>

- [ ] **Step 12: Extend tracker.py section parsing**

Update `_extract_section_number()`:
```python
def _extract_section_number(location: str) -> str | None:
    m = re.search(r"§(\d+(?:\.\d+)*)", location)
    if m:
        return m.group(1)
    m = re.search(r"\bD(\d+)\b", location)
    if m:
        return f"D{m.group(1)}"
    return None
```

Update `_find_section_range()` to match `## D<N>:` headings:
```python
num_match = re.match(r"[§S]?(\d+(?:\.\d+)*)", title)
if not num_match:
    num_match = re.match(r"(D\d+):", title)
if num_match and num_match.group(1) == section_ref:
```

- [ ] **Step 13: Run all tests**

Run: `python3 -m pytest design-review/tests/ -v`
Expected: All PASS (existing + new)

- [ ] **Step 14: Add pipeline state writes to review.py**

At review completion (where `_print_summary` and `REVIEW DONE` are logged),
add conditional pipeline.state write:

```python
if args.mode == "decision":
    spec_path_obj = Path(spec_path) if spec_path else None
    if spec_path_obj:
        pipeline_state = spec_path_obj.parent / "pipeline.state"
        if pipeline_state.exists():
            has_verified = any(
                issue.status == IssueStatus.VERIFIED
                for issue in tracker.issues()
            )
            new_state = "DECISION_REVISION" if has_verified else "SPEC_WRITING"
            _write_pipeline_state(pipeline_state, new_state, tracker)
```

Add `_write_pipeline_state()` helper that reads the existing file, updates
`state:` and `entered:`, and writes back.

Also add workspace path write at workspace creation in `setup_review()`:
```python
if spec_path and str(spec_path) != "":
    pipeline_state = spec_path.parent / "pipeline.state"
    if pipeline_state.exists():
        _update_pipeline_workspace(pipeline_state, mode, ws)
```

- [ ] **Step 15: Update review-tiers.md**

Update the Lifecycle Points table — change the Post-brainstorming row from
future to implemented:

```markdown
| Post-brainstorming | After approach selected, before spec | Decision | Approach fitness, prior art, platform conformance |
```

Add Decision to the Dimensions table:

```markdown
| Decision | Rationale soundness, unconsidered alternatives, platform coherence, implicit decisions *(post-brainstorming only)* |
```

- [ ] **Step 16: Run full test suite**

Run: `python3 -m pytest design-review/tests/ -v`
Expected: All PASS

- [ ] **Step 17: Commit**

```bash
git add design-review/review.py design-review/prompts.py design-review/setup.py design-review/tracker.py design-review/review-tiers.md design-review/tests/test_decision_review.py
git commit -m "feat(#198): decision review type — prompts, constraints, tracker parsing, pipeline state"
```

---

### Task 5: Decision Review Integration in Brainstorming

**Files:**
- Modify: `brainstorming/SKILL.md` — wire decision-review invocation,
  revision cycle logic, degree prompt

**Interfaces:**
- Consumes: `"decision"` mode from Task 4, decisions.md from Task 2
- Produces: Complete pipeline flow from decision capture through to
  spec writing, including revision cycles

- [ ] **Step 1: Add decision review invocation to brainstorming**

After the decision capture section (from Task 2), add the decision review
gate. This replaces the transition from "design approved" directly to
"write spec":

```markdown
### Decision Review Gate

After all decisions are captured and the user approves the overall design
direction, present the decision review depth prompt:

Count exploration depths from decisions.md:
- M decisions with `Exploration: quick`
- K decisions with `Exploration: deep-analysis`
- J decisions with `Exploration: multi-agent-debate`

Recommendation: many quick picks → Standard or Adversarial. All
deep/debate → Light or Skip.

\```python
AskUserQuestion(questions=[{
    "question": "N decisions captured (M quick, K deep, J debate). Review depth?",
    "header": "Decision review",
    "options": [
        {"label": "<Recommended> (Recommended)", "description": "<reasoning>"},
        {"label": "Skip", "description": "Proceed to spec writing"},
        {"label": "Light", "description": "~2 min — single pass"},
        {"label": "Standard", "description": "~5 min — 2-3 rounds"},
        {"label": "Adversarial", "description": "~12 min — 4-6 rounds"},
    ],
    "multiSelect": false,
}])
\```

If Skip: transition to SPEC_WRITING.

If review selected: launch decision review:
\```bash
python3 ~/.claude/skills/design-review/review.py \
  --spec $WORKSPACE/specs/<branch>/decisions.md \
  --title <branch>-decision \
  --type decision --degree <selected> \
  --stage <maturity_stage> \
  --source-dirs <project-dirs>
\```

Run with `run_in_background: true`. Set up watchdog cron to monitor.

When review completes, read pipeline.state:
- If `DECISION_REVISION`: re-enter APPROACH_EXPLORATION for dependent
  decisions (max 2 revision cycles — see spec revision cycle bounds)
- If `SPEC_WRITING`: proceed to write the spec
```

- [ ] **Step 2: Add revision cycle logic**

```markdown
### Revision Cycles

When decision-review revises decisions:

1. Read decisions.md — find entries with `Status: revised`
2. For each revised decision, find dependents (`Depends on:` pointing to it)
3. Re-evaluate each dependent: present the dependency change to the user,
   ask if the dependent decision still holds
4. If dependents change, capture updates (loop back to DECISION_CAPTURE)
5. Max 2 revision cycles. If reached, escalate to user with explanation
   of whether this is chain propagation or circular tension.
6. Null cycles (no substantive changes) don't count toward the max.
```

- [ ] **Step 3: Update the post-spec review prompt for ordered mode**

In the "Review Depth Prompt" section, add the ordered-mode variant:

```markdown
When the recommendation engine detects cross-module complexity in the
spec, replace the standard degree prompt with one that includes ordering:

\```python
AskUserQuestion(questions=[{
    "question": "Post-spec review depth?",
    "header": "Review",
    "options": [
        {"label": "Standard, ordered (Recommended)", "description": "Sequential dimensions — ~10 min"},
        {"label": "Standard, parallel", "description": "All dimensions simultaneous — ~5 min"},
        {"label": "Review it yourself", "description": "Self-review"},
        {"label": "Skip", "description": "No review"},
    ],
    "multiSelect": false,
}])
\```

If ordered mode selected: follow design-review SKILL.md ordered
orchestration (Task 6). Write `ordered: true` to pipeline.state.
```

- [ ] **Step 4: Update the flowchart**

Replace the brainstorming flowchart with the full pipeline flow from the
spec (CONTEXT → QUESTIONS → APPROACHES → CAPTURE → REVIEW → REVISION →
SPEC → SELF_REVIEW → POST_SPEC → PLANS).

- [ ] **Step 5: Commit**

```bash
git add brainstorming/SKILL.md
git commit -m "feat(#198): decision review gate and revision cycles in brainstorming"
```

---

### Task 6: Ordered Dimensional Reviews in Design-Review

**Files:**
- Modify: `design-review/SKILL.md` — ordered mode in Step 4, watchdog
  adaptation
- Modify: `design-review/setup.py` — cascading context injection in
  `_generate_context_md()`
- Test: `design-review/tests/test_cascading_context.py`

**Interfaces:**
- Consumes: `--arch-files` mechanism (already exists), `.mode` file in
  each workspace (already written by setup.py)
- Produces: Ordered dimension orchestration in SKILL.md, cascading
  context with prefixed issue IDs in context.md

- [ ] **Step 1: Write failing test for cascading context generation**

```python
# design-review/tests/test_cascading_context.py
import tempfile
from pathlib import Path

def test_generate_context_with_prior_dimension_tracker(tmp_path):
    from adversarial_design_review.setup import _generate_context_md

    structure_ws = tmp_path / "structure-ws"
    structure_ws.mkdir()
    (structure_ws / ".mode").write_text("structure")
    (structure_ws / "tracker.md").write_text(
        "## Issues\n\n"
        "### R1-01: Unclear boundary\n"
        "- **Status:** VERIFIED\n"
        "- **Location:** §2.1\n"
        "- **Priority:** HIGH\n"
    )

    coherence_ws = tmp_path / "coherence-ws"
    coherence_ws.mkdir()
    (coherence_ws / "responses").mkdir()
    (coherence_ws / "agents" / "reviewer").mkdir(parents=True)
    (coherence_ws / "agents" / "implementor").mkdir(parents=True)

    _generate_context_md(
        coherence_ws,
        source_dirs=["/tmp/project"],
        spec_path=Path("/tmp/spec.md"),
        arch_files=[str(structure_ws / "tracker.md")],
    )

    context = (coherence_ws / "context.md").read_text()
    assert "Prior dimension findings" in context
    assert "STR-R1-01" in context
    assert "Unclear boundary" in context

def test_no_cascading_without_mode_file(tmp_path):
    from adversarial_design_review.setup import _generate_context_md

    other_ws = tmp_path / "other-ws"
    other_ws.mkdir()
    (other_ws / "tracker.md").write_text("## Issues\n\n### R1-01: Something\n")

    target_ws = tmp_path / "target-ws"
    target_ws.mkdir()
    (target_ws / "responses").mkdir()
    (target_ws / "agents" / "reviewer").mkdir(parents=True)
    (target_ws / "agents" / "implementor").mkdir(parents=True)

    _generate_context_md(
        target_ws,
        source_dirs=["/tmp/project"],
        spec_path=Path("/tmp/spec.md"),
        arch_files=[str(other_ws / "tracker.md")],
    )

    context = (target_ws / "context.md").read_text()
    assert "Prior dimension findings" not in context
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest design-review/tests/test_cascading_context.py -v`
Expected: FAIL — _generate_context_md doesn't inject cascading context

- [ ] **Step 3: Add cascading context injection to setup.py**

Modify `_generate_context_md()` to detect prior dimension trackers in
arch_files and generate prefixed cascading context:

```python
DIMENSION_PREFIXES = {
    "structure": "STR",
    "coherence": "COH",
    "robustness": "ROB",
}

def _parse_prior_dimension_findings(arch_files: list[str]) -> str:
    """Parse arch-files that are dimension trackers and generate cascading context."""
    sections = []
    for af in arch_files:
        af_path = Path(af)
        ws_dir = af_path.parent
        mode_file = ws_dir / ".mode"
        if not mode_file.exists():
            continue
        mode = mode_file.read_text().strip()
        prefix = DIMENSION_PREFIXES.get(mode)
        if not prefix:
            continue
        tracker_content = af_path.read_text()
        findings = _extract_tracker_summaries(tracker_content, prefix)
        if findings:
            sections.append(f"{mode.title()} review found:\n" + "\n".join(findings))

    if not sections:
        return ""

    return (
        "\n## Prior dimension findings\n\n"
        "Read the full trackers for detail:\n\n"
        + "\n\n".join(sections)
        + "\n\nConsider these findings when evaluating this dimension. "
        "Where a prior finding creates a gap in this dimension, cite "
        "the finding by prefixed ID.\n"
    )

def _extract_tracker_summaries(content: str, prefix: str) -> list[str]:
    """Extract issue summaries from tracker.md with dimension prefix."""
    import re
    findings = []
    for m in re.finditer(
        r"### (R\d+-\d+): (.+)\n.*?Status:\*\* (\w+)",
        content, re.DOTALL,
    ):
        issue_id, title, status = m.group(1), m.group(2), m.group(3)
        findings.append(f"- {prefix}-{issue_id}: {title} [{status}]")
    return findings
```

Add the call in `_generate_context_md()` after the source_lines block:

```python
if arch_files:
    cascading = _parse_prior_dimension_findings(arch_files)
    if cascading:
        content += cascading
```

- [ ] **Step 4: Run test to verify it passes**

Run: `python3 -m pytest design-review/tests/test_cascading_context.py -v`
Expected: PASS

- [ ] **Step 5: Add ordered mode to design-review SKILL.md**

In Step 4 (Launch dimension reviews), add the ordered mode path:

```markdown
### Ordered mode (opt-in)

When the user selects an ordered review variant:

1. Write `ordered: true` and `dimensions_total: 3` to pipeline.state
2. Launch structure review with `run_in_background: true`
3. When structure completes:
   - Read pipeline.state for `workspace_structure` path
   - Present structure results to user
   - Ask: "Continue to coherence? (y/n)"
   - If yes: launch coherence with `--arch-files {structure_ws}/tracker.md`
   - Update pipeline.state: `dimensions_completed: 1`,
     `dimensions_done: structure`, `current_dimension: coherence`
4. When coherence completes:
   - Present coherence results
   - Ask: "Continue to robustness? (y/n)"
   - If yes: launch robustness with both tracker paths
   - Update pipeline.state accordingly
5. When robustness completes:
   - Present robustness results
   - Launch cross-cutting with all three tracker paths
6. When cross-cutting completes:
   - Present unified results table
   - Transition to PLANNING
```

- [ ] **Step 6: Add watchdog adaptation for ordered mode**

Update the watchdog cron prompt (Step 5) to parameterize by mode:

```markdown
In ordered mode, the watchdog monitors a single active dimension.
Checkpoint 1 (round 1 early-HIL gate) does not apply — SKILL.md
presents full results between dimensions. The watchdog's role is
health monitoring only: stalls, failures, timeouts.
```

- [ ] **Step 7: Add pipeline.state workspace write to review.py setup**

In `setup_review()`, after creating the workspace, add conditional
pipeline.state write:

```python
if spec_path and str(spec_path) != "":
    pipeline_state = spec_path.resolve().parent / "pipeline.state"
    if pipeline_state.exists():
        _update_pipeline_workspace(pipeline_state, mode, ws)

def _update_pipeline_workspace(state_path: Path, mode: str, ws: Path) -> None:
    content = state_path.read_text()
    key = f"workspace_{mode}"
    lines = content.split("\n")
    updated = []
    for line in lines:
        if line.startswith(f"{key}:"):
            updated.append(f"{key}: {ws}")
        else:
            updated.append(line)
    state_path.write_text("\n".join(updated))
```

- [ ] **Step 8: Run full test suite**

Run: `python3 -m pytest design-review/tests/ -v`
Expected: All PASS

- [ ] **Step 9: Commit**

```bash
git add design-review/SKILL.md design-review/setup.py design-review/review.py design-review/tests/test_cascading_context.py
git commit -m "feat(#198): ordered dimensional reviews with cascading context"
```

---

### Task 7: Validation and Documentation Sync

**Files:**
- Modify: `CLAUDE.md` — update Key Skills section
- Run: `python3 scripts/validate_all.py --tier commit`
- Run: `python3 scripts/claude-skill sync-local --all -y`

**Interfaces:**
- Consumes: All prior tasks
- Produces: Validated, synced skills ready for use

- [ ] **Step 1: Run commit-tier validation**

Run: `python3 scripts/validate_all.py --tier commit`
Expected: No CRITICAL findings. Fix any that appear.

- [ ] **Step 2: Check if README needs updating**

Follow `docs/development/readme-sync.md` workflow — SKILL.md files were
modified (brainstorming, design-review). The workflow determines whether
README changes are needed.

- [ ] **Step 3: Sync skills to ~/.claude/skills/**

Run: `python3 scripts/claude-skill sync-local --all -y`

- [ ] **Step 4: Commit any remaining changes**

```bash
git add -A
git commit -m "docs(#198): validation, README sync, and skill installation"
```
