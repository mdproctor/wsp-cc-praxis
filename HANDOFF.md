# HANDOFF — 2026-07-05

## Last Session

Superpowers fork integration — epic #68 complete, all issues closed.
Follow-up verification filed as epic #75 with three child issues.

## What Was Done

### Epic #68 — Superpowers fork integration (CLOSED)

Rewrote all 14 original obra/superpowers skills as first-class soredium
citizens. Two sessions, 7 commits:

| Commit | Issue | Skills |
|--------|-------|--------|
| `34febb8` | #68 | ide-tooling, using-superpowers, verification-before-completion, test-driven-development |
| `920e103` | #69 | systematic-debugging, dispatching-parallel-agents |
| `db8e470` | #70 | brainstorming, writing-plans |
| `0820e1c` | #71 | subagent-driven-development, executing-plans |
| `664ac74` | #72 | requesting-code-review, receiving-code-review |
| `1501a60` | #73 | using-git-worktrees, writing-skills |
| `b6d55e7` | #74 | cross-refs, CLAUDE.md, marketplace.json, ARC42STORIES.MD |

Soredium-native skills also updated: code-review, fix-ci, java-dev,
ts-dev, python-dev. work-end audited against finishing-a-development-branch
(no gaps). Four-phase pipeline spec updated. Comment filed on #66.

### What wasn't done

- No automated tests for the new skills (structural validators pass,
  but no functional or consistency tests)
- No fit-gap analysis against original obra/superpowers
- No alignment audit to find competing/overlapping areas
- README.md still describes superpowers as separate external tools
- Stale `superpowers:` references remain in test fixtures
- SDD scripts (review-package, task-brief, sdd-workspace) are bash, not
  Python as spec assumed — externalised-scripts protocol doesn't apply
- Local commits not yet pushed to origin

## Immediate Next — Epic #75

Post-integration verification. Three issues, in this order:

### 1. #76 — Functional testing + consistency checks

Do this first — it catches mechanical errors before deeper analysis.

- **Cross-reference consistency test** (`tests/test_skill_chaining.py`):
  parse every SKILL.md's Skill Chaining section, verify bidirectionality.
  If A says "Complements: B", B must mention A.
- **Marketplace completeness test** (`tests/test_marketplace.py`):
  every committed non-DEV-ONLY skill has a marketplace.json entry and
  vice versa.
- **sync-local smoke test**: run sync-local, verify all 13 new skills
  appear in `~/.claude/skills/`, invoke each in a fresh session.
- **Stale fixture cleanup**: `tests/regression/issue-001-cso-violation.json`
  references `superpowers:executing-plans`, `tests/test_validate_naming.py:93`
  tests `superpowers:brainstorming` — update both.
- **README.md update**: the Superpowers Integration table still frames
  these as separate external tools. Rewrite to integrate into the main
  skill tables.

### 2. #77 — Fit-gap analysis against original obra/superpowers

Do this second — it catches dropped content.

For each of the 13 skills:
1. Read the original obra/superpowers version (untracked files before
   the integration commits — check git stash or the state before
   `34febb8`)
2. Read the soredium rewrite
3. Compare section by section: methodology preserved? Content dropped?
   Behavioral intent changed?
4. Classify each gap: intentional removal (spec said to), intentional
   simplification (folded into CLAUDE.md), or unintentional drop

Also check supporting files: technique docs, prompt templates, scripts,
examples. A table per skill is the deliverable.

### 3. #78 — Alignment and unification audit

Do this last — it requires understanding both layers to see the seams.

Seven areas to examine:

1. **Review pipeline overlap** — code-review vs requesting-code-review
   vs design-review. Three review mechanisms. Clear to a user? Could
   any merge?
2. **Execution pipeline vs work lifecycle** — does work-start know
   about SDD? Is the handoff from writing-plans to SDD smooth?
3. **Debugging toolkit vs fix-ci** — clear routing? Does fix-ci
   duplicate systematic-debugging for CI-specific cases?
4. **Testing discipline layering** — TDD (process) vs language skills
   (framework) vs garden testing.md (principles). Is the layering
   clear or does content duplicate?
5. **Verification gate count** — a single SDD task passes through 5
   gates (TDD + self-review + task-review + VBC + final-review). Is
   this proportionate or redundant?
6. **Knowledge integration** — should more skills search the garden?
   Is the forage/protocol split clear?
7. **Workspace concepts** — four isolation-related concepts
   (workspace-init, using-git-worktrees, work-start, EnterWorktree).
   Can any merge?

Deliverable: current state, friction points, unification opportunities,
recommendation per area. Opportunities that survive scrutiny become
child issues.

## Key Context for Next Session

**Spec:** `docs/specs/2026-07-04-superpowers-fork-integration.md` —
the authoritative guide for what was done and why.

**ADR project CLAUDE.md:** `~/adr/CLAUDE.md` — has the agent constraint
design lessons (8 iterations, documented in detail) that apply to any
future multi-agent review work.

**Soredium repo state:** main branch, ahead of origin by ~7 commits
(not pushed). All 8 validators pass. 2077 tests pass (10 UI test
failures are pre-existing, unrelated).

**The rewrite conventions** (bare skill names, Mermaid flowcharts,
ide-tooling reference, focal issue, Skill Chaining format) are now
the standard for all soredium skills — not just the 13 new ones.
