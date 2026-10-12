# Decisions — Robust Pipeline Architecture (#443)

## D1: Migration approach

**Choice:** Evolve existing infrastructure
**Alternatives:**
- Clean-room pipeline — higher upfront cost, long integration tail, large blast radius
- Layered facade — wraps old code without changing it, but adds indirection without fixing root causes
**Rationale:** The primitives exist (worklog DB events table, lifecycle.py state machine with CAS, RepoState classification). The problem is they're bypassed, not missing. Evolution preserves working code and enables incremental migration where each step is independently shippable.
**Trade-offs:** Must maintain backward compatibility during migration. Coexistence of old and new patterns until migration completes.
**Sources:** worklog DB schema (events table), project/lifecycle.py (CAS writes), project/repo_state.py (classify_repo)
**Exploration:** quick
**Status:** captured

## D2: Unit of lifecycle tracking

**Choice:** Per-repo lifecycle
**Alternatives:**
- Work-item lifecycle — one lifecycle with per-repo sub-fields. Simpler coordination but the sub-state model is essentially per-repo anyway, just nested
- Single lifecycle (current) — one lifecycle per .plan with skip predicates for inactive repos. Doesn't fix the structural model mismatch (#437, #438, #431)
**Rationale:** Slots bundle multiple repos. Each repo has independent branch state, independent changes, independent landing. A slot-level coordinator checks "have ALL repos reached the gate?" before advancing. This naturally solves multi-repo scope violations (#437 sweep all repos, #438 archive partial, #431 not slot-aware).
**Trade-offs:** More state to manage (N event streams per work item). Coordination logic needed at slot level.
**Sources:** Issues #437, #438, #431, #387, #368, #361 (multi-repo blindspot pattern)
**Exploration:** quick
**Status:** captured

## D3: State source of truth

**Choice:** Events primary, .plan derived
**Alternatives:**
- Single-writer .plan with events as audit — simpler migration but .plan still vulnerable to git corruption (rebase, merge conflicts) and has no schema enforcement
- Dual-source with reconciliation — both authoritative with conflict resolution. Most complex but most resilient
**Rationale:** Eliminates the multi-writer problem entirely — only the event emitter writes .plan (as a derived snapshot). Pipeline reads state from DB events only. Fresh clones bootstrap from .plan + start new event sequence. 
**Trade-offs:** DB must be available for pipeline decisions. If DB is lost, state history is gone (but .plan snapshot provides recovery point). DB is local — doesn't travel with repo clones.
**Sources:** Worklog DB events table (already captures transitions, just write-only). .plan file (7 writers, 28+ readers, 12+ issues). Issues #441, #434, #410, #398, #396, #392 (state management failures)
**Exploration:** quick
**Status:** captured
**Depends on:** D1 (evolve existing — the DB and lifecycle.py are the infrastructure being evolved)

## D4: LLM interaction model

**Choice:** Pipeline drives, LLM serves
**Alternatives:**
- Current model + typed contracts — keeps yield/callback but adds JSON schema contracts. Fixes symptom (LLM doesn't know what to return) without changing control flow. State-passing bugs remain
- Hybrid: mechanical Python, judgment markdown — cleaner boundary but state still passed by LLM
**Rationale:** Python always controls the loop. For judgment steps, Python sends a structured request (JudgmentRequest with JSON schema) and waits for a structured response (JudgmentResponse). LLM never calls the orchestrator. Eliminates state-passing bugs (#434), missing completion hints (#391), and loops (#393) entirely.
**Trade-offs:** Requires typed interfaces for every judgment step. Handler .md files still exist for guidance but the interface is a contract, not free text. Migration requires updating SKILL.md orchestrator loops.
**Sources:** Issues #434 (meta_state bounce), #391 (missing completion hint), #393 (step_done missing), #433 (unnecessary round-trips)
**Exploration:** quick
**Status:** captured
**Depends on:** D1 (evolve existing — modifies the orchestrator engine, not replaces it)

## D5: Verification model

**Choice:** Git state + events, three-valued (PASS/FAIL/UNKNOWN)
**Alternatives:**
- Git state only — simpler but no audit trail. If git history rewritten, no backup
- Stamps + git fallback — least disruptive but maintains the proxy layer that causes failures
**Rationale:** Verify actual git state (tree SHA match, commit reachable from main, issue closed on GitHub) AND event sequence (land event exists with SHA). Three-valued: PASS when positive evidence exists, FAIL when counter-evidence exists, UNKNOWN when insufficient evidence. UNKNOWN never collapses to PASS — escalates to LLM via JudgmentRequest.
**Trade-offs:** Stamp files eliminated — all consumers must migrate. Git state checks are slower than stamp file existence checks (subprocess vs stat). UNKNOWN escalation adds a judgment step where none existed.
**Sources:** Issues #436 (empty = success), #435 (stamp never written), #375 (negative check), #386 (skip path missing SHA), #388 (ordering bug)
**Exploration:** quick
**Status:** captured
**Depends on:** D3 (events primary — verification reads event log), D2 (per-repo — verification is per-repo)
