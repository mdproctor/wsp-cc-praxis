# Tiered Design Review Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #98 — Tiered epic workflow strategies with scaled design-review depth
**Issue group:** #98

**Goal:** Replace the flat review phase model with a two-dimensional type x degree system, with AskUserQuestion prompts integrated into brainstorming and writing-plans.

**Architecture:** Add `--type` and `--degree` CLI flags to review.py, with backward-compat mapping from old `--mode`/`--depth`. Type-specific reviewer briefs in prompts.py. AskUserQuestion-based review prompt added to brainstorming and writing-plans skills. Escalation assessment added to light/standard reviewer briefs.

**Tech Stack:** Python (review.py, prompts.py), Markdown (SKILL.md files)

## Global Constraints

- Old `--mode` and `--depth` flags must remain accepted (backward compat)
- Existing review workspaces (`.mode` file) must still be resumable
- No changes to workspace layout, tracker format, or adversarial loop mechanics
- AskUserQuestion is the prompt mechanism (not text-based checklists)

---

### Task 1: CLI — Add `--type` and `--degree` flags with backward compat

**Files:**
- Modify: `design-review/review.py` (REVIEW_MODES, DEPTH_PRESETS, parse_args, main)
- Test: `design-review/tests/test_review.py`

**Interfaces:**
- Produces: `REVIEW_TYPES`, `DEGREE_PRESETS`, `TYPE_DEFAULTS` constants; updated `parse_args()` returning `args.review_type` and `args.degree`

- [ ] **Step 1: Write failing tests for new CLI flags**

```python
class TestParseArgs:

    def test_type_flag_accepted(self):
        sys.argv = ["review.py", "--spec", "x.md", "--title", "t",
                     "--source-dirs", "/tmp", "--type", "coherence"]
        args = parse_args()
        assert args.review_type == "coherence"

    def test_degree_flag_accepted(self):
        sys.argv = ["review.py", "--spec", "x.md", "--title", "t",
                     "--source-dirs", "/tmp", "--degree", "adversarial"]
        args = parse_args()
        assert args.degree == "adversarial"

    def test_degree_presets_applied(self):
        sys.argv = ["review.py", "--spec", "x.md", "--title", "t",
                     "--source-dirs", "/tmp", "--type", "coherence",
                     "--degree", "adversarial"]
        args = parse_args()
        assert args.max_rounds == 6
        assert args.min_rounds == 4

    def test_default_degree_per_type(self):
        sys.argv = ["review.py", "--spec", "x.md", "--title", "t",
                     "--source-dirs", "/tmp", "--type", "robustness"]
        args = parse_args()
        assert args.degree == "adversarial"
        assert args.max_rounds == 6

    def test_backward_compat_mode_maps_to_type(self):
        sys.argv = ["review.py", "--spec", "x.md", "--title", "t",
                     "--source-dirs", "/tmp", "--mode", "pre-review"]
        args = parse_args()
        assert args.review_type == "coherence"

    def test_backward_compat_depth_alias(self):
        sys.argv = ["review.py", "--spec", "x.md", "--title", "t",
                     "--source-dirs", "/tmp", "--type", "structure",
                     "--depth", "light"]
        args = parse_args()
        assert args.degree == "light"

    def test_mode_and_type_conflict_raises(self):
        sys.argv = ["review.py", "--spec", "x.md", "--title", "t",
                     "--source-dirs", "/tmp", "--mode", "pre-review",
                     "--type", "structure"]
        with pytest.raises(SystemExit):
            parse_args()
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest design-review/tests/test_review.py::TestParseArgs -v`
Expected: FAIL — `--type` and `--degree` flags don't exist yet

- [ ] **Step 3: Implement CLI changes in review.py**

Add constants:

```python
REVIEW_TYPES: Final = ("coherence", "structure", "robustness", "conformance", "readiness")

DEGREE_PRESETS: Final = {
    "light":       {"max_rounds": 1,  "min_rounds": 1, "budget_per_session": 1.5},
    "standard":    {"max_rounds": 3,  "min_rounds": 2, "budget_per_session": 5.0},
    "adversarial": {"max_rounds": 6,  "min_rounds": 4, "budget_per_session": 5.0},
    "deep":        {"max_rounds": 10, "min_rounds": 6, "budget_per_session": 8.0},
}

TYPE_DEFAULTS: Final = {
    "coherence": "light",
    "structure": "standard",
    "robustness": "adversarial",
    "conformance": "standard",
    "readiness": "standard",
}

MODE_TO_TYPE: Final = {
    "pre-review": ("coherence", "light"),
    "spec-review": ("structure", "adversarial"),
    "code-review": ("conformance", "standard"),
    "final-review": ("readiness", "standard"),
}
```

