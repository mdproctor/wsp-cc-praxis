# Design Journal — issue-222-repl-shell

## 2026-08-11 — Design + foundation implementation

Brainstormed from issue spec through 12 design decisions. Key pivots
from the original cmd.Cmd REPL concept: Textual from the start (maps
to Tamboui for Java port), event-driven command layer (not
request/response), action panel with keyboard navigation (guided UX,
no typing), home view as multi-repo/slot workspace manager, tmux-based
session multiplexing for concurrent LLM CLI sessions.

Standard 4-dimension design review ($50.65) found 68 issues, resolved
all — major improvements to lifecycle integration, error recovery model,
closing operation mapping, and intent file handling.

Implemented Tasks 1-3 (foundation): events module (21 event types),
script refactoring (stack.py, scaffold.py, branch_create.py, pause_exec.py,
resume_exec.py, work_end_execute.py, land_branch.py, quick_fix.py,
enrichment.py), and the full command layer (registry, action derivation,
10 command modules). 72 tests, all passing, no regressions.

Tasks 4-7 remaining: TUI Project View, Home View, Session SPI, CLI mode.
