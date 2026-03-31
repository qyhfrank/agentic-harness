# Orchestration Architecture

Shared architecture note for child-context orchestration across rules, skills, and platform adapters.

## Concepts

- `execution surface`: any way the parent can execute or launch work.
- `foreground tool execution`: the parent runs tools itself and blocks.
- `background tool execution`: the parent starts a tool-driven job and continues working while it runs.
- `child context`: any independent LLM execution context created by a parent agent.
- `foreground child context`: the parent blocks and waits for the child result before continuing.
- `background child context`: the parent continues meaningful independent work while the child runs.
- `runner adapter`: a platform-specific executor for a child context, such as `codex exec` via `codex-exec`.
- `workflow consumer`: a skill that uses child contexts as part of a larger workflow, but does not own the global orchestration contract.

These are concept-layer terms. Reusable rules and prompt templates should speak in these terms first, then map to platform surfaces only where needed.

For platform capability differences, read `application-matrix.md`.

## Ownership

- `rules/agent-orchestration.md` is the authoritative always-on contract for child-context hard rules (depth, budget, Agent Context Card).
- `skills/orchestrate/SKILL.md` owns parallel dispatch engine behavior: item mode, BoN, GSA, and aggregation flow.
- `skills/critique/SKILL.md` owns review policy and how review uses orchestration.
- `skills/plan-runner/SKILL.md` owns the per-task foreground child workflow for implementation and review loops.
- `~/.asb/skills/batch/SKILL.md` owns large-scale worktree fan-out, but consumes the shared orchestration contract.
- `~/.asb/skills/codex-exec/SKILL.md` is a Thinker agent type adapter, not the global orchestration contract.
- Skills such as `brainstorming`, `writing-plans`, `simplify`, and `ingest-pptx` are workflow consumers. They may spawn child contexts, but they should not define platform dispatch syntax or redefine orchestration policy.

## Two Orchestration Layers

Keep these layers distinct.

- `task orchestration` means driving a task toward completion through stages, gates, state, and evaluation. This is where `harness` lives: task loop, scaffold/plan/run, verification gates, evaluation, and task state.
- `execution orchestration` means choosing and coordinating execution surfaces such as tool execution, child contexts, review fan-out, GSA, or runner adapters. This is where `agent-orchestration` rule, `orchestrate`, `critique`, `plan-runner`, and `batch` live.

`harness` may invoke execution-orchestration skills, but it should not redefine foreground/background execution surfaces or platform dispatch semantics. Conversely, execution-orchestration rules should not absorb the harness task loop.

## Default Decision Order

1. Decide whether the work is `local-only` or needs a child context at all.
2. If the task is a deterministic wait or monitor, prefer tool execution over a child context.
3. If a child context is needed, decide whether it is foreground or background.
4. Use a background execution surface only when the parent has meaningful independent work to continue locally.
5. Keep orchestration at the highest context that already owns the full working context and final synthesis.
6. Do not push orchestration down into a child launched for content work unless the parent explicitly delegates orchestration responsibility and budget.
7. Treat review policy as top-level by default. A content worker should normally return findings or artifacts to the parent, not recursively invoke review orchestration.

## Platform Mapping

Platform surfaces differ. Keep shared skills and prompt templates platform-neutral, and use `application-matrix.md` for concrete capability mapping.

## Prompt Template Rule

Prompt templates should define:

- child role
- scope and inputs
- output contract
- stop rules

Prompt templates should not hardcode:

- raw tool syntax such as `Task tool (...)`
- platform-only flags such as `run_in_background=true`
- raw worker type strings unless the owning platform contract guarantees them

The controller skill or platform adapter is responsible for wrapping a prompt body in the correct dispatch syntax.

## Practical Boundaries

- deterministic waiters or scriptable monitors: prefer tool execution, not a child context
- `orchestrate`: use when multiple independent child contexts are needed and the current context owns synthesis.
- `critique`: use when review policy needs orchestrated coverage and the current context owns final review judgment.
- `plan-runner`: use for critical-path implementer/reviewer loops. These child contexts are foreground by default.
- `batch`: use for worktree-isolated background fan-out at large scale.
- `codex-exec`: use when Codex is explicitly requested or selected as the Thinker adapter.

If a workflow consumer only needs one reviewer or one implementer, it should usually dispatch a single foreground child context and stop there.
