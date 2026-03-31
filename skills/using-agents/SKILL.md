---
name: using-agents
description: >
  Use when spawning subagents, delegating work to child contexts, choosing
  between orchestration patterns, or deciding which workflow fits a multi-agent
  task. Triggers on: "use agents", "delegate", "spawn", "fan out", "parallel
  agents", "orchestrate", "which workflow", "subagent", "child context".
---

# Using Agents

Entry point for multi-agent orchestration in ASB. Provides agent type selection, orchestration primitive overview, workflow routing, and a complete skill index.

Hard constraints (max_depth, budget, Agent Context Card) live in the always-on `agent-orchestration` rule. This skill adds the detailed guidance loaded on demand.

## Agent Types

Two archetypes, selected by task nature:

- **Thinker**: depth-first reasoning. Use for planning, review, root-cause analysis, architecture decisions, and any task where quality of thought matters more than speed of execution.
- **Doer**: execution-first work. Use for implementation, testing, mechanical refactoring, and any task where fast file editing and command execution matter most.

Default to Thinker for tasks requiring multi-step reasoning or quality judgment. Default to Doer for tasks requiring rapid code changes.

Platform-specific adapters and capability differences are documented in `references/application-matrix.md`.

## Orchestration Primitives

Reusable coordination mechanisms. They define HOW to coordinate agents, not WHAT task to perform. The caller supplies the task content and decides what to do with results.

- **`orchestrate`**: parallel fan-out + aggregation. Three modes: Per-Item (split tasks), Best-of-N (sample and vote), GSA (sample and synthesize). Use when multiple independent subtasks or perspectives are needed and the current context owns final synthesis.
- **`critique`**: review policy layer over `orchestrate`. Adds role diversity or homogeneous cross-validation, reviewer output contracts, and main-agent source verification. Use for structured multi-agent review.

Selection: independent parallel work -> `orchestrate`. Review with coverage guarantees -> `critique`.

## Workflow Selection

See `references/workflow-selection.md` for the pattern-based routing table.

Each workflow skill declares its own trigger conditions. The skill matching protocol in `rules/engineering.md` handles concrete routing. This section provides orientation, not hard routing.

## Delegation Heuristics

Before spawning a child context, ask:

- Does the subagent need to reconstruct significant context the parent already has? If yes, do it locally.
- Is the task a deterministic wait or scriptable monitor? Prefer tool execution (background Bash) over a child context.
- Will the parent just idle while waiting? Use foreground (blocking) delegation, not background.
- Is the delegated result on the critical path? Block on it; do not background.

Once work is delegated, the parent's role becomes coordination only. Do not perform the delegated work in parallel with the subagent.

## Skill Index

### Primitives (coordination mechanisms)

| Skill | Purpose |
|---|---|
| `orchestrate` | Parallel dispatch engine: Per-Item, Best-of-N, GSA |
| `critique` | Structured multi-agent review with role diversity |

### Workflows (complete lifecycles)

| Skill | Purpose |
|---|---|
| `harness` | Autonomous verified iteration: scaffold, plan, run loop |
| `plan-runner` | Plan execution with implementer + staged review gates |
| `batch` | Large-scale parallel worktree fan-out (5-30 workers, each opens a PR) |

### Agent Type Adapters

| Skill | Purpose |
|---|---|
| `codex-exec` | Thinker runner adapter: wraps `codex exec` for deep reasoning tasks |

### Related Independent Skills

| Skill | Purpose |
|---|---|
| `worktree-dev` | Isolated development in git worktrees (setup, safety checks, baseline) |
| `writing-plans` | Implementation plan generation with structured review |
