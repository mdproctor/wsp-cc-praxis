# Design: Final Review Mode (Phase 4)

**Status:** Draft
**Date:** 2026-07-06
**Issue:** #66 (parent: #63 ADR four-phase review pipeline)

---

## Goal

Add `--mode final-review` to the design-review adversarial engine as the
branch-level code quality gate. Complements the existing `code-review` skill
(which stays for per-commit checklist review). Deprecates `requesting-code-review`
(whose branch-level role is absorbed by final-review).

## Artifact under review

Final-review reviews the **branch diff** — all code changes from the base branch
to HEAD. This is a standalone production-readiness check, not a spec-compliance
check. Unlike code-review mode (Phase 3), final-review does not require or
reference a companion spec.

| Mode | Artifact | Spec required? |
|------|----------|----------------|
| pre-review | Brainstorming output / outline | No |
| spec-review | Design spec | Yes — the spec is the artifact |
| code-review | Code + reviewed spec | Yes — checks implementation against spec |
| final-review | Branch diff (code only) | No — standalone production readiness |

The reviewer receives the branch diff via the source directories. The prompt
instructs the reviewer to:

1. Compute the branch diff: `git diff <base>..HEAD`
2. Read each changed file in the source directories
3. Apply production readiness criteria to the changes

The `--spec` flag is ignored for final-review mode. If passed, log a warning and
continue without using it.

## Decision summary

| Question | Answer |
|----------|--------|
| Replace code-review skill? | No — code-review stays for per-commit checklist review |
| Replace requesting-code-review? | Yes — deprecated, branch-level role absorbed by final-review |
| Sub-phases (4a/4b)? | Prompt-level only — reviewer brief structures both concerns, no infrastructure changes. Resolves open question #4 from the four-phase pipeline spec. Issue #66 text to be updated when this spec lands. |
| Degree system? | `--depth light\|standard\|deep` with auto-detection default and manual override |
| Work-end Step 3c? | Conditional — final-review for structural changes, code-review for body-only edits |

## Scope reduction from issue #66

The issue says "replace code-review skill and requesting-code-review." After
first-principles analysis, code-review and final-review serve different scopes
and complement each other:

| Tool | Scope | Depth | When |
|------|-------|-------|------|
| `code-review` (skill) | Commit-level | Checklist/mechanical | Before every commit |
| `design-review --mode final-review` | Branch-level | Adversarial | Before branch close (structural changes) |

`requesting-code-review` is the only skill being deprecated — its branch-level
independent-subagent review is replaced by final-review's adversarial model.

## §1 Depth system

### §1.1 Depth presets

| Depth | max_rounds | min_rounds | budget | Sub-phase treatment |
|-------|-----------|-----------|--------|---------------------|
| light | 1 | 1 | $1.50 | Combined — single pass, no 4a/4b split |
| standard | 3 | 2 | $5.00 | Structured — reviewer brief has main + test sections |
| deep | 5 | 3 | $8.00 | Full — both sections, expanded scope |

### §1.2 Auto-detection heuristic

When `--depth` is not specified, auto-detect from the branch diff:

```
Input: git diff <base>..HEAD --stat
```

| Signal | Depth |
|--------|-------|
| ≤50 lines changed AND ≤3 files | light |
| ≤300 lines changed AND ≤10 files | standard |
| >300 lines changed OR >10 files | deep |

New files count double — a 40-line branch with 2 new files is treated as 80
lines (→ standard, not light).

The `--depth` flag overrides auto-detection in all cases.

### §1.3 Prompt scope by depth

| Depth | Reviewer focus |
|-------|---------------|
| **light** | Correctness risks and security only. "Quick sanity check — anything obviously wrong or unsafe?" |
| **standard** | Full scope: architecture, correctness, edge cases, error handling, performance, concurrency, security, naming, structure, layer compliance. Test code: coverage completeness, assertion quality, missing scenarios. |
| **deep** | All standard concerns plus: cross-module impact, design drift from spec, concurrency under load, backward compatibility, test isolation, property-based testing gaps. |

## §2 Implementation in design-review Python code

### §2.1 review.py changes

**New CLI flag:**

```
--depth {light,standard,deep}   Review depth (default: auto-detect from diff stats)
```

Added to the argparse parser alongside `--mode`. Only applies when
`--mode final-review`.

**Behavior with non-final-review modes:**

```python
if args.depth and args.mode != "final-review":
    _log(f"WARNING: --depth is only supported for final-review mode, ignored for {args.mode}")
```

