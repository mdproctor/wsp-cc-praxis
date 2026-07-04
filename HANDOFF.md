# HANDOFF — 2026-07-04

## Last Session

Superpowers fork integration into soredium — epic #68. Completed steps 1-5 of 14.

## What Was Done

**Steps 1-4 (previous session):** Analysis, design review, and first 4 rewrites
(ide-tooling, using-superpowers, verification-before-completion,
test-driven-development). Committed as 34febb8.

**Step 5 (this session, committed as 920e103, Refs #69):**
Debugging toolkit — rewrote both skills from scratch:

1. `systematic-debugging/SKILL.md` — forage SEARCH/CAPTURE integration (Phase 1
   Step 0 + Phase 4 Step 6), TDD bug fix workflow reference, ide-tooling for code
   tracing and structural fixes, dispatching-parallel-agents for multi-failure
   (Phase 1 Step 6), fix-ci cross-ref, Skill Chaining with debugging toolkit
   framing.

2. `dispatching-parallel-agents/SKILL.md` — threshold aligned to 2+ (CSO/body
   match), dot→Mermaid flowchart, TDD and ide-tooling awareness in agent prompts,
   Skill Chaining (invoked by systematic-debugging, fix-ci, SDD).

3. Supporting files updated: dot→Mermaid in root-cause-tracing.md (2 graphs)
   and condition-based-waiting.md (1 graph). All hortora: prefixes stripped.

**Code review:** One important finding fixed (broken "Phase 4.5" internal
reference → "Phase 4 Step 5"). Cross-reference gaps to fix-ci and SDD are
expected — back-references will be added in issues #71 and #74.

## Immediate Next Step

Continue epic #68 — step 6: #70 (brainstorming + writing-plans — design pipeline).

## Remaining Issues

| Issue | Title | What to do |
|-------|-------|------------|
| #70 | Rewrite brainstorming + writing-plans | Design pipeline. See spec §1 and §2 |
| #71 | Rewrite subagent-driven-development + executing-plans | Execution pipeline. See spec §3 and §4. Scripts need tests per protocol |
| #72 | Rewrite requesting-code-review + receiving-code-review | Quality layer. See spec §6 and §7 |
| #73 | Rewrite using-git-worktrees + writing-skills | See spec §11 and §13 |
| #74 | Audit work-end + soredium-native cross-refs + artifacts | See spec §12 + Changes to soredium-native skills + Impact sections |

## Key Context for Continuation

**The spec is the guide.** `docs/specs/2026-07-04-superpowers-fork-integration.md`
has the per-skill rewrite approach for every remaining skill.

**Rewrite conventions (from spec):**
- Bare skill names (no `hortora:` prefix)
- Mermaid `flowchart TD` (not dot digraph)
- Supporting files (prompt templates, technique docs, scripts) in scope
- ide-tooling reference for IntelliJ capabilities; individual skills don't handle fallback
- Focal issue = descriptive (the issue currently being worked on from the branch's issue group)

**Pattern established by completed rewrites (5 skills done):**
- Each skill has a Skill Chaining section listing what invokes it and what it complements
- Process skills reference ide-tooling for IDE capabilities
- Language-specific content deferred to language dev skills (java-dev, ts-dev, python-dev)
- Garden approaches referenced as foundational reading where relevant
- Debugging toolkit framing: systematic-debugging (single root cause) +
  dispatching-parallel-agents (multiple independent) + fix-ci (CI-specific)

**Cross-reference gaps to fill in later issues:**
- fix-ci needs back-references to systematic-debugging and dispatching-parallel-agents (#74)
- subagent-driven-development needs dispatching-parallel-agents reference (#71)

## Files to Read First

1. `docs/specs/2026-07-04-superpowers-fork-integration.md` — the authoritative spec
2. The 5 completed SKILL.md rewrites — they establish the pattern
3. `docs/protocols/externalised-scripts-require-tests.md` — applies to #71 (SDD scripts)
