# Soredium TUI — Design Spec

**Issue:** #222 — REPL shell for mechanical lifecycle operations
**Branch:** issue-222-repl-shell
**Date:** 2026-08-11
**Scope:** Python Textual app (this issue); Java/Tamboui port is follow-on

## Overview

A Textual-based terminal application that exposes the soredium work/slot
lifecycle as a guided, panel-based interface. Runs without an LLM for all
mechanical operations (start, next, end, pause, resume, brief, what-next).
Optionally hands off to a pluggable LLM CLI for reasoning work (sweeps,
reviews, narrative, implementation).

The Python app is a stepping stone — the long-term target is a Quarkus
native CLI built on Tamboui. Both runtimes share the same architecture:
event-driven command layer, panel-based TUI, pluggable session provider.

## Architecture

Three layers. The command layer is the portable core — everything above
and below it changes between Python and Java. The Session SPI is a
TUI-owned adapter, not a separate architectural layer.

```
┌─────────────────────────────────┐
│  TUI (Textual / Tamboui)        │  tui/python/  tui/java/ (follow-on)
│  Panels, action list, events    │
│  SessionProvider (adapter)      │  Suspend / Automated / Embedded
├─────────────────────────────────┤
│  Command Layer (event-driven)   │  Portable orchestration
│  start(), next(), end(), etc.   │  Lifecycle state machine executor
├─────────────────────────────────┤
│  Script Layer (existing)        │  project/, work-start/, work-end/,
│  Refactored for library import  │  work-slot/, work-pause/, etc.
└─────────────────────────────────┘
```

**TUI** — Textual widget tree (Python) or Tamboui toolkit (Java).
Subscribes to events from the command layer. Renders panels. Handles
keyboard/mouse input. Never calls scripts directly. Owns the
SessionProvider adapter for LLM CLI handoff — suspends, launches the
configured CLI, resumes, then asks the command layer to refresh state.

**Command Layer** — orchestrates script calls by executing lifecycle
transitions, emits typed events. Language-agnostic concepts
(BranchCreated, PlanAdvanced, WorkEnded). This is what gets ported to
Java. The command layer is the effect executor for the lifecycle state
machine (see § Lifecycle Integration below).

**Script Layer** — existing Python scripts refactored with library APIs.
CLI entry points preserved for backward compatibility with Claude Code
skills. Java port reimplements this layer natively.

### Lifecycle Integration

The command layer is the effect executor for `lifecycle.py`'s state
machine. The lifecycle TRANSITION_TABLE is the single source of truth
for valid transitions and their effects. Every state-changing command
follows the three-phase commit protocol:

1. `lifecycle.transition(meta_path, event)` — validate, return `TransitionResult`
2. Execute effects listed in `TransitionResult.effects` by calling script library functions
3. `lifecycle.commit_transition(meta_path, result)` — verify state unchanged, write atomically
4. Execute `TransitionResult.post_commit_effects` (branch switches, stack operations)

The command layer maps abstract effect names from the transition table
to concrete script library calls:

| Effect | Script call |
|--------|------------|
| `create_branch` | `branch_create.create()` |
| `write_meta` | `scaffold.scaffold()` |
| `build_plan` | `plan_manager.create()` |
| `advance_issue` | `plan_manager.advance()` |
| `update_meta` | `scaffold.update_meta()` |
| `wip_commit` | `pause_exec.commit_wip()` |
| `push_stack` | `stack.push()` |
| `pop_stack` | `stack.pop()` |
| `reset_wip` | `resume_exec.reset_wip()` |
| `write_promotion_stamp` | `close_artifacts.close()` |
| `write_stamp` | `land_branch.stamp()` |
| `write_plan_closed` | `plan_manager.close()` |
| `return_to_main` | `branch_cleanup.cleanup()` |

**Multi-step commands.** The `end` command fires sequential lifecycle
transitions through the closing substates: `work_end` →
`review_pass` → `promote_pass` → `push_pass` → `merge_pass` →
`stamp_pass` → `cleanup_pass`. Each transition follows the three-phase
protocol. `StepProgress` events are emitted between transitions.
The TUI's mechanical close skips LLM-only effects (`pre_close_sweep`,
`record_review`) — these transitions still fire to advance state, but
the effects are no-ops.

