# Run

initialize -> [propose -> cleanup -> commit -> verify -> evaluate -> adapt -> record -> replan-check -> stop-check]* -> disposition

## State Schema

### `context.md`

```markdown
# Harness Context

## Current State
- phase:
- round:
- active_milestone:
- active_approach:
- version:
- current_objective:
- best_result:
- last_action:

## Working Memory
- ...

## Durable Notes
- [dead-end] ...
- [constraint] ...
- [tool-quirk] ...
- [code-map] ...

## Decisions
- ...

## Next Steps
- ...
```

Mutation rules:

- `Current State`: overwrite every round. `phase`: `setup (complete)` | `run`. `round`: `completed rounds / current stage` (`-` `preflight` `propose` `cleanup` `commit` `verify` `evaluate` `record` `stopped`)
- `Working Memory`: prune every round
- `Durable Notes`: persist across rounds
- `Decisions`: append-only
- `Next Steps`: rewrite every round

### `state.jsonl`

Append-only. `metric` field present only for optimize. Add `evaluation.reason` when `reverted|escalated`.

```json
{"event":"baseline_recorded","task_id":"…","ts":"…","round":0,"commit":"<sha>","verification":{"status":"pass","gates":{"<check>":"pass"}},"evaluation":{"result":"baseline"},"metric":{"value":0,"delta":0},"summary":"…"}
```

```json
{"event":"round_completed","task_id":"…","ts":"…","round":1,"commit":"<sha|null>","verification":{"status":"pass|fail|escalated","gates":{"<check>":"pass|fail|escalated"}},"evaluation":{"result":"kept|reverted|escalated|done"},"controller":{"version":1,"milestone_id":"M1","approach_id":"A1","approach_decision":"continue|demote|failed|complete|task_done|blocked","strategy_signal":"none|milestone_done|all_approaches_exhausted|new_constraint"},"metric":{"value":0,"delta":0},"summary":"…"}
```

```json
{"event":"harness_stopped","task_id":"…","ts":"…","round":3,"reason":"done|escalated|max_rounds|stagnation","summary":"…"}
```

```json
{"event":"task_disposed","task_id":"…","ts":"…","round":3,"disposition":"merged|kept|discarded","summary":"…"}
```

```json
{"event":"strategy_updated","task_id":"…","ts":"…","round":3,"version":2,"reason":"bootstrap|advance|replan","trigger":"initial|legacy_migration|milestone_done|all_approaches_exhausted|new_constraint|stagnation","active_milestone_id":"M2","summary":"…"}
```

Invariants:

- Exactly 1 `baseline_recorded`, at `round: 0`
- `round_completed.round` strictly increments by 1
- `harness_stopped`, if present, appears exactly once and immediately after the final `round_completed`
- `task_disposed`, if present, appears exactly once and must be the last event; must follow `harness_stopped`
- `kept|done` requires `verification.status: pass`
- `strategy_updated` may follow any `round_completed` or `baseline_recorded`; `version` strictly increments on `reason: replan`
- `state.jsonl` = event truth; `context.md` = live working state

### `plan.yaml`

See `references/plan.md` for canonical schema and approach lifecycle.

`current_objective` in `context.md` tracks the active milestone objective, not the global task goal.

## Initialize

### Route Detection

Locate `.harness/` and target task, read task state, classify route:

- **fresh**: no `baseline_recorded`
- **resume**: has recorded events
- **recovery**: `state.jsonl` empty but working directory shows progress beyond setup defaults

### Reconcile (all routes)

- dirty working dir -> [fresh: escalate] [resume/recovery: read diff, continue if aligned with task goal, otherwise escalate]
- `HEAD` ahead of most recent recorded commit -> read new commits, reconcile with state.jsonl if they look like harness rounds, otherwise escalate
- recovery: do not delete existing task files or artifacts
- **resume: if last `round_completed.evaluation.result` is `done` or `escalated`, treat as missing `harness_stopped` and escalate (do not enter Round Lifecycle)**
- **resume: after reconcile, enter Round Lifecycle directly**

### Preflight (fresh + recovery)

