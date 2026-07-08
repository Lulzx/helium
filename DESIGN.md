# Helium — a lighter-than-Air agentic development environment

*Working name: **Helium** (lighter than Air). Rust-native, open-source, built on GPUI.*

## 0. Design principles

The whole system obeys five rules; every section below is a consequence of them.

1. **One loop, one queue, one owner.** All orchestration state is owned by a single engine
   thread running a single event loop over a single queue (the QEMU main-loop shape). No locks,
   no shared mutable state, no async runtime. Concurrency exists only at the edges — dumb
   threads that block on a pipe or a socket and push events into the queue.
2. **The engine is sans-IO.** The core is a pure state machine:
   `step(State, Event) -> Vec<Command>`. It never touches a process, socket, file, or clock —
   every event record carries its timestamp, and time reaches the engine *only* through
   events (`Tick` included); randomness doesn't exist. IO lives in a thin shell that feeds
   events in and executes emitted commands. The engine is therefore deterministic, replayable,
   and testable at millions of steps per second with a fake clock.
3. **The event log is the database.** Every event is appended to a JSONL log before it is
   applied. State is a fold over the log; restart = replay. There is no second store to drift
   out of sync. Snapshots are an optimization, added only if replay ever gets slow.
4. **Canonical protocols, shelled-out tools.** ACP comes from the official
   `agent-client-protocol` crate — Zed's own SDK, runtime-agnostic (futures-based, no tokio),
   so it rides our edge threads without dragging in an async ecosystem, and spec drift is
   tracked by the spec's authors. Git and GitHub are driven through the `git` and `gh` CLIs —
   stable, documented, battle-tested interfaces — not through library bindings.
5. **A dependency must pay rent.** GPUI + gpui-component are the point of the project (native
   cross-platform GPU UI) and are accepted. Beyond them the budget is small and enumerated
   (§3.7). Anything a competent programmer can write in a weekend at the size we need — HTTP/1.1
   client, JSON-RPC framing, diff parsing — we write.

What this buys: the hardest subsystem of the naive design (a tokio↔GPUI executor bridge) simply
does not exist; the engine runs headless, under test, or under the GUI without changing a line;
crash recovery is replay; and the binary stays one small artifact.

## 1. Why this should exist

