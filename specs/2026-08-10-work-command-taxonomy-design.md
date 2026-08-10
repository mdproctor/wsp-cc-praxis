# Work Command Taxonomy — Design Spec

**Issue:** #202
**Date:** 2026-08-10
**Status:** Draft
**Scope:** work lifecycle commands, `/brief` orientation, HANDOFF.md integration
**Related Specs:** `2026-08-06-unified-work-state-design.md` (#188) — that spec's
"stop caching, start deriving" principle is complementary: `continue` loads
HANDOFF.md for its narrative value (reasoning, failed attempts), not for state.
Issue progress, branch status, and queue position derive from their authoritative
sources (`.plan`, GitHub, git).

---

## Problem Statement

Three issues with the current work lifecycle commands:

1. **"resume" is semantically overloaded** — used for both pause-stack restoration
   (`work-resume` skill, lifecycle event `work_resume`) AND reading HANDOFF.md to
   continue on an existing branch (Step 4 option in `work/SKILL.md`). Two different
   operations, one word.

2. **"start" appears when work is already started** — after `work start` in one
   session, `/clear`, then `/work`, Step 4 shows "start — begin working (first
   session on this branch)" because `HAS_HANDOFF=no`. The label is wrong — the
   branch exists, scaffold exists, work has been done. The underlying behavior
   is correct (runs the resume path), but the label misleads.

3. **No "continue" concept** — the distinction between "resume" (has HANDOFF.md)
   and "start" (no HANDOFF.md) in Step 4 is an implementation detail the user
   shouldn't see. Both mean "keep working on this branch."

Additionally, HANDOFF.md's role has been largely mechanised. Context that used to
require explicit handover narrative is now captured by `.meta`, `.plan`, specs,
JOURNAL.md, and `work_health.py`. The remaining unique value of HANDOFF.md is
the 2-3 line "Last Session" narrative (what was tried, what reasoning drove
decisions) — and `continue` can auto-load this.

---

## Decisions

### D1: Command taxonomy

**Choice:** Three distinct verbs with non-overlapping semantics

| Verb | Meaning | Lifecycle? |
|------|---------|-----------|
| `continue` | Keep working on the current branch | Yes — auto-loads all context |
| `resume` | Restore a paused branch from the pause stack | Yes — pause-stack only |
| `brief` | Orientation summary — what's the state of things? | No — purely informational |

**Alternatives:**
- Single `resume` for both pause-stack and branch continuation — ambiguous, current problem
- `continue` + `resume` only, no `brief` — loses the orientation use case

**Rationale:** Each verb maps to exactly one operation. No overloading. The user
never has to wonder which "resume" they're invoking.

**Exploration:** deep-analysis
**Status:** captured

### D2: `continue` subsumes HANDOFF.md reading

**Choice:** `continue` reads HANDOFF.md when present and presents a brief
summary (what was done, what reasoning drove decisions) before proceeding —
along with `.meta`, `.plan`, specs, and `work_health.py` output. No separate
"resume handover" step.

`resume handover` still works as an explicit manual invocation if the user wants
to interrogate the handover document directly. But the automatic context
restoration is intrinsic to `continue`.

**Alternatives:**
- Keep `resume handover` as a distinct lifecycle action — adds a step the user
  doesn't need, since `continue` already loads everything
