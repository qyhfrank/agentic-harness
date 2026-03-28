# Evaluation Protocol

Evaluation decides what the verified round means for task progress.

## Role Split

| Role | Question it answers |
|---|---|
| Verification | Is this candidate safe and acceptable enough to consider? |
| Evaluation | Does this candidate improve the frontier, satisfy the objective, or require termination? |

Verification and evaluation may consume some of the same evidence, but they are not the same decision.

## Evaluation Inputs

Evaluators consume:

1. Verification verdicts and anchored findings
2. Acceptance criteria from config
3. Metric observations, if any
4. Stop guard state (budget, stagnation)
5. Prior frontier from `state.jsonl`

## Config Shape

```yaml
evaluation:
  objective: satisfy                # satisfy | optimize
  acceptance_criteria: []           # concrete completion checks for satisfy mode
  metric:
    name: ""                       # optional human-readable metric name
    direction: increase             # increase | decrease | none
    volatile: false
    samples: 3
    min_delta: 0
    confirmation_runs: 2
  close_authority:
    type: command_backed            # command_backed | human_review | confirmed_review
```

`termination.stop_guards.budget.max_rounds` uses these semantics:

- positive integer: hard stop after that many rounds
- `-1`: unlimited budget, rely on stagnation or human stop

## Close Authority

`close_authority` answers who may declare the task complete. When running in a worktree, `complete` triggers the Completion flow in `SKILL.md` (merge/keep/discard), not direct integration:

| Type | Meaning |
|---|---|
| `command_backed` | Completion requires deterministic command evidence or acceptance checks. |
| `human_review` | Human explicitly approves task closure. |
| `confirmed_review` | Review-based completion is allowed only after confirmation rules succeed. |

`agent_review` discovery alone never closes a task.

## Evaluation Results

The evaluator returns one of these round results:

- `baseline`
- `keep`
- `discard`
- `crash`
- `no_op`
- `hook_blocked`
- `needs_escalation`
- `complete`

## Decision Rules

Apply in order:

1. Verification failed -> `discard` or `needs_escalation`
2. Verification crashed -> `crash`
3. Verification passed but improvement is below `min_delta` -> `no_op`
4. Verification passed and candidate improves or preserves the frontier -> `keep`
5. Objective met and close authority satisfied -> `complete`

For `satisfy` tasks, `complete` depends on `acceptance_criteria`, not on the absence of reviewer complaints.

For `optimize` tasks, `keep` updates the current frontier. The evaluator does not autonomously produce `complete`; the run ends via budget, stagnation, or user-initiated stop, all of which transition to the Completion flow.

## Metric Interpretation

When `metric.direction` is:

- `increase`: higher value is better
- `decrease`: lower value is better
- `none`: metric is informational only; frontier changes rely on acceptance criteria or reviewer/human judgment

When `volatile: true`, use median sampling and confidence checks from `verification-gate.md` before deciding `keep` vs `no_op`.

## `/critique` Positioning

`/critique` belongs on the verification side as a structured discovery and review provider.

It may:
- produce anchored findings
- request escalation
- block a round when a confirmed issue exists

It may not by itself:
- declare objective satisfaction
- choose the best candidate among multiple non-blocking options
- close the task without the configured close authority
