# Decisions — #271 Mechanise work-end close sequence

## D1: Orchestration model

**Choice:** Stateless re-entrant Python script
**Alternatives:**
- Long-running coroutine (stdin/stdout streaming) — fights Claude Code's execution model; no bidirectional streaming support
- Claude Code Workflow (JavaScript) — wrong layer; this is a single-agent skill, not multi-agent orchestration
**Rationale:** Each invocation reads `.close-progress`, runs mechanical steps until a judgment point, prints one action, exits. The LLM calls it again after doing the work. Simple, crash-safe, matches how Claude Code already runs scripts.
**Trade-offs:** Re-entry overhead (re-reading progress, re-resolving paths) on each call — negligible vs correctness gains.
**Sources:** Issue #271, Claude Code Workflows docs, work-end restructure spec
**Exploration:** quick
**Status:** captured

## D2: Control inversion — LLM sees one action at a time

**Choice:** The SKILL.md is a ~20-line loop. The LLM calls the orchestrator, reads one ACTION, executes it, loops. The LLM never knows the full step sequence.
**Alternatives:**
- LLM reads full sequence from SKILL.md (current model) — LLM skips steps; 4/4 audited slots had failures
- LLM reads step list but scripts enforce order — partial fix; LLM can still skip calling scripts
**Rationale:** The LLM cannot skip what it cannot see. Python decides what's next. The 660-line SKILL.md becomes ~20 lines.
**Trade-offs:** The LLM loses situational awareness of the full close sequence. Mitigated by the orchestrator providing context in each action's output.
**Sources:** Issue #271 (audit evidence), Oracle AI Agent Loop, AgenticStateMachines (GitHub), Google Cloud agentic AI patterns
**Exploration:** quick
**Status:** captured

## D3: Action granularity — coarse actions, fine-grained progress

**Choice:** 5 coarse LLM actions internally (review, sweep, squash, trajectory, user_input) plus mechanical steps between them. Fine-grained sub-step tracking in `.close-progress` for crash recovery.
**Alternatives:**
- 11+ fine-grained actions (one per judgment step) — more round-trips, more orchestrator complexity, no proportional safety gain
- Monolithic single action (LLM does everything in one turn) — no validation checkpoints
**Rationale:** Coarse actions minimise LLM round-trips. Fine-grained progress enables resume from any sub-step after crash. Orchestrator validates outputs after each coarse action — catches skips within groups.
**Trade-offs:** Less granular LLM validation per sub-step. Mitigated by mechanical validation of outputs (file exists, findings resolved, plan JSON valid).
**Sources:** Work-end SKILL.md current step inventory
**Exploration:** quick
**Status:** captured

## D4: Sweep ownership — LLM-owned, orchestrator-validated

**Choice:** The orchestrator yields `ACTION=sweep`. The LLM owns the toggle UI, item ordering, and execution of all sweep sub-steps (forage, protocol, update-claude-md, impl-doc-sync, ADR, write-content). The orchestrator validates outputs on return (blog file exists, etc.) and re-yields if validation fails.
**Alternatives:**
- Orchestrator micro-manages sweep sub-steps (6 separate yield points) — unnecessary; sweep is pure judgment work
- Orchestrator presents toggle UI via structured output — over-engineering; LLM already handles this naturally
**Rationale:** Sweep is entirely creative/judgment work. The LLM already sets up defaults and handles the user interaction. The only thing the orchestrator adds is the completion gate — the LLM can't move past sweep until outputs exist.
**Trade-offs:** LLM could skip individual sweep sub-steps. Mitigated by output validation (blog file, garden entries, etc.) and re-yield on failure.
**Sources:** Current SKILL.md Step 3 sweep implementation
**Exploration:** quick
**Status:** captured

## D5: Validation stack — mechanical first, Haiku secondary

**Choice:** Python structural checks (file exists, frontmatter valid, JSON schema, git status, findings resolved) as the primary validation layer. Haiku via Vertex AI as a secondary semantic validation layer, used only where mechanical checks are proven insufficient.
**Alternatives:**
- Haiku validation for every step — unnecessary cost and latency for steps that are mechanically validatable
- No semantic validation (mechanical only) — misses content quality issues (placeholder blog entries, hallucinated findings)
**Rationale:** Most judgment steps produce mechanically validatable artifacts. Haiku earns its place only where "file exists" doesn't mean "file is useful" — likely write-content, possibly ADR. Proven during implementation, not assumed upfront.
**Trade-offs:** May miss some content quality issues with mechanical-only validation. Acceptable — the current system has zero validation.
**Sources:** Vertex AI Haiku availability in current environment
**Exploration:** quick
**Status:** captured

