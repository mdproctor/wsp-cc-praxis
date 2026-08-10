# Design Journal — issue-202-work-command-taxonomy

## 2026-08-10 — Session 1

### Design decisions (D1-D6)

Brainstormed and captured 6 decisions for the work command taxonomy.
Three distinct verbs: `continue` (keep working), `resume` (pause-stack
only), `brief` (orientation). Design review ran at standard depth across
4 dimensions (coherence, structure, robustness, cross-cutting) — 59
issues raised, 48 verified, 3 accepted into spec, 8 deferred. $52.41.

Key spec refinements from review:
- D3 done-detection: mechanical definition uses `check_plan_state` in
  entry scope (not a new GitHub API call)
- D4 wrong-context: added `workspace_dirty` and `resume on main with
  no stack` error cases
- `work_continue` lifecycle event: self-transition for observability

### Phase 1 implementation (complete)

7 commits landed:
1. `work_continue` self-transition in lifecycle.py
2. Word-boundary fix for handoff issue matching (#42 vs #421)
3. `check_plan_state` in entry scope when `owner_repo` provided
4. work/SKILL.md: routing table, Step 1c errors, Step 4 continue menu
5. work-resume CSO: pause-stack only
6. handover: continue auto-reads HANDOFF.md
7. README sync

### Phase 2 progress (in progress)

ctx.py refactored to expose `resolve(cwd=None) -> dict`. 106 tests
passing, CLI backward-compatible. Remaining: `brief.py` aggregator,
`brief/SKILL.md`, slash command, validation.