- Worktree: create if missing (`.worktree/<task_slug>/`, branch = `<task_slug>`); path anomaly -> escalate
- stale `index.lock` -> delete
- scope: `boundary.immutable` paths must exist, missing -> escalate
- run all configured checks; failure -> attempt fix then rerun; unfixable -> update `context.md` (`last_action`, `Next Steps`) then escalate

### Baseline (fresh + recovery)

After Preflight passes:

1. Collect baseline summary; also collect baseline metric when `optimize`
2. Append `baseline_recorded` to `state.jsonl`
3. Update `context.md`: `phase: run`, `round: 0 / propose`, `current_objective`, `best_result`, `last_action: "baseline recorded"`
4. Rewrite `Next Steps`, point to round 1

`baseline_recorded` anchors the starting point; it does not mean the task is complete.

### Bootstrap Strategy (fresh + recovery)

After Baseline passes:

1. Decompose goal into 2-5 ordered milestones with verifiable `exit_criteria`
2. For the first active milestone, generate 2-3 ranked approaches (initial score spread >= 15)
3. Write `plan.yaml` with `strategy.status: active`, `active_milestone_id: M1`
4. Emit `strategy_updated(reason=bootstrap, trigger=initial)`
5. Update `context.md`: `active_milestone`, `active_approach`, `version`

## Round Lifecycle

### Propose

- Read `plan.yaml`: take `active_milestone_id` and its `active` approach
- If approach has `steps[]` populated and `current_step < len(steps)`: scope the round to `steps[current_step]`
- If `steps[]` exhausted (`current_step >= len(steps)`) or empty: scope to the approach `hypothesis`
- Scan `Durable Notes` for `[dead-end]`, `[constraint]` relevant to this milestone/approach
- Non-trivial changes: converge on approach first; invoke `/brainstorming` when needed
- Obey `task.protocol` and `execution_policy` (`dangerous_commands` require human approval; `secret_patterns` never read or staged)
- When `task.protocol` is `tdd_required` or `tdd_preferred`, load `references/tdd-discipline.md`
- One atomic round at a time; if the description needs "and" to explain, split into multiple rounds

### Cleanup

- Skip for small changes (< ~20 LOC, <= 3 files, no prior-round revert)
- Re-read diff, check reuse / simplicity / efficiency

### Commit

- Stage only `boundary.mutable`
- commit message: `chore(harness): round-{N} <description>`
- hook blocked: save patch to `artifacts/round-{N}/`, reset to pre-round HEAD, mark as `reverted` (reason: `hook_blocked`)

### Verify

Execute by `cost` group: `cheap -> medium -> expensive`, in list order within each group. Any check fail short-circuits the current group and all higher-cost groups.

Checks with `action` starting with `/` are dispatched as skill calls; verdict written to `verification.gates` (`pass` / `fail` / `escalated`). Map skill verdict `needs_escalation` to gate value `escalated`.

Evidence: stdout/stderr + `artifacts/round-{N}/manifest.json`. Short-circuited checks are not written to gates.

### Evaluate

First match wins:

1. Any failure (check fail / crash / hook blocked / timeout) -> `reverted` (reason field distinguishes)
2. verification escalated -> `escalated`, pause for human
3. `optimize` and `metric.delta` is non-null and < `min_delta` -> `reverted` (reason: `below_threshold`)
4. objective met -> `done` (satisfy: all checks pass; optimize: all pass + target reached)
5. otherwise -> `kept`

After `reverted`, rerun cheap checks to confirm baseline is intact.

Rollback: `revert_commit` = `git revert HEAD --no-edit`; `reset_to_last_pass` = `git reset --hard <last_kept_commit>` (requires human approval). When `preserve_failed_experiments: true`, save `artifacts/round-{N}/attempted.patch` before revert.

`round_completed.commit` = current HEAD SHA at time of recording (after any rollback). For pre-commit failures, `commit` = `null`. `last_kept_commit` = `commit` of the most recent `round_completed` where `evaluation.result` is `kept` or `done`; falls back to `baseline_recorded.commit` if none exists.

Metric runtime (optimize):

- `volatile: true`: rerun `N-1` additional times within the round, take median
- `metric.delta`: increase = `value - best_value`, decrease = `best_value - value`
- `best_value`: best `metric.value` among baseline and historical `kept` events; frontier updated only on `kept`
- `target`: increase = `best_value >= target`, decrease = `best_value <= target`
- `reading` cannot yield a unique float -> `escalated` (reason: `escalation`)

