# cc-praxis Workspace

**Name:** cc-praxis
**Project repo:** /Users/mdproctor/claude/cc-praxis
**Workspace type:** public

## Session Start

Run `add-dir /Users/mdproctor/claude/cc-praxis` before any other work.

## Artifact Locations

| Skill | Writes to |
|-------|-----------|
| brainstorming (specs) | `specs/` |
| writing-plans (plans) | `plans/` |
| handover | `HANDOFF.md` |
| idea-log | `IDEAS.md` |
| adr | `adr/` |
| write-content | `blog/` |

**Spec path override:** The brainstorming skill defaults to `docs/superpowers/specs/` in the project repo. This project overrides that: specs promote to `docs/specs/` (not `docs/superpowers/specs/`). When writing a spec, use `docs/specs/YYYY-MM-DD-<topic>-design.md`.

## Structure

- `HANDOFF.md` — session handover (single file, overwritten each session)
- `IDEAS.md` — idea log (single file)
- `specs/` — brainstorming / design specs (superpowers output)
- `plans/` — implementation plans (superpowers output)
- `snapshots/` — design snapshots with INDEX.md
- `adr/` — architecture decision records with INDEX.md
- `blog/` — project diary entries with INDEX.md

## Git Discipline

Two git repositories are active in every session:
- **Workspace** (`~/claude/public/cc-praxis`) — plans, blog (staging), snapshots, handover
- **Project repo** (`/Users/mdproctor/claude/cc-praxis`) — source code, ADRs (`docs/adr/`)

Never rely on CWD for git operations — the session may have started in either repo. Always use explicit paths:
```bash
git -C ~/claude/public/cc-praxis add <file>          # workspace artifacts
git -C /Users/mdproctor/claude/cc-praxis add <file>  # project artifacts
```

## Rules

- All methodology artifacts go here, not in the project repo
- Promotion to project repo is always explicit — never automatic
- Workspace branches mirror project branches — switch both together

## Routing

| Artifact   | Destination | Notes |
|------------|-------------|-------|
| adr        | project     | lands in `docs/adr/` — promoted at work end |
| specs      | project     | lands in `docs/specs/` — promoted at work end |
| blog       | workspace   | staged here; routed via publish-blog and blog-routing.yaml at epic close |
| plans      | workspace   | stay in workspace permanently |
| design     | workspace   | work journal stays in workspace |
| snapshots  | workspace   | stay in workspace permanently |
| handover   | workspace   | |

**Blog directory:** `/Users/mdproctor/claude/public/cc-praxis/blog/`

---

# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## ⛔ No AI Attribution in Commits — Ever

**NEVER add AI attribution to any commit message unless the user explicitly requests it for that specific commit.**

No `Co-Authored-By: Claude`, no `Generated-by:`, no `AI-assisted:`, no mention of Claude, AI, or tooling in commit messages. This applies to every commit in every project. Commit messages describe WHAT changed and WHY — not who or what wrote them.

---

## Project Identity

