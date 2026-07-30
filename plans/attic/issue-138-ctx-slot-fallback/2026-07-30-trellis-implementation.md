# Trellis Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #117 — UI for work lifecycle management
**Issue group:** #117

**Goal:** Build Trellis — a local Electron app backed by a Quarkus
sidecar that serves as an epic delivery engine for spec-driven parallel
development across multi-repo organisations.

**Architecture:** Electron shell launches a Quarkus sidecar on a dynamic
port (sparge pattern). The sidecar provides workspace scanning, tmux
session management, GitHub issue intelligence with critical path
analysis, and work lifecycle operations via REST/SSE. The UI is built
with pages + blocks-ui, served by Quinoa. The LLM coordinator is a
Claude Code session in a managed tmux session.

**Tech Stack:** Java 21, Quarkus 3.x, Quinoa, pages, blocks-ui,
pages-push, Electron, xterm.js, tmux, GitHub API (gh CLI + REST)

**Spec:** `specs/issue-117-work-ui/2026-07-30-trellis-design.md`

## Global Constraints

- Repository: `hortora/trellis` (own repo, not inside soredium)
- Electron shell: no Node.js application logic, only window management
  and sidecar lifecycle (sparge pattern)
- Quarkus sidecar: Quinoa serves pages/blocks-ui frontend, pages-push
  for SSE, REST for API
- Pages/blocks-ui consumed as Maven SNAPSHOT artifacts via portal:
  resolutions (quinoa-convention.md)
- Tmux management: own TmuxManager, `trellis-*` naming convention, no
  claudony dependency
- Python scripts: existing soredium lifecycle scripts called via
  ProcessBuilder, path via `trellis.skills.path` config property
- GitHub API: gh CLI for issue operations, conditional requests
  (ETag/If-None-Match) for rate limit management
- Layout persistence: electron-store (JSON), per-workspace
- All real-time UI updates via pages-push SSE (TopicRegistry,
  EventBroadcaster, PushMessage)

---

## Task 1: Project Skeleton (Batch 0)

**Creates the hortora/trellis repository with working Electron + Quarkus
sidecar. After this task, `npm start` launches the app and shows a blank
pages-based UI.**

**Files:**
- Create: `hortora/trellis/` (new repository)
- Create: `sidecar/pom.xml` — Quarkus app with Quinoa, pages-push deps
- Create: `sidecar/src/main/java/io/hortora/trellis/TrellisApp.java`
- Create: `sidecar/src/main/webui/package.json` — Quinoa frontend
- Create: `sidecar/src/main/webui/src/index.ts` — blank pages app entry
- Create: `sidecar/src/main/resources/application.properties`
- Create: `shell/package.json` — Electron app
- Create: `shell/main.js` — Electron main process (sparge pattern)
- Create: `shell/java-server.js` — sidecar lifecycle (port, spawn, poll)
- Create: `shell/preload.js` — Electron preload script
- Create: `CLAUDE.md` — project conventions
- Create: `.github/workflows/build.yml` — CI

**Interfaces:**
- Produces: Quarkus sidecar on dynamic port serving pages UI via Quinoa
- Produces: Electron shell that launches sidecar and opens BrowserWindow
- Produces: `GET /api/health` endpoint for readiness polling

**Steps:**

- [ ] **Step 1: Create GitHub repo**

```bash
gh repo create Hortora/trellis --public --description "Epic delivery engine for spec-driven parallel development"
git clone git@github.com:Hortora/trellis.git ~/claude/hortora/trellis
```

- [ ] **Step 2: Create sidecar Maven project**

`sidecar/pom.xml` with:
- `io.hortora:trellis-sidecar`
- Quarkus 3.x BOM
- Dependencies: `quarkus-rest`, `quarkus-rest-jackson`, `quarkus-quinoa`,
  `casehub-pages-push` (SNAPSHOT), `casehub-pages-npm` (SNAPSHOT),
  `casehub-blocks-ui-npm` (SNAPSHOT)
