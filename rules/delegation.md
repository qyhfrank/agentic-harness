# Delegation

Global contract for any workflow that creates new independent LLM contexts: Task/subagent calls, worktree workers, external runners (`codex exec`), and any skill that spawns subagents. Skill-specific arguments, prompt templates, and domain procedures stay in the relevant skill or agent.

## Orchestration Surfaces

- `Task` / subagent tools: primitive that creates a new agent context.
- `orchestrate`: orchestration engine for parallel dispatch and aggregation.
- `critique`: review-policy layer over `orchestrate`.
- `batch`: large-scale worktree orchestrator.
- `codex-cli`: delegates to `codex exec`; counts as a new agent context.
- Unlisted skills that spawn subagents inherit the default contract unless explicitly excepted below.
- Prefer one-hop references between layers instead of copying behavior.

## Hard Rules

1. **max_depth = 1** by default. Children are `leaf` (`child_budget = 0`). No orchestrator-to-orchestrator.
2. **Leaf agents cannot spawn.** `role = leaf`, `child_budget = 0`, or `depth >= max_depth` → no Agent, Task, or orchestration skill calls.
3. **Parent owns verification.** Subagent findings are hypotheses; parent verifies against source before surfacing to user.
4. **No cycles.** Do not invoke an orchestrator or runner already in the call stack.
5. **Budget is hard.** Every new agent context (including retries) consumes `child_budget`. Budget exhausted → continue locally or return blocked. A child may tighten but never expand inherited budget.

## Delegation

- Every subagent cold-starts without the parent's accumulated context. Before spawning, ask: does the subagent need to reconstruct significant context the parent already has? If yes, do it locally.
- Use subagents for isolation, context-window reduction, decomposable work, independent validation, or materially deeper reasoning.
- Once work is delegated to subagents, the parent's role becomes coordination only. Do not perform the delegated work in parallel with the subagent.
- If subagents are running, wait for them before yielding to the user, unless the user asks an explicit question.
- When the task is about evolving long-lived rules, memory, or skills, follow the active `self-evolve` contract: present proposals first and wait for explicit confirmation before writing to `~/.asb/` or syncing changes. If a write happened early, disclose it, audit the current state, and ask whether to keep or revert it before making further persistence changes.

## Dispatch Planning

- Before dispatching, classify each unit of work as `critical-path`, `parallelizable`, `backgroundable`, or `local-only`.
- `critical-path` work blocks the next decision or implementation step; use blocking delegation only when the parent cannot safely continue without the result.
- `parallelizable` work has no ordering dependency and no shared-write conflict; launch it in the same wave when the active workflow allows concurrency.
- `backgroundable` work is read-only or advisory and not needed immediately for the next step; prefer background delegation so the parent can continue coordinating or verifying locally.
- `local-only` work already fits in the parent's context or would cost more to reconstruct in a fresh agent than to do directly; keep it local.
- Default to `local-only` for single-file UI tweaks, narrow follow-up fixes, small serialization changes, and quick read-only checks unless there is a clear isolation or independent-review payoff that outweighs subagent cold-start cost.
- Subagents can discover and load skills. When a subagent's task may trigger skills that require further sub-delegation (e.g., `/skill-evals` running self-play, `/systematic-debugging` spawning investigation agents), set `max_depth` and `child_budget` to accommodate the expected depth. The default leaf values (`max_depth = 1`, `child_budget = 0`) are for pure leaf work; skill-aware tasks typically need at least `max_depth = 2` and `child_budget >= 1`.

## Stage Gates

- If an active skill or workflow defines stage ordering, gate checks, or review prerequisites, those constraints override the general preference for parallelism.
- Do not run work from a later gate before the earlier gate passes. Example: if a workflow says spec review must pass before code-quality review, keep those reviews sequential even when both are individually parallelizable.
- Only parallelize within the same stage when doing so does not violate the workflow's explicit contract.

## Agent Context Card

Every child prompt **must** begin with this card. Missing or malformed card → child defaults to `leaf` with `child_budget = 0`.

```
## Agent Context
- role: <leaf>              # default leaf; orchestrator only with approved exception
- depth: <parent + 1>
- max_depth: <inherited>    # may tighten, never expand
- child_budget: <0>         # leaf = 0; others = parent remaining - 1
- parent: <parent role>
```

## Approved Exceptions

| Skill / Runner | Exception |
|---|---|
| `orchestrate` | May fan out to multiple leaf workers in one wave. Deeper budget only if caller explicitly grants it. |
| `critique` | Fans out via `orchestrate`. Reviewer workers are always leaf; no orchestration skills. |
| `batch` | May spawn isolated worktree leaf workers. Extra depth only if parent explicitly grants it. |
| `skill-forge`, `self-play-optimizer` | One extra eval layer when parent sets `max_depth = 2`; second-layer workers are leaf. |
| `codex-cli` | No extra privilege; counts as a child context under caller's existing budget. |
