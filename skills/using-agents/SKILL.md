---
name: using-agents
description: >
  Use when spawning subagents, delegating work to child contexts, choosing
  between orchestration patterns, or deciding which workflow fits a multi-agent
  task. Triggers on: "use agents", "delegate", "spawn", "fan out", "parallel
  agents", "orchestrate", "which workflow", "subagent", "child context".
---

# Using Agents

Entry point for multi-agent orchestration in ASB.

Hard constraints (max_depth, budget, Agent Context Card) live in the always-on `agent-orchestration` rule. This skill adds detailed guidance loaded on demand.

Platform capabilities are documented in `references/application-matrix.md`. The actual primitives are platform-provided tools (Agent, Task, TeamCreate, background Bash, codex exec), not ASB skills.

## Agent Types

Two archetypes, selected by task nature:

- **Thinker**: depth-first reasoning. Planning, review, root-cause analysis, architecture decisions.
- **Doer**: execution-first. Implementation, testing, mechanical refactoring.

Default to Thinker for multi-step reasoning or quality judgment. Default to Doer for rapid code changes. See `references/application-matrix.md` for per-platform adapter mapping.

## Workflow Selection

See `references/workflow-selection.md` for the routing table.

Domain-specific workflows are checked first. If none match, use direct agent dispatch (1 agent or N via `orchestrate`). Not every task needs a workflow -- single child contexts use platform tools directly.

## Delegation Heuristics

Before spawning a child context:

- Subagent needs to reconstruct significant parent context? Do it locally.
- Deterministic wait or monitor? Prefer tool execution over a child context.
- Parent would just idle? Use foreground, not background.
- Result on critical path? Block on it.

Once delegated, parent becomes coordinator only. Do not duplicate the delegated work locally.

## Consuming Agent Outputs

Thinker agents (deep reasoning, review, analysis) are strong at finding problems but their fix suggestions tend to add complexity. Use their findings, apply subtraction to their fixes.

When consuming multi-agent outputs (orchestrate, critique, GSA):
- Treat agent findings as hypotheses. Parent verifies against source before surfacing.
- Consensus across agents strengthens a finding. Single-agent findings need more scrutiny.
- Prefer the simplest fix that addresses the identified problem, not the agent's proposed fix.

## Skill Index

### Workflows

Domain-specific:

| Skill | Purpose |
|---|---|
| `critique` | Structured multi-agent review with role diversity |
| `plan-runner` | Plan execution with implementer + staged review gates |
| `harness` | Autonomous verified iteration: scaffold, plan, run loop |
| `batch` | Large-scale parallel worktree fan-out (5-30 workers, each opens a PR) |
| `orchestrate` | Parallel dispatch + aggregation (Per-Item / BoN / GSA). Also composable by other workflows. |

### Agent Type Adapters

| Skill | Purpose |
|---|---|
| `codex-exec` | Thinker adapter: wraps `codex exec` for deep reasoning tasks |

### Related Independent Skills

| Skill | Purpose |
|---|---|
| `worktree-dev` | Isolated development in git worktrees (setup, safety checks, baseline) |
| `writing-plans` | Implementation plan generation with structured review |