- Quinoa config in `application.properties`:
  `quarkus.quinoa.package-manager=yarn`
  `quarkus.quinoa.build-dir=dist`

- [ ] **Step 3: Create Quinoa frontend**

`sidecar/src/main/webui/package.json` with portal: resolutions for
pages and blocks-ui packages (per quinoa-convention.md). Minimal
`src/index.ts` that renders a pages `split-layout` with a heading:
"Trellis — no workspace loaded."

- [ ] **Step 4: Create health endpoint**

`TrellisApp.java` — `@Path("/api")` REST resource with `GET /health`
returning `{"status": "ok"}`.

- [ ] **Step 5: Verify sidecar boots**

```bash
cd sidecar && mvn quarkus:dev
# In another terminal: curl http://localhost:8080/api/health
# Expected: {"status": "ok"}
# Browser: http://localhost:8080 shows "Trellis — no workspace loaded"
```

- [ ] **Step 6: Create Electron shell**

`shell/package.json` with electron dependency.
`shell/main.js` — following sparge pattern:
- `findFreePort()` — net.createServer on port 0
- `spawnServer(port)` — `java -Dquarkus.http.port=${port} -jar sidecar.jar`
- `pollUntilReady(port)` — poll `GET /api/health` every 200ms, 20s timeout
- `createWindow(port)` — BrowserWindow at `http://127.0.0.1:${port}/`
- `before-quit` — SIGTERM → 5s → SIGKILL (sidecar survives for dev;
  killed in production mode)

`shell/java-server.js` — sidecar process lifecycle with crash recovery
(3 retries, exponential backoff, 60s stability reset).

- [ ] **Step 7: Build sidecar uber-jar**

```bash
cd sidecar && mvn package -DskipTests
# Produces: sidecar/target/trellis-sidecar-runner.jar
```

- [ ] **Step 8: Wire npm start**

Root `package.json` with `"start": "cd shell && electron ."`.
`shell/package.json` points `main` to `main.js`, `extraResources` to
`../sidecar/target/trellis-sidecar-runner.jar`.

- [ ] **Step 9: End-to-end test**

```bash
npm start
# Expected: Electron window opens, shows "Trellis — no workspace loaded"
# Sidecar log visible in terminal
```

- [ ] **Step 10: Write CLAUDE.md**

Project type: `java, ts`. Stage: `pre-release`.
Build commands, test commands, key conventions.

- [ ] **Step 11: Commit and push**

```bash
git add -A && git commit -m "feat: project skeleton — Electron + Quarkus sidecar with Quinoa frontend"
git push -u origin main
```

---

## Task 2: Workspace Scanner (Batch 1a)

**Scans a root directory and builds the org model — repos, slots, paused
work, adhoc branches, epics. Pushes live updates via pages-push SSE.**

**Files:**
- Create: `sidecar/src/main/java/io/hortora/trellis/scanner/WorkspaceScanner.java`
- Create: `sidecar/src/main/java/io/hortora/trellis/scanner/RepoInfo.java` (record)
- Create: `sidecar/src/main/java/io/hortora/trellis/scanner/SlotInfo.java` (record)
- Create: `sidecar/src/main/java/io/hortora/trellis/scanner/PauseEntry.java` (record)
- Create: `sidecar/src/main/java/io/hortora/trellis/scanner/EpicInfo.java` (record)
- Create: `sidecar/src/main/java/io/hortora/trellis/scanner/WorkspaceModel.java`
- Create: `sidecar/src/main/java/io/hortora/trellis/scanner/FileWatcherService.java`
- Create: `sidecar/src/main/java/io/hortora/trellis/scanner/ScannerResource.java`
- Create: `sidecar/src/main/webui/src/views/org-dashboard.ts` — repo list + slot map
- Test: `sidecar/src/test/java/io/hortora/trellis/scanner/WorkspaceScannerTest.java`
- Test: `sidecar/src/test/java/io/hortora/trellis/scanner/ScannerResourceTest.java`

