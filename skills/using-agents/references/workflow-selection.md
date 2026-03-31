# Workflow Selection

Match first applicable pattern. Domain-specific workflows are checked first; direct agent dispatch is the fallback when no domain workflow fits.

## Domain-specific workflows

| Pattern | Workflow | Key Differentiator |
|---|---|---|
| Structured review with role diversity or cross-validation | `critique` | Composes `orchestrate`, adds review strategy, severity, output contracts |
| Iterating toward a metric or goal with verified feedback loops | `harness` | Propose-verify-evaluate-keep/discard cycle with state ledger |
| Executing a known plan with staged quality gates per task | `plan-runner` | Each task: implementer + spec review + quality review |
| Large-scale parallel changes, each independently mergeable | `batch` | 5-30 isolated worktree workers, each opens its own PR |

## Direct agent dispatch (no domain workflow needed)

When the task doesn't match a domain-specific workflow but still benefits from agent delegation:

| Pattern | Surface |
|---|---|
| Single advisory opinion, research, or analysis | 1 agent: foreground read-only child context (Agent tool / codex-exec) |
| Single implementation helper in isolation | 1 agent: foreground editable child context or worktree |
| Large mechanical refactor in one changeset | 1 agent: worktree-dev + single Doer, sequential batches |
| Multiple independent subtasks or perspectives | N agents: `orchestrate` (Per-Item / BoN / GSA) |

## Local execution (default)

| Pattern | Surface |
|---|---|
| Single focused task that fits the current context | No agent delegation needed |

## When NOT to delegate

- The task fits in the current context and cold-start cost outweighs isolation benefit.
- The parent would just idle waiting for results (use foreground, not background).
- The task is a deterministic wait or monitor (use tool execution, not a child context).
- Subagents already running cover the work (do not duplicate effort locally).
- Tasks have overlapping write ownership or shared mutable state (sequential or single agent).
- The problem domain is not yet isolated -- no crisp standalone output contract per subtask.
- Merge or synthesis cost exceeds the parallelism benefit.
