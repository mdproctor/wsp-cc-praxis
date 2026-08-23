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
