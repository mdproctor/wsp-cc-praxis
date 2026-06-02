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

**Spec path override:** The brainstorming skill defaults to `docs/superpowers/specs/` in the project repo. This project overrides that: specs promote to `docs/specs/` (not `docs/superpowers/specs/`). When writing a spec, use `docs/specs/YYYY-MM-DD-<topic>-design.md`.
| writing-plans (plans) | `plans/` |
| handover | `HANDOFF.md` |
| idea-log | `IDEAS.md` |
| design-snapshot | `snapshots/` |
| adr | `adr/` |
| write-blog | `blog/` |

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
| blog       | workspace   | staged here; routed via publish-blog and blog-routing.yaml at work end |
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

📖 **[docs/development/skill-architecture.md](docs/development/skill-architecture.md)** — frontmatter requirements, naming conventions, chaining patterns, canonical path resolution, supporting files, consistency rules, developer-only and third-party skill policies.

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

### When Adding a Skill to a Bundle (or Changing Bundles)

Bundle membership is defined in **`.claude-plugin/marketplace.json` § `bundles`** — the single source of truth. The install/uninstall wizard skills read this at runtime, so the menus and counts update automatically.

**To add a skill to an existing bundle:**
1. Edit `.claude-plugin/marketplace.json` — add the skill name to the relevant `bundles[].skills` array
2. Commit and push — the wizard picks it up immediately

**To create a new bundle:**
1. Edit `.claude-plugin/marketplace.json` — add a new entry to the `bundles` array with `name`, `displayName`, `description`, and `skills`
2. No changes needed to `install-skills/SKILL.md` or `uninstall-skills/SKILL.md` — they render bundles dynamically

**To remove a skill from a bundle:**
1. Edit `.claude-plugin/marketplace.json` — remove the skill name from the bundle's `skills` array

**Never** add bundle membership or skill counts directly to `install-skills/SKILL.md` or `uninstall-skills/SKILL.md` — they will drift.

### When Adding a New Project Type

**Adding a new project type (e.g. `python`, `go`) requires updating ALL of these:**

- [ ] **`CLAUDE.md` § Project Types table** — add the new type row (this is the canonical source of truth)
- [ ] **`docs/PROJECT-TYPES.md`** — full type documentation and routing logic
- [ ] **`git-commit/SKILL.md`** — routing logic in Step 0 (adds new type branch)
- [ ] **`install-skills/SKILL.md`** — hook script it creates (`Choices:` line)
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

Before applying any fix, ask: is this problem specific to one context, or could it occur in any project type? If uncertain → ask the user.

**Stay narrow for:** domain-specific knowledge, language syntax, tool-specific features.
**Go universal for:** process failures, quality gates, cognitive biases, infrastructure.

The test: if the problem could plausibly occur in a different project type, it's probably universal. See ADR-0001.

## Workspace Model

Skills write methodology artifacts to a companion workspace, not the project repo.
Full design: `docs/superpowers/specs/2026-04-09-workspace-model-design.md`

- **cc-praxis workspace:** `~/claude/public/cc-praxis/` → GitHub: `mdproctor/wsp-cc-praxis`
- Claude opens in the workspace; project loaded via `add-dir` in workspace CLAUDE.md
- Navigation: `proj/` in workspace → project; `wksp/` in project → workspace

## Key Skills

📖 **[README.md § Key Skills](README.md#key-skills)** — complete skill directory with chaining relationships.

## Quality Assurance Framework

19 validators across 3 tiers. Scripts handle mechanical checks; Claude handles semantic analysis.

📖 **[QUALITY.md](QUALITY.md)** — full framework, validator specs, deep analysis procedures, and success criteria.

**Run validators locally:**
```bash
python scripts/validate_all.py --tier commit   # pre-commit suite
python scripts/validate_document.py README.md  # single file
```


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