**Interfaces:**
- Produces: `GET /api/workspace/{root}` → `WorkspaceModel` (repos, slots, pauses, epics)
- Produces: SSE topic `workspace/{root}/repos`, `workspace/{root}/slots`
- Produces: CDI event `@WorkspaceChanged` for direct notification from lifecycle manager

**Key implementation details:**
- `WorkspaceScanner.scan(Path root)` — walks directory tree, finds `.git/`
  dirs (repos), `worktrees/` dirs (slots), `.pause-stack` files, `.meta`
  files, `.slot` files, `.epic` files
- `FileWatcherService` — Java WatchService on key directories + 60s
  rescan fallback for macOS kqueue event loss
- `WorkspaceModel` — immutable snapshot, rebuilt on each scan/watch event
- Pages-push integration: `TopicRegistry` + `EventBroadcaster` for SSE
- Failure modes: corrupted files → warn + exclude, `.git/index.lock` →
  skip, WatchService overflow → full rescan
- Org dashboard UI: `pages-table` for repo list, `work-item-inbox` for
  slot/work map, `kpi-metric-row` for top-bar stats

**TDD approach:**
- Unit test scanner with a tmp directory tree containing sample repos,
  slots, .meta, .pause-stack files
- Integration test REST endpoint with @QuarkusTest
- Test failure modes: corrupted .slot file, missing .git, locked index

---

## Task 3: Tmux Manager + Terminal WebSocket (Batch 1b)

**Trellis-owned tmux session management with xterm.js streaming via
WebSocket. FIFO pipe architecture matching claudony's proven pattern.**

**Files:**
- Create: `sidecar/src/main/java/io/hortora/trellis/terminal/TmuxManager.java`
- Create: `sidecar/src/main/java/io/hortora/trellis/terminal/SessionInfo.java` (record)
- Create: `sidecar/src/main/java/io/hortora/trellis/terminal/SessionRegistry.java`
- Create: `sidecar/src/main/java/io/hortora/trellis/terminal/TerminalWebSocket.java`
- Create: `sidecar/src/main/java/io/hortora/trellis/terminal/TerminalResource.java`
- Create: `sidecar/src/main/webui/src/components/terminal-panel.ts` — xterm.js wrapper
- Test: `sidecar/src/test/java/io/hortora/trellis/terminal/TmuxManagerTest.java`
- Test: `sidecar/src/test/java/io/hortora/trellis/terminal/SessionRegistryTest.java`

**Interfaces:**
- Produces: `POST /api/sessions` (create), `DELETE /api/sessions/{id}` (destroy)
- Produces: `GET /api/sessions` (list), `GET /api/sessions/{id}` (detail)
- Produces: `WS /ws/terminal/{sessionId}/{cols}/{rows}` — bidirectional terminal stream
- Produces: `POST /api/sessions/{id}/input` — send keystrokes (for LLM coordinator)
- Produces: tmux session options `@trellis_slot`, `@trellis_repo`, `@trellis_issue`

**Key implementation details:**
- `TmuxManager` — ProcessBuilder wrapper: `createSession(name, workDir)`,
  `killSession(name)`, `sendKeys(name, text)`, `capturePane(name)`,
  `listSessions()`, `hasSession(name)`, `setOption(name, key, val)`,
  `getOption(name, key)`
- `SessionRegistry` — ConcurrentHashMap, `trellis-*` prefix filter,
  `bootstrapRegistry()` re-discovers on restart
- `TerminalWebSocket` — `@WebSocket(path="/ws/terminal/{id}/{cols}/{rows}")`:
  `tmux pipe-pane -t {id} "cat > {fifo}"` → virtual thread reads FIFO →
  WebSocket send. Input: WebSocket → `tmux send-keys -t {id} -l "{text}"`
