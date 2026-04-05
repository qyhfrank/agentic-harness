# Using Agents

## Agent Types

Two archetypes, selected by task nature:

- **Thinker**: depth-first reasoning. Planning, review, root-cause analysis, architecture decisions. Strong at finding problems; fix suggestions tend to add complexity -- use findings, apply subtraction to fixes.
- **Doer**: execution-first. Implementation, testing, mechanical refactoring.

Default to Thinker for multi-step reasoning or quality judgment. Default to Doer for rapid code changes. When consuming Thinker outputs, treat findings as hypotheses, verify them against source, and prefer the simplest fix over the agent's proposed fix.

Per-platform mapping:

| Type | Claude Code | OpenCode | Codex CLI |
|---|---|---|---|
| Thinker | `/codex-exec` | `@agent-gpt-5.4-xhigh` | native `gpt-5.4 xhigh` |
| Doer | self / Agent / Task | `@agent-opus` | native editable child context |

Platform-specific notes:

- **Claude Code**: Thinker delegates to Codex via `/codex-exec`. Model and effort are configurable (`--model`, `--effort`); defaults are `gpt-5.4` + `xhigh`. Doer uses the native editable child-context surfaces.
- **OpenCode**: Thinker uses `@agent-gpt-5.4-xhigh`. Doer uses `@agent-opus`.
- **Codex CLI**: Only OpenAI models are available. Thinker maps to `gpt-5.4 xhigh`. Doer should prefer a faster or lower-effort native coding worker when configurable, such as `gpt-5.1-codex-mini` or `gpt-5.4` at lower effort.

When delegating to Thinker: structure the prompt with XML blocks (`<task>`, `<structured_output_contract>`, `<verification_loop>`, etc.) from `/codex-exec` `references/prompt-blocks.md`. Prefer tighter prompt contracts over raising reasoning effort.

## Execution Surfaces

Choose the execution surface separately from the agent type.

| Capability | Claude Code | OpenCode | Codex CLI |
|---|---|---|---|
| Foreground tool execution (Bash etc.) | R/W | R/W | R/W |
| Background tool execution | R/W | Not supported | R/W |
| Foreground child context (blocking agent) | R/W | R/W | R/W |
| Background child context | R/W | Read-only | R/W |
| Team orchestration (shared task list / messaging) | Supported | Not supported | Not supported |

Notes:

- **Claude Code**: Full tool and child-context coverage, plus coordinated team orchestration.
- **OpenCode**: Background child contexts are read-only only. No background tool execution.
- **Codex CLI**: Full foreground/background support for tools and child contexts, but no Claude-family runners.
- Unverified applications default to the safest assumption: foreground tool execution only.
- Do not treat platform-specific names such as `delegate` or `run_in_background` as cross-platform concepts.

## Hard Rules

1. **max_depth = 1** by default. Children are `leaf` (`child_budget = 0`). No orchestrator-to-orchestrator nesting.
2. **Leaf agents cannot spawn.** `role = leaf`, `child_budget = 0`, or `depth >= max_depth` prohibits Agent, Task, or orchestration skill calls.
3. **Parent owns verification.** Subagent findings are hypotheses; parent verifies against source before surfacing to the user.
4. **No cycles.** Do not invoke an orchestrator or runner already in the call stack.
5. **Budget is hard.** Every new agent context (including retries) consumes `child_budget`. Budget exhausted means continue locally or return blocked. A child may tighten but never expand inherited budget.
6. **Background delegation is read-only/advisory by default.** Code edits require foreground child contexts, Task/subagent workflows, or worktrees. Exception: `/batch` may use editable background child contexts when each worker is isolated in its own worktree with no shared mutable state.
7. **No overlapping writes across concurrent children.** Parent must partition file ownership before dispatch. Concurrent children must not write to the same files or resources.
8. **Child prompts must be self-contained.** Every child prompt must include: scope (one problem domain), goal, constraints (what not to touch), and expected output format. No placeholders or implicit context inheritance.
9. **Child failure must be surfaced.** Do not silently swallow child failures. Failed or timed-out children still consume budget. Assess root cause: task-scoped failure -> attempt locally; systemic failure -> surface to user.

## Pre-Flight Gate

