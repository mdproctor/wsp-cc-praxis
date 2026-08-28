## D1: Module boundaries

**Choice:** 11-module extraction matching natural call-graph clusters
**Alternatives:**
- 6-module coarser split — fewer files but `slot_infra.py` would be ~400 lines with 4 tangled concerns, recreating the original problem at smaller scale
- 8-module hybrid — merges maven+ISX into `slot_infra.py`, reasonable but ISX has its own lifecycle distinct from build tooling
**Rationale:** Each module should be small enough to reason about independently. 11 modules where each has a single clear purpose prevents re-accumulation. The extra files are cheap — clear names and focused imports.
**Trade-offs:** More import statements across the codebase; slightly more files to navigate
**Sources:** work-slot/slot_manager.py call graph audit, issue #306 responsibility inventory
**Exploration:** quick
**Status:** captured
