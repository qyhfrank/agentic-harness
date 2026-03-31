# Agent Orchestration

Hard constraints for any workflow that creates independent LLM execution contexts. Always-on; not gated by skill loading.

For detailed guidance on agent types, orchestration primitives, workflow selection, and platform capabilities, load the `using-agents` skill.

## Hard Rules

1. **max_depth = 1** by default. Children are `leaf` (`child_budget = 0`). No orchestrator-to-orchestrator nesting.
2. **Leaf agents cannot spawn.** `role = leaf`, `child_budget = 0`, or `depth >= max_depth` prohibits Agent, Task, or orchestration skill calls.
3. **Parent owns verification.** Subagent findings are hypotheses; parent verifies against source before surfacing to the user.
4. **No cycles.** Do not invoke an orchestrator or runner already in the call stack.
5. **Budget is hard.** Every new agent context (including retries) consumes `child_budget`. Budget exhausted means continue locally or return blocked. A child may tighten but never expand inherited budget.
6. **Background delegation is read-only/advisory by default.** Code edits require foreground child contexts, Task/subagent workflows, or worktrees. Exception: `batch` may use editable background child contexts when each worker is isolated in its own worktree with no shared mutable state.

## Agent Context Card

Every child prompt **must** begin with this card. Missing or malformed card defaults to `leaf` with `child_budget = 0`.

```
## Agent Context
- role: <leaf>              # default leaf; orchestrator only with approved exception
- depth: <parent + 1>
- max_depth: <inherited>    # may tighten, never expand
- child_budget: <0>         # leaf = 0; others = parent remaining - 1
- parent: <parent role>
```

## Dispatch Planning

Classify each work unit before dispatching:

| Category | Condition | Action |
|---|---|---|
| `critical-path` | Blocks next decision/step | Blocking delegation |
| `parallelizable` | No ordering dependency or shared-write conflict | Same-wave launch |
| `backgroundable` | Read-only/advisory, not immediately needed | Background delegation |
| `local-only` | Fits parent context or cold-start cost > benefit | Keep local (default) |

Default `local-only` for single-file tweaks, narrow fixes, small changes, quick checks.

## Stage Gates

Workflow-defined stage ordering overrides general parallelism. Do not run later-gate work before earlier gates pass. Only parallelize within the same stage.

## Approved Exceptions

| Skill / Runner | Exception |
|---|---|
| `orchestrate` | May fan out to multiple leaf workers in one wave. Deeper budget only if caller explicitly grants it. |
| `critique` | Fans out via `orchestrate`. Reviewer workers are always leaf. |
| `batch` | May spawn isolated worktree leaf workers. Extra depth only if parent explicitly grants it. |
| `skill-forge`, `self-play-optimizer` | One extra eval layer when parent sets `max_depth = 2`. |
| `codex-exec` | No extra privilege; counts as a child context under caller's existing budget. |