Warn and continue — not an error. The user may have a shell alias or script that
passes `--depth` generically. Silent ignore would be confusing; hard error would
break workflows.

**Depth resolution order:**
1. Explicit `--depth` flag → use it
2. No `--depth` and `--mode final-review` → auto-detect from diff stats
3. No `--depth` and other modes → use `MODE_DEFAULTS` as today

**Auto-detection function:**

```python
def _auto_detect_depth(workspace: str) -> str:
    """Detect review depth from branch diff stats."""
    # Read diff stats from workspace (base stored in .meta or .diff-base)
    # Count lines changed, files changed, new files
    # Apply heuristic from §1.2
    # Return "light" | "standard" | "deep"
```

**Depth presets integration:**

```python
DEPTH_PRESETS: Final = {
    "light":    {"max_rounds": 1, "min_rounds": 1, "budget_per_session": 1.5},
    "standard": {"max_rounds": 3, "min_rounds": 2, "budget_per_session": 5.0},
    "deep":     {"max_rounds": 5, "min_rounds": 3, "budget_per_session": 8.0},
}
```

When mode is `final-review` and depth is resolved, `DEPTH_PRESETS[depth]`
overrides `MODE_DEFAULTS["final-review"]` for that run. The `.depth` file
in the workspace persists the resolved depth for resume.

**Resume behavior with `.depth` file:**

1. **Explicit `--depth` on resume** — always overrides stored depth. The user
   knows better; they may want to re-run at a different depth.
2. **No `--depth` on resume, `.depth` exists** — use stored depth. Do not
   re-detect from diff stats (the diff may have changed, but the review should
   complete at the depth it started).
3. **No `--depth` on resume, `.depth` missing** — re-detect from diff stats.
   This handles resuming a legacy workspace created before `.depth` was added.

**Budget semantics:**

The `budget_per_session` in `DEPTH_PRESETS` uses the same semantics as
`MODE_DEFAULTS` — it is a per-agent-invocation limit passed to
`claude -p ... --max-turns`. When the budget is exhausted:

1. The agent's turn ends (Claude Code's built-in budget enforcement)
2. The PM receives whatever output was generated
3. The round continues to the next agent (or ends if this was the implementor)

This is not a hard stop — it's a soft cap per agent call. The total review
cost may exceed the preset if many rounds are needed.

### §2.2 setup.py changes

**New mode generator registration:**

```python
_MODE_GENERATORS["final-review"] = {
    "reviewer": _final_review_reviewer_md,
    "implementor": _final_review_implementor_md,
}
```

**New constraint constants** (following the existing pattern of composable
text blocks assembled via `_assemble_constraints()`):

- `_FINAL_REVIEW_APPROACH_REVIEWER` — reviewer behavioral constraints for
  final-review mode. Includes: review the full branch diff for production
  readiness; apply both main-code and test-code lenses; scale focus to
  depth level.
- `_FINAL_REVIEW_APPROACH_IMPLEMENTOR` — implementor constraints. Includes:
  defend implementation decisions with evidence; acknowledge genuine issues;
  do not capitulate on correct code. **Implementor fixes code directly**
  (in the source directories) rather than updating a spec.
- `_FINAL_REVIEW_MAIN_CODE_FOCUS` — structured section for main code review
  areas: architecture, correctness, edge cases, error handling, performance,
  concurrency, security, naming, structure, layer compliance.
- `_FINAL_REVIEW_TEST_CODE_FOCUS` — structured section for test code review
  areas: coverage completeness, assertion quality, missing scenarios, test
  isolation, fixture patterns.

**Naming rationale:** The four-phase pipeline spec uses `_FINAL_REVIEW_STARTING_POINTS`
as a single block. This spec splits it into `_FINAL_REVIEW_MAIN_CODE_FOCUS` and
`_FINAL_REVIEW_TEST_CODE_FOCUS` because the Decision Summary chose "prompt-level
only" for sub-phases — the reviewer brief structures both concerns within a
single phase, and separate focus blocks make that structure explicit.

**Generator functions:**

`_final_review_reviewer_md(context)` and `_final_review_implementor_md(context)`
assemble the CLAUDE.md for each agent using the constraint blocks. The depth
level (available in context) controls which blocks are included:

- **light**: `_FINAL_REVIEW_APPROACH_*` + `_CORE_APPROACH_*` only. Omit the
  detailed focus sections — the prompt handles scope narrowing.
- **standard**: all blocks including both focus sections.
- **deep**: all blocks plus `_CROSS_MODULE_IMPACT` (new block for deep-only
  cross-cutting analysis).

