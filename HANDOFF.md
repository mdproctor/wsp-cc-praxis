# HANDOFF — 2026-07-06

## Last Session

ADR four-phase pipeline — closed #60, #61, #62, #64, #65. Two branches
across one session. Epic #63 has one remaining issue (#66).

## What Was Done

### Branch 1: issue-60-design-review-fixes (landed as 7814ea4)

Batched bugfix + test coverage for the adversarial design-review skill.
Closed 4 issues:

| Issue | What | Status at branch start |
|-------|------|----------------------|
| #60 | `record_round()` cumulative vs delta counts | Already fixed — `_previous_snapshot` delta computation in place. Closed with evidence. |
| #61 | Decision file missing issue ID + recursive retry | Issue ID: `_handle_decision_needed` now auto-extracts from signal description via regex. Recursive retry: already a `while True` loop. |
| #62 | `--arch-files` CLI flag | Added to `parse_args()`, persisted in `.arch-files`, injected into `context.md` as "Architectural Files" section. |
| #64 | Pre-review mode test coverage | Implementation was complete. Added 8 tests covering prompts, templates, mode generators. |

### Branch 2: issue-65-code-review-mode (landed as a6b401f)

Phase 3 of the four-phase review pipeline — code review against spec.
Verifies implementation matches the reviewed design. Added:

- **setup.py**: 4 constraint constants (`_CODE_REVIEW_APPROACH_REVIEWER`,
  `_CODE_REVIEW_STARTING_POINTS`, `_CODE_REVIEW_APPROACH_IMPLEMENTOR`,
  `_CODE_REVIEW_IMPLEMENTOR_SKEPTICISM`) + 2 generator functions +
  `_MODE_GENERATORS["code-review"]` registration
- **prompts.py**: `_build_code_review_reviewer_prompt()` and
  `_build_code_review_implementor_prompt()` + mode dispatch branches
- **review.py**: `--diff-base` parameter (stored in `.diff-base`, loaded on resume)
- **SKILL.md**: phase 3 marked active, optional flags table updated
- **tests**: 12 new tests (TestCodeReviewPrompts, TestCodeReviewTemplates, TestDiffBase)

Infrastructure was already wired — `REVIEW_MODES` and `MODE_DEFAULTS` already
included `code-review` with defaults (max=4, min=2, budget=$5).

## What Wasn't Done

- `--diff-base` is stored and loaded but not yet injected into prompt text.
  Agents discover code via `--source-dirs` and IntelliJ. A future enhancement
  could inject `git diff <diff-base>..HEAD` into the prompt for focused review.
- No end-to-end test of `--mode code-review` running against a real spec + code.
  All tests are unit-level (prompt content, template content, CLI parsing).

## Immediate Next — #66

**Issue:** Phase 4: Final code review — replace code-review skill and superpowers review

**Parent epic:** #63 (ADR four-phase review pipeline)

**Scope (from issue):**
- Add `--mode final-review` with sub-phase support (4a main code, 4b test code)
- New reviewer templates for main and test review briefs
- **Deprecation:** replace `code-review` skill and `requesting-code-review`
- Update `git-commit` workflow to reference final-review instead of code-review

**Why this needs its own brainstorming session:**
The deprecation is the hard part. `code-review` is invoked by `work-end` (Step 3c,
mandatory gate), `git-commit` (when no review done), and multiple dev skills. Replacing
it means updating all those integration points. `requesting-code-review` dispatches a
fresh subagent — different mechanism from the adversarial multi-round approach. The
migration path needs design before implementation.

**Key questions for brainstorming:**
1. Does final-review fully replace code-review, or do they coexist? The current
   code-review is a single-pass checklist. Final-review is multi-round adversarial.
   Different weight classes.
2. Does the sub-phase (4a/4b) run as two sequential `--mode` invocations or as
   a single run with an internal phase switch?
3. What happens to `work-end` Step 3c? Does it invoke final-review (heavy) or
   keep code-review (lightweight)?

## Key Context

**Epic #63 status:** 3 of 4 phases complete.
- Phase 1 (pre-review): done — `--mode pre-review`
- Phase 2 (spec-review): done — `--mode spec-review` (original)
- Phase 3 (code-review): done — `--mode code-review` (this session)
- Phase 4 (final-review): #66, not started

**Test count:** 98 tests across `test_adr_tracker.py` and `test_adr_review.py`, all passing.

**Skill sync:** `sync-local` was run after each commit. `~/.claude/skills/design-review/`
has the latest version.

**Repo state:** main branch, up to date with origin. Clean working tree.