- Session metadata stored as tmux options (not a separate file)
- Naming convention: `trellis-slot-{N}-{repo}` for slot sessions,
  `trellis-coordinator` for LLM coordinator
- xterm.js component: Lit web component wrapping xterm.js + addon-fit +
  addon-webgl, connects to `ws://localhost:{port}/ws/terminal/{id}/{cols}/{rows}`

**TDD approach:**
- Unit test TmuxManager with real tmux (requires tmux installed)
- Unit test SessionRegistry with mock TmuxManager
- Integration test WebSocket with @QuarkusTest + WebSocket client
- Test failure modes: session dies, FIFO stalls, restart recovery

---

## Task 4: Multi-Window Electron (Batch 1c)

**Electron multi-window management: detach/attach panels, tab management,
layout persistence, multi-monitor support.**

**Files:**
- Create: `shell/window-manager.js` — multi-window lifecycle
- Create: `shell/layout-store.js` — electron-store persistence
- Modify: `shell/main.js` — integrate window manager
- Create: `sidecar/src/main/webui/src/components/tab-bar.ts`
- Create: `sidecar/src/main/webui/src/components/detachable-panel.ts`
- Test: `shell/test/window-manager.test.js` — Jest unit tests

**Interfaces:**
- Produces: `WindowManager.createWindow(route, opts)` — new BrowserWindow
- Produces: `WindowManager.detachPanel(panelId)` — panel → new window
- Produces: `WindowManager.attachPanel(panelId, windowId)` — panel → existing window
- Produces: `LayoutStore.save(workspaceId, layout)` / `LayoutStore.load(workspaceId)`
- Produces: Electron IPC channels: `panel:detach`, `panel:attach`,
  `layout:save`, `layout:restore`

**Key implementation details:**
- `WindowManager` — tracks all BrowserWindows, routes each to
  `http://127.0.0.1:{port}/#{panelRoute}`
- `LayoutStore` — electron-store JSON, keyed by workspace root path.
  Stores: window positions, sizes, screen index, tab order per window
- Electron `screen` API for monitor detection and per-monitor memory
- Terminal transfer semantics: detaching a terminal sends IPC
  `terminal:transfer` to close WebSocket in old window, open in new
- Tab bar: Lit component managing tab state, drag-to-reorder

**TDD approach:**
- Jest tests for WindowManager with mocked Electron APIs
- Jest tests for LayoutStore serialization/deserialization
- Manual test: detach panel, move to second monitor, restart app,
  verify layout restored

---

## Task 5: Project Bootstrapper (Batch 1d)

**First-run onboarding: pick a known project, clone and set up
everything automatically.**

**Files:**
- Create: `sidecar/src/main/java/io/hortora/trellis/bootstrap/BootstrapperResource.java`
- Create: `sidecar/src/main/java/io/hortora/trellis/bootstrap/ProjectRegistry.java`
- Create: `sidecar/src/main/java/io/hortora/trellis/bootstrap/BootstrapRunner.java`
- Create: `sidecar/src/main/resources/projects.json` — built-in registry
- Create: `sidecar/src/main/webui/src/views/launcher.ts` — workspace picker
- Test: `sidecar/src/test/java/io/hortora/trellis/bootstrap/ProjectRegistryTest.java`

**Interfaces:**
- Produces: `GET /api/projects` — list known projects
- Produces: `POST /api/bootstrap/{projectId}` — start bootstrap (async)
- Produces: SSE topic `bootstrap/{projectId}/progress` — progress updates
- Consumes: WorkspaceScanner (Task 2) — after bootstrap, scanner picks up new tree

**Key implementation details:**
- `projects.json` registry: `[{id, name, description, parentRepoUrl,
  setupCommand, expectedStructure}]`. Initial entry: casehub.
- `BootstrapRunner` — runs in a managed tmux session:
  `git clone {parentRepoUrl} ~/claude/{id}/parent` →
  runs parent's setup script → workspace scanner picks up the result