## D7: Protocol format — action-specific context

**Choice:** KEY=VALUE lines with action-specific context only. The orchestrator provides values the LLM needs to execute THIS action correctly (diff range, repo list, resume point). Does not repeat generic context already in the conversation (project path, workspace path, branch name).
**Alternatives:**
- Full context every time (Option A) — verbose, repeats what the LLM already knows
- Minimal ACTION + RESUME_FROM only (Option C pure) — risks LLM hallucinating action-specific values like diff ranges or repo lists
**Rationale:** The LLM can't be trusted to derive action-specific values correctly. The orchestrator has computed them from progress state and ctx.py. Generic paths are already in conversation context and don't need repeating. KEY=VALUE format is consistent with the soredium script ecosystem.
**Trade-offs:** Requires judgment about what's "action-specific" vs "already known" — but the rule is clear: if getting it wrong causes a failure, include it.
**Sources:** ctx.py output format, existing soredium script conventions
**Exploration:** quick
**Status:** captured

## D8: Progress file format

**Choice:** Flat KEY=VALUE, one line per sub-step, append-only. Same convention as `.execute-progress`.
**Alternatives:**
- JSON object — easier atomic read/write but inconsistent with existing `.execute-progress` format
**Rationale:** Append-only is crash-safe (partial writes don't corrupt earlier entries). Consistent with the pattern already used by `work_end_execute.py`. Simple to parse with grep or Python.
**Trade-offs:** Append-only means the file grows — but work-end has ~20 sub-steps max, so size is irrelevant.
**Sources:** work-end/work_end_execute.py `.execute-progress` format
**Exploration:** quick
**Status:** captured

## D9: Slot mode handling — unified sequence

**Choice:** Single orchestrator, same sequence for slots and non-slots. Slot-specific mechanics (two-hop transport, merge into original, `.landed` marker) are already encapsulated in the existing scripts (`land_flow.py`, `slot_manager.py`). The orchestrator calls the same scripts — they handle the mode difference internally.
**Alternatives:**
- Two orchestrator scripts (slot vs non-slot) — duplicates sequence logic for no gain
- Slot-aware branching in orchestrator — unnecessary; the scripts already handle this
**Rationale:** Slots and non-slots follow the same sequence. The only difference is the extra merge into the original repo, which is already handled by the landing scripts. The orchestrator doesn't need to know.
**Trade-offs:** None — this is the simpler and more correct approach.
**Sources:** land_flow.py, slot_manager.py, prior unification work
**Exploration:** quick
**Status:** captured

## D10: Error and retry policy

**Choice:** Re-yield with reason, max 3 attempts, then skip-with-warning. The orchestrator re-yields the same action with `REASON=` explaining what failed validation. After 3 failures, logs a warning and continues.
**Alternatives:**
- Re-yield indefinitely — risks infinite loops
- Hard stop after 3 — too disruptive; forces manual intervention
**Rationale:** Matches the existing non-blocking philosophy. The orchestrator records the skip in `.close-progress` so verify can catch it downstream. Critical steps (review, verify) already have their own hard gates in the existing scripts.
**Trade-offs:** A skipped step could leave artifacts unpromoted or reviews incomplete. Mitigated by verify_slot_close.py catching gaps at the end.
**Sources:** Current work-end non-blocking patterns (trajectory, forage)
**Exploration:** quick
**Status:** captured

## D6: Existing scripts reused as-is

**Choice:** The orchestrator calls the existing 17 work-end Python scripts without modification. It is a wrapper/sequencer, not a rewrite.
**Alternatives:**
- Rewrite scripts into monolithic orchestrator — high risk, no gain; scripts already work correctly
- Partial rewrite (merge some scripts) — premature; scripts are independently testable as-is
**Rationale:** The scripts are not the problem. The LLM skipping calls to scripts is the problem. The orchestrator adds sequencing and validation around scripts that already work.
**Trade-offs:** More subprocess calls (orchestrator → script → subprocess). Negligible latency.
**Sources:** work-end/*.py inventory (17 scripts), issue #271 audit evidence
**Exploration:** quick
**Status:** captured