**Hard gate.** Before starting any implementation task with 3+ independent subtasks or 3+ unrelated files, evaluate this gate. Do not skip it in favor of serial execution. This checkpoint pairs with the pre-commit checkpoint in the engineering rule.

### Reasoning Checkpoint

Before evaluating dispatch strategy, assess whether the task needs a Thinker reasoning pass. **Default to yes** for non-trivial tasks where the approach is not obvious.

Route to Thinker (`/codex-exec` on Claude Code) when any of these hold:

- User describes a problem without specifying a solution path
- Multiple valid approaches exist and the best one is not obvious
- Task involves design decisions, tradeoffs, or architecture choices
- Understanding the problem requires reasoning across multiple files or systems
- Jumping to implementation would likely require backtracking

Skip the Thinker when:

- Task is trivial (single file, clear path, < 3 steps)
- User has already provided a specific implementation plan
- Task is mechanical (rename, reformat, apply a known pattern)
- Context setup cost for the Thinker exceeds the reasoning benefit

When the Thinker produces analysis and a plan, return to the dispatch evaluation below to decide execution strategy.

### Dispatch Evaluation

Evaluate in order. Stop at first match:

1. Fits in parent context, or cold-start cost > benefit? -> **local, do not spawn**
2. Subagent would need to reconstruct significant parent context? -> **local, do not spawn**
3. Deterministic wait with scriptable condition? -> **tool execution** (background if parent has work, foreground otherwise). Only escalate to a child context when the condition needs ongoing reasoning or synthesis.
4. Single child's result is the current critical path? -> **foreground child context**
5. Work decomposition will evolve during execution -- subtasks emerge from intermediate findings, workers' outputs reshape sibling scope, or convergence requires multiple dispatch-assess-redispatch waves? -> **team orchestration** (Claude Code only). If the full partition is known upfront and workers are independent, prefer `/fanout`.
6. No ordering dependency or shared-write conflict? -> **same-wave parallel launch** (use `/fanout` for independent parallel work with no inter-agent communication)
7. Read-only/advisory, parent has independent work? -> **background child context**
8. None of the above? -> **local** (default)

Quick test for steps 4-6: Can I write all child prompts before any child starts? Yes -> `/fanout` (step 6). Am I mainly waiting on one result to decide the next move? Yes -> foreground child (step 4). Do I expect at least one round of "mid-flight discovery -> revise task list -> reassign" while multiple agents stay active? Yes -> team (step 5).

Depth > 1 only when a child itself needs to fan out (e.g., `/planning` task needs `/critique` internally). Most tasks work fine with depth 1 + leaf workers. Once delegated, parent becomes coordinator only -- do not duplicate the delegated work locally.

## Agent Context Card

Every child prompt **must** open with this card. Missing or malformed card defaults to `leaf` with budget 0.

> Agent: <role> | depth <N>/<max> | budget <B> | parent: <parent_role>

## Approved Exceptions

| Skill | Exception |
|---|---|
| `/fanout` | May dispatch multiple leaf workers in one wave. Deeper budget only if caller explicitly grants it. |
| `/critique` | Fanout engine fans out via `/fanout`. Single engine uses one reviewer child. Reviewer workers are always leaf. |
| `/batch` | May spawn isolated worktree leaf workers. Extra depth only if parent explicitly grants it. |
| `/skill-forge`, `/skill-evals` | One extra eval layer when parent sets `max_depth = 2`. |

## Skill Index

Workflows compose in layers: task loop -> pluggable stage -> `/fanout` -> leaf agents.

| Skill | Layer | Purpose | Composes |
|---|---|---|---|
| `/harness` | Task loop | Autonomous verified iteration: scaffold, plan, run | `/critique` at review gates |
| `/planning` | Task loop | Sequential plan execution: task-by-task with `/critique` gates (standalone) or implementer dispatcher (embedded in harness) | `/critique` (standalone); harness verification gates (embedded) |
| `/batch` | Task loop | Parallel worktree fan-out (5-30 workers, each opens a PR) | own dispatch |
| `/critique` | Pluggable stage | Review stage: spec/quality/plan/full profiles, single/fanout engines | `/fanout` (fanout engine only) |
| `/fanout` | Dispatch engine | Parallel dispatch + aggregation (split / sample) | -- |
| `/codex-exec` | Thinker adapter | Wraps `codex exec` for deep reasoning tasks | -- |