**Transient states.** `scaffolded` and `transitioning` are auto-fired
by the command layer — the user never sees these states. When `start`
reaches `scaffolded`, the command layer immediately fires `auto_setup`
to reach `active`. When `next` reaches `transitioning`, the command
layer immediately fires `auto_refresh`. Both auto-transitions emit
`StepProgress` events for their effects.

### Action Derivation

The command layer owns the mapping from lifecycle state to available
actions. After every state change, it computes `available_actions` and
`suggested_action` based on the lifecycle TRANSITION_TABLE and includes
them in the `StateChanged` event. The TUI renders whatever the command
layer provides — it has no independent derivation logic.

| State | Available actions | Suggested |
|-------|-------------------|-----------|
| `idle` (no stack) | start, what-next, status | start |
| `idle` (stack > 0) | start, resume, what-next, status | resume |
| `active` (has queue) | brief, next, pause, end, session, status | next |
| `active` (no queue) | brief, pause, end, session, status | end |
| `paused` | resume, start, what-next, status | resume |
| `closing:*` | (continue close sequence), status | (next close step) |

`scaffolded` and `transitioning` are never surfaced — the command
layer auto-fires through them before emitting `StateChanged`.

## Directory Structure

```
soredium/
  tui/
    python/
      __main__.py          # Entry point: python -m tui.python
      app.py               # Textual App subclass
      ui/
        header.py          # Branch, state, queue position
        action_panel.py    # Context-sensitive action list
        content.py         # Scrollable output area
        footer.py          # Keybinding hints
        modals.py          # Input overlays (issue numbers, messages)
      commands/
        __init__.py        # Command registry
        start.py           # start #N [#M ...]
        next.py            # advance plan
        end.py             # mechanical close
        pause.py           # WIP commit + stack push
        resume.py          # stack pop + rebase
        continue_.py       # keep working on current branch
        brief.py           # context summary
        quick_fix.py       # ephemeral branch landing
        what_next.py       # enrichment recommendations
        status.py          # full context dump
      events.py            # All typed event dataclasses
      session/
        __init__.py        # SessionProvider protocol
        suspend.py         # Default: suspend TUI, run CLI
      styles/
        app.tcss           # Textual CSS
    java/                  # Follow-on issue (empty for now)
  project/                 # Existing — refactored for import
  work-start/              # Existing — refactored for import
  work-end/                # Existing — refactored for import
  work-slot/               # Existing — refactored for import
  work-pause/              # Existing — refactored for import
  work-resume/             # Existing — refactored for import
  quick-fix/               # Existing — refactored for import
  scripts/                 # Existing — enrichment.py etc.
```

## Event Model

Commands emit typed dataclass events. The TUI subscribes and updates
panels. Events are the contract between command and UI layers.

```python
from dataclasses import dataclass

# State management
@dataclass
class StateChanged:
    old_state: str
    new_state: str
    available_actions: list[str]
    suggested_action: str | None

# Command results
@dataclass
class BriefReady:
    issue: int | None
    branch: str
    state: str
    queue_position: str | None   # "1/3"
    health: dict[str, str]

@dataclass
class BranchCreated:
    branch: str
    issues: list[int]
    plan_path: str | None

@dataclass
class PlanAdvanced:
    completed_issue: int
    next_issue: int | None
    next_title: str | None
    position: str               # "2/3"
    queue_complete: bool

@dataclass
class WorkEnded:
    branch: str
    issues_closed: list[int]

@dataclass
class WipCommitted:
    repo: str
    message: str

@dataclass
class Paused:
    branch: str
    stack_depth: int

@dataclass
class Resumed:
    branch: str
    rebased: bool

@dataclass
class QuickFixLanded:
    branch: str
    message: str

@dataclass
class Recommendation:
    issue: int
    title: str
    strategic_role: str | None   # "quick-win", "load-bearing", etc.
    readiness: str | None        # "ready", "needs-design", etc.
    reason: str | None

@dataclass
class WhatNextReady:
    recommendations: list[Recommendation]

@dataclass
class StatusReady:
    branch: str
    state: str
    on_main: bool
    in_slot: bool
    has_plan: bool
    plan_position: str | None    # "2/3"
    stack_depth: int
    owner_repo: str | None
    base_branch: str

@dataclass
class ContinueReady:
    issue: int | None
    branch: str
    state: str
    handoff_summary: str | None  # Last Session narrative, if HANDOFF.md present
    done_detected: bool          # True if active plan issue is complete
    suggest_next: bool           # True if queue has remaining items
    suggest_end: bool            # True if queue is exhausted

@dataclass
class CommandFailed:
    command: str                 # "end", "start", "resume", etc.
    step: str | None             # For multi-step commands: which step failed
    error: str                   # Machine-readable error code
    detail: str                  # Human-readable message
    recoverable: bool            # True if retry or abort is possible

# Multi-step progress
@dataclass
class StepProgress:
    command: str                # "end", "start"
    step: str                   # "promoted", "rebased", "pushed", "stamped"
    detail: str | None

@dataclass
class StepFailed:
    command: str
    step: str
    error: str
    recoverable: bool           # True if retry/resume is possible
    detail: str | None

# Session lifecycle
@dataclass
class SessionStarted:
    provider: str
    issue: int | None

@dataclass
class SessionEnded:
    provider: str
```

