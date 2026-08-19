# Unified Review and Findings Persistence

**Date:** 2026-08-19
**Status:** Draft
**Decisions:** [decisions.md](decisions.md)

## Problem

Work-end misses gates. Code review is buried in Step 3 (Execute), after
the entire Sweep (forage, protocol, doc sync, write-content). The review
itself is a per-line checklist — it catches mutable defaults and unawaited
coroutines but not "you didn't implement requirement 3" or "this branch
has three deferred TODOs nobody tracked."

Findings are ephemeral. Code review findings, design review deferred
items, completeness audit gaps, and loose ends all die with the session.
The next session has no idea what was deferred, dismissed, or left
unfinished. Only hygiene findings (stale branches, unrecovered artifacts)
persist via `findings.json`.

The blog entries from 2026-08-17 ("The Feedback Loop That Wasn't" and
"Four Fixes, One Already Done") diagnosed this gap and identified the
architecture: extend `findings.json` to cover all finding categories,
with accumulation across sessions and a forcing function at work-end.

## Solution Overview

Six components:

1. **Branch audit** — new skill for holistic branch-level code review
2. **Loose ends sweep** — captures deferred/skipped/missing items at
   every lifecycle gate
3. **Findings persistence** — extends `findings.json` to all categories
4. **Forcing function** — drains all accumulated findings at work-end
5. **Lifecycle integration** — reorders work-end and adds sweep to handover
6. **Skill cleanup** — removes deprecated skills, preserves VBC as protocol

## Component 1 — Branch Audit

New skill `branch-audit`. Reviews the full branch diff across four
dimensions (D1, D6).

### Dimensions

Each dimension includes (but is not limited to) the listed sub-concerns.

**Conformance** — did we build what we said we'd build?

The reference document varies by context:
- With spec: implementation vs spec
- With issue only: implementation vs issue description and acceptance criteria
- With neither: implementation vs stated intent

Includes: issue conformance, completeness, missing requirements, gaps in
edge case coverage, untested scenarios, acceptance criteria not met.

**Coherence** — does the branch hold together as a whole?

Includes: internal consistency, uniform patterns across changes, naming
consistency, architectural fit with the rest of the codebase.

**Structure** — are the boundaries right?

Includes: decomposition, module boundaries, dependency cleanliness,
separation of concerns, file organisation.

**Robustness** — what could go wrong?

Includes: failure modes, error paths, surface-level security (auth,
input validation, data exposure), regression risk (downstream callers
affected), boundary conditions, silent failures.

When Robustness identifies security concerns (auth, PII, payment, user
input code), it offers to escalate to `security-audit` for the full
OWASP pass (D7).

### Execution Model

Runs inline in the current session. Single pass per dimension. No
external sessions, no adversarial rounds, no watchdog crons.

Produces findings with severity (CRITICAL / WARNING / NOTE). Unresolved
findings are written to `findings.json` (Component 3).

### Relationship to design-review

`design-review/review-tiers.md` planned a "Post-implementation" lifecycle
point with the same four dimensions. Branch-audit fills this gap with a
simpler execution model. `review-tiers.md` should be updated to reference
branch-audit for the post-implementation lifecycle point.

The four dimensions (Conformance, Coherence, Structure, Robustness) are
shared vocabulary between design-review and branch-audit. Cross-cutting
remains a design-review synthesis step and does not apply to branch-audit.

### Relationship to code-review

Both run at work-end (D10). code-review is the per-line checklist (safety,
types, async, testing, performance). Branch-audit is the holistic review
(conformance, coherence, structure, robustness). Different concerns, both
valuable at the final gate.

During development, only code-review runs (per-commit). Branch-audit
requires the full branch diff and only runs at lifecycle gates.

## Component 2 — Loose Ends Sweep

First-class concept for capturing unfinished work (D8). Distinct from
epic hygiene, which checks workspace/branch lifecycle state (branches
closed correctly, artifacts promoted, blogs published).

### What it checks

- Deferred plan items (from `.plan`)
- Skipped or deferred review findings from this session
- TODOs in code referencing this branch/issue
- Open findings from prior sessions (reads `findings.json`)
- Uncommitted changes
- Unresolved "I'll come back to this" items from conversation context

### Lifecycle behavior

**At handover:** capture and persist only. New findings written to
`findings.json`. No forcing function — session is ending, work continues.

**At work-end:** capture, persist, then force-resolve all accumulated
findings (Component 4).

### Relationship to epic hygiene

