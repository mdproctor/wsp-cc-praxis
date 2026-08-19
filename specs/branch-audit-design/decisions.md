# Decisions — Unified Review and Findings Persistence

## D1: New branch-audit skill

**Choice:** Create a new `branch-audit` skill for holistic branch-level review
**Alternatives:**
- Extend `code-review` with `--mode branch-audit` — mixes two different review approaches (line-level vs holistic) in one skill
- Extend `design-review` with `--type code` — design-review is adversarial spec review; wrong execution model (external sessions, watchdog crons, ~/reviews/ workspaces) and no ADR/debate needed for code
**Rationale:** Branch-level audit is a genuinely different kind of review from per-line checklist (code-review) and adversarial spec review (design-review). Clean separation of concerns.
**Trade-offs:** One more skill in the review space. Mitigated by removing two deprecated skills (D2, D3).
**Sources:** work-end/SKILL.md, code-review/SKILL.md, design-review/SKILL.md
**Exploration:** quick
**Status:** captured

## D2: Remove requesting-code-review

**Choice:** Remove the deprecated `requesting-code-review` skill entirely
**Alternatives:**
- Un-deprecate and evolve into branch-audit — name doesn't convey holistic audit, implementation is narrowly scoped (subagent dispatch template)
- Keep as backward-compat shim — dead code that misleads
**Rationale:** Deprecated, superseded by design-review --mode final-review (itself a poor fit). Gap now properly filled by branch-audit (D1).
**Trade-offs:** Skills referencing it need cross-reference updates.
**Sources:** requesting-code-review/SKILL.md (deprecated notice at line 10)
**Exploration:** quick
**Status:** captured

## D3: Retire verification-before-completion

**Choice:** Fold VBC principles into branch-audit forcing function and lifecycle gates, retire as standalone skill
**Alternatives:**
- Keep VBC as standalone — referenced by 18 files, but it's a behavioral discipline ("run the command, read the output"), not a review skill with steps or artifacts
**Rationale:** VBC's principle ("evidence before claims") is embedded in the forcing function's requirement that every finding shows resolution evidence. Having a separate skill for "don't lie about results" is redundant when the forcing function enforces it structurally.
**Trade-offs:** 18 files reference VBC and need updating. The principle must not be lost in the transition.
**Sources:** verification-before-completion/SKILL.md
**Exploration:** quick
**Status:** captured

## D4: Forcing function at work-end only

**Choice:** Handover captures and persists findings; work-end forces resolution of all accumulated findings
**Alternatives:**
- Same forcing function at both gates — too strict for handover (session ending, work continues)
- No forcing function anywhere — findings accumulate indefinitely with no resolution pressure
**Rationale:** Handover is mid-work — forcing resolution would block session wrap for findings that can't be fixed yet. Work-end is branch close — all accumulated findings must be addressed before merging. "Fix, file as issue, or dismiss with reason" — no finding survives branch close unaddressed.
**Trade-offs:** Findings can accumulate across many sessions before being forced. Mitigated by work_health.py surfacing them at every session entry.
**Sources:** work-end/SKILL.md, handover/SKILL.md, blog 2026-08-17-mdp01
**Exploration:** quick
**Status:** captured

## D5: Move review early in work-end

**Choice:** Review becomes Step 2 (right after Context), before Sweep
**Alternatives:**
- Keep review in Step 3 (Execute) — current position; code review buried after forage, protocol, doc sync, write-content
**Rationale:** Finding bugs after spending 10+ minutes on knowledge capture and blog writing wastes time. Review first, then sweep. Bugs found early can be fixed before sweep captures the session narrative.
**Trade-offs:** None identified — strictly better ordering.
**Sources:** work-end/SKILL.md (current Step 3.1)
**Exploration:** quick
**Status:** captured

## D6: Four shared review dimensions

**Choice:** Conformance, Coherence, Structure, Robustness — same vocabulary for design-review and branch-audit
**Alternatives:**
- Three dimensions (collapse Structure into Coherence) — loses decomposition as first-order concern for spec review
- Different dimensions per review type — vocabulary diverges, findings harder to compare across lifecycle
**Rationale:** Structure asks "are boundaries right?" (decomposition). Coherence asks "does it hold together?" (composition). Opposite directions of the same concern — both deserve explicit attention. For code review, Structure is lighter but still worth checking.
**Trade-offs:** Four dimensions is more than three — slightly heavier per review. Structure will often have fewer findings for code reviews than spec reviews.
**Sources:** design-review/SKILL.md (current dimensions: coherence, structure, robustness)
**Exploration:** quick
**Status:** captured

## D7: Security-audit stays as deep-dive

**Choice:** Robustness dimension does surface-level security; escalates to security-audit for full OWASP pass
**Alternatives:**
- Fold security-audit into Robustness entirely — loses the deep OWASP checklist
- Keep security fully separate from Robustness — fragments the review
**Rationale:** Same pattern as current code-review ("offers security-audit for auth/PII code"). Surface-level security is part of every review. Full OWASP audit is opt-in when warranted.
**Trade-offs:** Two-tier security review could miss things if Robustness doesn't escalate when it should. Mitigated by explicit escalation triggers (auth, PII, payment, user input).
**Sources:** security-audit/SKILL.md, code-review/SKILL.md
**Exploration:** quick
**Status:** captured

## D8: Loose ends sweep is first-class

**Choice:** Separate first-class concept — runs at every handover (capture + persist) and work-end (capture + force-resolve)
**Alternatives:**
- Fold into branch-audit as a dimension — loose ends are session state, not code quality; branch-audit reviews the diff, can't see "a review finding was deferred 3 hours ago"
- Fold into work-end only — misses capture at handover, losing accumulation across sessions
**Rationale:** Loose ends sweep reviews session/project state for unfinished work. Branch-audit reviews code quality in the diff. Different inputs, different concerns. Both feed into the same findings.json persistence layer.
**Trade-offs:** One more concept to invoke at lifecycle gates. Mitigated by both handover and work-end orchestrating it as a standard step.
**Sources:** blog 2026-08-17-mdp01 ("ephemeral findings"), blog 2026-08-17-mdp03 ("first piece of the broader persistent audit system"), hygiene_scan.py, work_health.py
**Exploration:** quick
**Status:** captured