Every state-changing command emits `StateChanged` as its final event.
`StateChanged.available_actions` drives the action panel.
`StateChanged.suggested_action` highlights the recommended next step.

Informational commands (`brief`, `what-next`, `status`) do NOT emit
`StateChanged` — they read state without changing it.

Multi-step commands (end, start) emit `StepProgress` events as each
step completes — the content panel updates in real time.

When any command fails, the command layer emits `CommandFailed`.
For multi-step commands, `CommandFailed.step` identifies where the
failure occurred. The TUI uses `recoverable` to decide whether to
offer retry/abort or just display the error.

## UI Layout

Four persistent zones:

```
┌─ Header ──────────────────────────────────────────┐
│ soredium   [issue-42] active   Queue: 1/3         │
├────────────┬──────────────────────────────────────-┤
│ Actions    │ Content                               │
│            │                                       │
│ > brief    │ (output from selected/last action)    │
│   next     │                                       │
│   pause    │                                       │
│   end      │                                       │
│   session  │                                       │
│   status   │                                       │
│            │                                       │
├────────────┴──────────────────────────────────────-┤
│ ↑↓ select  Enter run  ? help  q quit              │
└───────────────────────────────────────────────────-┘
```

**Header** — always visible. Branch, lifecycle state, plan position.
Updates reactively on `StateChanged`.

**Action Panel** (left) — valid actions for the current state.
Arrow keys to navigate, Enter to run. Mouse click also works.
Derived from lifecycle state machine transitions + static commands.
Suggested next action is highlighted.

**Content** (right, scrollable) — output from the last action.
Brief shows issue/branch/health. End shows step-by-step progress.
What-next shows ranked recommendations.

**Footer** — keybinding hints. Context-sensitive.

### Bootstrap Sequence

On startup, the TUI runs a deterministic initialisation before accepting
user input:

1. Resolve topology — call `topology.resolve()` for project/workspace paths
2. Detect work state — call `work_state.detect(topo)` for routing, plan, stack
3. Read lifecycle state — call `lifecycle.read_state(meta_path)` (None → idle)
4. Derive initial available actions from the state (same logic as `StateChanged`)
5. Populate header (branch, state, queue position) from the detection results
6. Populate action panel from available actions; highlight suggested action
7. Content panel starts empty — displays a welcome line with branch/state summary

The bootstrap emits a synthetic `StateChanged` event internally (not from
a command) to drive the initial UI render. No command runs automatically —
the user chooses their first action from the panel.

### Action Panel Derivation

The lifecycle state machine defines valid transitions. The action
panel maps state to available actions:

| State | Available actions |
|-------|-------------------|
| `main` (no work) | start, quick-fix, what-next, status, resume (if stack) |
| `scaffolded` / `active` | continue, brief, next (if queue), pause, end, session, status |
| `transitioning` | _(no user actions — progress indicator while context refreshes; auto-resolves to `active`)_ |
| `paused` (on main) | resume, start, what-next, status |
| `closing:review` | abort, status |
| `closing:verified` | abort, status |
| `closing:promoted` | status _(no abort — artifacts already promoted)_ |
| `closing:pushed` | status _(no abort — branch pushed)_ |
| `closing:merged` | status _(no abort — content merged)_ |
| `closing:stamped` | status _(cleanup only — auto-completes)_ |