### §2.3 prompts.py changes

**New prompt builder functions:**

```python
def _build_final_review_reviewer_prompt(
    spec_path, source_dirs, round_num, history, mode, arch_files, depth
) -> str:
    ...

def _build_final_review_implementor_prompt(
    spec_path, source_dirs, round_num, history, mode, arch_files, depth
) -> str:
    ...
```

Both `build_reviewer_prompt()` and `build_implementor_prompt()` get a new
dispatch branch:

```python
if mode == "final-review":
    return _build_final_review_reviewer_prompt(...)
```

**Prompt content by depth:**

The `depth` parameter shapes the prompt text:

- **light**: "Quick review of the branch diff. Focus on correctness risks
  and security issues. One round — flag only items that would cause bugs
  or vulnerabilities in production."
- **standard**: structured brief with main code section and test code
  section (see §1.3). The reviewer applies both lenses throughout.
- **deep**: standard brief plus cross-module analysis, design-drift
  detection (if spec available), and backward compatibility checks.

### §2.4 Depth parameter threading

The `depth` value must flow through the existing call chain. Affected
function signatures gain a `depth: str | None` parameter (None for
non-final-review modes):

- `setup_review(...)` — gains `depth` param; writes `.depth` file; passes
  depth to `_generate_agent_claude_mds()` which passes to mode generators
- `build_reviewer_prompt(...)` — gains `depth` param; dispatches to
  `_build_final_review_reviewer_prompt()` which uses it for content scaling
- `build_implementor_prompt(...)` — same pattern
- `_final_review_reviewer_md(context)` — reads `context["depth"]` to select
  constraint blocks
- Resume: `_load_depth()` reads `.depth` file, returns stored value

## §3 Workflow integration changes

### §3.1 work-end Step 3c — conditional gate

Current: always invokes `code-review` on branch diff.

New logic:

```
1. Get branch diff: git diff <base>..HEAD --stat
2. Classify changes:
   - New files added? → structural
   - Files deleted? → structural (API surface may have changed)
   - Files renamed? → structural (package/namespace may have changed)
   - New classes/interfaces in diff? → structural
   - Method signature changes? → structural
   - Only method body changes? → body-only
   - Config files changed (pom.xml, application.properties, etc.)? → structural
   - Imports-only changes → body-only (dependency usage, not contract)
3. If structural → invoke design-review --mode final-review
   (depth auto-detected, --depth override available)
4. If body-only → invoke code-review (checklist, as today)
```

The classification uses diff stats and simple content heuristics (grep for
`class `, `interface `, method signatures in the `+` lines). It does not
need semantic analysis — false positives (running adversarial review on a
body-only change) are harmless, just slower.

Edge case: user can force final-review on a body-only branch if the changes
are algorithmically complex. work-end should surface the classification:

> "Branch diff is body-only (N files, M lines). Running code-review checklist.
>  Override with final-review? (y/n)"

### §3.2 subagent-driven-development

Current: final whole-branch review uses `requesting-code-review`'s
`code-reviewer.md` template.

Change: switch to `design-review --mode final-review --depth standard`.
The subagent-driven-development skill invokes design-review directly
instead of dispatching its own reviewer subagent.

### §3.3 requesting-code-review — deprecation

Add deprecation notice to SKILL.md:

```markdown
> **Deprecated:** This skill is superseded by `design-review --mode final-review`.
> Use `--mode final-review` for branch-level adversarial code review.
> This skill remains for backward compatibility but will not receive updates.
```

**Artifact disposition:**

| Artifact | Action |
|----------|--------|
| `requesting-code-review/SKILL.md` | Keep with deprecation notice |
| `requesting-code-review/code-reviewer.md` | Keep as historical reference — not moved to design-review since final-review uses the composable constraint system instead |

The skill directory is not deleted. Existing scripts and other projects may
reference it. Deletion would be a breaking change with no benefit — deprecation
is sufficient.

### §3.4 Skill Chaining updates

| Skill | Section | Change |
|-------|---------|--------|
| `design-review` | Skill Chaining, phase table | Add Phase 4 as active |
| `work-end` | Step 3c, Skill Chaining | Conditional gate logic, add final-review reference |
| `subagent-driven-development` | Final review step, Skill Chaining | Switch from `requesting-code-review/code-reviewer.md` to `design-review --mode final-review --depth standard` |
| `requesting-code-review` | Full SKILL.md | Deprecation notice |
| `code-review` | Complements section | Add: "For branch-level adversarial review, use `design-review --mode final-review`" |

