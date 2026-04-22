# Plan

## `plan.yaml` Schema

```yaml
version: 1

strategy:
  version: 1
  status: pending              # pending|active|done|blocked
  active_milestone_id: null
  last_replan_round: 0

milestones:
  - id: M1
    title: "..."
    objective: "..."
    exit_criteria: "verifiable condition"
    status: active             # pending|active|done|blocked|dropped
    approaches:
      - id: A1
        hypothesis: "..."
        score: 70              # 0-100; initial spread >= 15 between ranks
        status: active         # candidate|active|failed|done|blocked
        steps: []              # ordered sub-steps; optional, populated when approach has multi-step structure
        current_step: 0        # index into steps[]; 0 when steps is empty or first step
        attempts: 0
        revert_streak: 0
        last_failure_family: null
        evidence_for: []
        evidence_against: []
```

Field notes: `version` bumps only on structural replan (not on milestone advance). `exit_criteria` must reference a check, artifact, or observable code-state claim. `score` is a coarse ranking heuristic, not a probability. `evidence_for/against` store one-line summaries; full evidence stays in `artifacts/`.

### Approach steps

`steps[]` is an optional ordered list of sub-objectives within an approach. Each entry is a short string describing one verifiable sub-goal. When populated, Propose consumes `steps[current_step]` instead of reasoning from the hypothesis alone. On `kept`, advance `current_step` if the sub-goal is met. On `reverted`, retry the same step (not the same patch). When all steps are consumed, evaluate the milestone `exit_criteria`.

Steps are generated alongside the approach during bootstrap or replan. They may be empty for simple approaches where the hypothesis is self-explanatory. Steps are advisory, not binding: Propose may skip ahead or merge steps if evidence warrants, but must note the deviation in `Decisions`.

When `current_step >= len(steps)` but milestone `exit_criteria` is not met: fall back to hypothesis-level Propose (ignore steps, scope round to the approach hypothesis directly). If the next round also fails to advance toward exit_criteria, treat as normal adapt logic (demote/failed based on failure_scope).

Example:

```yaml
steps:
  - "implement RateLimiter class with sliding window core logic"
  - "integrate as middleware on route layer"
  - "add unit tests covering boundary cases"
  - "add config options (window size, max requests)"
current_step: 1
```

## Approach Lifecycle

Status transitions:

```
candidate -> active -> done
                    -> failed
                    -> blocked
```

Only one approach per milestone has `status: active`. On failure, promote highest-score `candidate`. No approach is ever resurrected after `failed` unless new evidence explicitly removes the original blocker.

## Score Mechanics

| Round outcome | Condition | Score delta | Approach decision |
|---|---|---|---|
| `kept` | milestone not done | +5 | `continue` |
| `kept` | exit_criteria met | +5 | `complete` -> advance milestone |
| `done` | final task objective met | 0 | `task_done` -> strategy.status=done, all active milestones/approaches → done |
| `reverted` | first occurrence of this failure family | -10 | `demote` (retry same approach) |
| `reverted` | same failure family repeated OR `[dead-end]` evidence | -30 | `failed` -> switch approach |
| `reverted` | `[constraint]` invalidates hypothesis | -30 | `failed` -> switch approach |
| `escalated` | any | 0 | `blocked` -> stop |

Hysteresis: active approach keeps priority unless a candidate exceeds it by >= 15 points.

## Failure Scope

`last_failure_family` classifies revert cause:

- `execution`: implementation bug, typo, wrong file. Retry same approach.
- `hypothesis`: evidence shows this approach direction is unviable. Switch approach.
- `environment`: tool quirk, flaky infra. Retry; repeated → escalate.

Default to `execution` when ambiguous. Only upgrade to `hypothesis` on repeated same-family failure or explicit `[constraint]`/`[dead-end]` evidence.

## Adapt Decision Table

The `adapt` step runs after Evaluate, before Record. It translates round verdicts into plan actions:

1. Read `evaluation.result` and current `plan.yaml` state
2. Classify `failure_scope` for reverted rounds
3. Update approach counters (`attempts`, `revert_streak`, `score`)
4. Determine `approach_decision`: `continue|demote|failed|complete|blocked`
5. Determine `strategy_signal`: `none|milestone_done|all_approaches_exhausted|new_constraint`
6. If `failed`: mark approach, activate next candidate by score
7. If no candidates remain: set `strategy_signal = all_approaches_exhausted`
8. Write updated `plan.yaml`; emit `strategy_updated` event if strategy changed