- Remove HANDOFF.md entirely — loses the narrative context ("what was tried and
  failed") that git log can't capture

**Rationale:** HANDOFF.md's unique value (narrative context) is thin — 2-3 lines.
It doesn't warrant a separate command. Auto-loading it during `continue` gives
the LLM the context without user ceremony.

**Depends on:** D1 (continue exists as a distinct verb)
**Exploration:** deep-analysis
**Status:** captured

### D3: `continue` done-detection with auto-suggest

**Choice:** When `continue` detects the current issue is complete, it
auto-suggests the next action:
- If queue has remaining items: "Current issue complete. `next` (N remaining) or `end`?"
- If queue is empty: "Current issue complete. `end` to close the branch?"

**Mechanical definition:** "current issue is complete" = the active `.plan`
leaf issue's GitHub state is CLOSED. Derivation: `gh issue view
<PLAN_ACTIVE_ISSUE> --repo <OWNER_REPO> --json state --jq '.state'`. This
aligns with the unified work state spec's principle: derive from the
authoritative source (GitHub), don't cache.

**When `HAS_PLAN=no`:** Done-detection is not available. `ISSUE_DONE` is not
emitted by the router. `continue` proceeds directly to context loading
without auto-suggest. This covers single-issue branches, branches started
before the plan system, and legacy epic branches. No degradation — these
branches work exactly as today's resume path does.

**Alternatives:**
- Report only — "Current issue is complete." Then wait for user to type the
  next command. More control, extra round-trip.

**Rationale:** Saves a round-trip for the common case. The user can always ignore
the suggestion and do something else. Done-detection is scoped to `.plan`
branches because they're the only ones with a well-defined "current issue"
concept.

**Depends on:** D1 (continue exists)
**Exploration:** quick
**Status:** captured

### D4: Wrong-context command behavior

**Choice:** Forgiving for `start`, strict for `resume`, error for `continue` on main

| Command | Wrong context | Behavior |
|---------|--------------|----------|
| `work start` | On feature branch | **Redirect** → `continue` + note: "Already on `<branch>` — continuing." |
| `work resume` | On feature branch (not paused) | **Error** → "Not paused — use `continue` to keep working, or `work pause` first." |
| `work continue` | On main, no stack | **Error** → "No active branch — use `work` to start new work." |
| `work continue` | On main, with stack | **Error** → "No active branch — use `work` to start new work or `work resume` to return to a paused branch." |
| `work resume` | On main, no stack | **Error** → "Nothing to resume — pause stack is empty. Use `work` to start new work." (**Intentional behavioral change:** currently this falls through the router to `ROUTE=start` and redirects to work-start. The new behavior is semantically stricter — "resume" with nothing to resume is an error, not a silent redirect.) |

**Alternatives:**
- All strict (error on every wrong-context) — punishes `start` users unnecessarily
- All forgiving (redirect everything) — undermines the semantic distinction for `resume`

**Rationale:** `start` redirect is forgiving because the intent is clear. `resume`
is strict because the semantic distinction is the whole point. `continue` on main
has no reasonable interpretation.

**Exploration:** quick
**Status:** captured

### D5: `/brief` as standalone skill with structured data layer

**Choice:** `brief.py` script outputs structured data (branch, issue, plan state,
recent commits, HANDOFF summary, stack depth, notes, health checks). `/brief`
skill is a thin CLI wrapper that formats this output for the terminal.

The real delivery mechanism will be the Trellis frame — showing this information
ambiently. `brief.py` is the reusable data contract; `/brief` is the interim CLI
interface.

```
brief.py (structured data output)
  ├── /brief skill (CLI formatting — lightweight wrapper)
  └── Trellis frame (future — reads same structured output)
```

`brief.py` aggregates from existing scripts: `ctx.py` + `work_router.py` +
`work_health.py` + HANDOFF.md parsing. The pieces exist — `brief.py` is
mostly composition, plus new branch enumeration logic for the "main, no work"
state (see D6).

**Output format:** KEY=VALUE lines for scalars, consistent with `ctx.py` and
`work_router.py`. List data uses established section conventions:

```
# Scalar fields (always emitted)
STATE=feature_branch|main_with_stack|main_idle
BRANCH=<name>                    # empty if on main
ISSUE=<number>                   # empty if none
STACK_DEPTH=<N>

# Plan fields (only when HAS_PLAN=yes)
HAS_PLAN=yes|no
PLAN_POSITION=<completed>/<total>
PLAN_ACTIVE_ISSUE=<N>
PLAN_BATCH=<batch name>
ISSUE_DONE=yes|no                # GitHub-derived, see D3

# Context fields
HAS_HANDOFF=yes|no
HANDOFF_SUMMARY=<first 2-3 lines of Last Session section>
RECENT_COMMITS=<N>               # count; commit lines follow

# Commit lines (one per line, most recent first, max 6)
COMMIT=<sha7> <subject>

# Health check lines (from work_health.py, same format)
CHECK=<name> STATUS=ok|warn|changed [DETAIL=<text>]

# Completed branches (only when STATE=main_idle, max 5)
CLOSED_BRANCH=<name> ISSUE=<N> CLOSED=<relative date>
```

**CSO description for `brief/SKILL.md`:**
```
Use when the user says "/brief", "brief me", "what's the state",
"where are we", or "orient me" — provides an orientation summary of the
current work state, branch, issue, plan progress, and health checks.
Not for writing content (use write-content for that).
```

**Alternatives:**
- Full skill with rich formatting — over-invests in CLI UX that Trellis replaces
- No skill at all, just the script — need a CLI interface before Trellis lands

**Rationale:** Thin CLI wrapper avoids wasted effort. Structured data layer serves
both CLI (now) and Trellis (soon). The script pattern matches `ctx.py` and
`work_router.py`.

**Exploration:** quick
**Status:** captured

### D6: `brief` works from any state

**Choice:** `brief` adapts to context:

| State | What brief shows |
|-------|-----------------|
| Feature branch with `.meta` | Branch, issue, plan queue, recent commits, HANDOFF summary (if present), specs loaded, health checks |
| Main with pause stack | Stack depth, paused branch summaries, what-next recommendations |
| Main, no stack, no work | Recent completed branches, what-next recommendations |

**"Recent completed branches" enumeration:** This requires new logic beyond
composing existing scripts. `is_closed()` in `lifecycle.py` checks a specific
branch but doesn't enumerate. `brief.py` implements:

1. List local branches: `git branch --list 'issue-*'` (convention-filtered)
2. For each, call `is_closed(project, branch, workspace)` — sub-second per branch
3. Filter for `CLOSED` or `MERGED_UNSTAMPED`
4. Sort by last commit date, take most recent 5
5. Output as `CLOSED_BRANCH=<name> ISSUE=<N> CLOSED=<relative date>` lines

**Alternatives:**
- Feature branch only — misses the "picking up cold on main" use case

**Rationale:** Orientation is useful in every state. The data is all available
from existing scripts, except the branch enumeration which is new but
straightforward.

**Exploration:** quick
**Status:** captured

---

## HANDOFF.md Coverage Audit

Verified no gaps in mechanical coverage:

| HANDOFF.md section | Mechanical replacement | Gap? |
|---|---|---|
| Last Session (narrative) | Git log + JOURNAL.md + blog | **Partial** — git log gives facts but not reasoning or failed attempts. HANDOFF.md still captures this; `continue` auto-loads it |
| Immediate Next Step | `.plan` active issue | None |
| Cross-Module blockers | GitHub issues with labels | None |
| References (paths) | `find` on workspace dirs | None |
| Queue progress | `.plan` + `format_resume_display()` | None — already moved |
| What's Left / What's Next | `.plan` + GitHub issues | None — already moved |
| State validation | `work_health.py --scope entry` | None — already moved |

**Session wrap checklist** (forage sweep, protocol sweep, write-content, etc.)
stays with the `wrap` command. These are session-bound operations at wrap time,
not resume time. No change needed.

**HANDOFF.md is still written** at session end via the handover skill. The change
is only that reading it on resume is automatic (part of `continue`) rather than
a separate command.

---

## Implementation Scope

### Changes to existing files

| File | Change |
|------|--------|
| `work/SKILL.md` | Routing table: add `continue`, separate from `resume`. Step 4: replace "resume"/"start" with "continue". Add done-detection auto-suggest. Add wrong-context redirects (D4). Update CSO description to: `Use when the user says "work", "work end", "work pause", "work resume", "work continue", or "work next" — detects current branch state and routes to the correct work lifecycle skill automatically.` |
| `work-resume/SKILL.md` | CSO description updated to: `Use when returning to a paused branch from the pause stack — user says "work-resume", "resume", or "go back to that branch". Pause-stack restoration only, not general branch continuation (use "work continue" for that).` |
| `handover/SKILL.md` | Update resume section (Step R1-R3): note that `continue` subsumes automatic HANDOFF.md reading. `resume handover` remains as explicit manual invocation. |
| `work_router.py` | Add `ISSUE_DONE=yes/no` output — emitted only when `HAS_PLAN=yes`. Checks active leaf issue's GitHub state via `gh issue view`. When `HAS_PLAN=no`, field is omitted entirely. |

### New files

| File | Purpose |
|------|---------|
| `brief/SKILL.md` | Thin CLI skill — calls `brief.py`, formats output |
| `brief/brief.py` | Structured data aggregator — composes `ctx.py` + `work_router.py` + `work_health.py` + HANDOFF.md |
| `brief/commands/brief.md` | Slash command registration |

### Not changed

| File | Why |
|------|-----|
| `lifecycle.py` | No state machine changes — the transitions are correct. The overloading was in the skill presentation layer, not the state machine. |
| `work-pause/SKILL.md` | No changes — pause semantics are unchanged |
| `work-end/SKILL.md` | No changes — end semantics are unchanged |
| `work-start/SKILL.md` | No changes — start semantics are unchanged (it's the internal skill; the user-facing `start` redirect is handled in `work/SKILL.md`) |

---

## Updated `/work` Menu (Step 4 — on feature branch)

```
1. continue — keep working (loads context automatically)
2. switch — you have N paused branch(es) — resume one instead     ← only if STACK_DEPTH > 0
N. next — mark current issue done, advance to next in queue       ← only if HAS_PLAN=yes
N+1. end — close this branch, merge, push, return to main
N+2. pause — commit WIP, push to stack, switch to main
N+3. wrap — end session but keep branch open (write handover)
```

The "resume"/"start" distinction is gone. `continue` is always option 1
regardless of HANDOFF.md existence. Internally, `continue` reads
HANDOFF.md if present, runs resume checks if not — same behavior as
today, unified label.

---

## Updated Routing Table (Step 1)

| Invocation | Route to |
|------------|---------|
| `work end` | → **work-end** immediately |
| `work pause` | → **work-pause** immediately |
| `work next` | → advance to next issue in `.plan` queue (Step 5) |
| `work resume` / `resume` | → **work-resume** (pause-stack only; error if on active branch) |
| `work continue` / `continue` | → run router → Step 4 `continue` action directly |
| `work` / `work start` | → run router (Step 1b) |
| `resume handover` | → handover skill directly (manual invocation) |

When router returns `resume_branch` and invocation was `work start`:
redirect to `continue` with note "Already on `<branch>` — continuing."