The `end` command runs the full closing sequence automatically —
individual `closing:*` sub-states are not user-driven. The action
panel shows the current close step as a progress indicator. `abort`
is available only in `closing:review` and `closing:verified` (before
artifacts are promoted), matching the lifecycle's `abort_close` transitions.

Transient states (`scaffolded`, `transitioning`) auto-resolve via
lifecycle events (`auto_setup`, `auto_refresh`). The TUI shows a
progress indicator during these states. `scaffolded` appears
momentarily during `start` before auto-transitioning to `active`.

### Input Modals

Actions that need input use modal overlays:
- `start` — issue number input (accepts `#42 #43 #44` or `42 43 44`)
- `quick-fix` — commit message input
- `resume` — branch picker (if multiple paused branches)

## Session SPI

Pluggable interface for LLM CLI handoff. The TUI doesn't know or care
which LLM or CLI is behind the SPI.

```python
from typing import Protocol
from dataclasses import dataclass

@dataclass
class IssueContext:
    issue: int
    title: str
    branch: str
    plan_position: str | None
    project_path: str
    workspace_path: str | None

class SessionProvider(Protocol):
    def start(self, context: IssueContext) -> None: ...
    def is_active(self) -> bool: ...
    def stop(self) -> None: ...
```

### Default Implementation: Suspend

```python
class SuspendingProvider:
    """Suspend TUI, launch configured CLI, resume on exit."""

    def __init__(self, cli_command: str = "claude"):
        self.cli_command = cli_command
        self._active = False

    def start(self, context: IssueContext) -> None:
        self._active = True
        # TUI calls app.suspend() before this
        # Launch CLI with issue context
        subprocess.run([
            self.cli_command,
            "--system-prompt", f"Working on #{context.issue}: {context.title}",
        ], cwd=context.project_path)
        self._active = False

    def is_active(self) -> bool:
        return self._active

    def stop(self) -> None:
        self._active = False
```

After `start()` returns, the command layer must re-detect state before
emitting `SessionEnded` + `StateChanged`. The CLI session may have
advanced the plan, closed the branch, or changed lifecycle state.
Re-detection sequence: re-read `.meta`, call `work_state.detect()`,
derive new available actions. The resulting `StateChanged` reflects
the post-session state.

Future implementations:
- **AutomatedProvider** — drives a third-party CLI via I/O (reading
  output, submitting prompts programmatically)
- **EmbeddedProvider** — native conversation panel inside the TUI

Configuration via environment or config file (precedence: env > config
file > default):
```
SOREDIUM_SESSION_CLI=claude          # default
SOREDIUM_SESSION_CLI=aider
SOREDIUM_SESSION_CLI=gemini
```

## Script Refactoring Pattern

Each script gets a library function alongside its CLI entry point.
The library function returns a typed result. The CLI entry point
calls the library function and prints KEY=value output.

```python
# Pattern for all refactored scripts

@dataclass
class ScaffoldResult:
    meta_path: str
    journal_path: str
    plan_path: str | None

def scaffold(workspace: Path, issue: int, branch: str,
             covers: list[int], **opts) -> ScaffoldResult:
    """Library API — called by command layer and CLI."""
    ...
    return ScaffoldResult(...)

def main() -> int:
    """CLI entry point — called by skills via python3 scaffold.py."""
    workspace = Path(sys.argv[1]).resolve()
    params = parse_args(sys.argv[2:])
    result = scaffold(workspace, **params)
    print(f"META_PATH={result.meta_path}")
    print(f"JOURNAL_PATH={result.journal_path}")
    ...
```

For scripts that need interactive decisions, use the `decide_fn`
callback pattern (garden entry GE-20260421-690e47):

```python
def land_branch(project: Path, branch: str,
                decide_fn: Callable[[str], bool] = None,
                **opts) -> LandResult:
    """Library API. decide_fn handles confirmations."""
    ...
    if needs_confirmation and decide_fn:
        if not decide_fn(f"Force push to {branch}?"):
            return LandResult(aborted=True)
    ...
```

### Scripts to Refactor