- Launcher UI: recent workspaces list (from `~/.trellis/workspaces.json`)
  + "Set up a project" button → project picker → bootstrap progress

**TDD approach:**
- Unit test ProjectRegistry loading and validation
- Integration test bootstrap with a mock git repo
- Test: launcher shows recent workspaces, bootstrap progress streams

---

## Task 6: Issue Engine + Dependency Graph (Batch 2a)

**GitHub API integration: fetch issues, parse dependencies, build DAG,
cache locally.**

**Files:**
- Create: `sidecar/src/main/java/io/hortora/trellis/issues/IssueEngine.java`
- Create: `sidecar/src/main/java/io/hortora/trellis/issues/IssueInfo.java` (record)
- Create: `sidecar/src/main/java/io/hortora/trellis/issues/DependencyGraph.java`
- Create: `sidecar/src/main/java/io/hortora/trellis/issues/DependencyParser.java`
- Create: `sidecar/src/main/java/io/hortora/trellis/issues/IssueCache.java`
- Create: `sidecar/src/main/java/io/hortora/trellis/issues/IssueResource.java`
- Create: `sidecar/src/main/webui/src/views/issue-browser.ts`
- Test: `sidecar/src/test/java/io/hortora/trellis/issues/DependencyParserTest.java`
- Test: `sidecar/src/test/java/io/hortora/trellis/issues/DependencyGraphTest.java`
- Test: `sidecar/src/test/java/io/hortora/trellis/issues/IssueCacheTest.java`

**Interfaces:**
- Produces: `GET /api/repos/{owner}/{repo}/issues` → grouped issue list
- Produces: `GET /api/repos/{owner}/{repo}/issues/{n}/graph` → dependency DAG
- Produces: `DependencyGraph.criticalPath(epicIssueN)` → ordered issue list
- Produces: `DependencyGraph.bottlenecks()` → ranked by unlocking impact
- Consumes: WorkspaceScanner (Task 2) — repo list for multi-repo issue fetching

**Key implementation details:**
- `DependencyParser` — parse `blocked-by:#N` labels, `## Dependencies`
  checklist in issue body, cross-repo `owner/repo#N` references
- `DependencyGraph` — DAG with Kahn's algorithm for cycle detection,
  longest-path for critical path computation
- `IssueCache` — `~/.trellis/cache/{owner}/{repo}/issues.json`, ETag
  conditional requests, refresh on demand + 5-minute interval
- `gh` CLI for issue fetching: `gh issue list --repo {owner}/{repo}
  --json number,title,body,labels,state,closedAt --limit 500`
- Issue browser UI: `pages-table` with grouping by epic/label, blocked-by
  indicators, search/filter

**TDD approach:**
- Unit test DependencyParser with sample issue bodies and labels
- Unit test DependencyGraph: linear chain, diamond, cycle, cross-repo
- Unit test IssueCache: hit/miss, ETag refresh, staleness
- Integration test IssueResource with @QuarkusTest

---

## Task 7: Work Lifecycle Manager (Batch 2b)

**REST API wrapping soredium's externalised lifecycle scripts. Button-click
operations without needing Claude Code.**

**Files:**
- Create: `sidecar/src/main/java/io/hortora/trellis/lifecycle/LifecycleManager.java`
- Create: `sidecar/src/main/java/io/hortora/trellis/lifecycle/ScriptRunner.java`
- Create: `sidecar/src/main/java/io/hortora/trellis/lifecycle/LifecycleResource.java`
- Create: `sidecar/src/main/java/io/hortora/trellis/lifecycle/OperationResult.java` (record)
- Test: `sidecar/src/test/java/io/hortora/trellis/lifecycle/ScriptRunnerTest.java`
- Test: `sidecar/src/test/java/io/hortora/trellis/lifecycle/LifecycleResourceTest.java`