**Specific file changes in `subagent-driven-development/SKILL.md`:**

1. Line 315-316: Replace reference to `requesting-code-review/code-reviewer.md`
   with invocation of `design-review --mode final-review --depth standard`
2. Line 366: Update Skill Chaining to reference `design-review` instead of
   `requesting-code-review`

**Specific file changes in `code-review/SKILL.md`:**

1. Complements section: Add entry for `design-review --mode final-review` with
   scope differentiation (per-commit vs branch-level)

### §3.5 design-review SKILL.md

Update the phase table to mark Phase 4 as active. Add `--depth` flag to the
optional flags table. Document the depth auto-detection and the relationship
with code-review (complementary, not replacement).

## §4 What does NOT change

- `code-review` skill — untouched, keeps per-commit role
- `git-commit` — no changes (doesn't invoke review)
- `executing-plans` — stays with code-review (per-task scope)
- Phases 1-3 of the pipeline — untouched
- review.py loop structure — no sub-phase state, no new infrastructure
- tracker.py — no changes (convergence protection works as-is)
- parser.py — no changes
- **Workspace structure** — final-review uses the existing flat workspace format,
  not the hierarchical structure from the four-phase pipeline spec. The
  hierarchical structure will be implemented by pre-review (#64) which requires
  the full phase orchestration infrastructure. Final-review is a single mode
  addition that fits the existing flat structure.

## §5 Test plan

### §5.1 Unit tests (in test_adr_review.py)

**Depth auto-detection:**
- Small diff (30 lines, 2 files) → light
- Medium diff (150 lines, 6 files) → standard
- Large diff (500 lines, 15 files) → deep
- New files double-count: 40 lines + 2 new files → standard (not light)
- Explicit `--depth` overrides auto-detection

**Depth presets:**
- `DEPTH_PRESETS` values are applied when mode is final-review
- `MODE_DEFAULTS` used when mode is not final-review (depth ignored)
- `.depth` file persists and restores on resume

**Prompt content:**
- Final-review reviewer prompt includes main-code and test-code sections
- Light depth omits detailed focus sections
- Standard depth includes both sections
- Deep depth includes cross-module analysis section
- Prompt builders dispatch correctly for final-review mode

**Mode generators:**
- `_MODE_GENERATORS["final-review"]` registered
- Reviewer CLAUDE.md includes correct constraint blocks for each depth
- Implementor CLAUDE.md includes correct constraint blocks for each depth

**Classification heuristic (new test class: `TestClassifyBranchDiff`):**
- `test_new_file_triggers_structural`: `A src/NewClass.java | 50 ++` → structural
- `test_deleted_file_triggers_structural`: `D src/OldClass.java | 100 -` → structural
- `test_renamed_file_triggers_structural`: `R src/Old.java → src/New.java` → structural
- `test_body_only_change`: method body edits only → body-only
- `test_signature_change_triggers_structural`: method signature change → structural
- `test_config_file_triggers_structural`: `pom.xml`, `application.properties` → structural
- `test_imports_only_is_body_only`: only import statements changed → body-only
- `test_mixed_changes_is_structural`: any structural signal → structural (conservative)

### §5.2 Integration touchpoints (manual verification)

- work-end Step 3c correctly classifies structural vs body-only changes
- subagent-driven-development invokes final-review instead of requesting-code-review
- requesting-code-review SKILL.md shows deprecation notice
- design-review SKILL.md shows Phase 4 as active with --depth documented

## §6 File change inventory

| File | Change type | Description |
|------|------------|-------------|
| `design-review/review.py` | Modify | Add `--depth` flag, `DEPTH_PRESETS`, auto-detection function, depth file persistence |
| `design-review/setup.py` | Modify | Add `_MODE_GENERATORS["final-review"]`, constraint constants, generator functions |
| `design-review/prompts.py` | Modify | Add `_build_final_review_*_prompt()`, dispatch branches, depth-scaled content |
| `design-review/SKILL.md` | Modify | Phase 4 active, `--depth` flag documented |
| `work-end/SKILL.md` | Modify | Step 3c conditional gate logic |
| `subagent-driven-development/SKILL.md` | Modify | Switch to final-review |
| `requesting-code-review/SKILL.md` | Modify | Deprecation notice |
| `code-review/SKILL.md` | Modify | Complements section update |
| `tests/test_adr_review.py` | Modify | New test classes for depth and final-review |
