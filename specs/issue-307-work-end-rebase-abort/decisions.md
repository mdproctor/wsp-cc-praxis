## D1: Land postcondition location

**Choice:** Inside `land_flow.py:land_batch()`, after merge+push, before stamp
**Alternatives:**
- verify_fn on orchestrator's land step — lifecycle transitions fire before verify catches the problem
**Rationale:** Catches failure at the earliest point, before stamp, before lifecycle transitions, before the orchestrator can fall through
**Trade-offs:** land_flow.py gets slightly more complex (~10 lines)
**Sources:** work-end/land_flow.py:548-683, work-end/verify_stamp.py, work-end/work_end_orchestrator.py:831-840
**Exploration:** quick
**Status:** captured