Update `parse_args()`:
- Add `--type` with choices from REVIEW_TYPES
- Add `--degree` with choices from DEGREE_PRESETS
- Keep `--mode` and `--depth` as backward compat aliases
- If `--mode` provided without `--type`: map via MODE_TO_TYPE
- If both `--mode` and `--type`: error (mutually exclusive)
- If `--depth` provided: treat as `--degree` alias
- If no `--degree`: use TYPE_DEFAULTS[review_type]
- Apply DEGREE_PRESETS to set max_rounds, min_rounds, budget_per_session

Update `main()`:
- Use `args.review_type` instead of `args.mode` for prompt selection
- Write `.type` and `.degree` files alongside `.mode` for resume
- On resume: read `.type` and `.degree` if they exist, fall back to `.mode`

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest design-review/tests/test_review.py::TestParseArgs -v`
Expected: PASS

- [ ] **Step 5: Commit**

```
feat(#98): add --type and --degree CLI flags to review.py

Refs #98
```

---

### Task 2: Reviewer briefs — Type-specific prompts with escalation

**Files:**
- Modify: `design-review/prompts.py`
- Test: `design-review/tests/test_review.py` (or new `test_prompts.py`)

**Interfaces:**
- Consumes: `review_type` string from Task 1
- Produces: `build_reviewer_prompt()` accepts `review_type=` kwarg and returns type-focused brief; light/standard include escalation instruction

- [ ] **Step 1: Write failing tests for type-specific prompts**

```python
class TestReviewerBriefsByType:

    def test_coherence_brief_focuses_on_completeness(self):
        prompt = build_reviewer_prompt(
            round_num=1, focus_items=[], handover_path=None,
            review_type="coherence",
        )
        assert "completeness" in prompt.lower()
        assert "failure mode" not in prompt.lower()

    def test_structure_brief_focuses_on_decomposition(self):
        prompt = build_reviewer_prompt(
            round_num=1, focus_items=[], handover_path=None,
            review_type="structure",
        )
        assert "decomposition" in prompt.lower() or "boundaries" in prompt.lower()

    def test_robustness_brief_focuses_on_failure(self):
        prompt = build_reviewer_prompt(
            round_num=1, focus_items=[], handover_path=None,
            review_type="robustness",
        )
        assert "failure" in prompt.lower() or "break" in prompt.lower()

    def test_light_degree_includes_escalation_instruction(self):
        prompt = build_reviewer_prompt(
            round_num=1, focus_items=[], handover_path=None,
            review_type="coherence", degree="light",
        )
        assert "escalation" in prompt.lower()
        assert "ESCALATE:" in prompt

    def test_deep_degree_does_not_include_escalation(self):
        prompt = build_reviewer_prompt(
            round_num=1, focus_items=[], handover_path=None,
            review_type="robustness", degree="deep",
        )
        assert "ESCALATE:" not in prompt
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python3 -m pytest design-review/tests/test_review.py::TestReviewerBriefsByType -v`
Expected: FAIL — `review_type` kwarg not accepted

- [ ] **Step 3: Implement type-specific briefs in prompts.py**

Update `build_reviewer_prompt()` to accept `review_type` kwarg. Add dispatch to type-specific brief builders:

- `_build_coherence_reviewer_prompt()` — completeness, consistency, gaps
- `_build_structure_reviewer_prompt()` — decomposition, boundaries, dependencies
- `_build_robustness_reviewer_prompt()` — failure modes, edge cases, error paths

For `conformance` and `readiness`: reuse existing `_build_code_review_reviewer_prompt` and `_build_final_review_reviewer_prompt`.

For light and standard degrees: append escalation assessment instruction to the brief:

```
After your review, include an escalation assessment:

## Escalation Assessment

ESCALATE: yes|no
RECOMMENDED_TYPE: <type>
RECOMMENDED_DEGREE: <degree>
REASON: <one sentence>
```

For adversarial and deep degrees: omit escalation (you're already at full depth).

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest design-review/tests/test_review.py::TestReviewerBriefsByType -v`
Expected: PASS

- [ ] **Step 5: Commit**

```
feat(#98): type-specific reviewer briefs with escalation assessment

Refs #98
```

---

### Task 3: Reference doc and SKILL.md — Tiered review flow

**Files:**
- Create: `design-review/review-tiers.md`
- Modify: `design-review/SKILL.md`

**Interfaces:**
- Consumes: REVIEW_TYPES, DEGREE_PRESETS from Task 1
- Produces: Reference doc for brainstorming/writing-plans to follow; updated SKILL.md with AskUserQuestion flow

- [ ] **Step 1: Create review-tiers.md**

Reference file defining the type x degree model, recommendation signals table, escalation rules, and the three-part prompt flow (recommendation text → type AskUserQuestion → degree AskUserQuestion). Include the AskUserQuestion tool call examples.

- [ ] **Step 2: Update SKILL.md Step 0.5**

Replace the flat phase checklist with the new tiered flow:

1. When invoked directly (`/design-review`): analyze the spec, present full recommendation text, then AskUserQuestion for type, then degree.
2. When invoked with explicit flags (`/design-review --type robustness --degree deep`): skip the prompt, use the flags directly.
3. Update the `--mode` mapping table to show both old and new flags.
4. Update the "Optional flags" table to include `--type` and `--degree`.

- [ ] **Step 3: Verify SKILL.md renders correctly**

Read the updated SKILL.md end to end. Check for broken references, stale phase names, contradictions with the new model.

- [ ] **Step 4: Commit**

```
feat(#98): review-tiers reference doc and updated SKILL.md

Refs #98
```

---

### Task 4: Integration — Brainstorming review prompt

**Files:**
- Modify: `brainstorming/SKILL.md`

**Interfaces:**
- Consumes: review-tiers.md from Task 3
- Produces: Step 7b in brainstorming that prompts for review depth after spec approval

- [ ] **Step 1: Add Step 7b between User Review Gate and Implementation**

Insert after Step 7 (User reviews written spec), before Step 8 (Transition to implementation):

```markdown
### Step 7b — Review depth prompt

After the user approves the spec, assess complexity and prompt for
review depth before transitioning to writing-plans.

1. Read the spec and identify complexity signals (see
   design-review/review-tiers.md for signal table).
2. Present the full recommendation as text — type, degree, and reasoning
   specific to this spec.
3. Use AskUserQuestion for type selection (Skip as an option, recommended
   option first with "(Recommended)" suffix).
4. If not Skip: use AskUserQuestion for degree selection.
5. If a review is selected: invoke design-review with `--type` and
   `--degree` flags. Handle escalation if returned.
6. After review completes (or Skip selected): proceed to writing-plans.
```

- [ ] **Step 2: Update the process flow diagram**

Add the review depth node between APPROVE and PLANS in the flowchart.

- [ ] **Step 3: Update Skill Chaining section**

Add `design-review` as an invoked skill (conditionally, when review is not skipped).

- [ ] **Step 4: Commit**

```
feat(#98): add review depth prompt to brainstorming

Refs #98
```

---

### Task 5: Integration — Writing-plans review prompt

**Files:**
- Modify: `writing-plans/SKILL.md`

**Interfaces:**
- Consumes: review-tiers.md from Task 3

- [ ] **Step 1: Add review prompt step after plan is written**

Insert before "Execution Handoff":

```markdown
### Step N — Review depth prompt (plans)

After the plan is written and the user has approved it, assess whether
the plan warrants review. Plans are typically lighter than specs —
default recommendation is Coherence / Light or Skip.

Follow the same three-part flow as brainstorming Step 7b (see
design-review/review-tiers.md). If a review surfaces issues, revise
the plan before proceeding to execution.
```

- [ ] **Step 2: Update Skill Chaining section**

Add `design-review` as an optionally invoked skill.

- [ ] **Step 3: Commit**

```
feat(#98): add review depth prompt to writing-plans

Refs #98
```

---

### Task 6: Sync and verify

**Files:**
- All modified skill files

- [ ] **Step 1: Run existing tests**

Run: `python3 -m pytest design-review/tests/ -v`
Expected: All pass (no regressions)

- [ ] **Step 2: Run commit-tier validators**

Run: `python3 scripts/validate_all.py --tier commit`

- [ ] **Step 3: Sync skills**

Run: `python3 scripts/claude-skill sync-local --all -y`

- [ ] **Step 4: Final commit and push**

```
feat(#98): tiered design review — type x degree model

Closes #98
```
