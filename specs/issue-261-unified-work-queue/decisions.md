# Decisions — #261 Unified Work Queue

## D1: Bidirectional chaining direction

**Choice:** Commands chain both forward (fallthrough) and backward (guard)
**Alternatives:**
- Forward-only chaining — simpler but allows skipping unfinished work
- No chaining (current) — commands error in wrong context, user must know the right one
**Rationale:** Ensures no matter which command is invoked, the user lands at the right place. Forward prevents dead ends, backward prevents skipping.
**Trade-offs:** More complex routing logic, but implemented in Python so it's testable
**Sources:** Conversation with user, current work/SKILL.md routing table
**Exploration:** deep-analysis
**Status:** captured

## D2: .plan on main

**Choice:** Allow .plan files on main, with the same queue semantics as branches
**Alternatives:**
- Branch-only .plan (current) — main has ad-hoc what-next via enrichment.py
- Separate main-queue mechanism — different format/logic for main vs branch
**Rationale:** Unifies queue semantics. Same commands work identically regardless of context. Eliminates the branch/main routing split.
**Trade-offs:** .plan lifecycle on main is different (no merge/stamp at close)
**Sources:** lifecycle.py audit, ctx.py audit, conversation
**Exploration:** deep-analysis
**Status:** captured

## D3: Chaining logic in Python, not LLM

**Choice:** All mechanical chaining decisions (forward/backward routing, guards, state checks) implemented in Python
**Alternatives:**
- LLM-driven routing (current) — skill instructions tell Claude what to check
- Hybrid — some checks in Python, some in skill
**Rationale:** Python is deterministic, testable, doesn't hallucinate. The LLM reads directives and executes them.
**Trade-offs:** More Python code to maintain, but dramatically more reliable
**Sources:** User directive, evidence-before-claims protocol
**Exploration:** quick
**Status:** captured
**Depends on:** D1 (chaining design determines what the Python needs to implement)

## D4: New `ended` state in lifecycle

**Choice:** Add `ended` state — work-end ceremony completed, .plan persists, can transition back to `active`
**Alternatives:**
- Reuse `idle` — but idle means "no .plan exists"
- Delete .plan after end — but then we lose queue history and can't chain to find
**Rationale:** `ended` is distinct from `idle`: the .plan still exists, ceremony is done, but new items can be added. The `ended → active` transition via `work_find` event enables the find chain.
**Trade-offs:** One more state in the machine, but semantically clean
**Sources:** lifecycle.py audit — `idle` is never written to .plan, `ended` can be
**Exploration:** deep-analysis
**Status:** captured
**Depends on:** D2 (.plan on main requires a state for "done but plan persists")

## D5: work-end on main

**Choice:** work-end works on main with same ceremony minus merge/stamp
**Alternatives:**
- Branch-only work-end (current) — requires a branch to close
- Separate main-close command — different ceremony for main
**Rationale:** Unifies the close experience. Closing issues, updating docs, creating blogs are not branch-specific. Only merge/stamp are branch-specific.
**Trade-offs:** work-end becomes context-aware (branch mode vs main mode). Several steps become conditional.
**Sources:** work-end/SKILL.md audit — 12 touchpoints identified
**Exploration:** deep-analysis
**Status:** captured
**Depends on:** D4 (main needs `ended` state since there's no branch to stamp)

## D6: `work find` as new verb

**Choice:** Add `work find` as a discovery/population command, separate from queue execution
**Alternatives:**
- Keep enrichment/what-next inline in `work` routing (current Step 2a)
- Auto-populate queue on every `work` invocation
**Rationale:** Separates "execute the queue" from "populate the queue." Clean chain terminal: end → find discovers new work and appends to .plan.
**Trade-offs:** One more verb to learn, but the chain means you're naturally guided to it
**Sources:** Conversation, current Step 2a logic
**Exploration:** quick
**Status:** captured

## D7: Corruption recovery in Python

**Choice:** Python-based triage that detects inconsistencies and outputs resolution directives
**Alternatives:**
- LLM-driven triage (current) — CorruptedState exception, LLM figures it out
- Hard fail on corruption — stop and require manual intervention
**Rationale:** Python can check state against reality (GitHub issue status, git branch existence, uncommitted changes) and propose specific resolutions. The LLM presents the resolution to the user.
**Trade-offs:** Need to enumerate corruption scenarios, but they're finite and testable
**Sources:** lifecycle.py audit — 5 exception types, stale .plan on main encountered this session
**Exploration:** deep-analysis
**Status:** captured
**Depends on:** D3 (Python-driven routing is the foundation for Python-driven recovery)
