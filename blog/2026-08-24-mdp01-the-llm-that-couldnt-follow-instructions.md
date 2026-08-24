---
entry_type: note
subtype: diary
title: "The LLM that couldn't follow instructions"
date: 2026-08-24
issue: 271
branch: issue-271-mechanise-work-end-close
project: soredium
tags: [architecture, agent-design, inverted-control, reliability]
status: partial
---

# The LLM that couldn't follow instructions

Four slot closures audited. Four failures. Every one caused by the same thing: the LLM skipping steps in a 660-line instruction set.

The evidence was damning. Repos not switched to main. Artifacts not promoted. Code not pushed. Blogs left uncommitted. A branch deleted instead of stamped. Not edge cases — the happy path, consistently broken. The failure mode wasn't "the LLM made a bad decision." It was "the LLM didn't execute the step at all."

I'd been tightening the instructions for months. Checklists. Hard gates. Red flag tables. Common pitfalls sections. None of it mattered, because the architecture was wrong. Telling an LLM to follow a 20-step sequence is like handing someone a 20-item shopping list and hoping they don't skip the boring items. They will.

## The inversion

The fix isn't better instructions. It's removing the LLM from the control loop.

A Python script owns the close sequence now. It reads a progress file, runs mechanical steps (push, stamp, verify, promote), and when it hits something that needs judgment — write a blog, review code, classify commits for squash — it prints one instruction and exits. The LLM reads that instruction, does the work, calls the script again. The script validates the output before continuing.

The key insight: the LLM never sees the step list. It gets one instruction at a time. It can't skip step 7 because it doesn't know step 7 exists. Python decides what's next. Python validates. The 660-line skill becomes a 20-line dispatch loop.

This is the opposite of how Claude Code normally works. The standard model is LLM-as-orchestrator: the skill describes what to do, the LLM does it. Claude Code Workflows offer deterministic orchestration, but they spawn a fresh agent per step — conversation context doesn't carry forward. Review findings can't inform the sweep. User decisions evaporate between steps.

The re-entrant script preserves context because it's one continuous conversation. The LLM calls the same script repeatedly, building up session knowledge across yield points. Forage discoveries are available during protocol capture. Protocol captures inform the diary entry. Nothing is lost.

## The crash-safety accident

While designing the orchestrator, we discovered that the existing progress files — the ones tracking which repos had been pushed, stamped, merged — weren't crash-safe. `Path.write_text()` truncates the file before writing. Process crash between truncation and write: empty file. All prior progress gone.

The design spec called this "append-only." It wasn't. The code read the file into a dict, modified it, and rewrote the entire file. The gap had been there since the scripts were written. The fix was the same pattern `lifecycle.py` already uses: write to `.tmp`, then `os.replace()` — atomic on POSIX.

Finding this while designing the orchestrator was fortunate. If we'd built the orchestrator on top of the same pattern, the crash-safety problem would have propagated into a system that runs more steps, not fewer.

## What's built, what's next

The orchestrator core is working: `close_progress.py` for atomic progress tracking with stale detection, and `work_end_orchestrator.py` with the step walker, action yielding, validation, retry escalation, and abort handling. The crash-safety fix is shipped independently.

Still to come: updating `close_report.py` with the new step names so the close summary renders properly, rewriting the SKILL.md itself, and integration tests with real git repos. The rewrite is where it gets real — 660 lines of hard-won judgment instructions need to become ~470 lines: a dispatch loop, pre-close context handling, and an Action Handlers reference section that tells the LLM what each action actually means.

The question I can't answer yet: does this work in practice? The architecture is sound — the LLM genuinely can't skip what it can't see. But the judgment steps still need the LLM to execute them correctly. Code review. Forcing function triage. Blog writing. The orchestrator validates outputs, but "blog file exists with >100 words" doesn't guarantee "blog captures the session's narrative." That's where the optional Haiku validation layer comes in — a cheap semantic check that the mechanical heuristics can't provide. Whether it earns its place is an implementation question, not a design one.
