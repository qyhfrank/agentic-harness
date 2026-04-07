# Using Agents

Shared delegation contract for child-agent work.

This rule owns only the always-on cross-skill invariants: agent archetypes, execution surfaces, delegation safety, and dispatch checkpoints. Skill-specific execution detail such as prompt wrappers, reviewer layouts, fanout shapes, and workflow-local exceptions stays with the owning skills.

## Agent Types

- **Thinker**: depth-first reasoning for planning, review, root-cause analysis, and architecture choices. Treat findings as hypotheses and prefer simpler fixes than the Thinker proposes.
- **Doer**: execution-first implementation for code changes, tests, and mechanical refactors.

Per-platform mapping:

| Type | Claude Code | OpenCode | Codex CLI |
|---|---|---|---|
| Thinker | `/codex-exec` | `@agent-gpt-5.4-xhigh` | native `gpt-5.4 xhigh` |
| Doer | self / Agent / Task | `@agent-opus` | native editable child context |

When a skill owns the procedure for Thinker prompting or Doer dispatch, follow that skill instead of re-deriving it from this rule.

## Execution Surfaces

Choose the execution surface separately from the agent type.

| Capability | Claude Code | OpenCode | Codex CLI |
|---|---|---|---|
| Foreground tool execution (Bash etc.) | R/W | R/W | R/W |
| Background tool execution | R/W | Not supported | R/W |
| Foreground child context (blocking agent) | R/W | R/W | R/W |
| Background child context | R/W | Read-only | R/W |
| Team orchestration (shared task list / messaging) | Supported | Not supported | Not supported |

Prefer the simplest surface that satisfies the current step. Deterministic waits and scriptable monitors stay with tools unless ongoing reasoning is required.

## Hard Rules

1. **max_depth = 1** by default. Children are `leaf` (`child_budget = 0`) unless the caller explicitly grants deeper budget.
2. **Leaf agents cannot spawn.** `role = leaf`, `child_budget = 0`, or `depth >= max_depth` prohibits Agent, Task, or orchestration skill calls.
3. **Parent owns verification.** Subagent findings are hypotheses until the parent verifies them against source, code, or tool output.
4. **No cycles.** Do not invoke an orchestrator or runner already in the call stack.
5. **Budget is hard.** Every new agent context, including retries, consumes `child_budget`. A child may tighten but never expand inherited budget.
6. **Background delegation is read-only or advisory by default.** Concurrent code edits require isolated ownership and a surface that explicitly supports safe writes.
7. **No overlapping writes across concurrent children.** Partition file and resource ownership before dispatch.
8. **Child prompts must be self-contained.** Include scope, goal, constraints, and expected output format. Do not rely on implied context.
9. **Child failure must be surfaced.** Failed or timed-out children still consume budget; do not silently swallow them.

## Reasoning Checkpoint

Before any non-trivial task, decide whether it needs a Thinker pass.

Route to Thinker when any of these hold:

- The user describes a problem without a clear solution path.
- Multiple valid approaches exist and the best one is not obvious.
- The task involves architecture, tradeoffs, or quality judgment.
- Understanding the task requires reasoning across multiple files or systems.

Skip the Thinker only for clearly trivial work: single-file mechanical changes, direct lookups, or tasks that already have a specific implementation plan.

When a design decision has multiple plausible answers, prefer multi-perspective fanout over a single Thinker.

## Pre-Flight Gate

Before implementation work with 3+ independent subtasks or 3+ unrelated files, choose the execution shape explicitly:

1. **Local** when the work fits in the parent context, or a child would mostly reconstruct context.
2. **Tool execution** when the work is a deterministic wait or scriptable monitor.
3. **Foreground child** when one child result is the current critical path.
4. **Same-wave fanout** when subtasks are independent and there are no shared-write conflicts.
5. **Background child** for read-only or advisory work when the parent has independent work to do.
6. **Team orchestration** only when task decomposition will evolve mid-flight and the platform supports it.

Quick test:

- If you can write all child prompts before any child starts, prefer fanout.
- If you are mainly waiting on one answer to decide the next move, prefer one foreground child.
- If workers would need repartitioning after new findings, use team orchestration where available or stay local.

## Agent Context Card

Every child prompt must open with this card:

> Agent: <role> | depth <N>/<max> | budget <B> | parent: <parent_role>
