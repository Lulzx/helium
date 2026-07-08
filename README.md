<div align="center">

# ⚛️ Helium

**Lighter than Air.**

An open-source, Rust-native agentic development environment —
run a fleet of AI coding agents in parallel, each in its own git worktree,
from a GPU-accelerated native cockpit. Or walk away and let it run your backlog.

*Built with [GPUI](https://www.gpui.rs/), the UI framework behind [Zed](https://zed.dev).*

`status: designing` · [Design document](DESIGN.md)

</div>

---

## Why

[JetBrains Air](https://air.dev/) proved the category: an orchestration layer — not an editor —
where AI coding agents work tickets in parallel, in isolation, under review. But Air is closed
source, JVM-based, has no Windows build, no local models, and ties you to a subscription.

Helium is the version of that idea we actually want to use:

|  | JetBrains Air | **Helium** |
|---|---|---|
| Source | Closed | **Open** |
| Runtime | Kotlin/JVM | **Rust + GPUI, native on Metal/Vulkan/DX11** |
| Platforms | macOS, Linux | **macOS, Linux, Windows** |
| Local models | No ("no ETA") | **Built-in Ollama agent** |
| Agents | Fixed four | **Any [ACP](https://agentclientprotocol.com) agent — Claude Code, Gemini CLI, …** |
| Unattended mode | No | **Conductor mode: issue → PR, while you sleep** |
| Pricing | Subscription credits | **Free. Your keys, your agents, your machine.** |

## What it does

**🃏 Cockpit mode** — a kanban board of agent tasks. Describe a task, pick an agent, and Helium
spins up an isolated git worktree and sets the agent loose. Watch live transcripts, answer
permission prompts, steer workers mid-flight, review real diffs, and ship — merge locally or
open a PR with one action.

**🎼 Conductor mode** — point Helium at your GitHub Project and walk away. It polls for
candidate issues, claims them, dispatches workers under a concurrency cap, drives each through
**implement → review → rework → PR**, retries failures with backoff, detects stalls, and
raises an *escalation* when a worker genuinely needs a human. Runs headless on a server
(`helium serve`); the desktop app attaches to it like a cockpit to an engine.

### Under the hood

- **ACP-native** — one [Agent Client Protocol](https://agentclientprotocol.com) client, every
  compatible agent. Adding an agent is three lines of TOML, not a plugin.
- **Local models first-class** — a built-in agent loop (read/edit/grep/bash, permission-gated)
  drives models through [Ollama](https://ollama.com). Use a local model as a cheap reviewer for
  a frontier implementer, or go fully offline.
- **Worktree isolation** — every task gets its own branch and working copy. Agents never fight
  over files; cleanup is bulletproof.
- **GitHub through `gh`** — PRs, checks, issues, and Projects v2 sync ride your existing
  `gh auth login`. Helium never sees a token.
- **Headless-first engine** — the orchestrator is a UI-free Rust library with a typed
  command/event API. The GPUI app, the CLI, and the server are all just clients.
- **Repo-local workflow file** — `.helium/workflow.toml` declares your tracker, filters,
  runners, per-state prompts, hooks (`on_working_complete = "cargo test"`), and limits.
  Your orchestration policy is code-reviewed like everything else.

## Design

The full architecture — crate layout, the GPUI/tokio concurrency spine, the ACP session
lifecycle, the task state machine, and the milestone plan — lives in
**[DESIGN.md](DESIGN.md)**. The short version:

```
GPUI cockpit / CLI  ⇄  command/event API  ⇄  helium-engine (tokio)
                                              ├─ control loop: dispatch, retries,
                                              │    stall detection, escalations, hooks
                                              ├─ agent sessions: ACP · Ollama
                                              ├─ git worktrees · gh (PRs, Projects)
                                              └─ sqlite + JSONL event log
```

## Status

Helium is in the **design phase** — the blueprint is public before the code, on purpose.
Read [DESIGN.md](DESIGN.md), open an issue, tell us where we're wrong.

Roadmap (abridged): GPUI skeleton → headless engine → ACP client → worktrees & diff review →
lifecycle & persistence → `gh` ship flow → conductor mode → Ollama agent → v1.

## Inspiration

Standing on the shoulders of: [Kata Symphony](https://kata.sh/symphony)'s headless
issue-to-PR discipline, [Zed](https://zed.dev)'s GPUI and the Agent Client Protocol,
[gpui-component](https://github.com/longbridge/gpui-component)'s widget library, and the
worktree-orchestration lineage of Vibe Kanban, Claude Squad, and Conductor.

## License

MIT
