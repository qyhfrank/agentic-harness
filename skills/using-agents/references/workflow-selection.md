# Workflow Selection

Match first applicable pattern. Each workflow skill declares its own trigger conditions; the skill matching protocol handles concrete routing.

## General-purpose

| Pattern | Workflow | Key Differentiator |
|---|---|---|
| Multiple independent subtasks or perspectives, caller owns synthesis | `orchestrate` | Pure fan-out + aggregation, no domain logic. Composable by other workflows. |
| Structured review with role diversity or cross-validation | `critique` | Composes `orchestrate`, adds review strategy, severity, output contracts. |

## Domain-specific

| Pattern | Workflow | Key Differentiator |
|---|---|---|
| Iterating toward a metric or goal with verified feedback loops | `harness` | Propose-verify-evaluate-keep/discard cycle with state ledger |
| Executing a known plan with staged quality gates per task | `plan-runner` | Each task: implementer + spec review + quality review |
| Large-scale parallel changes, each independently mergeable | `batch` | 5-30 isolated worktree workers, each opens its own PR |
| Single focused task, no orchestration overhead justified | Local execution | Default when the task fits the current context |

## When NOT to orchestrate

- The task fits in the current context and cold-start cost outweighs isolation benefit.
- The parent would just idle waiting for results (use foreground execution instead of background).
- The task is a deterministic wait or monitor (use tool execution, not a child context).
- Subagents already running cover the work (do not duplicate effort locally).