Same persistence mechanism (findings.json), different concerns:

| Check | Epic hygiene | Loose ends sweep |
|-------|-------------|-----------------|
| Branches closed correctly | Yes | No |
| Artifacts promoted | Yes | No |
| Main diverged from remote | Yes | No |
| Deferred review findings | No | Yes |
| Skipped plan items | No | Yes |
| TODOs referencing this branch | No | Yes |
| Open findings from prior sessions | No | Yes |

Same pipe, different taps.

## Component 3 — Findings Persistence

Extends the existing `$WORKSPACE/.audit/findings.json` architecture (D9).

### Current format (hygiene only)

```json
{
  "category": "hygiene",
  "check": "unrecovered_artifact",
  "detail": "blog entry on closed branch issue-200",
  "status": "open",
  "timestamp": "2026-08-17T14:30:00Z"
}
```

### Extended format

```json
{
  "category": "review|audit|loose-end|hygiene",
  "dimension": "conformance|coherence|structure|robustness|null",
  "severity": "critical|warning|note",
  "check": "missing-requirement|dead-code|deferred-plan-item|...",
  "detail": "Requirement 3 (user notifications) not implemented",
  "source": "branch-audit|code-review|loose-ends-sweep|hygiene-scan",
  "branch": "issue-123-feature-name",
  "status": "open|resolved|dismissed|filed",
  "resolution": "fixed in abc1234|filed as #456|dismissed: out of scope|null",
  "timestamp": "2026-08-19T10:00:00Z"
}
```

### Semantics

- **Dedup** by `(check, detail)` — same as current hygiene behavior
- **Accumulate** across sessions — findings persist until explicitly resolved
- **Read at session entry** — `work_health.py` `check_prior_findings()` surfaces open findings (already implemented for hygiene; same code path handles extended categories)
- **Resolution statuses:**
  - `resolved` — fixed in code (include commit SHA)
  - `filed` — created as GitHub issue (include issue number)
  - `dismissed` — not a real problem (include reason)

### Who writes

| Source | Category | When |
|--------|----------|------|
| `hygiene_scan.py` | `hygiene` | work-end, handover |
| `branch-audit` | `audit` | work-end |
| `code-review` | `review` | work-end (unresolved findings only) |
| loose ends sweep | `loose-end` | work-end, handover |

### Who reads

- `work_health.py` at session entry — surfaces all open findings
- Forcing function at work-end — presents all open findings for resolution
- Loose ends sweep — reads prior findings as part of its scan

## Component 4 — Forcing Function

Runs at work-end only (D4). Presents all accumulated findings from
`findings.json` and requires resolution for each.

### Presentation

```
Open findings — 9 items require resolution before branch close

AUDIT (branch-audit):
  1. [conformance/WARNING] Requirement 3 (notifications) not implemented
  2. [robustness/NOTE] No error handling for network timeout in sync.py

REVIEW (code-review):
  3. [WARNING] Mutable default argument in user_service.py:42

LOOSE-END:
  4. [WARNING] Plan item "add integration tests" deferred
  5. [NOTE] TODO on line 88 of handler.py: "come back to edge case"

HYGIENE:
  6. [WARNING] Blog entry on closed branch issue-200 never promoted

Prior sessions (accumulated):
  7. [WARNING] Deferred review finding from session 2026-08-16
  8. [NOTE] Skipped plan item from session 2026-08-15
  9. [NOTE] TODO referencing issue-123 in utils.py:14
```

### Resolution options per finding

| Option | What happens |
|--------|-------------|
| **Fix** | Fix the issue now. Finding status → `resolved`, resolution includes commit SHA |
| **File** | Create a GitHub issue. Finding status → `filed`, resolution includes issue number |
| **Dismiss** | Not a real problem. Finding status → `dismissed`, resolution includes reason |

No finding survives branch close with status `open`. The forcing function
is a hard gate — work-end cannot proceed to Execute until all findings
are resolved.

### Batch operations

For efficiency when many findings exist:
- "Fix all" — not available (each fix is different)
- "File all remaining" — creates one GitHub issue per finding
- "Dismiss all NOTEs" — dismisses all NOTE-severity findings with a
  blanket reason (user provides)

## Component 5 — Lifecycle Integration

### Work-end flow (D5)

```
Current:  Context → Sweep → Execute (review in 3.1) → Verify → Close
Proposed: Context → Review → Sweep → Execute → Verify → Close
```

