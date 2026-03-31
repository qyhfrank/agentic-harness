---
name: using-agents
description: >
  Use when spawning subagents, delegating work to child contexts, choosing
  between orchestration patterns, or deciding which workflow fits a multi-agent
  task. Triggers on: "use agents", "delegate", "spawn", "fan out", "parallel
  agents", "orchestrate", "which workflow", "subagent", "child context".
---

# Using Agents

Entry point for multi-agent orchestration in ASB. Provides agent type selection, workflow overview, routing, and a complete skill index.

Hard constraints (max_depth, budget, Agent Context Card) live in the always-on `agent-orchestration` rule. This skill adds the detailed guidance loaded on demand.

Platform capabilities (what each application supports) are documented in `references/application-matrix.md`. The actual primitives are platform-provided tools (Agent, Task, TeamCreate, background Bash, codex exec), not ASB skills. All ASB skills that coordinate agents are workflows at various levels of generality.

## Agent Types

Two archetypes, selected by task nature:

- **Thinker**: depth-first reasoning. Use for planning, review, root-cause analysis, architecture decisions, and any task where quality of thought matters more than speed of execution.
- **Doer**: execution-first work. Use for implementation, testing, mechanical refactoring, and any task where fast file editing and command execution matter most.

Default to Thinker for tasks requiring multi-step reasoning or quality judgment. Default to Doer for tasks requiring rapid code changes.

Platform-specific adapters and capability differences are documented in `references/application-matrix.md`.

## Workflows

All agent-coordination skills are workflows. They compose platform primitives (Agent tool, Task, TeamCreate, background Bash, codex exec) into reusable patterns with varying levels of domain specificity.

### General-purpose

- **`orchestrate`**: parallel fan-out + aggregation. Three modes: Per-Item (split tasks), Best-of-N (sample and vote), GSA (sample and synthesize). No domain logic; caller supplies content and decides what to do with results. Often composed by other workflows internally.

### Domain-specific

- **`critique`**: structured multi-agent review. Adds review strategy (Role-Diverse / Homogeneous), reviewer output contracts, severity levels, and main-agent source verification. Composes `orchestrate` for dispatch.
- **`plan-runner`**: plan execution with staged review gates. Per-task: implementer → spec review → quality review. Sequential foreground child contexts.
- **`harness`**: autonomous verified iteration. Scaffold → plan → run loop with propose-verify-evaluate-keep/discard cycle, state ledger, and worktree isolation.
- **`batch`**: large-scale parallel worktree fan-out. Research → decompose → spawn 5-30 isolated workers → each opens a PR. Distinct from `orchestrate` by having a research/planning phase, worktree isolation, and PR-per-worker output.

## Workflow Selection

See `references/workflow-selection.md` for the pattern-based routing table. Domain-specific workflows are checked first; `orchestrate` is the general-purpose fallback.

Not every agent task needs a multi-agent workflow. Single advisory child contexts (research, second opinion) and single implementation helpers use platform primitives directly (Agent tool, codex-exec), not a workflow skill.

## Delegation Heuristics

Before spawning a child context, ask:

- Does the subagent need to reconstruct significant context the parent already has? If yes, do it locally.
- Is the task a deterministic wait or scriptable monitor? Prefer tool execution (background Bash) over a child context.
- Will the parent just idle while waiting? Use foreground (blocking) delegation, not background.
- Is the delegated result on the critical path? Block on it; do not background.

Once work is delegated, the parent's role becomes coordination only. Do not perform the delegated work in parallel with the subagent.

## Skill Index

### Workflows

General-purpose:

| Skill | Purpose |
|---|---|
| `orchestrate` | Parallel dispatch + aggregation (Per-Item / BoN / GSA). Composable by other workflows. |

Domain-specific:

| Skill | Purpose |
|---|---|
| `critique` | Structured multi-agent review with role diversity |
| `plan-runner` | Plan execution with implementer + staged review gates |
| `harness` | Autonomous verified iteration: scaffold, plan, run loop |
| `batch` | Large-scale parallel worktree fan-out (5-30 workers, each opens a PR) |

### Agent Type Adapters

| Skill | Purpose |
|---|---|
| `codex-exec` | Thinker adapter: wraps `codex exec` for deep reasoning tasks |

### Related Independent Skills

| Skill | Purpose |
|---|---|
| `worktree-dev` | Isolated development in git worktrees (setup, safety checks, baseline) |
| `writing-plans` | Implementation plan generation with structured review |