### Adapt

Translate round verdict into plan-level actions. See `references/plan.md` for the full decision table.

1. Classify `failure_scope` for reverted rounds (`execution|hypothesis|environment`)
2. Update approach: `attempts++`, adjust `revert_streak` and `score` per delta table
3. If `kept` and approach has `steps[]`: advance `current_step` when sub-goal is met
4. Determine `approach_decision` and `strategy_signal`
4. If `approach_decision = failed`: mark approach, activate highest-score candidate
5. If no candidates remain: `strategy_signal = all_approaches_exhausted`
6. If `approach_decision = complete`: mark milestone done, advance to next pending milestone; emit `strategy_updated(reason=advance, trigger=milestone_done)`
7. If `evaluation.result = done`: set `strategy.status = done`, mark current milestone and approach as `done`
8. Update `plan.yaml`

`evaluation.result = done` is reserved for final task completion. Milestone completion uses `kept` + `approach_decision = complete`. `approach_decision` maps to `approach.status`: `failed → failed`, `blocked → blocked`, `complete → done`.

### Record

Update `state.jsonl` (append event) and `context.md` (round state, objective, best_result, Next Steps). Write `artifacts/round-{N}/` evidence.

## Stop Conditions

Checked after each round's Record, first match wins:

1. `done`
2. `escalated`
3. round >= `termination.max_rounds` (`-1` disables)
4. stagnation (satisfy: last N rounds all `reverted` and replan-check has run; optimize: last N metric samples did not advance frontier)

`kept` is not a reason to stop.

When a stop condition fires:

1. Append `harness_stopped` to `state.jsonl` with the matching `reason`
2. Update `context.md`: set `round: N / stopped`, `last_action` to stop summary, rewrite `Next Steps` to disposition prompt
3. Present disposition options to user

### Doom loop (tactics-level)

Same check fails with same error pattern N times (`doom_loop_threshold`) within the current milestone:

1. Record pattern in `Durable Notes` (`[dead-end][M#/A#]`)
2. `/fanout -a codex:6 -m sample` for independent diagnosis with GSA synthesis (in Claude Code, load `/codex-exec` first)
3. Fanout candidates become new approaches in current milestone (initial `score: 40`, `status: candidate`)
4. Subsequent rounds consume candidates through normal lifecycle
5. All exhausted -> `strategy_signal = all_approaches_exhausted` -> triggers replan

### Replan Protocol

**Triggers** (checked at `replan-check`, after Record):

1. `milestone_done`: activate next pending milestone; generate tactics for it. Not a structural replan; does not bump `plan_version`.
2. `all_approaches_exhausted`: current milestone has no `active|candidate` approaches.
3. `new_constraint`: `[constraint][strategy]` in Durable Notes invalidates milestone order or feasibility.
4. `stagnation`: current milestone has N consecutive rounds with no `kept` progress across multiple approaches.

**Not triggers**: single `reverted`, single approach switch, `[code-map]` without order implications, doom-loop candidates not yet consumed.

**Replan execution**:

1. Freeze all `status: done` milestones. Never rewrite completed prefix.
2. Suffix-only: rewrite active + pending milestones. Allowed ops: split, reorder, drop, insert prerequisite.
3. Generate 2-3 ranked approaches for affected milestones.
4. `strategy.version += 1`, emit `strategy_updated(reason=replan, trigger=<matching trigger>)`.

**Anti-thrash**: at least 1 completed round between replans; only hard triggers (`new_constraint`, `escalated`) bypass cooldown.

## Disposition

After user selects `merge|keep|discard`:

- **`merged`** (worktree mode only): squash-merge back to `task.base_branch` (Conventional Commits), delete branch and worktree
- **`kept`**: preserve final code state; worktree mode reports path + branch, in-place preserves current state
- **`discarded`**: worktree mode deletes branch + worktree; in-place reverts to `baseline_recorded.commit` (**requires explicit user confirmation**)

`.harness/` is always preserved.

After applying: append `task_disposed` to `state.jsonl`; update `context.md` (`round: N / stopped`, `last_action`: final summary, clear `Next Steps`).