**Name:** cc-praxis
**GitHub:** [github.com/mdproctor/cc-praxis](https://github.com/mdproctor/cc-praxis)
**Marketplace:** `/plugin marketplace add github.com/mdproctor/cc-praxis`
**Status:** Approaching v1.0 release. The collection is feature-complete for its initial scope. Active development continues on the main branch.

## Repository Purpose

This is a skill collection for Claude Code, providing specialized guidance for professional software development workflows. Skills are markdown files with YAML frontmatter that Claude Code loads to execute specific development tasks.

## Project Type

**Type:** skills

## Project Type Awareness

**CRITICAL: Skills must handle different project types with appropriate workflows.**

This repository follows the **type: skills** project model. All repositories using these skills declare their type in CLAUDE.md to enable appropriate commit workflows, documentation sync, and validation.

### Project Types

| Type | When to Use |
|------|-------------|
| **`skills`** | Claude Code skill repositories |
| **`java`** | Java/Maven/Gradle projects |
| **`blog`** | GitHub Pages / Jekyll blogs |
| **`custom`** | Working groups, research, docs with custom sync |
| **`generic`** | Everything else |

This repository is **type: skills**. See 📖 **[docs/PROJECT-TYPES.md](docs/PROJECT-TYPES.md)** for full type definitions, routing logic, and when to use each.

The remainder of this document focuses on **type: skills** specific guidance.

## Skill Architecture

### Frontmatter Requirements

Every `SKILL.md` requires YAML frontmatter with exactly two fields:

```yaml
---
name: skill-name-with-hyphens
description: >
  Use when [specific triggering conditions and symptoms]
---
```

**Critical: Claude Search Optimization (CSO)**

The `description` field determines when Claude loads the skill. Follow these rules:

- **Start with "Use when..."** to focus on triggering conditions
- **NEVER summarize the skill's workflow** in the description
- Describe the *problem* or *symptoms*, not the solution
- Keep under 500 characters if possible
- Third person only (no "I" or "you")

**Why this matters:** If the description summarizes the workflow, Claude may follow the description instead of reading the full skill content. Descriptions are for *when to use*, skill body is for *how to use*.

❌ Bad: `description: Use when executing plans - dispatches subagent per task with code review between tasks`

✅ Good: `description: Use when executing implementation plans with independent tasks in the current session`

### Naming Conventions

Skills follow a hierarchical naming pattern:

**Router skills** (language-agnostic entry points with lazy-loaded content files):
- `code-review` — routes to `java.md`, `typescript.md`, or `python.md` based on project type
- `security-audit` — same pattern
- `dependency-update` — routes to `maven.md`, `npm.md`, or `pip.md`
- `git-commit` — routes to `java.md`, `custom.md`, or generic
- `update-design` — routes to `java.md`, `typescript.md`, or `python.md`
- `project-health` — routes to `java.md`, `typescript.md`, `python.md`, `skills-repo.md`, `blog.md`, `custom.md`

**Language-specific dev skills** (one per language — auto-trigger on file type):
- `java-dev` — Java/Quarkus development
- `ts-dev` — TypeScript development
- `python-dev` — Python development
- `quarkus-flow` — Quarkus Flow patterns

**Why this matters:** Language content lives in `.md` files inside the router skill's directory, not in separate skills. A Java developer only loads Java content — Python/TS content files are never read. When adding a new language to an existing router, add a content file (e.g. `code-review/go.md`) and update the dispatch table in `SKILL.md`.

#### Extending to New Languages

When adding a new language, apply these principles consistently:

1. **Canonical names** — `go-dev` not `golang-dev`, `ts-dev` not `javascript-dev`
2. **1-word base suffix** — `*-dev`, `*-code-review`, `*-security-audit` (not `*-development`)
3. **Tool prefix for tools** — `cargo-*`, `npm-*`, `pip-*`, `gomod-*`
4. **Framework prefix for frameworks** — `react-*`, `vue-*`, `django-*`
5. **Consistency over brevity** — match existing patterns

Quick check: `ls -d *-dev *-code-review *-security-audit 2>/dev/null` should show a consistent pattern.

### Skill Chaining

Skills explicitly reference each other to create workflows. The README documents the complete chaining matrix, but when editing skills:

1. **Add cross-references in "Skill Chaining" sections** (capitalized, not "Skill chaining")
2. **Make references bidirectional** when appropriate (e.g., `security-audit` ↔ `code-review`)
3. **Use Prerequisites sections** for layered skills (e.g., quarkus-flow-testing builds on java-dev and quarkus-flow-dev)
4. **Generic principles skills are never invoked directly** — they're referenced via Prerequisites by language/framework-specific skills

Example chaining patterns:
```
# Java repository:
java-dev → code-review → git-commit → update-design + update-claude-md (automatic)

# Any repository:
git-commit → update-claude-md (automatic)
```

### Canonical Path Resolution Block

Several workspace-aware skills (`work-start`, `work-end`, `work-pause`, `work-resume`,
`java-update-design`, `handover`, `adr`) resolve workspace and project paths via symlinks.
The canonical form — identical across all of them — is:

```bash
WORKSPACE=$(git rev-parse --show-toplevel 2>/dev/null)
PROJECT=$(readlink -f proj 2>/dev/null)
[ -z "$PROJECT" ] && { echo "⚠️ No proj/ symlink found. Run workspace-init to set up."; exit 1; }
```

**Do not use CLAUDE.md parsing** (`grep "**Workspace:**"`, `grep "add-dir"`) for path
detection. The `proj/` → project and `wksp/` → workspace symlinks (created by `workspace-init`)
are the single source of truth. CLAUDE.md fields are human documentation only.

If you edit any of these skills, keep this block identical. Verify with:
```bash
grep -l "readlink -f proj" ~/claude/cc-praxis/*/SKILL.md
```

There is no include mechanism in cc-praxis — this duplication is intentional and the block
is short enough to audit by eye. If a shared-snippet mechanism is ever added, this is the
first candidate. (Tracked in cc-praxis#35 or similar.)

### Supporting Files

When skill content exceeds ~200 words or includes heavy reference material:

- Extract to separate `.md` files (e.g., `funcDSL-reference.md`)
- Reference from main `SKILL.md`
- Keep skill body focused on workflow and principles

Pattern:
```
skill-name/
  SKILL.md              # Main workflow (required)
  reference-name.md     # Heavy API/reference docs
  defaults/             # Bundled defaults (optional) — mandatory rules,
                        #   common voice, or other files the skill always loads
                        #   regardless of user configuration
```

### Skills-Repository-Specific Documentation

**This repository has modularized documentation for skills-specific workflows to avoid token waste in other projects.**

Skills-repository-specific logic (SKILL.md validation, README synchronization) is NOT implemented as portable skills. Instead, it lives in standalone documentation files at the repository root:

| File | Purpose | Used By |
|------|---------|---------|
| **docs/development/skill-validation.md** | SKILL.md validation workflow (frontmatter, CSO, flowcharts, cross-references) | `git-commit` when type: skills AND SKILL.md files staged |
| **docs/development/readme-sync.md** | README.md synchronization workflow (skill collection changes) | `git-commit` when type: skills AND skill changes detected |

**Why modularized (not skills):**
- These workflows only apply to THIS repository
- Loading them as skills wastes tokens in all other projects (java, custom, generic)
- They contain heavy reference material (checklists, tables, patterns)
- CLAUDE.md loads automatically in every conversation
- git-commit can reference these files when operating in type: skills mode

**When git-commit operates in type: skills mode:**
1. Check for staged SKILL.md files → Follow docs/development/skill-validation.md workflow
2. Check for skill collection changes → Follow docs/development/readme-sync.md workflow
3. Both workflows maintain the same confirmation pattern (propose → user YES → apply)

**These files are NOT:**
- Loaded in java/custom/generic projects (zero token cost)
- Duplicated in skills collection (single source of truth)
- Skills themselves (not in skills/ directory, not user-invocable via Skill tool)

### Flowcharts

Skills use Mermaid `flowchart TD` notation for decision flows. Add flowcharts when:

- Decision points are non-obvious
- Process has loops where you might stop too early
- "When to use A vs B" decisions exist

Never use flowcharts for:
- Reference material (use tables)
- Code examples (use markdown blocks)
- Linear instructions (use numbered lists)

Flowcharts must have semantic labels, not generic ones like `step1`, `helper2`.

### Success Criteria

Skills that produce artifacts (commits, ADRs, dependency updates) include explicit "Success Criteria" sections with checkboxes. This prevents premature completion claims.

Example pattern:
```markdown
## Success Criteria

Dependency update is complete when:

- ✅ User has confirmed changes with **YES**
- ✅ BOM alignment verified (no version drift)
- ✅ Compilation succeeds (`mvn compile` passes)
- ✅ pom.xml changes committed

**Not complete until** all criteria met and changes committed.
```

## Modular Documentation

**Note:** This repository does not use modular documentation. Skills use single `SKILL.md` files.

For projects that split large documents into linked modules (e.g., `DESIGN.md` → `docs/design/architecture.md` + `api.md` + `data-model.md`), with automatic discovery, validation, and sync across all modules:

**See:** 📖 **[QUALITY.md § Modular Documentation Quality Assurance](QUALITY.md#modular-documentation-quality-assurance)** and **[README.md § Modular Documentation](README.md#modular-documentation)**

## Consistency Patterns

When editing skills, maintain these conventions:

### Section Naming

- "Skill Chaining" (capitalized C)
- "Prerequisites" (for layered skills)
- "Success Criteria" (for artifact-producing skills)
- "Common Pitfalls" (table format: Mistake | Why It's Wrong | Fix)

### Cross-Reference Format

```markdown
## Prerequisites

**This skill builds on `skill-name`**. Apply all rules from:
- **skill-name**: [specific rules that apply]
```

or

```markdown
## Skill Chaining

**Triggered by skill-name:**
When [condition], it should invoke this skill for [reason].

**Chains to skill-name:**
After [milestone], invoke skill-name for [purpose].
```

### Common Mistakes Tables

All major skills include "Common Pitfalls" tables documenting real mistakes:

```markdown
| Mistake | Why It's Wrong | Fix |
|---------|----------------|-----|
| [Anti-pattern] | [Consequence] | [Correct approach] |
```

### Developer-Only Skills

Some skills in this repository are **developer-only** — they require a cloned repository and are not available to marketplace plugin users.

**Rules:**
- ✅ **In `skills/` directory** — auto-discovered and synced for developers
- ❌ **Not in `marketplace.json`** — invisible to the web installer and not installable via plugin
- ❌ **Not in README.md** — not documented as a user-facing skill
- ✅ **SKILL.md includes a prominent DEV-ONLY note** — so Claude knows not to recommend it to marketplace users

**Current developer-only skills:**
- `sync-local` — syncs installed skills from the cloned repo; requires `scripts/claude-skill`

**When adding a new developer-only skill:**
1. Create it in `skills/<name>/` as normal
2. Do NOT add it to `.claude-plugin/marketplace.json`
3. Do NOT add it to README.md
4. Add "DEV-ONLY" prominently in the SKILL.md and CSO description
5. Add it to the "Current developer-only skills" list above

---

### Third-Party Skill Exclusion

**CRITICAL: Only document skills authored by the repository owner.**

**Rules:**
- ✅ **Document only user-authored skills** in README.md and CLAUDE.md
- ❌ **Never document third-party skills** (skills from superpowers:*, external repos, etc.)
- ✅ **Add third-party skills to .gitignore** immediately upon discovery
- ❌ **Never reference third-party skills** in documentation (chaining tables, workflows, repository structure)

**Rationale:**
- Third-party skills are maintained elsewhere and can change without notice
- Documenting them creates maintenance burden and stale references
- Users already see third-party skills in their skill list
- README/CLAUDE.md should reflect only what this repository provides

**Implementation:**
1. Add skill directory to .gitignore with comment: `# Third-party skill from [source]`
2. Remove all references from README.md (Skills section, Chaining table, Repository Structure)
3. Remove all references from CLAUDE.md (Key Skills section)
4. Verify removal with: `grep -r "skill-name" README.md CLAUDE.md`

**Examples of third-party skills to exclude:**
- `superpowers:*` (all superpowers skills)
- `skill-creator` (external skill authoring tool)
- `frontend-design:*` (third-party frontend skills)
- Any skill not in version control for this repository

**Quality check:** During deep analysis, verify no third-party skills are documented anywhere. *This is part of the `consistency` category in `project-health`.*

## Blog

**Blog directory:** `docs/_posts/`

Blog posts are Jekyll posts published at `mdproctor.github.io/cc-praxis/blog/`. Each post needs frontmatter: `layout: post`, `title`, `date`, `type`.

## Writing Style Guide — Mandatory for Blog Content

**The writing style guide at `~/claude-workspace/writing-styles/blog-technical.md` is mandatory for all blog and diary entries. It is not a soft suggestion.**

When invoking `write-blog`:
- Load the guide in full before drafting (Step 0 of the write-blog workflow)
- Complete the pre-draft voice classification before generating any prose (Step 4)
- Verify the draft against "What to Avoid" before showing it — do not show a draft that fails

Do not produce a draft without completing these steps. The guide constrains vocabulary, register, heading style, and what to avoid. A Claude that reads the guide and then generates anyway without classification has not followed it.

---

## Developer Workflow

When working on this repository, use these commands:

```bash
# Sync all skills to ~/.claude/skills/ (do this after any skill change)
python3 scripts/claude-skill sync-local --all -y
# Or use the slash command: /sync-local

# Launch the web skill manager UI
python3 scripts/web_installer.py          # opens http://localhost:8765
# (or: cc-praxis, if bin/ is on PATH via plugin install)

# Regenerate web app data after chaining changes
python3 scripts/generate_web_app_data.py

# Run all tests (1152 tests; ~2m including Playwright UI tests)
python3 -m pytest tests/ -v

# Run commit-tier validators
python3 scripts/validate_all.py --tier commit

# Generate missing slash command files after adding a new skill
python3 scripts/generate_commands.py

# Validate a specific document
python3 scripts/validate_document.py README.md

# Check project type list consistency
python3 scripts/validation/validate_project_types.py --verbose

# Check if a primary doc needs modularising
python3 scripts/validation/validate_doc_structure.py CLAUDE.md

# Revert stray subtype: log → subtype: diary (run periodically until sessions converge)
python3 scripts/revert_diary_subtype.py          # dry-run: shows what needs changing
python3 scripts/revert_diary_subtype.py --apply  # apply changes
```

**After editing any skill:** run `sync-local` so `~/.claude/skills/` has the latest version.
**After adding a new skill:** run `generate_commands.py` AND add to `marketplace.json` plugins list.
**After chaining changes:** run `generate_web_app_data.py` to sync `docs/index.html` CHAIN data.
**Worktrees for feature development:** use `.worktrees/` (gitignored). Create with `git worktree add .worktrees/<name> -b <branch>`. Always use `--force` when removing after subagent use.

## How Claude Code Loads Skills

Skills live in `~/.claude/skills/<skill-name>/` and are **auto-discovered** — no registration in `settings.json` or `installed_plugins.json` is needed.

**Skills must be real directories, not symlinks.** Claude Code does not follow symlinks in `~/.claude/skills/`. A symlink at `~/.claude/skills/java-dev` pointing elsewhere will be silently ignored — the skill will not load. Always copy skill content into the directory.

**Skills need a `commands/` directory for slash commands.** A skill with only a `SKILL.md` is loadable by Claude via the Skill tool (triggered automatically by description matching) but does NOT create a `/skill-name` slash command in the UI. To register a slash command, the skill must have `commands/<skill-name>.md`. Use `python scripts/generate_commands.py` to create missing command files.

**`~/.claude/plugins/cache/` is the wrong location.** That directory is managed by Claude Code's marketplace package manager (`/plugin install`). Skills placed there work but require versioned subdirectories, `installed_plugins.json` registration, and `enabledPlugins` entries — unnecessary complexity. Always use `~/.claude/skills/` instead.

## Editing Skills

When modifying existing skills:

1. **Check README first** — the Skill Chaining Reference table shows the complete dependency graph
2. **Update cross-references** — if you add chaining, update both skills (source and target)
3. **Preserve CSO descriptions** — don't add workflow summaries to frontmatter
4. **Test flowcharts** — invalid dot syntax breaks skill loading
5. **Maintain Prerequisites** — layered skills (quarkus-flow-*) must reference their foundations

## Pre-Commit Checklist for Skills

**CRITICAL: All items are MANDATORY by default unless explicitly marked (advisory).**

**This is NOT a judgment call. This is NOT optional. This is a DISCIPLINE.**

Failing to execute these checks before committing is a failure, not a decision. The bigger the change, the more critical the checklist becomes. Infrastructure changes especially require documentation sync.

### Mandatory Checks

Run these checks **before every commit** to this repository. *These map to `project-health` categories — see [docs/project-health.md](docs/project-health.md). For a full pre-release health check: `/project-health --all`.*

- [ ] **Commit message** → No `Co-Authored-By`, `Generated-by`, or AI attribution of any kind. Commit messages describe WHAT and WHY only.
- [ ] **New skill added?** → Run `python scripts/generate_commands.py` to create its slash command file `[coverage]`
- [ ] **SKILL.md files modified?** → Follow docs/development/readme-sync.md workflow (NEVER skip, let it decide if README needs updates) `[docs-sync]`
- [ ] **CLAUDE.md modified?** → Follow relevant update workflow if applicable `[docs-sync]`
- [ ] **New validation/testing added?** → Update README.md § Skill Quality & Validation AND QUALITY.md § Implementation Status `[infrastructure]`
- [ ] **New scripts/ files added?** → Update README.md § Repository Structure `[coverage]`
- [ ] **New chaining relationships?** → Update README.md § Skill Chaining Reference `[cross-refs]`
- [ ] **New features added to skills?** → Update README.md § Key Features `[docs-sync]`
- [ ] **Framework changes** (same pattern across multiple skills)? → Document in README.md AND QUALITY.md if validation-related `[consistency]`
- [ ] **Infrastructure changes** (validators, test infrastructure, orchestrators)? → Update README.md § Repository Structure, QUALITY.md § Implementation Status, and this file § Validation Script Roadmap `[infrastructure]`

### When Work Tracking Is Configured

If a project's CLAUDE.md contains `## Work Tracking` with `Issue tracking: enabled`, Claude must:
- **Before any significant task**: check if the request spans multiple concerns; if so, help break it into separate GitHub issues before starting work
- **Before any commit**: check if staged changes span multiple issues; if so, suggest splitting with `git add -p`
- **In every commit message**: include `Refs #N` or `Closes #N` for the related issue
- **For changelogs**: use `gh release create --generate-notes` — do NOT maintain a CHANGELOG.md manually

The `issue-workflow` skill handles setup of this configuration and the pre-commit analysis.

### Skill edits always go to this project first

**NEVER write skill edits directly to `~/.claude/skills/`.** Always edit in this
repository (`~/claude/cc-praxis/`), then run `sync-local` to propagate. This keeps
git history, validation, and marketplace metadata in sync.

### When Adding a New Project Type

**Adding a new project type (e.g. `python`, `go`) requires updating ALL of these:**

- [ ] **`CLAUDE.md` § Project Types table** — add the new type row (this is the canonical source of truth)
- [ ] **`docs/PROJECT-TYPES.md`** — full type documentation and routing logic
- [ ] **`git-commit/SKILL.md`** — routing logic in Step 0 (adds new type branch)
- [ ] **`~/.claude/hooks/check_project_setup.sh`** — live hook (`Choices:` line)
- [ ] **Run `python scripts/validation/validate_project_types.py --verbose`** — confirms no hardcoded lists were missed

**The validator (`validate_project_types.py`) catches hardcoded lists automatically at commit time,
but this checklist ensures the new type is fully wired into all workflows.**

### Known Regressions (Learn From These)

**Regression 1:** Validation framework added to 4 sync workflows but README.md not updated. Root cause: Skipped docs/development/readme-sync.md workflow, rationalized "just internal changes". Result: Documentation drift. Fix: Never skip workflows when SKILL.md files change.

**Regression 2:** Created 14 validators + test infrastructure (17 new files, ~2,800 LOC), committed completion document, but didn't update README.md § Repository Structure or QUALITY.md § Implementation Status until user asked. Root cause: Tunnel vision on Options A/B/C, didn't check Pre-Commit Checklist, rationalized "completion doc is sufficient". Result: Primary documentation out of date. Fix: Infrastructure changes require MORE documentation sync, not less. Checklist is mandatory, not advisory.

---

## Meta-Rule: Checklists Are Mandatory

All checklists in this file and in skills are **MANDATORY** unless explicitly marked `(advisory)`, `Optional:`, or `Consider:`. If not marked, do it.

If a checklist item seems inapplicable: re-read it, check for rationalization, execute it anyway if uncertain. If it truly doesn't apply, document why in the commit message. Never skip silently.

---

## Meta-Rule: Consider Universality First

**CRITICAL: Before applying any fix, ask "Should this be universal?"**

**The principle:**
When you identify a problem and prepare a solution, STOP and consider:
1. **Is this problem specific to one context** (skills repo only, Java only, this file only)?
2. **Or is this a universal problem** (could happen in any project type, any AI assistant, any workflow)?

**If uncertain whether to make it universal → ASK THE USER.**

**Examples:**

| Situation | Narrow Fix | Universal Fix | Correct Choice |
|-----------|------------|---------------|----------------|
| README.md sync missing | Add check to docs/development/readme-sync.md only | Add framework change detection to ALL sync workflows | Universal (ADR-0001) |
| Java-specific BOM issue | Fix in maven-dependency-update | Make dependency management universal | Narrow (Java-specific) |
| Rationalization bypass | Strengthen git-commit Step 2b | Add mandatory checks to all workflows | Universal |
| Quarkus event loop bug | Fix in quarkus-flow-dev | Make concurrency universal | Already universal (code-review-principles) |

**Why this matters:**
- **Prevents repeated regressions** - fix once, applies everywhere
- **Scales to future project types** - type: python, type: rust get the fix automatically
- **Reduces user burden** - don't re-discover same problem per project
- **Forces better design** - universal solutions are often cleaner

**When to stay narrow:**
- Domain-specific knowledge (Java annotations, Quarkus configuration)
- Language syntax (Java vs Python vs Go)
- Tool-specific features (Maven BOM vs npm package.json)
- Framework patterns (Quarkus vs Spring)

**When to go universal:**
- Process failures (rationalization, skipping workflows)
- Quality gates (validation, testing, documentation)
- Human behavior patterns (cognitive biases, shortcuts)
- Infrastructure (scripts, automation, validation)

**The test:** If the problem could plausibly occur in a different project type with different technologies, it's probably universal.

**See:** ADR-0001: Documentation Completeness Must Be Universal, Not Project-Specific

## Workspace Model

Skills write methodology artifacts to a companion workspace, not the project repo.
Full design: `docs/superpowers/specs/2026-04-09-workspace-model-design.md`

- **cc-praxis workspace:** `~/claude/public/cc-praxis/` → GitHub: `mdproctor/wsp-cc-praxis`
- Claude opens in the workspace; project loaded via `add-dir` in workspace CLAUDE.md
- Navigation: `proj/` in workspace → project; `wksp/` in project → workspace

## Key Skills

**Workspace:**
- `workspace-init` — one-time setup; creates `~/claude/private/<project>/` or
  `~/claude/public/<project>/` with routing CLAUDE.md, gitignored project symlink
  via `.git/info/exclude`, and all subdirectories
- `work` — **unified lifecycle entry point**; detects current state and routes automatically: `work` alone starts or resumes, `work end` closes the branch, `work pause` saves state and returns to main. Single command replaces needing to know which lifecycle skill to invoke
- `work-start` — unified entry point for all work; detects branch state (6 states including paused, orphaned, misaligned); creates `issue-NNN-<slug>` branches in both repos atomically; scaffolds `.meta` + `JOURNAL.md` with SHA baseline and design routing; runs platform coherence, protocols, IntelliJ pre-checks; replaces the former "work-start + /epic begin" two-step
- `work-end` — closes the current branch; promotes artifacts per routing config; merges `design/JOURNAL.md` into DESIGN.md with three-way diff preview; posts specs to GitHub issue; closes issue; marks branch with `design/EPIC-CLOSED.md`; returns both repos to main. Replaces "epic close"
- `work-pause` — commits all WIP as a `WIP:` commit on the branch; pushes an entry onto `.pause-stack` on workspace main (supports multiple paused branches); switches both repos to main. No stash used — WIP commit is durable and visible in history
- `work-resume` — reads `.pause-stack`; shows picker if multiple paused branches; rebases selected branch onto current main (picks up work that landed while paused); resets the WIP commit to restore working state; removes entry from stack
- `epic` — **deprecated**. Use `work-start` (replaces `/epic begin`) and `work-end` (replaces `/epic close`). Retained for reference during migration

**Router skills** (dispatch to per-language content files — never load cross-language content):
- `code-review` — Java/TS/Python review via `java.md`, `typescript.md`, `python.md`
- `security-audit` — Java/TS/Python OWASP audit, same pattern
- `dependency-update` — Maven/npm/pip management via `maven.md`, `npm.md`, `pip.md`
- `git-commit` — routes to `java.md`, `custom.md`, or generic
- `update-design` — DESIGN.md sync via `java.md`, `typescript.md`, `python.md`
- `project-health` — universal checks + per-type content files

**Language dev skills** (auto-trigger on file type):
- `java-dev` — Java/Quarkus development
- `ts-dev` — TypeScript development
- `python-dev` — Python development
- `quarkus-flow` — Quarkus Flow patterns

**Workflow integrators:**
- `git-commit` — entry point for all commits; routes by project type
- `issue-workflow` — GitHub issue tracking; invoked automatically when Work Tracking is enabled
- `retro-issues` — on-demand retrospective mapping of git history to epics and issues
- `update-claude-md` — CLAUDE.md sync, invoked by all commit skills
- `docs/development/readme-sync.md` — README.md sync, invoked by `git-commit` for type: skills only
- `adr` — Architecture Decision Records in MADR format
- `design-snapshot` — immutable dated record of design state; links to ADRs rather than duplicating them
- `idea-log` — lightweight living log for undecided possibilities; park ideas before they evaporate, promote to ADR when ready
- `write-content` — universal content creation skill; determines content type (Note/log/musing/idea, Article/explanation/commentary/essay/tutorial/how-to, Brief, News), applies structure principles, anti-slop guidance, and form-specific rules; consumed by write-blog and any future content skill
- `write-blog` — living project diary; captures decisions, pivots, and discoveries in diary voice as they happen; consumes write-content for content type and writing guidance; never revised in hindsight
- `publish-blog` — routes blog entries to external git destinations via blog-routing.yaml; Level 2 blog routing (per-entry cross-posting), independent of epic Level 1 routing
- `handover` — end-of-session HANDOFF.md generator; lazy references to blog, design-snapshot, and CLAUDE.md rather than loading them; invokes write-blog, design-snapshot, and update-claude-md via user-confirmed wrap checklist

**TypeScript/Node.js skills:**
- `ts-dev` — TypeScript development; strict mode, async patterns, error handling, testing
- `ts-code-review` — extends `code-review-principles` for TypeScript/Node.js
- `ts-security-audit` — OWASP Top 10 for TypeScript/Node.js, triggered by `ts-code-review`
- `npm-dependency-update` — npm/yarn/pnpm dependency management, builds on `dependency-management-principles`
- `ts-project-health` — extends project-health for TypeScript projects

**Python skills:**
- `python-dev` — Python development; type hints, async patterns, safety, testing with pytest
- `python-code-review` — extends `code-review-principles` for Python
- `python-security-audit` — OWASP Top 10 for Python, triggered by `python-code-review`
- `pip-dependency-update` — pip/poetry/pipenv dependency management, builds on `dependency-management-principles`
- `python-project-health` — extends project-health for Python projects

**Health & quality skills** (correctness and improvement):
- `project-health` — universal health check; answers "is the project correct, complete, and consistent?"; auto-chains to type-specific skill at tier 3+
- `project-refine` — companion to project-health for improvement opportunities; never blocks work
- `skills-project-health` — extends project-health for type: skills (skill craft, cross-refs, marketplace, validators)
- `java-project-health` — extends project-health for type: java
- `blog-project-health` — extends project-health for type: blog
- `custom-project-health` — extends project-health for type: custom

## Quality Assurance Framework

**Comprehensive validation ensures skills maintain structural integrity, logical soundness, and documentation accuracy.**

Skills are infrastructure code guiding AI behavior across millions of invocations. Quality issues compound exponentially. This framework provides automated checks (scripts), semantic analysis (Claude), and functional testing.

### Quality Assurance Philosophy

**Scripts handle mechanical checks (syntax, structure, format).**
**Claude handles semantic analysis (logic, contradictions, completeness).**

For complete details on the division of labor, validation tiers, what gets checked, and integration patterns:
📖 **[QUALITY.md § The Architecture: Scripts + Claude](QUALITY.md#the-architecture-scripts--claude)**

### Automated Validation

**19 validators across 3 tiers (COMMIT/PUSH/CI):**

For complete validator specifications, tier assignments, and implementation details:
📖 **[QUALITY.md § Implementation Status](QUALITY.md#implementation-status)**

**Pre-Commit Checklist integration:** See § Pre-Commit Checklist for Skills below for mandatory validation hooks.

### Deep Analysis Procedures

When user requests deep analysis of skills ("/skill-review", "do a deep analysis", "comprehensive review"):

*These checks map to the `project-health` skill. Invoke as `/project-health --deep` for a full deep analysis, or with specific categories.*

📖 **[QUALITY.md § Deep Analysis Validation](QUALITY.md#deep-analysis-validation-level-2)**

**Mandatory checklist:**
- [ ] Run all automated checks (`python scripts/validate_all.py --tier commit`)
- [ ] Perform all manual analysis procedures (reference accuracy, logical soundness, completeness, etc.)
- [ ] Document findings by severity (CRITICAL/WARNING/NOTE)
- [ ] Create action items for CRITICAL/WARNING findings

### Validation Infrastructure

For complete inventory of validation scripts by tier:
📖 **[QUALITY.md § Validation Script Roadmap](QUALITY.md#validation-script-roadmap)**

**Quick reference (19 validators total):**
- **COMMIT tier (<2s)**: frontmatter, CSO, references, naming, sections, structure, project-types, blog-frontmatter
- **PUSH tier (<30s)**: flowcharts, cross-document, temporal, usability, edge-cases, behavior, readme-sync, external-links, code-examples, web-app
- **CI tier (<5min)**: Python quality (mypy, flake8, bandit) + functional tests
- **On disk, not yet registered**: validate_blog_commit.py (git hook only — needs commit message), validate_doc_structure.py (on-demand — needs target path)

### Success Criteria

Validation framework is working when:
- ✅ Zero CRITICAL findings pass pre-commit
- ✅ All skills have functional tests
- ✅ All known issues have regression tests
- ✅ Deep analysis finds ≤5 WARNING issues per 40 skills
- ✅ No duplicate issues across releases
- ✅ CI blocks PRs with validation failures

For complete success criteria and implementation status:
📖 **[QUALITY.md § Success Criteria for QA Framework](QUALITY.md#success-criteria-for-qa-framework)**

---

## Document Sync Quality Assurance

**Universal validation prevents document corruption during sync operations across all project types.**

For complete details on validation rules, integration points, corruption patterns, and regression prevention:
📖 **[QUALITY.md § Document Sync Quality Assurance](QUALITY.md#document-sync-quality-assurance)**

### Running Validators Locally

```bash
# Validate a specific document
python scripts/validate_document.py README.md
python scripts/validate_document.py CLAUDE.md

# Validate all staged .md files
git diff --staged --name-only | grep '\.md$' | xargs -I{} python scripts/validate_document.py {}

# Run full commit-tier validation suite
python scripts/validate_all.py --tier commit
```

Exit codes: `0` = clean · `1` = CRITICAL (blocks commit) · `2` = WARNING (review recommended) · `3` = NOTE

Local validation is optional but recommended — pre-commit validation runs automatically and is mandatory.


## Work Tracking

**Issue tracking:** enabled
**GitHub repo:** mdproctor/cc-praxis
**Changelog:** GitHub Releases (run `gh release create --generate-notes` at milestones)

**Automatic behaviours (Claude follows these at all times in this project):**
- **Before implementation begins** — when the user says "implement", "start coding",
  "execute the plan", "let's build", or similar: check if an active issue or epic
  exists. If not, run issue-workflow Phase 1 to create one **before writing any code**.
- **Before writing any code** — check if an issue exists for what's about to be
  implemented. If not, draft one and assess epic placement (issue-workflow Phase 2)
  before starting. Also check if the work spans multiple concerns.
- **Before any commit** — run issue-workflow Phase 3 (via git-commit) to confirm
  issue linkage and check for split candidates. This is a fallback — the issue
  should already exist from before implementation began.
- **All commits should reference an issue** — `Refs #N` (ongoing) or `Closes #N` (done).
  If the user explicitly says to skip ("commit as is", "no issue"), ask once to confirm
  before proceeding — it must be a deliberate choice, not a default.

## Project Artifacts

Paths that are project content (not workspace noise). Skills use this to avoid
filtering or dropping commits that touch these paths.

| Path | What it is |
|------|------------|
| `CLAUDE.md` | Project conventions (build, test, naming) |
| `docs/adr/` | Architecture decision records |
| `docs/protocols/` | Standing rules for taxonomy and conventions (e.g. subtype naming, cleanup patterns) |