| Script | Current API | Library API needed |
|--------|------------|-------------------|
| `project/ctx.py` | `resolve()` returns dict | Already importable — no change |
| `project/lifecycle.py` | `transition()` returns `TransitionResult` | Already importable — no change |
| `project/work_state.py` | `detect()` | Already importable — no change |
| `project/work_health.py` | `run_checks()` | Already importable — no change |
| `project/topology.py` | `resolve()` returns `Topology` | Already importable — no change |
| `project/stack.py` | `cmd_*()` via sys.argv, prints KEY=value | Expose `read_entries(path) -> list[StackEntry]`, `push_entry(path, entry) -> int`, `pop_entry(path, branch) -> tuple[bool, int]` — pure data functions without stdout |
| `work-start/scaffold.py` | `main()` via sys.argv | `scaffold(workspace, issue, branch, covers) -> ScaffoldResult` |
| `work-start/branch_create.py` | `main()` via sys.argv | `create(project, workspace, branch, issues) -> CreateResult` |
| `work-end/work_end_execute.py` | `main()` via sys.argv | `promote(opts) -> PromoteResult`, `rebase(opts) -> RebaseResult`, `land(opts) -> LandResult` |
| `work-end/close_artifacts.py` | subprocess call | `close(workspace, project, branch) -> CloseResult` |
| `work-end/land_branch.py` | `main()` via sys.argv | `rebase(project, branch, base) -> RebaseResult`, `push(project, branch) -> PushResult`, `stamp(project, branch, sha) -> StampResult` |
| `work-end/branch_cleanup.py` | `main()` via sys.argv | `cleanup(project, workspace, branch) -> CleanupResult` |
| `work-end/verify_promotion.py` | script call | `verify(workspace, project, branch) -> VerifyResult` |
| `work-pause/pause_exec.py` | `commit_wip()`, `push_and_stack()` | Already has callable functions — add typed results |
| `work-resume/resume_exec.py` | `checkout_branches()`, `rebase()` | Already has callable functions — add typed results |
| `work-slot/plan_manager.py` | parse/advance/detect | Already importable — add typed `AdvanceResult` (exists) |
| `work-slot/slot_manager.py` | `main()` via sys.argv (68KB) | Extract key functions with typed results |
| `quick-fix/quick_fix.py` | `run()` function | Already importable — add typed result |
| `scripts/enrichment.py` | argparse `main()` | `what_next(repo) -> list[Recommendation]` |

## Commands → Script Calls → Events

| Command | Script calls | Events emitted |
|---------|-------------|----------------|
| `brief` | `ctx.resolve()`, `work_health.run_checks()` | `BriefReady` |
| `continue` | `ctx.resolve()`, `work_health.run_checks()`, HANDOFF.md parse, `plan_manager.detect()` | `ContinueReady`, `StateChanged` |
| `start #N [#M]` | `branch_create.create_branches()`, `scaffold.scaffold()`, `plan_manager.create_main_plan()` | `StepProgress` × N, `BranchCreated`, `StateChanged` |
| `next` | `plan_manager.advance()`, `lifecycle.transition()` | `PlanAdvanced`, `StateChanged` |
| `end` | `close_artifacts.close()`, `land_branch.rebase()`, `land_branch.push()`, `land_branch.stamp()`, `branch_cleanup.cleanup()` | `StepProgress` × N, `WorkEnded`, `StateChanged` |
| `pause` | `pause_exec.commit_wip()`, `pause_exec.push_and_stack()` | `WipCommitted`, `Paused`, `StateChanged` |
| `resume` | `stack.pop()`, `resume_exec.checkout_branches()`, `resume_exec.rebase()`, `resume_exec.reset_wip()` | `Resumed`, `StateChanged` |
| `quick-fix` | `quick_fix.run()` | `QuickFixLanded`, `StateChanged` |
| `what-next` | `enrichment.what_next()` | `WhatNextReady` |
| `status` | `ctx.resolve()` | `StatusReady` |
| `session` | `SessionProvider.start()` | `SessionStarted`, `SessionEnded`, `StateChanged` |

Any command may emit `CommandFailed` instead of its success events if
an error occurs. For multi-step commands (`end`, `start`), partial
`StepProgress` events may precede the `CommandFailed`.

## What the TUI Skips (LLM-only)

