# Decisions — Classify-Once RepoContext (#449)

## D1: Where to place get_lifecycle_state()

**Choice:** Add to worklog.py
**Alternatives:**
- project/lifecycle.py — closer to state machine but lifecycle.py is about transitions, not queries; would need connection management
- New project/lifecycle_query.py — clean separation but adds indirection for a single function
**Rationale:** worklog.py is the DB access layer — all DB reads/writes belong there. Query functions like active_work() and find_work_item() already live there. Adding get_lifecycle_state() follows the same pattern.
**Trade-offs:** worklog.py lives in ~/.claude/lib/ (shared across projects). Changes there affect all projects. But this is a read-only query — low risk.
**Sources:** worklog.py (line 749: active_work, line 200: find_work_item), lifecycle.py (line 286: _emit_to_worklog already imports worklog)
**Exploration:** quick
**Status:** captured

## D2: RepoContext construction approach

**Choice:** Two-phase construction — classify then enrich
**Alternatives:**
- Extend classify_repo() to return RepoContext directly — fewer subprocess calls but couples classification to DB access, violates single-responsibility
- RepoContext.from_state() classmethod — OOP-style, similar to chosen approach but less explicit
**Rationale:** Keeps classify_repo() focused on pure git classification. build_repo_context() wraps RepoState + computes tree_sha, main_tree_sha, remote_synced + queries lifecycle_state from DB. Callers who don't need lifecycle_state can skip the DB query.
**Trade-offs:** Two function calls instead of one. Extra git subprocess calls for tree SHAs (but these are cheap local operations).
**Sources:** repo_state.py (classify_repo lines 47-138), parent spec Phase 2 design (RepoContext extends RepoState via composition)
**Exploration:** quick
**Status:** captured
**Depends on:** D1 (get_lifecycle_state lives in worklog.py, called by build_repo_context)

## D3: OrchestratorContext.repos type change

**Choice:** Change repos: list[RepoState] → repos: list[RepoContext]
**Alternatives:**
- Keep list[RepoState], add parallel contexts: dict[str, RepoContext] — dual data structures with drift risk
**Rationale:** Single source of truth. RepoContext wraps RepoState via composition (context.state.disposition), so existing consumers update mechanically: r.disposition → r.state.disposition, r.path → r.state.path. No dual structures, no divergence risk.
**Trade-offs:** All existing RepoState access paths in the orchestrator must be updated. Mechanical but touches many lines.
**Sources:** work_end_orchestrator.py (OrchestratorContext line 134, repos usage at lines 277, 321, 1561, 1734, 1866)
**Exploration:** quick
**Status:** captured
**Depends on:** D2 (build_repo_context produces the RepoContext instances stored in repos)

## D4: lifecycle_state dependency handling

**Choice:** Build Phase 1b (get_lifecycle_state) as part of this issue
**Alternatives:**
- Defer lifecycle_state field — build RepoContext without it, add when Phase 1b lands later. Keeps Phase 2 unblocked but incomplete.
- Stub from .plan — read lifecycle_state from .plan's state: field as temporary bridge. Adds migration step later.
**Rationale:** User preference. The DB query function is small scope — one function in worklog.py. Building it now makes RepoContext complete from day one.
**Trade-offs:** Increases issue scope. log_transition is still @safe in worklog.py — writes can silently fail while reads succeed, creating potential stale-state window. Acceptable because lifecycle_state is informational in Phase 2 (not a control-flow gate).
**Sources:** worklog.py events table schema (line 66-78), parent spec Phase 1b
**Exploration:** quick
**Status:** captured

## D5: Step 2d scope — skip predicates only

**Choice:** Replace skip predicates and fallback git-probe path with disposition checks; postconditions deferred to Phase 3
**Alternatives:**
- Skip predicates + postconditions — also refactor postconditions to use disposition where applicable. Increases scope but reduces Phase 3 surface area.
- Minimal (fallback path only) — only eliminate the git-probe fallback at lines 1770-1800. Leaves existing skip predicates unchanged.
**Rationale:** Skip predicates are the BEFORE-execution checks that should use classify-once data. Postconditions are AFTER-execution checks that verify outcomes — Phase 3 replaces them with VerifyResult three-valued logic. Mixing the two blurs the phase boundary.
**Trade-offs:** Postconditions continue to shell out to git independently until Phase 3. Acceptable — their correctness doesn't depend on classification.
**Sources:** step_postconditions.py (12 functions), _skip_empty_branch (line 315), _skip_multi_repo_partial_close (line 264), _close_per_repo_mechanical fallback (lines 1770-1800)
**Exploration:** quick
**Status:** captured

## D6: Empty repos handling

**Choice:** Hard error if classify_repos returns empty for non-main pipeline
**Alternatives:**
- Keep fallback as safety net — defensive, but contradicts classify-once philosophy
- Warn and skip — no git fallback, no hard error, but silently drops steps
**Rationale:** The classify-once contract means repos are always populated at pipeline entry. An empty list in a non-main pipeline is a classification bug that should be caught immediately, not papered over with git fallback probes.
**Trade-offs:** Must ensure classify_repos is robust. A classification failure becomes a pipeline failure instead of degraded operation.
**Sources:** _close_per_repo_mechanical fallback path (lines 1770-1800), classify_repos (repo_state.py lines 141-168)
**Exploration:** quick
**Status:** captured
**Depends on:** D3 (repos: list[RepoContext] — empty check is on this list)

## D7: Disposition filtering on StepDef

**Choice:** Separate dispositions: frozenset[RepoDisposition] field on StepDef
**Alternatives:**
- Encode in skip_fn — keep modes as-is, per-step skip_fn checks disposition. Simple but distributed across lambda closures instead of declarative.
- Unified filter — replace both modes and skip_fn with a single predicate system. Most elegant but highest refactoring cost and risk.
**Rationale:** PipelineMode (BRANCH/SLOT/MAIN/SYNC/CYCLE) and RepoDisposition (HAS_WORK/EMPTY_BRANCH/NO_BRANCH/ALREADY_LANDED) are orthogonal concepts. Separate fields keep them orthogonal. Engine evaluates modes first (pipeline-level), then dispositions (repo-level) — both declarative, no skip_fn needed for these concerns.
**Trade-offs:** Adds a field to StepDef. Engine run_loop needs updating to evaluate dispositions. Per-repo steps may need different disposition semantics than single-step disposition checks.
**Sources:** shared_steps.py StepDef (line 31, modes field at line 43), RepoDisposition enum (repo_state.py line 22)
**Exploration:** quick
**Status:** captured
