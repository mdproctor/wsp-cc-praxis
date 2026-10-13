# Decisions — Outcome-Based Verification (#450)

## D1: Verification layer unification

**Choice:** Unify into a single verification layer
**Alternatives:**
- Keep two layers, upgrade both — postconditions stay as fast-path (VerifyResult instead of bool), close-checks stay as reporting layer (Verification instead of Finding). Preserves separation but maintains the duplication that caused #436
- Replace postconditions, extend close-checks — eliminate postconditions, preserve Finding pattern for close-time reports. Hybrid approach but close-checks still diverge from the orchestrator's verification needs
**Rationale:** The two-layer split (step_postconditions.py returning bool, close_checks.py returning Finding) is historical accident. Both check overlapping things (content_landed, branch_stamp, etc.) and edge-case fixes need to happen in two places — exactly the problem that caused #436 (empty = success). A single verification/outcomes.py with verify_*() returning Verification gives the orchestrator PASS/FAIL/UNKNOWN for check-execute-verify, and close_verify.py composes the same functions for reporting.
**Trade-offs:** Migration touches both the orchestrator engine (postcondition_fn signature changes) and close_verify.py (swaps Finding composition for Verification composition). Larger blast radius per step, but each step is self-contained.
**Sources:** verification/step_postconditions.py (15 bool functions), verification/close_checks.py (Finding functions), verification/close_verify.py (composed reporter). Issues #436, #375, #435
**Exploration:** quick
**Status:** captured

## D2: UNKNOWN resolution ownership

**Choice:** Engine-level resolution
**Alternatives:**
- Per-check resolution — each verify_*() function embeds its own retry/fallback logic (e.g., verify_content_landed() does wait-and-recheck internally). Engine only sees PASS/FAIL after resolution. Simpler engine contract but scatters retry policies across N functions, making them harder to test and harder to change the retry policy
**Rationale:** Keep verify functions pure — they observe git/event state and report what they see (PASS/FAIL/UNKNOWN with evidence). The orchestrator engine's run_loop() handles UNKNOWN centrally: retry up to MAX_MECHANICAL_RETRIES, then yield to LLM via the existing ACTION=user_input pattern with UNKNOWN condition details. This centralizes retry/escalation policy in one place. The UNKNOWN conditions table from the parent spec (partial landing → retry unfinished repos, ambiguous branch state → reattach/abort, missing evidence → wait-and-recheck with backoff) maps to engine-level retry with condition-specific backoff.
**Trade-offs:** Engine becomes slightly more complex (three-way dispatch instead of binary). But verify functions become simpler and fully testable without mocking retries.
**Sources:** project/orchestrator_engine.py run_loop() (current binary dispatch), shared_steps.py MAX_MECHANICAL_RETRIES=3. Parent spec D5 (UNKNOWN conditions table)
**Exploration:** quick
**Status:** captured
**Depends on:** D1 (unified layer — engine consumes Verification, not bool)