**Step 2 — Review (new, before Sweep):**

```
2.1  code-review         per-line checklist on branch diff
2.2  branch-audit        four dimensions on branch diff
2.3  loose ends sweep    session + prior findings scan
2.4  forcing function    present all findings, require resolution
```

All four sub-steps are hard gates. Step 2 does not complete until
the forcing function has resolved all findings.

**Trade-off acknowledged:** Review runs without forage/protocol sweep
context. For per-line code-review this is negligible. For branch-audit's
holistic dimensions, a gotcha surfaced by forage could explain unusual
code. Accepted: the cost of reviewing without sweep context is lower
than the cost of sweeping before reviewing.

### Handover flow

Add loose ends sweep to the wrap checklist (Step 0), defaulting ON:

```
[x] 1  Loose ends sweep   capture deferred/skipped/missing items
[x] 2  Knowledge capture  (forage then protocol — sequential)
[x] 3  ADR               record architectural decisions
[x] 4  Doc sync           (update-claude-md then implementation-doc-sync)
[x] 5  write-content      capture branch narrative as diary entry
```

Loose ends sweep runs first (while session context is full). Captures
and persists to `findings.json`. No forcing function at handover —
capture only.

### Session entry

No changes needed. `work_health.py` `check_prior_findings()` already
reads `findings.json` and surfaces open findings. Extended format is
backward compatible — new fields are additive.

## Component 6 — Skill Cleanup

### Remove requesting-code-review (D2)

- Delete `requesting-code-review/` directory
- Update cross-references in: `code-review/SKILL.md`,
  `receiving-code-review/SKILL.md`, `subagent-driven-development/SKILL.md`,
  `CLAUDE.md`, `README.md`
- `receiving-code-review` stays — update its Skill Chaining to reference
  branch-audit instead

### Retire verification-before-completion (D3)

- Delete `verification-before-completion/` directory
- Create `docs/protocols/evidence-before-claims.md` preserving the
  principle: "Run the command. Read the output. THEN claim the result."
- Update cross-references in all 25 referencing files to point to the
  protocol
- `git-commit` and `executing-plans` reference the protocol directly
- The forcing function (Component 4) is additive — it handles finding
  resolution at work-end but does NOT replace the per-boundary evidence
  gate

### Update design-review

- Update `review-tiers.md` post-implementation lifecycle point to
  reference branch-audit
- Rename dimensions to match shared vocabulary:
  - Current: Coherence, Structure, Robustness
  - New: Conformance, Coherence, Structure, Robustness (add Conformance
    for post-implementation)
- No changes to execution model, adversarial machinery, or degree system

## Implementation Order

1. **Findings persistence** (Component 3) — extend `findings.json` format.
   Everything else writes to it.
2. **Loose ends sweep** (Component 2) — can ship independently. Writes to
   `findings.json`, runs at handover.
3. **Branch audit** (Component 1) — new skill with four dimensions.
4. **Forcing function** (Component 4) — reads `findings.json`, presents
   findings, requires resolution.
5. **Lifecycle integration** (Component 5) — reorder work-end, add sweep
   to handover.
6. **Skill cleanup** (Component 6) — remove deprecated skills, create
   protocol, update references.

Steps 1-2 can ship first and provide value (findings persist, surfaced
at session entry) before the full system lands.

## Success Criteria

- No findings survive branch close with status `open`
- Findings from session N are visible at session N+1 entry
- Code review runs before sweep in work-end
- Branch audit covers four dimensions on the full branch diff
- Loose ends sweep runs at every handover and work-end
- `requesting-code-review` and `verification-before-completion` removed
- VBC principle preserved as protocol, referenced by git-commit and
  executing-plans
- design-review dimensions aligned with branch-audit vocabulary

## References

- [decisions.md](decisions.md) — 10 design decisions with rationale
- work-end/SKILL.md — current work-end flow
- code-review/SKILL.md — per-line checklist
- design-review/SKILL.md — adversarial spec review
- design-review/review-tiers.md — lifecycle points and dimensions
- work-end/hygiene_scan.py — existing findings persistence
- project/work_health.py — existing findings surfacing
- Blog: 2026-08-17-mdp01 "The Feedback Loop That Wasn't"
- Blog: 2026-08-17-mdp03 "Four Fixes, One Already Done"
- handover/SKILL.md — epic hygiene checks
- verification-before-completion/SKILL.md — evidence-before-claims principle
- requesting-code-review/SKILL.md — deprecated skill to remove
