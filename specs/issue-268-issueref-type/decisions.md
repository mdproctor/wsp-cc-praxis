# Decisions — IssueRef type (#268)

## D1: Scope

**Choice:** Full stack — plan_manager.py + events.py + commands + work_health.py + work_chain.py
**Alternatives:**
- Core only (plan_manager + direct consumers) — delays events.py, leaves bare ints in TUI/CLI contract
- plan_manager only — minimises change but perpetuates the split in downstream code
**Rationale:** One pass eliminates all bare `int` issue references. No follow-up issue needed. The event types are the public contract — they should carry `IssueRef` strings from the start.
**Trade-offs:** Larger change surface. More test updates. But the change is mechanical, not architectural.
**Sources:** issue #268 body, plan_manager.py, events.py, work_health.py, commands/next.py
**Exploration:** quick
**Status:** captured

## D2: IssueRef implementation

**Choice:** Frozen dataclass with construction-time validation
**Alternatives:**
- NewType wrapper around string — no construction validation, relies on convention
- Validation on existing fields (__post_init__ on QueueItem/LeafItem) — scatters the invariant across multiple classes
**Rationale:** Frozen dataclass gives immutability + hash/equality for free. Construction-time validation means invalid refs can't exist — caught at the source. `parse()` classmethod gives a single parse point replacing scattered regex matching.
**Trade-offs:** 73+ references to update in plan_manager.py alone, plus downstream. But this is mechanical.
**Sources:** issue #268 body (proposed IssueRef design), GE-20260811-7e119c (cross-repo resolution bug)
**Exploration:** quick
**Status:** captured

## D3: Backward compatibility — hard fail

**Choice:** Parser raises on bare `#N` in .plan files — forces all .plan files to be repo-qualified
**Alternatives:**
- Warn and backfill from issue-repo — allows gradual migration but preserves ambiguity
- Silent backfill — same as today with IssueRef wrapping
**Rationale:** Clean break. No ambiguity about which repo an issue belongs to. Existing .plan files are transient (created per branch, deleted at close) — there's no long-lived migration concern.
**Trade-offs:** Any active .plan with bare numbers will fail to parse until repo-qualified. In practice these are rare — `_write_item()` already requires repo since the write guard was added.
**Sources:** issue #268 body
**Exploration:** quick
**Status:** captured

## D4: Serialisation in events.py

**Choice:** String field — `issue: "Hortora/soredium#268"` matching `IssueRef.__str__()`
**Alternatives:**
- Nested object `{repo, number}` — structured but verbose, breaks existing TUI consumers
- Two fields `issue: 268, issue_repo: "Hortora/soredium"` — backward compatible but perpetuates the split
**Rationale:** Single field, matches the canonical string representation. TUI deserialises with `IssueRef.parse()`. Clean contract.
**Trade-offs:** Breaks TUI consumers that expect bare `int` — but those need updating anyway for cross-repo correctness.
**Sources:** events.py current structure
**Exploration:** quick
**Status:** captured

## D5: Git hook enforcement

**Choice:** Pre-commit hook validates .plan file content — every queue line must have `owner/repo#N`
**Alternatives:**
- Both pre-commit + PreToolUse hook — redundant if pre-commit catches everything
- PreToolUse only — doesn't catch git operations outside Claude
**Rationale:** Pre-commit catches any source of bad data, including manual edits that bypass the lifecycle guard. Runs on every commit in the workspace repo where .plan lives.
**Trade-offs:** Adds a hook to the workspace repo. But .plan format validation is fast (regex scan) and the hook already exists for lifecycle guarding.
**Sources:** User request, existing lifecycle guard hook pattern
**Exploration:** quick
**Status:** captured