**Interfaces:**
- Produces: `POST /api/lifecycle/start` — create slot + scaffold
- Produces: `POST /api/lifecycle/end/{slotId}` — land branch (fast lane, skip review)
- Produces: `POST /api/lifecycle/pause/{slotId}` — WIP commit + stack
- Produces: `POST /api/lifecycle/resume/{slotId}` — pop stack + rebase
- Produces: `POST /api/lifecycle/slot/create` — create worktree slot
- Produces: `POST /api/lifecycle/slot/{id}/merge` — land ready slot
- Produces: `POST /api/lifecycle/epic/setup` — parse children + batch plan
- Produces: `POST /api/lifecycle/epic/{id}/next` — advance to next issue
- Produces: CDI event `@WorkspaceChanged` → consumed by WorkspaceScanner
- Consumes: TmuxManager (Task 3) — operations execute in tmux sessions
- Consumes: WorkspaceScanner (Task 2) — direct notification after mutations

**Key implementation details:**
- `ScriptRunner` — ProcessBuilder wrapper with KEY=value stdout parsing,
  non-zero exit → error. `trellis.skills.path` config for script location
  (default: `~/.claude/skills/`)
- `LifecycleManager` — per-slot mutex (ConcurrentHashMap of locks), HTTP
  409 on concurrent access. Two-tier model: button-click (fast lane) vs
  terminal (full-featured via Claude Code session)
- Operations table maps REST endpoints to script calls (see spec §4.4)
- After each mutation, fires `@WorkspaceChanged` CDI event so scanner
  updates immediately

**TDD approach:**
- Unit test ScriptRunner with a mock Python script
- Unit test LifecycleManager mutex behaviour
- Integration test endpoints with @QuarkusTest (mock script execution)
- Test concurrent access → 409 response

---

## Task 8: Terminal Panels in UI (Batch 2c)

**xterm.js terminal panels integrated into the slot detail view.
Tab layout for multi-repo slots.**

**Files:**
- Modify: `sidecar/src/main/webui/src/components/terminal-panel.ts` (from Task 3)
- Create: `sidecar/src/main/webui/src/views/slot-detail.ts`
- Create: `sidecar/src/main/webui/src/components/terminal-tab-group.ts`
- Modify: `shell/window-manager.js` — terminal transfer on detach

**Interfaces:**
- Consumes: TmuxManager (Task 3) — WebSocket terminal streaming
- Consumes: WorkspaceScanner (Task 2) — slot metadata for sidebar
- Consumes: WindowManager (Task 4) — detach terminal → new window

**Key implementation details:**
- Slot detail view: left = terminal tab group (one tab per repo in slot),
  right sidebar = issue context, .meta info, epic progress, spec list
- Terminal tab group: Lit component wrapping N terminal-panels, tab bar
  for switching, active terminal gets WebSocket connection
- Terminal transfer: on detach, close WebSocket in old window, new
  window reconnects to same tmux session via new WebSocket
- Actions bar: work-end (skip review), work-pause, work-next (if epic)

**TDD approach:**
- Lit component tests for terminal-tab-group
- Manual test: open slot detail, switch tabs, detach terminal to new
  window, verify streaming continues

---

## Task 9: Critical Path + Recommendations (Batch 3)

**The core value: critical path computation, bottleneck detection,
parallel opportunity analysis, and algorithmic recommendations.**

**Files:**
- Create: `sidecar/src/main/java/io/hortora/trellis/planning/CriticalPathEngine.java`
- Create: `sidecar/src/main/java/io/hortora/trellis/planning/RecommendationEngine.java`
- Create: `sidecar/src/main/java/io/hortora/trellis/planning/Recommendation.java` (record)
- Create: `sidecar/src/main/java/io/hortora/trellis/planning/PlanningResource.java`
- Create: `sidecar/src/main/webui/src/views/epic-dashboard.ts`
- Create: `sidecar/src/main/webui/src/components/dag-view.ts`
- Test: `sidecar/src/test/java/io/hortora/trellis/planning/CriticalPathEngineTest.java`
- Test: `sidecar/src/test/java/io/hortora/trellis/planning/RecommendationEngineTest.java`

