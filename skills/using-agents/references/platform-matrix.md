# Orchestration Platform Matrix

Platform capability matrix for execution surfaces. This file is platform-specific by design; keep runtime skills and prompt templates platform-neutral and map to these capabilities only when routing.

## Execution Surfaces

| Capability | Claude Code | OpenCode | Codex CLI |
|---|---|---|---|
| Foreground tool execution (Bash etc.) | R/W | R/W | R/W |
| Background tool execution (background Bash) | R/W | Not supported | R/W |
| Foreground child context (blocking agent) | R/W | R/W | R/W |
| Background child context (background agent) | R/W | Read-only (no shell, no file edit) | R/W |
| Team orchestration (TeamCreate + Task + SendMessage) | Supported | Not supported | Not supported |

Notes:
- OpenCode `delegate` is a read-only background child-context surface. Not tool-enabled, unsuitable for implementation work.
- Claude Code `run_in_background` supports full tool access including file edits and shell.
- Codex CLI supports both foreground and background for tools and child contexts, all with R/W access. Background child contexts use async spawn-and-wait patterns.
- Codex CLI does not support Claude models. All agents run on OpenAI models only.
- Do not treat platform-specific names (`delegate`, `run_in_background`) as cross-platform concepts.

## Agent Type Platform Mapping

| Type | Claude Code | OpenCode | Codex CLI |
|---|---|---|---|
| Thinker (depth-first reasoning) | `/codex-exec` | `@agent-gpt-5.4-xhigh` | native (`gpt-5.4 xhigh`) |
| Doer (execution-first work) | self / `Agent` / `Task` | `@agent-opus` | `gpt-5.1-codex-mini` or `gpt-5.4` with lower reasoning effort |

Platform-specific notes:
- **Claude Code**: Thinker delegates to Codex via `/codex-exec` adapter. Doer uses Claude Opus natively (self, Agent tool, Task tools). Both model families available.
- **OpenCode**: Thinker uses `@agent-gpt-5.4-xhigh`. Doer uses `@agent-opus`. Both model families available.
- **Codex CLI**: Only OpenAI models available. Claude Opus is not supported as a runner. Thinker uses `gpt-5.4` with `xhigh` reasoning effort natively. Doer uses `gpt-5.1-codex-mini` (fast, lower cost) or `gpt-5.4` with reduced reasoning effort.

Team orchestration (`TeamCreate` + `Task` + `SendMessage`) is available only on Claude Code. Use for tasks where agents need shared task lists, inter-agent messaging, and coordinated state.

## Routing Rules

1. Deterministic wait or monitor + platform supports background tool execution + parent has meaningful work -> background tool execution.
2. Deterministic wait or monitor + no background tool support -> foreground tool execution.
3. Use a child context for monitoring only when the wait condition needs ongoing reasoning or synthesis.
4. Result on the critical path and parent would otherwise just wait -> foreground execution surface.
5. Multiple agents need coordination and shared state -> TeamCreate (Claude Code only).
6. Independent parallel work with no inter-agent communication -> `orchestrate` fan-out.

## Quick Reference

| Need | Preferred Surface | Avoid |
|---|---|---|
| Read-only research | read-only child context, local reads | file-edit surfaces when not needed |
| Second opinion review | read-only child context | changing code during review |
| Deterministic wait or monitor | background tool execution (if supported); else foreground tool | child context for pure waiting |
| Implementation | current session, editable Task, worktree, harness | OpenCode `delegate` |
| Multi-step verified iteration | `harness` | ad-hoc looping with no ledger |
| Isolated rewrite or branch surgery | worktree + durable checkpoint | editing directly on busy workspace |
| Coordinated multi-agent project | TeamCreate + Task (CC only) | uncoordinated parallel agents with shared state |
