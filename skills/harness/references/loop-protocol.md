# Loop Protocol

Phase B core: the autonomous propose-verify-evaluate-record cycle.

## Prerequisites

- Preflight passed (see `preflight-protocol.md`)
- Baseline recorded as round 0 in `state.jsonl`
- config.yaml is finalized (no `draft: true`)

## Single Round Lifecycle

```
1. PROPOSE  -->  2. COMMIT  -->  3. VERIFY  -->  4. EVALUATE  -->  5. RECORD  -->  6. ENTROPY CHECK
     ^                                                                                             |
     |_____________________________________________________________________________________________|
                                                   continue
```

### 1. Propose

- Read context.md Next Steps for direction.
- If the change is non-trivial (multi-file, architectural, or unclear path), invoke the `brainstorming` skill to explore alternatives before committing to an approach. Skip for simple, obvious changes.
- Respect `implementation.protocol` from config:
  - `tdd_required`: before writing production code, create one minimal failing test or repro, run it, and confirm it fails for the expected reason. Then write the minimal code to pass.
  - `tdd_preferred`: follow the same flow when practical. If you cannot start with a failing test or repro, record why in the round summary before proceeding.
  - `direct`: full red-first sequencing is not required, but behavior changes still need appropriate verification coverage.
- When writing or changing tests:
  - test real behavior, not mock existence
  - do not add test-only methods to production code
  - do not mock dependencies until you understand which side effects the test relies on
- Make code changes within `boundary.mutable`. Never touch `boundary.immutable`.
- Apply `atomicity_test`: if the change description needs `and`, split into separate rounds.
- Apply `simplicity_rule` from config if set.

### 2. Commit

- Stage only files within `boundary.mutable`.
- Commit with message: `harness(round-{N}): <description>`
- If pre-commit hook blocks: record evaluation result `hook_blocked`, do not retry. Move to Record.

### 3. Verify

Filter gates by frequency before execution (see `verification-gate.md` Verification Tiers > Gate Frequency):
- `every_round`: always run (Tier 0 and Tier 1 gates).
- `milestone`: run when `current_round % termination.milestone_interval == 0` (default interval 10), and always on the final round.
- `final`: only run during the completion verification pass (step 4 reaches a `complete` candidate).

When a gate is skipped due to frequency, note it in `artifacts/round-{N}/` but do not record a verdict for it.

Execute eligible gates per `verification-gate.md` composite execution order:

1. Run all mandatory `command` gates.
2. On all pass, run `agent_review` gates (if configured). Use the `critique` skill as the review engine for discovery-heavy agent-review gates.
3. On pass or escalation, queue `human_review` if configured.
4. Short-circuit on first mandatory deterministic `fail`.

Capture full stdout/stderr to `<harness_root>/.harness/tasks/<task_id>/artifacts/round-{N}/`.

### 4. Evaluate

Use `evaluation-protocol.md` to decide what the verification result means.

| Verification + evaluation outcome | Result |
|---|---|
| Deterministic gate failed | `discard` |
| Review produced unresolved escalation | `needs_escalation` |
| Verification command crashed | `crash` |
| Verified, but improvement is below `min_delta` | `no_op` |
| Verified and frontier improved or held acceptably | `keep` |
| Objective met and close authority satisfied | `complete` |

After revert (`discard` or `crash`): verify the revert itself does not break baseline. If it does, stop and escalate.

### 5. Record

Append one event to `state.jsonl` per `state-ledger.md`.

Update context.md per `context-protocol.md` update discipline:
- Overwrite Current State
- Curate Working Memory
- Rewrite Next Steps
- Append to Decisions if a new decision was made

### 6. Entropy Check

After recording, evaluate whether to continue or stop.

#### Stop Conditions (check in order)

1. **Objective met** (`satisfy` mode): all `acceptance_criteria` pass and `close_authority` is satisfied.
2. **Budget exhausted**: current round >= `max_rounds`. Stop, report best result.
   - If `max_rounds == -1`, budget stop is disabled.
3. **Stagnation**: last N rounds (where N = `stagnation.rounds`) all have result `no_op` or `discard`. Stop, report best result.
4. **Escalation**: result is `needs_escalation`. Pause, wait for human input.
5. **Complete**: latest round result is `complete`. Stop with success.

#### Doom Loop Detection

Track error patterns across rounds. If the same guard fails with the same error signature N times (where N = `doom_loop_threshold`):

Before executing any doom_loop_action, invoke the `systematic-debugging` skill to diagnose the root cause. The diagnosis informs the pivot -- do not pivot blindly.

| `doom_loop_action` | Behavior |
|---|---|
| `reread_and_pivot` | Re-read all files in `boundary.mutable`. Discard current approach. Invoke `brainstorming` skill to generate alternative strategies, then pick one. Note the pivot and reasoning in context.md Decisions. |
| `widen_boundary` | Suggest expanding `boundary.mutable`. Requires user approval. Pause loop until approved. |
| `ask_human` | Pause loop. Present the repeated failure pattern and ask for guidance. |

Error signature matching: normalize guard output by stripping line numbers and timestamps. Two errors match if they share the same failing test or rule name and error category.

#### Continue

If no stop condition fires, return to step 1 (Propose) for the next round.

## Volatile Metric Handling

When `evaluation.metric.volatile: true` in config:

- Step 3 (Verify): run measurement `evaluation.metric.samples` times, take median.
- Step 4 (Evaluate): compare median against the previous frontier. Apply `min_delta` and `confirmation_runs`.
- Step 5 (Record): store the metric payload in `state.jsonl`.

## Rollback Safety

- `rollback.granularity: commit` (default): `git revert HEAD --no-edit`
- `rollback.strategy: revert_commit` (default): create a revert commit (preserves history)
- `rollback.strategy: reset_to_last_pass`: `git reset --hard <last_keep_commit>` (destructive, loses intermediate history)
- `rollback.preserve_failed_experiments: true`: before reverting, save the diff to `artifacts/round-{N}/attempted.patch`

## Git Isolation

Each active task works on branch `<task_slug>` inside a worktree (see Worktree Isolation in `SKILL.md`). All commits stay on the task branch until the user merges via the Completion flow. Harness state (`.harness/`) lives at the harness root, not in the worktree.

## Session Boundary

When ending a session mid-loop:

1. Complete the current round (do not leave a half-committed state).
2. Update context.md with full current state.
3. If mid-verify, record as `crash` and revert.
4. Write a structured feedback note per `feedback-protocol.md`.
5. Auto-detect: check `state.jsonl` for repeated failure signatures. Append `repeat_loop` events.
6. Report: current round, best result, and how to resume (`/harness run`).

When the loop terminates (not mid-session), hand off to the Completion flow in `SKILL.md` for merge/keep/discard.