**Interfaces:**
- Produces: `GET /api/epics/{owner}/{repo}/{n}/critical-path` → ordered issue list
- Produces: `GET /api/epics/{owner}/{repo}/{n}/bottlenecks` → ranked issues
- Produces: `GET /api/epics/{owner}/{repo}/{n}/recommendations` → scored suggestions
- Produces: `GET /api/portfolio/recommendations` → cross-epic suggestions
- Consumes: DependencyGraph (Task 6) — DAG for computation
- Consumes: WorkspaceScanner (Task 2) — active slots for capacity analysis

**Key implementation details:**
- `CriticalPathEngine` — longest path in DAG (DFS with memoisation),
  handles closed issues (zero remaining cost), cross-repo edges
- `RecommendationEngine` — scoring factors:
  - Critical path position (highest weight)
  - Dependency unlocking impact (how many issues unblocked)
  - Epic momentum (near-complete epics get priority)
  - Scale fit (match issue scale label to available slot time)
  - Idle capacity (repos with no active slots)
- Cross-epic portfolio: shared dependencies, resource contention
- Epic dashboard UI: `kpi-metric-row` top bar, DAG visualisation
  (ECharts graph), recommendations panel, batch timeline
- DAG view: ECharts graph chart with node colouring (done/active/
  unblocked/blocked), critical path edge highlighting, click → detail

**TDD approach:**
- Unit test CriticalPathEngine: linear, diamond, parallel branches,
  closed nodes, cycles (excluded), cross-repo
- Unit test RecommendationEngine: verify scoring factors, ranking order
- Integration test PlanningResource with sample epic data
- Manual test: epic dashboard with real casehub epic

---

## Task 10: One-Click Work-Start (Batch 4)

**Integration batch: click an issue in the browser → create a slot →
terminals open automatically.**

**Files:**
- Modify: `sidecar/src/main/webui/src/views/issue-browser.ts` — add "Start work" button
- Create: `sidecar/src/main/webui/src/components/start-work-dialog.ts`
- Modify: `sidecar/src/main/webui/src/views/slot-detail.ts` — auto-open after creation
- Modify: `shell/window-manager.js` — open slot window on creation

**Interfaces:**
- Consumes: IssueEngine (Task 6) — issue context for slot creation
- Consumes: LifecycleManager (Task 7) — `POST /api/lifecycle/slot/create`
- Consumes: TmuxManager (Task 3) — terminal sessions for new slot
- Consumes: WindowManager (Task 4) — open slot detail in window

**Key implementation details:**
- "Start work" button on each issue row in the issue browser
- Dialog: confirm issue, choose slot or branch, pick repos for multi-repo
- On confirm: POST lifecycle/slot/create → scanner updates → slot detail
  view opens → terminals created for each repo → Electron window opens
  (or tab added to existing slot window)
- Epic context: if starting from a recommendation, pre-populate the
  dialog with the recommended issue and batch context

**TDD approach:**
- Integration test: create slot via dialog, verify scanner sees it
- Manual test: full flow from issue browser → slot → working terminal

---

## Task 11: LLM Coordinator L1 (Batch 5)

**Claude Code session with org awareness: queries the recommendations
engine, adds natural language reasoning, answers "what should I work on?"**

**Files:**
- Create: `sidecar/src/main/java/io/hortora/trellis/coordinator/CoordinatorService.java`
- Create: `sidecar/src/main/java/io/hortora/trellis/coordinator/CoordinatorResource.java`
- Create: `sidecar/src/main/webui/src/views/coordinator-panel.ts`
- Create: `sidecar/src/main/webui/src/components/recommendation-card.ts`

