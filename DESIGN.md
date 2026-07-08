# Helium — a lighter-than-Air agentic development environment

*Working name: **Helium** (lighter than Air). Rust-native, open-source, built on GPUI.*

## 1. Why this should exist

[JetBrains Air](https://air.dev/) (public preview, March 2026) defined a new category: the
**Agentic Development Environment (ADE)** — not an editor, but an orchestration layer that runs
multiple AI coding agents in parallel against your codebase, each in an isolated environment,
with a kanban board, diff review, and permission controls on top.

Air is good, but its most-voiced complaints are structural, not fixable by JetBrains:

| Air's weakness | Our answer |
|---|---|
| Closed source, trust deficit after Fleet's cancellation | Apache-2.0 / MIT, open from day one |
| Kotlin/JVM — "lightweight" only relative to IntelliJ | Rust + GPUI: native GPU-rendered UI, ~15 MB binary, instant startup |
| No Windows (macOS + Linux only as of mid-2026) | GPUI ships Metal/Vulkan/DX11 — all three platforms from v1 |
| No local-model support (top requested feature, "no ETA") | Built-in Ollama-backed agent |
| Subscription lock-in (JetBrains AI credits) | BYOK / bring-your-own-agent; we never proxy tokens |
| Coarse permission controls | Fine-grained per-task permission policies |

The open-source field validates the demand but leaves the slot open:
**Vibe Kanban** (web UI, company shut down, community-maintained), **Claude Squad** (terminal-only),
**Crystal/Nimbalyst** (Electron), **Conductor** (polished but closed, macOS-only).
Nobody has built the *native, cross-platform, open* one. That's the gap.

**[Kata Symphony](https://kata.sh/symphony)** (open-source, Rust) is the strongest design
influence: a *headless* issue-to-PR orchestrator that polls GitHub/Linear, dispatches parallel
workers in isolated workspaces, and drives tickets through implementation → review → rework →
merge unattended, with a real operational control loop (concurrency limits, retries with
exponential backoff, stall detection, escalations, hooks, structured logs) and a console you
*attach* to. Symphony proves the category's endgame is **unattended backlog execution**, not
hand-launched tasks — but it has no GUI. Helium = Symphony's headless-first orchestration
discipline + a first-class native GPUI cockpit on top.

## 2. Product scope (v1)

A **lean orchestrator**. The user's editor stays their editor; Helium manages the agents.

Two operating modes over one engine (Symphony-inspired):
- **Cockpit mode** (interactive): you create tasks, watch the board, review diffs, approve merges.
- **Conductor mode** (unattended): Helium polls the tracker (GitHub issues/Projects), auto-claims
  candidate issues, dispatches workers, drives each through implement → review → rework → PR,
  and only surfaces **escalations** when a worker is blocked. The GUI is a client you attach to;
  the same engine runs headless on a server (`helium serve`).

**In scope:**
1. **Kanban board** of agent tasks (To do / In progress / In review / Done), one agent session per card.
2. **Git worktree isolation** per task — each task gets its own worktree + branch; approve-and-merge when done.
3. **Task detail view** — live agent transcript (streamed markdown), tool-call log, and a **diff tab** reviewing the worktree against the base branch.
4. **ACP-native agent integration** — implement the [Agent Client Protocol](https://agentclientprotocol.com) client once, get Claude Code, Gemini CLI, and every future ACP agent free. Agents are external processes speaking JSON-RPC over stdio; we spawn and manage them.
5. **Built-in Ollama agent** — a minimal native agent loop (read/write/edit/glob/grep/bash tools with permission gates) driving local models through the Ollama API, exposed through the same internal interface as ACP agents. This is the headline differentiator.
6. **Permission modes** per task: Ask / Auto-edit / Full-auto, plus per-request approval dialogs surfaced from ACP `session/request_permission`.
7. **Context attachment** on task creation: files and pasted text (symbols/commits later).
8. **GitHub integration via the `gh` CLI** — no OAuth or token handling of our own; we shell out
   to the user's authenticated `gh`:
   - **Push / pull / PR flow**: when a task is approved, push its branch and open a PR
     (`gh pr create`) instead of (or in addition to) local merge; PR status, review state, and
     CI checks (`gh pr checks`) render live on the kanban card.
   - **Issues as tasks**: import a GitHub issue as a task (`gh issue list/view`) — its title/body
     become the agent prompt context; the PR auto-links `Fixes #N`.
   - **GitHub Projects (v2) sync**: two-way sync between the kanban board and a GitHub Project
     via `gh project item-list / item-add / item-edit` — moving a card in Helium updates the
     Project field, and remote changes are picked up on poll.

9. **Task lifecycle state machine** (Symphony-style): `queued → working → review → rework →
   ready → merged/pr-opened`, plus `blocked` (escalation) and `failed` (retry queue). Every
   state transition is an event — logged, persisted, and rendered.
10. **Escalations & steering**: a blocked worker raises an escalation card (question + context)
    that the user answers to unblock; any running worker can be **steered** mid-flight with an
    injected instruction (maps to a follow-up ACP prompt / conversation turn).
11. **Operational control loop**: per-project concurrency limit, retry with exponential backoff,
    stall detection (no session events for N minutes → nudge, then escalate), structured JSONL
    event log per task.
12. **Repo-local workflow file** — `.helium/workflow.toml` declares tracker, candidate filter
    (labels/status), workspace strategy, runner (agent) per state, per-state prompts, concurrency,
    and hooks (shell commands on state transitions, e.g. `cargo test` gate before `ready`).

**Explicitly out of scope for v1** (designed to be addable later):
embedded terminal (alacritty_terminal, v1.x), Docker/SSH isolation, cloud sandboxes, app preview,
built-in git client beyond diff/merge, team features, Linear tracker support (GitHub first;
tracker is a trait).

## 3. Architecture

### 3.0 Headless-first (the Symphony rule)

The orchestration engine is a library with **zero UI dependencies**. Everything — task state
machine, agent sessions, git, gh, tracker polling, control loop — lives in the engine and
communicates exclusively through a typed command/event API. Two hosts consume it:

- **Embedded**: the GPUI app runs the engine in-process (default desktop experience).
- **Server**: `helium serve --port …` runs the same engine headless; the GPUI app (and a minimal
  CLI: `helium status|steer|escalations`) attaches over HTTP + SSE using the same command/event
  types. Unattended conductor mode on a box, cockpit at home.

This costs one serialization boundary and buys: testability without a GUI, a CLI for free,
remote operation, and immunity of core logic to gpui API churn.

### 3.1 The concurrency spine

GPUI runs its own smol-based executors; tokio futures panic inside `cx.spawn`. The ACP crate,
reqwest (Ollama), and process management all want tokio. So:

- One **dedicated tokio runtime** on background threads owns all I/O: agent child processes, ACP
  JSON-RPC sessions, HTTP to Ollama, git operations.
- The GPUI side talks to it via channels: commands in (`tokio::mpsc::UnboundedSender<Command>`),
  events out. Events are pumped into GPUI with `cx.spawn` + `smol` channel adapters (or
  `cx.background_executor` polling an async-channel receiver) and dispatched to entities via
  `Entity::update`, which triggers re-render.
- Rule: **no blocking or tokio-dependent call ever runs on the GPUI thread.**

```
┌─────────────── GPUI main thread ───────────────┐
│  Workspace ─ KanbanBoard ─ TaskDetail ─ Diff   │
│        Entity<TaskStore>, Entity<Task>          │
└───────────────▲───────────────┬────────────────┘
        events  │               │ commands
┌───────────────┴───────────────▼────────────────┐
│         helium-engine (tokio runtime)           │
│  ControlLoop: dispatch, concurrency, retries,   │
│    stall detection, escalations, hooks          │
│  TaskStateMachine (queued→working→review→…)     │
│  AgentManager ── AcpSession (per task)          │
│              └── OllamaAgent (per task)         │
│  Tracker (gh: issues/Projects poll+reconcile)   │
│  GitService (worktrees, diffs, merge)           │
│  Store (sqlite + JSONL event log)               │
└──────────────────────▲──────────────────────────┘
                       │ same command/event API over HTTP+SSE
              helium serve ⇄ remote cockpit / helium-cli
```

### 3.2 Workspace layout

```
helium/
├── Cargo.toml                # workspace
├── crates/
│   ├── helium/               # bin: GPUI app (cockpit) — embeds engine or attaches to a server
│   ├── helium-cli/           # bin: `helium serve|status|steer|escalations` (headless + remote control)
│   ├── helium-core/          # Task, lifecycle states, Transcript, events, workflow/config types (no UI/IO deps)
│   ├── helium-engine/        # tokio runtime, task state machine, control loop (concurrency,
│   │                         #   retries/backoff, stall detection, escalations, hooks), AgentManager,
│   │                         #   tracker polling/dispatch; exposes command/event API + HTTP/SSE server
│   ├── helium-acp/           # ACP client: spawn agent process, JSON-RPC session, permission callbacks
│   ├── helium-ollama/        # built-in agent: tool loop, tool impls, Ollama HTTP client
│   ├── helium-git/           # worktree lifecycle, branch naming, diff, merge (gitoxide + git CLI fallback)
│   ├── helium-gh/            # GitHub via `gh` CLI: PRs, checks, issues, Projects v2 sync (tracker impl)
│   └── helium-store/         # sqlite (rusqlite) persistence: tasks, transcripts, event log, settings
└── assets/                   # icons, themes
```

### 3.3 Agent abstraction

One trait, two implementations:

```rust
trait AgentSession: Send {
    async fn prompt(&mut self, blocks: Vec<ContentBlock>) -> Result<()>;
    async fn cancel(&mut self) -> Result<()>;
    fn events(&mut self) -> impl Stream<Item = SessionEvent>; // text deltas, tool calls,
                                                              // permission requests, plan updates, done
}
```

- **`AcpAgentSession`** (helium-acp): spawns the configured agent command (e.g.
  `npx @zed-industries/claude-code-acp`, `gemini --experimental-acp`), runs
  `initialize → session/new → session/prompt` over stdio JSON-RPC via the official
  `agent-client-protocol` Rust crate. `session/update` notifications become `SessionEvent`s;
  `session/request_permission` becomes a `SessionEvent::PermissionRequest` that blocks on a
  oneshot answered from the UI (or auto-answered by the task's permission mode).
- **`OllamaAgentSession`** (helium-ollama): a classic tool-use loop against
  `POST /api/chat` with `tools`: send conversation → model returns tool calls → execute
  (through the same permission gate) → append results → repeat until a plain message.
  v1 toolset: `read_file`, `write_file`, `edit_file` (find/replace), `list_dir`, `grep`,
  `bash` (cwd locked to the worktree). Recommended default models: qwen3-coder / devstral-class.

The UI never knows which kind it's driving.

### 3.4 Git layer

- Worktree per task: `git worktree add .helium/worktrees/<task-slug> -b helium/<task-slug> <base>`.
  Prefer shelling out to the `git` CLI for worktree/merge (worktree support in libraries is the
  weakest corner; the CLI is what Vibe Kanban and Claude Squad ship on), **gitoxide** for fast
  read-side ops (status, diff enumeration).
- Diff tab renders `base...worktree` unified diffs (recomputed on file-change events, debounced).
- Merge flow: squash-merge or plain merge back to base branch, then
  `git worktree remove` + branch cleanup. Conflicts → surface and leave the worktree for manual fixing.

### 3.4b GitHub layer (helium-gh)

All GitHub access shells out to the user's **`gh` CLI** — Helium never stores or requests
tokens. On startup we probe `gh auth status`; if absent, GitHub features grey out with a
"run `gh auth login`" hint (everything local keeps working — GitHub is an enhancement, not
a requirement).

- **Structured output everywhere**: every call uses `--json`/`--format json`
  (`gh pr view --json state,statusCheckRollup,reviewDecision`, `gh project item-list --format json`)
  parsed with serde into typed structs in `helium-core`. No screen-scraping.
- **Ship flow**: approve task → `git push -u origin helium/<slug>` → `gh pr create --fill --head …`
  (title/body drafted from the task prompt + agent summary, `Fixes #N` when the task came from
  an issue). Card then tracks the PR: state, review decision, and check rollup polled via
  `gh pr view` on a backend interval (~60s, plus manual refresh).
- **Issues as tasks**: "New task from issue" lists open issues (`gh issue list --json`), and the
  selected issue's title/body/labels are attached as prompt context.
- **Projects v2 sync** (opt-in per project, config maps board columns ↔ the Project's Status
  field options): local card moves call `gh project item-edit`; a poll of `gh project item-list`
  reconciles remote moves. Conflict rule: **last-writer-wins, remote preferred** — the board is a
  view over GitHub, not a second source of truth. Requires the `project` scope
  (`gh auth refresh -s project`); we detect the missing-scope error and show the exact command.
- All `gh` invocations run on the tokio runtime through a single rate-limit-aware queue
  (serialized per repo, exponential backoff on 4xx/secondary-rate-limit errors), emitting the
  same backend events the UI already consumes.

### 3.4c Control loop & lifecycle (helium-engine)

The heart of conductor mode, and cockpit mode uses the same machinery with manual dispatch:

- **Lifecycle**: `queued → working → review → rework* → ready → pr-opened/merged`, with
  `blocked` and `failed` as side states. `review` runs a configurable gate: hooks (e.g. build/test
  commands) and optionally a second agent session prompted as reviewer; findings send the task to
  `rework` with the review output as the next prompt. Max rework cycles before escalating.
- **Dispatch**: tracker poll (`gh issue list` / `gh project item-list`) → candidate filter from
  `workflow.toml` (labels, status column, assignee) → claim (move Project status, comment on
  issue) → create worktree → start agent session. Respects `max_concurrent_workers`.
- **Retries**: failed sessions (process death, API errors) re-queue with exponential backoff and
  a retry cap; the retry queue is visible in the UI.
- **Stall detection**: no session events for `stall_after` (default 10 min) → inject a nudge
  prompt; still silent → mark `blocked` with an auto-generated escalation.
- **Escalations**: a first-class queue. Each carries the agent's question (or the stall/failure
  context) and accepts a text answer, which resumes the session as the next prompt turn.
- **Steering**: `steer(task, instruction)` cancels the in-flight turn if needed and injects the
  instruction as a user turn — works identically for ACP and Ollama sessions.
- **Hooks**: shell commands on state transitions (`on_working_complete = "cargo test"`,
  `on_ready = "./notify.sh"`), run in the task's worktree; non-zero exit routes to `rework`
  with stderr as context.
- **Structured logs**: every event appended to `.helium/logs/<task>.jsonl` — replayable,
  greppable, and the source for the transcript view after restarts.

### 3.4d Workflow file (`.helium/workflow.toml`)

```toml
[tracker]
kind = "github"                # trait-based; linear later
project = "acme/webapp#3"      # Projects v2 board
candidate_filter = { labels = ["agent-ok"], status = "Todo" }

[workspace]
strategy = "worktree"          # local | worktree (docker/ssh later)
base = "main"

[limits]
max_concurrent_workers = 3
stall_after = "10m"
max_retries = 2
max_rework_cycles = 3

[runner]
agent = "claude-code"          # key into the agent registry
permission_mode = "auto-edit"

[runner.per_state.review]
agent = "ollama:qwen3-coder"   # cheap local reviewer
prompt = "Review the diff for correctness and scope creep…"

[hooks]
on_working_complete = "cargo test --workspace"

[ship]
mode = "pr"                    # pr | merge
draft = true
```

### 3.5 UI (gpui-component mapping)

| Screen | Widgets |
|---|---|
| Workspace shell | `Dock` layout, `Sidebar` (projects), titlebar, theme switcher (20+ built-in themes) |
| Kanban board | columns of cards via `VirtualList`; card = status badge, agent icon, branch, cost/turn count, PR state + CI check pill |
| New task form | `Form`, `Input`, agent `Select`, base-branch `Select`, permission-mode radio, file-context picker |
| Task detail — transcript | Markdown `TextView` stream, tool-call accordion rows, sticky prompt `Input` at bottom |
| Task detail — diff | file tree (`Tree`) + per-file diff using the `Editor` component (tree-sitter highlighting) in read-only two-tone mode |
| Permission prompts | `Dialog` (blocking request) + `Notification` toasts for background events |
| Escalation queue | dedicated `Sidebar` section + badge count; escalation card = question, context excerpt, answer `Input` |
| Steering | command palette action / per-card button → `Dialog` with instruction input |
| Retry/blocked queues | `DataTable` views under an "Operations" tab (mirrors Symphony's console) |
| Settings | agent registry editor (name, command, args, env), Ollama endpoint/model, workflow.toml editor, defaults |

### 3.6 Persistence & config

- **sqlite** (`rusqlite`, bundled) at the project level (`.helium/state.db`): tasks, status,
  transcript events (append-only), token/cost tallies. Survives restarts; sessions are
  resumable where the agent supports `session/load`.
- **Config**: global `~/.config/helium/config.toml` (agent registry, theme, defaults) +
  optional per-repo `.helium/config.toml`. Agent registry is data, not code — adding an ACP
  agent is three lines of TOML.

## 4. Milestones

Each independently runnable/verifiable:

1. **Skeleton** — workspace compiles; GPUI window with dock layout, theme, empty kanban. *(small)*
2. **Engine spine** — headless `helium-engine` with tokio runtime, command/event API, task state machine; fake "echo agent" streams text into a task card's transcript. Engine has integration tests with no GUI. *(medium)*
3. **ACP client** — drive claude-code-acp end-to-end: create task → prompt → streamed transcript → tool-call rendering → permission dialog → cancel → steer. *(large — the heart)*
4. **Worktrees + diff** — task creation makes a worktree/branch; diff tab; merge & cleanup flow. *(medium)*
5. **Lifecycle + persistence** — full state machine (review/rework/blocked/failed), sqlite store + JSONL event logs, restart-safe. *(medium)*
6. **GitHub via `gh`** — auth probe, push + `gh pr create` ship flow, PR/checks status on cards, issue→task import. *(medium)*
7. **Conductor mode** — `workflow.toml`, tracker polling + candidate dispatch, Projects v2 two-way sync, control loop (concurrency, retries, stall detection, escalations, hooks), `helium serve` + HTTP/SSE attach + minimal CLI. *(large)*
8. **Ollama agent** — tool loop, permission gates, model picker; parity in the UI incl. per-state reviewer role. *(large)*
9. **Polish & release** — Gemini CLI config preset, settings UI, packaging (cargo-bundle / MSI / AppImage), docs, CI for the three platforms. *(medium)*

Milestone 3 is the risk concentrator — do it early, everything else is conventional.

## 5. Risks

- **gpui/gpui-component pre-1.0 churn** — pin exact versions, upgrade deliberately; keep UI code thin over `helium-core` types so rewrites are contained.
- **ACP spec drift** — track the official crate version; integration-test against claude-code-acp in CI.
- **Windows quirks** (GPUI newest backend; git worktree paths, process spawning) — CI on all three OSes from milestone 1, not at the end.
- **Ollama agent quality** — local models are weaker at agentic loops; scope the toolset small, make the system prompt public and tunable, set expectations in docs (it's for smaller tasks / privacy-sensitive work).
- **No accessibility in GPUI** — known ecosystem gap; document it, track upstream.
- **`gh` CLI as a dependency** — output schemas can shift between gh versions and Projects v2
  commands need an extra auth scope; pin a minimum gh version, integration-test the JSON
  parsing in CI, and degrade gracefully when gh is missing or under-scoped.

## 6. What makes the loved tools loved (design north stars)

From user sentiment around Air, Conductor, Vibe Kanban, Claude Squad:

1. **Instant, glanceable status** — you look at the board and know which agent needs you. Optimize for the *review* moment, not the launch moment.
2. **Worktree isolation that never bites** — cleanup must be bulletproof; orphaned worktrees are the #1 trust killer in this category.
3. **Frictionless task launch** — one keystroke from "idea" to "agent running" (global new-task shortcut, sensible defaults).
4. **Real diffs, not walls of text** — reviewing the change matters more than reading the transcript.
5. **Respecting the user's stack** — BYO agent, BYO keys, BYO editor ("Open in $EDITOR" on every worktree).
6. **Being native and fast** — Conductor wins praise purely for feeling like a real Mac app; GPUI gives us that on all three platforms.
7. **Unattended must mean unattended** (Symphony's lesson) — retries, stall detection, and escalations make it safe to walk away; a tool you have to babysit is just a slower terminal.
