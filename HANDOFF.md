# HANDOFF — 2026-07-04

## Last Session

Superpowers fork integration into soredium — epic #68. Completed steps 1-4 of 14.

## What Was Done

**Analysis:** Discovered all 14 original obra/superpowers skills exist in soredium
as untracked files (never committed). Mapped each skill's intent, gaps, and
integration points with soredium's ecosystem.

**Design review:** Spec at `docs/specs/2026-07-04-superpowers-fork-integration.md`
went through adversarial review (3 rounds, 23 issues, 21 verified, 2 accepted,
$14.01). Review artifacts at `~/adr/hortora-soredium/superpowers-fork-integration-20260704-220319/`.

**Completed rewrites (committed as 34febb8, Refs #68):**
1. `ide-tooling/SKILL.md` — restructured around capability layers (Navigate, Read,
   Edit, Refactor, Verify, Project). New structural editing tools documented.
2. `using-superpowers/SKILL.md` — all 5 process skills named with gates, common
   flows, lifecycle integration. Stale references/ directory removed.
3. `verification-before-completion/SKILL.md` — Skill Chaining section added.
4. `test-driven-development/SKILL.md` + `testing-anti-patterns.md` — made
   language-neutral, dot→Mermaid, Skill Chaining as process layer.

## Immediate Next Step

Continue epic #68 — steps 5-14. Start with #69 (systematic-debugging +
dispatching-parallel-agents).

## Remaining Issues

| Issue | Title | What to do |
|-------|-------|------------|
| #69 | Rewrite systematic-debugging + dispatching-parallel-agents | Debugging toolkit. See spec §10 and §8 |
| #70 | Rewrite brainstorming + writing-plans | Design pipeline. See spec §1 and §2 |
| #71 | Rewrite subagent-driven-development + executing-plans | Execution pipeline. See spec §3 and §4. Scripts need tests per protocol |
| #72 | Rewrite requesting-code-review + receiving-code-review | Quality layer. See spec §6 and §7 |
| #73 | Rewrite using-git-worktrees + writing-skills | See spec §11 and §13 |
| #74 | Audit work-end + soredium-native cross-refs + artifacts | See spec §12 + Changes to soredium-native skills + Impact sections |

## Key Context for Continuation

**The spec is the guide.** `docs/specs/2026-07-04-superpowers-fork-integration.md`
has the per-skill rewrite approach for every remaining skill, including gaps,
supporting files, and rewrite conventions.

**Rewrite conventions (from spec):**
- Bare skill names (no `hortora:` prefix)
- Mermaid `flowchart TD` (not dot digraph)
- Supporting files (prompt templates, technique docs, scripts) in scope
- ide-tooling reference for IntelliJ capabilities; individual skills don't handle fallback
- Focal issue = descriptive (the issue currently being worked on from the branch's issue group)

**New IntelliJ editing tools** (not yet visible in all sessions):
- `ide_edit_member` — replace entire member (signature + body)
- `ide_replace_member` — replace body only, signature preserved
- `ide_insert_member` — insert new member at structural position
- `ide_file_structure` now returns `endLine` per member

**Pattern established by completed rewrites:**
- Each skill has a Skill Chaining section listing what invokes it and what it complements
- Process skills reference ide-tooling for IDE capabilities
- Language-specific content deferred to language dev skills (java-dev, ts-dev, python-dev)
- Garden approaches referenced as foundational reading where relevant

**User feedback baked into spec:**
- Branch covers 1..n issues; each activity has a focal issue from the group
- IntelliJ structural editing is default for code authoring (preference hierarchy in ide-tooling)
- ide-tooling restructured by capability layer, not MCP server identity

## Files to Read First

1. `docs/specs/2026-07-04-superpowers-fork-integration.md` — the authoritative spec
2. The 4 completed SKILL.md rewrites — they establish the pattern
3. `docs/protocols/externalised-scripts-require-tests.md` — applies to #71 (SDD scripts)