**Interfaces:**
- Produces: `POST /api/coordinator/start` — start coordinator tmux session
- Produces: `GET /api/coordinator/status` — running/stopped
- Produces: SSE topic `coordinator/output` — coordinator's observations
- Consumes: TmuxManager (Task 3) — `trellis-coordinator` session
- Consumes: RecommendationEngine (Task 9) — algorithmic rankings
- Consumes: WorkspaceScanner (Task 2) — org state
- Consumes: IssueEngine (Task 6) — issue context

**Key implementation details:**
- Coordinator is a Claude Code session in tmux (`trellis-coordinator`)
- Has access to: recommendation API endpoints, workspace state API,
  issue API. Can call these via curl/fetch from within the session.
- L1 = read-only advisory: analyses recommendations, adds reasoning
  ("I recommend #42 because it unblocks 3 downstream issues"), answers
  questions about the org state
- Coordinator panel in UI: xterm.js showing the coordinator's terminal,
  recommendation cards showing its suggestions, observation feed
- User can type questions to the coordinator via the terminal panel

**TDD approach:**
- Unit test CoordinatorService session lifecycle
- Integration test: coordinator starts, can query APIs
- Manual test: ask coordinator "what should I work on?" and verify it
  reasons about the dependency graph

---

## Post-MVP Tasks (Batches 6-9)

These are filed as issues but not detailed here. Each gets its own
implementation plan when work reaches that point.

### Task 12: Garden Service + Provenance (Batch 6a)
Garden search integration, `.provenance.yaml` convention, lineage queries.
Depends on: Task 2 (scanner), new skill convention for provenance recording.

### Task 13: Artifact Browser (Batch 6b)
Resolve and render specs, ADRs, journals, handovers. Markdown viewer.
Depends on: Task 2 (scanner for path resolution).

### Task 14: Drafthouse Integration (Batch 6c)
Launch design reviews from artifact browser, monitor progress, status display.
Depends on: Task 13 (artifact browser).

### Task 15: LLM Coordinator L2 — Propose Actions (Batch 7)
Coordinator proposes actions with approve/reject UI. Action execution pipeline.
Depends on: Task 11 (L1 coordinator), Task 7 (lifecycle manager).

### Task 16: LLM Coordinator L3 + ISX (Batch 8)
Autonomous action with observation mode. ISX integration for permission-free execution.
Depends on: Task 15 (L2 coordinator).

### Task 17: Velocity Tracking + Projections (Batch 9)
Historical velocity per epic, completion projections based on throughput + critical path.
Depends on: Task 9 (critical path engine).

---

## Dependency Summary

```
Task 1 (skeleton) ──┬──→ Task 2 (scanner)    ─┐
                    │                          ├──→ Task 6 (issues)  ──→ Task 9 (critical path)
                    │                          │                              │
                    │                          └──→ Task 7 (lifecycle)        │
                    │                                    │                    │
                    ├──→ Task 3 (tmux)         ──→ Task 8 (terminal UI)      │
                    │                                    │                    │
                    ├──→ Task 4 (windows)      ──────────┘                    │
                    │                                                         │
                    └──→ Task 5 (bootstrap)                                   │
                                                                              │
Task 6 + Task 7 ──→ Task 10 (one-click start)                                │
                                                                              │
Task 9 ──→ Task 11 (LLM coordinator L1) ═══ MVP ═══                          │
```

## Parallelism Guide (for slot allocation)

| Phase | Slots | Tasks (parallel) |
|-------|-------|-----------------|
| 1 | 1 | Task 1 (skeleton) — must be first |
| 2 | 4 | Task 2, Task 3, Task 4, Task 5 — fully independent |
| 3 | 3 | Task 6, Task 7, Task 8 — partially parallel |
| 4 | 1 | Task 9 (critical path) — needs Task 6 |
| 5 | 1 | Task 10 (integration) — needs Tasks 6+7+8 |
| 6 | 1 | Task 11 (LLM L1) — needs Task 9 |
| Post-MVP | 3 | Tasks 12, 13, 14 — parallel |
| Post-MVP | 1 each | Tasks 15, 16, 17 — sequential |
