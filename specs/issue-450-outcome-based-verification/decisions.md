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