| Operation | Why it needs an LLM |
|-----------|---------------------|
| Forage SWEEP | Scans conversation memory for non-obvious knowledge |
| Protocol SWEEP | Identifies conventions from session reasoning |
| Write-content | Narrative synthesis (diary, blog) |
| Brainstorming | Design exploration before implementation |
| Design review | Adversarial spec analysis |
| Code review (semantic) | Understanding intent, catching logic errors |
| Squash analysis | Classifying commits by semantic concern |
| HANDOFF.md writing | Summarizing session context and reasoning |

The TUI's `end` runs the mechanical close only: promote, rebase,
push, stamp, close issues. Commits stay as-is unless the user
pre-squashes. These LLM operations can be invoked via the session
SPI when the user selects `session`.

## Slot-Specific Workflows

The lifecycle state machine supports slot creation via
`('idle', 'slot_create')`. Slot-specific features (epic management,
batch cycling) operate within the same lifecycle states but add
context:

- The TUI's `start` command detects slot context via `topology.resolve()`.
  In a slot workspace, `start` fires `slot_create` instead of `work`;
  the lifecycle transition table handles the distinction.
- The header shows slot context when `in_slot` is true: queue position
  includes batch info from `work_state.detect()` (e.g., "Queue: 1/3
  Batch: 2 of 4").
- Epic/plan detection in `work_state.py` already resolves
  `epic_batch`, `epic_active_issue`, `plan_batch` — the TUI header
  and `BriefReady` event surface these fields.
- `slot_manager.py` orchestration (68KB) requires library API
  extraction as noted in the refactoring table. The TUI command layer
  calls the extracted functions; detailed slot management TUI (batch
  cycling, epic dashboard) is a follow-on issue.

## Stateless CLI Mode

The same command layer supports stateless invocation for scripting
and automation:

```bash
soredium start 42 43 44
soredium next
soredium brief
soredium end
soredium what-next
```

Entry point: `python -m tui.python start 42` detects non-interactive
mode (no TTY or explicit `--cli` flag), runs the command, prints
JSON events to stdout, exits.

The `soredium` command is installed via `pyproject.toml`:
```toml
[project.scripts]
soredium = "tui.python.__main__:main"
```

## Testing Strategy

- **Command layer** — unit tests with mocked script functions.
  Each command gets a test that verifies correct script calls
  and event emission.
- **Script refactoring** — existing tests continue to pass (CLI
  entry points unchanged). New tests for library API functions.
- **Events** — serialisation/deserialisation roundtrip tests.
- **UI** — Textual's built-in test framework (`async with app.run_test()`).
  Verify action panel contents for each state. Verify content panel
  updates on events.
- **Session SPI** — `decide_fn` pattern: tests inject a mock provider.
- **Integration** — end-to-end test: start → next → end cycle via
  command layer against a test git repo.

## Dependencies

**Python (tui/python/):**
- `textual` — TUI framework
- No other new dependencies. All script imports are from the existing
  soredium codebase.

**Future Java (tui/java/):**
- `tamboui` — TUI framework
- `casehub platform agent-api` — LLM session SPI
- `picocli` — CLI mode
- Quarkus for CDI, native compilation

## Configuration

Precedence: environment variable > config file > default.

```
# Environment variables
SOREDIUM_SESSION_CLI=claude       # CLI to launch for LLM sessions
SOREDIUM_PROJECT=<path>           # Override project path detection
SOREDIUM_WORKSPACE=<path>         # Override workspace path detection

# Or config file: ~/.soredium/config.toml
[session]
cli = "claude"

[paths]
project = "/path/to/project"     # Optional override
workspace = "/path/to/workspace" # Optional override
```

Defaults: `cli = "claude"`, paths auto-detected via `topology.resolve()`.

## Decisions Reference

See [decisions.md](decisions.md) for the full decision log (D1–D10).

Key decisions:
- **D1:** Textual from the start (maps to Tamboui)
- **D2:** `tui/python/` and `tui/java/` location
- **D3:** Refactor scripts for library APIs
- **D4:** Event-driven command layer
- **D5:** LLM via casehub AgentProvider SPI (Java); Python equivalent
- **D7:** Action panel with keyboard navigation (guided UX)
- **D8:** Persistent panels over scrolling
- **D9:** Pluggable SessionProvider SPI (CLI-agnostic)
- **D10:** Suspend/resume as default, automated/embedded as future