[JetBrains Air](https://air.dev/) (public preview, March 2026) defined a new category: the
**Agentic Development Environment (ADE)** — not an editor, but an orchestration layer that runs
multiple AI coding agents in parallel against your codebase, each in an isolated environment,
with a kanban board, diff review, and permission controls on top.

Air is good, but its most-voiced complaints are structural, not fixable by JetBrains:

| Air's weakness | Our answer |
|---|---|
| Closed source, trust deficit after Fleet's cancellation | MIT, open from day one |
| Kotlin/JVM — "lightweight" only relative to IntelliJ | Rust + GPUI: native GPU-rendered UI, small single binary, instant startup |
| No Windows (macOS + Linux only as of mid-2026) | GPUI ships Metal/Vulkan/DX11 — all three platforms from v1 |
| No local-model support (top requested feature, "no ETA") | Built-in Ollama-backed agent |
| Subscription lock-in (JetBrains AI credits) | BYOK / bring-your-own-agent; we never proxy tokens |
| Coarse permission controls | Fine-grained per-task permission policies |

The open-source field validates the demand but leaves the slot open:
**Vibe Kanban** (web UI, company shut down, community-maintained), **Claude Squad**
(terminal-only), **Crystal/Nimbalyst** (Electron), **Conductor** (polished but closed,
macOS-only). Nobody has built the *native, cross-platform, open* one. That's the gap.

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
  and only surfaces **escalations** when a worker is blocked. `helium --headless` runs the same
  binary without a window; the GUI (or CLI subcommands) attach over a local control socket.

**In scope:**
1. **Kanban board** of agent tasks (To do / In progress / In review / Done), one agent session per card.
2. **Git worktree isolation** per task — each task gets its own worktree + branch; approve-and-merge when done.
3. **Task detail view** — live agent transcript (streamed markdown), tool-call log, and a **diff tab** reviewing the worktree against the base branch.
4. **ACP-native agent integration** — the official [Agent Client Protocol](https://agentclientprotocol.com) SDK behind a thin adapter; get Claude Code (via claude-code-acp), Gemini CLI, and every future ACP agent from a three-line TOML registry entry.
5. **Built-in Ollama agent** — a minimal native agent loop (read/write/edit/glob/grep/bash tools with permission gates) driving local models through the Ollama API, exposed through the same internal interface as ACP agents. Headline differentiator.
6. **Permission modes** per task: Ask / Auto-edit / Full-auto, plus per-request approval dialogs surfaced from ACP `session/request_permission`.
7. **Context attachment** on task creation: files and pasted text (symbols/commits later).
8. **GitHub integration via the `gh` CLI** — no OAuth or token handling of our own:
   - **Ship flow**: push the task branch, `gh pr create`; PR state, review decision, and CI
     checks render live on the kanban card.
   - **Issues as tasks**: import an issue as a task; its title/body become prompt context; the
     PR auto-links `Fixes #N`.
   - **Projects v2 sync**: two-way sync between the board and a GitHub Project (poll + reconcile).
9. **Task lifecycle state machine**: `queued → working → review → rework* → ready →
   pr-opened/merged`, plus `blocked` (escalation) and `failed` (retry queue). Every transition
   is an event — logged, persisted, rendered.
10. **Escalations & steering**: a blocked worker raises an escalation card the user answers to
    unblock; any running worker can be **steered** mid-flight with an injected instruction.
11. **Operational control loop**: per-project concurrency limit, retry with exponential backoff,
    stall detection (no session events for N minutes → nudge, then escalate).
12. **Repo-local workflow file** — `.helium/workflow.toml` declares tracker, candidate filter,
    workspace strategy, runner per state, per-state prompts, concurrency, and hooks (shell
    commands on state transitions, e.g. `cargo test` gate before `ready`).

**Explicitly out of scope for v1** (designed to be addable later):
embedded terminal, Docker/SSH isolation, cloud sandboxes, app preview, built-in git client
beyond diff/merge, team features, Linear tracker (GitHub first; tracker is a trait).

## 3. Architecture

### 3.1 Process shape

One binary. Threads, not runtimes:

```
┌────────────────────────── helium (one process) ─────────────────────────┐
│                                                                          │
│  GPUI main thread            engine thread              edge threads     │
│  ┌──────────────────┐   cmds  ┌──────────────────┐      ┌─────────────┐  │
│  │ Workspace         │ ─────▶ │  loop {           │ ◀──  │ pipe reader │  │
│  │  KanbanBoard      │        │    ev = q.recv()  │      │ (per agent  │  │
│  │  TaskDetail/Diff  │ ◀───── │    log.append(ev) │      │  stdout/err)│  │
│  │  Escalations      │  evs   │    cmds =         │      ├─────────────┤  │
│  └──────────────────┘         │      step(st, ev) │      │ timer       │  │
│         ▲                     │    exec(cmds)     │      ├─────────────┤  │
│         │ Entity updates      │  }                │      │ gh/git/hook │  │
│   (foreground executor)       └──────────────────┘      │ runners     │  │
│                                       ▲                  ├─────────────┤  │
│                                       │                  │ ctl socket  │  │
│                                       └── one mpsc queue │ (NDJSON)    │  │
│                                                          └─────────────┘  │
└──────────────────────────────────────────────────────────────────────────┘
```

- **Engine thread**: owns all `State`, runs the loop. `step()` is the sans-IO core
  (`helium-core`); `exec()` is the IO shell — it spawns processes, writes to stdin pipes,
  starts edge threads, appends to logs.
- **Edge threads** are disposable and dumb: block on one fd or one command, translate bytes
  into typed `Event`s, push to the queue, die. A stuck agent process can never wedge the
  engine — the reader thread blocks, the engine keeps looping (and its `Tick` handler is what
  notices the stall).
- **GPUI thread**: subscribes to the engine's event broadcast (a channel drained via
  `cx.spawn` on GPUI's own executor — no foreign runtime, no bridge), applies events to
  `Entity<_>` view models, sends commands back. The UI is a *client* of the engine, in-process.
- **Headless / attach**: `helium --headless` is the same process minus the GPUI thread. The
  control socket (§3.5) speaks the identical command/event types as NDJSON over a Unix socket
  (named pipe on Windows) — QMP-style. `helium status|steer|escalations` and a remote cockpit
  are just socket clients. No HTTP server, no SSE, no web framework.

### 3.2 The sans-IO core (`helium-core`)

```rust
// The entire engine contract. Time comes from the event, never from a clock —
// an Instant/SystemTime parameter would be a replay-breaking side channel.
pub struct Stamped { pub at_ms: u64, pub event: Event }   // what gets logged & applied
pub fn step(state: &mut State, ev: &Stamped) -> Vec<Command>;

pub enum Event {
    // user/UI intents (also arrive from the control socket)
    CreateTask { spec: TaskSpec },
    Approve { task: TaskId }, Answer { task: TaskId, text: String },
    Steer { task: TaskId, instruction: String }, Cancel { task: TaskId },
    // edges reporting back
    AgentOutput { task: TaskId, chunk: SessionUpdate },   // parsed ACP/Ollama updates
    ProcExited { task: TaskId, code: i32 },
    GitDone { task: TaskId, op: GitOp, result: Result<String, String> },
    GhDone { op: GhOp, result: Result<Json, String> },
    HookDone { task: TaskId, hook: String, code: i32, stderr: String },
    Tick,                                                  // 1 Hz; with Stamped.at_ms this is the
}                                                          //   engine's only source of time

pub enum Command {
    SpawnAgent { task: TaskId, runner: RunnerCfg, worktree: PathBuf },
    SendToAgent { task: TaskId, msg: Json },               // prompt / permission reply / cancel
    KillAgent { task: TaskId },
    Git { task: TaskId, op: GitOp },                       // worktree add/remove, diff, merge, push
    Gh { op: GhOp },                                       // pr create/view, issue list, project ops
    RunHook { task: TaskId, hook: String, cmd: String },
    Emit { event: UiEvent },                               // broadcast to UI + control socket
}
```

Consequences, all load-bearing:
- **Every feature in §2.9–2.11 is pure code in `step()`**: the lifecycle machine, retry
  backoff, stall detection (compare `last_activity_ms` against `Tick`'s `at_ms`), rework
  cycles, concurrency caps, tracker-poll scheduling. All of it unit-tests with synthetic
  events and fabricated timestamps — no processes, no git repos, no network, no clock.
- **Replay is trivial**: `state = events.fold(State::default(), step_ignoring_commands)`.
  Startup, crash recovery, and "reopen a 3-week-old task" are the same code path.
- **The control protocol falls out for free**: `Event` in, `UiEvent` out, serde on both.

### 3.3 The event log is the database

- Per project: `.helium/log/<task-id>.jsonl` (session events) + `.helium/log/engine.jsonl`
  (task-level transitions, dispatch decisions). Append-only, fsync'd on state transitions,
  line-buffered otherwise.
- Startup: replay `engine.jsonl` (small — transitions only) to rebuild the board; a task's
  transcript replays lazily when its detail view opens.
- No sqlite, no schema migrations, no cache invalidation. `grep`/`jq` are the debug tools.
  If a log is corrupted (torn final line), we truncate to the last valid line and continue.
- Config: `~/.config/helium/config.toml` (agent registry, theme) + `.helium/workflow.toml`
  (per-repo policy). Both are data; adding an ACP agent is three lines of TOML.

### 3.4 Protocols we own

**ACP client**: the official [`agent-client-protocol`](https://docs.rs/agent-client-protocol)
crate (Apache-2.0, maintained by Zed — the spec's authors). It is **runtime-agnostic**:
futures-based with no tokio dependency, which is why it fits this architecture — each ACP
session gets one edge thread running a local futures executor (`futures::executor`) that owns
the child process (agent command from the registry, e.g. `npx @zed-industries/claude-code-acp`,
`gemini --experimental-acp`) and the connection: `initialize` → `session/new` →
`session/prompt`. The crate's `session/update` callbacks become `SessionUpdate` events pushed
to the engine queue; `session/request_permission` parks on a oneshot answered by a
`SendToAgent` command. We keep a thin adapter (~150 LOC) so `helium-core` sees only our own
event types — the crate never leaks past the edge.

**Ollama agent** (~700 LOC): the same `SessionUpdate` stream produced by a native loop:
`POST /api/chat` (streaming NDJSON via `ureq` — a real HTTP client, because real Ollama
deployments sit behind HTTPS and reverse proxies; one edge thread per session) with a fixed
tool set: `read_file`, `write_file`,
`edit_file` (find/replace), `list_dir`, `grep`, `bash` (cwd locked to the worktree). Tool
executions route through the same permission gate as ACP tool calls. Default models:
qwen3-coder / devstral-class. The engine cannot tell an Ollama session from an ACP session.

**Control socket**: NDJSON over `$XDG_RUNTIME_DIR/helium/<project-hash>.sock`
(Windows: named pipe). Line in = `Event` (auth: filesystem permissions, same as QMP), line
out = `UiEvent`. ~150 LOC including the client side used by the CLI subcommands.
**Remote attach** (conductor on a server, cockpit at home) is SSH socket forwarding —
`ssh -L /tmp/helium.sock:$XDG_RUNTIME_DIR/helium/<hash>.sock server` — which supplies
authentication and encryption for free; we ship the one-liner in docs, not a network server
in the binary.

### 3.5 Tools we shell out to

**git** — worktrees (`git worktree add .helium/worktrees/<slug> -b helium/<slug> <base>`),
diffs (`git diff <base>...` parsed into hunks — the unified format is a stable interface),
status (`--porcelain=v2`), merge, push, cleanup (`git worktree remove` + branch delete;
conflicts surface as an escalation and leave the worktree for manual fixing). No git library:
the CLI is the one interface every git feature is guaranteed to support, and it's what the
proven orchestrators (Vibe Kanban, Claude Squad) ship on.

**gh** — all GitHub access rides the user's authenticated `gh`; Helium never stores tokens.
Probe `gh auth status` at startup; without it, GitHub features grey out and everything local
still works. Every call uses `--json`/`--format json` into typed structs — no screen-scraping.
Ship flow: `git push -u origin helium/<slug>` → `gh pr create --fill` (body drafted from task
prompt + agent summary, `Fixes #N` when issue-born) → card tracks
`gh pr view --json state,statusCheckRollup,reviewDecision` on a poll driven by `Tick`.
Issues-as-tasks: `gh issue list/view --json`. Projects v2 sync: column ↔ Status-field mapping
from `workflow.toml`; local moves call `gh project item-edit`, a reconcile poll applies remote
moves; conflict rule: **remote wins** — the board is a view over GitHub, not a second source of
truth. Missing `project` scope is detected and answered with the exact
`gh auth refresh -s project` command. One serialized queue per repo with exponential backoff
on rate-limit errors — scheduled by `step()`, executed by a runner thread.

### 3.6 UI (GPUI + gpui-component)

The UI holds **no business state** — view models are folds over `UiEvent`s, commands go back
to the engine. gpui-component (Apache-2.0, production-proven in Longbridge Pro) supplies:

| Screen | Widgets |
|---|---|
| Workspace shell | `Dock` layout, `Sidebar` (projects), titlebar, theme switcher |
| Kanban board | columns via `VirtualList`; card = status badge, agent icon, branch, turn count, PR state + CI pill |
| New task form | `Form`, `Input`, agent `Select`, base-branch `Select`, permission-mode radio, file-context picker |
| Task detail — transcript | Markdown `TextView` stream, tool-call accordion rows, prompt `Input` |
| Task detail — diff | file `Tree` + read-only `Editor` (tree-sitter) rendering parsed hunks |
| Permission prompts | `Dialog` + `Notification` toasts |
| Escalation queue | `Sidebar` section + badge; card = question, context, answer `Input` |
| Operations tab | retry/blocked queues as `DataTable` (Symphony's console, native) |
| Settings | agent registry, Ollama endpoint/model, workflow editor, defaults |

### 3.7 Dependency budget

Accepted: `gpui`, `gpui-component` (the project's raison d'être),
`agent-client-protocol` (official, runtime-agnostic — canonical types beat hand-rolled
framing), `futures` (local executors on edge threads), `serde`/`serde_json`, `toml`, `ureq`
(Ollama HTTP — TLS and proxies are table stakes, we don't hand-roll them), `notify`
(worktree file-watch for diff refresh), `dirs`. Explicitly rejected: tokio (threads suffice),
sqlite (log is the store), axum/hyper (no HTTP server), git2/gitoxide (CLI), octocrab (gh
CLI). Target: **< 20 direct dependencies, one binary, cold start < 100 ms, idle RSS dominated
by GPUI itself.**

### 3.8 Workflow file (`.helium/workflow.toml`)

```toml
[tracker]
kind = "github"
project = "acme/webapp#3"
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

## 4. Crate layout

Two crates. Split further only when compile times or ownership demand it.

```
helium/
├── Cargo.toml                 # workspace
├── crates/
│   ├── helium-core/           # sans-IO: State, Event, Command, step(); deps: serde only.
│   │                          #   lifecycle, control loop, dispatch policy, permission policy —
│   │                          #   the whole brain, fully deterministic, fuzzable.
│   └── helium/                # bin: everything with side effects.
│       ├── engine/            #   event loop thread, command executor, log append/replay
│       ├── edges/             #   pipe readers, timer, git/gh/hook runners, control socket
│       ├── acp.rs             #   adapter over the official agent-client-protocol crate
│       ├── ollama.rs          #   native agent loop + HTTP client
│       └── ui/                #   GPUI app: entities, views, actions
└── assets/
```

## 5. Milestones

Each independently verifiable; `helium-core` grows tests from milestone 1.

1. **Core + replay** — `helium-core` with lifecycle machine, `step()`, event log
   append/replay; property tests (random event sequences never violate state invariants) and
   fake-clock tests for backoff/stall. No UI, no processes. *(medium — but it's the brain)*
2. **Skeleton** — GPUI window, dock, empty kanban fed by a scripted engine replaying a canned
   log. Proves UI-as-fold-over-events. *(small)*
3. **ACP client** — drive claude-code-acp end-to-end: create task → prompt → streamed
   transcript → tool-call rendering → permission dialog → cancel → steer. *(large — the heart)*
4. **Worktrees + diff** — worktree/branch per task, diff tab, merge & cleanup. *(medium)*
5. **GitHub via `gh`** — auth probe, push + PR ship flow, checks on cards, issue→task import. *(medium)*
6. **Conductor mode** — `workflow.toml`, tracker poll + dispatch, Projects sync, hooks,
   `--headless` + control socket + CLI subcommands. Mostly `step()` logic already tested in
   milestone 1; this wires the edges. *(medium)*
7. **Ollama agent** — tool loop, permission gates, model picker, reviewer role. *(large)*
8. **Polish & release** — Gemini preset, settings UI, packaging (cargo-bundle / MSI /
   AppImage), docs, CI on all three platforms. *(medium)*

Milestone 1 is deliberately first: in this architecture the orchestrator is *finished and
tested before it ever touches IO*. Symphony-grade operational behavior comes from tests, not
from production incidents.

## 6. Risks

- **gpui/gpui-component pre-1.0 churn** — pin exact versions; UI is a thin fold over
  `UiEvent`, so rewrites are contained by construction.
- **ACP spec drift** — carried by the official crate (maintained by the spec's authors);
  our exposure is the ~150-line adapter. Version-check at `initialize`, integration-test
  against claude-code-acp in CI.
- **Windows quirks** (named pipes, process spawning, worktree paths, DX11 backend) — CI on all
  three OSes from milestone 2.
- **Ollama agent quality** — local models are weaker at agentic loops; small toolset, public
  and tunable system prompt, honest docs (it shines as reviewer / small-task worker).
- **`gh`/`git` output drift** — pin minimum versions, parse only `--json`/porcelain
  interfaces, integration-test the parsers in CI, degrade gracefully when absent.
- **No accessibility in GPUI** — known ecosystem gap; document, track upstream.
- **Event log growth** — transcripts are append-only JSONL per task; rotate on task
  completion (archive dir), snapshot `engine.jsonl` if replay ever exceeds ~50 ms.

## 7. What makes the loved tools loved (design north stars)

1. **Instant, glanceable status** — you look at the board and know which agent needs you.
   Optimize for the *review* moment, not the launch moment.
2. **Worktree isolation that never bites** — cleanup must be bulletproof; orphaned worktrees
   are the #1 trust killer in this category.
3. **Frictionless task launch** — one keystroke from "idea" to "agent running".
4. **Real diffs, not walls of text** — reviewing the change matters more than the transcript.
5. **Respecting the user's stack** — BYO agent, BYO keys, BYO editor ("Open in $EDITOR" on
   every worktree).
6. **Being native and fast** — Conductor wins praise purely for feeling like a real Mac app;
   GPUI gives us that on all three platforms.
7. **Unattended must mean unattended** (Symphony's lesson) — retries, stall detection, and
   escalations make it safe to walk away; a tool you babysit is just a slower terminal.
8. **Comprehensible when it breaks** (the Bellard clause) — one loop to read, one log to
   `jq`, one deterministic replay to reproduce any bug. Simplicity is an operational feature.
