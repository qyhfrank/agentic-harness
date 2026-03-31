# Workflow Selection

Match first applicable pattern. Domain-specific workflows are checked first; `orchestrate` is the general-purpose fallback when no domain workflow fits.

## Domain-specific workflows

| Pattern | Workflow | Key Differentiator |
|---|---|---|
| Structured review with role diversity or cross-validation | `critique` | Composes `orchestrate`, adds review strategy, severity, output contracts |
| Iterating toward a metric or goal with verified feedback loops | `harness` | Propose-verify-evaluate-keep/discard cycle with state ledger |
| Executing a known plan with staged quality gates per task | `plan-runner` | Each task: implementer + spec review + quality review |
| Large-scale parallel changes, each independently mergeable | `batch` | 5-30 isolated worktree workers, each opens its own PR |

## General-purpose workflow

| Pattern | Workflow | Key Differentiator |
|---|---|---|
| Multiple independent subtasks or perspectives, none of the above fits | `orchestrate` | Pure fan-out + aggregation, no domain logic. Composable. |

## Single child context (not a multi-agent workflow)

| Pattern | Surface | Notes |
|---|---|---|
| Single advisory opinion, research, or analysis | Foreground read-only child context | Not a workflow skill; use Agent tool / codex-exec directly |
| Single implementation helper in isolation | Foreground editable child context or worktree | Use Agent tool + worktree-dev |
| Large mechanical refactor (rename, migrate) in one changeset | worktree-dev + current session or single Doer | Do not split into parallel PRs; sequential batches in one workspace |

## Local execution (default)

| Pattern | Surface |
|---|---|
| Single focused task that fits the current context | No orchestration needed |

## When NOT to orchestrate

- The task fits in the current context and cold-start cost outweighs isolation benefit.
- The parent would just idle waiting for results (use foreground execution, not background).
- The task is a deterministic wait or monitor (use tool execution, not a child context).
- Subagents already running cover the work (do not duplicate effort locally).
- Tasks have overlapping write ownership or shared mutable state (sequential execution or single agent instead).
- The problem domain is not yet isolated -- no crisp standalone output contract per subtask.
- Merge or synthesis cost exceeds the parallelism benefit.
