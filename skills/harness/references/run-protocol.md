# Run Protocol

The autonomous propose-verify-evaluate-record cycle, plus checks model and evaluation rules.

## Prerequisites

- Preflight passed (see `setup-protocol.md` Preflight)
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
- Check context.md Durable Notes: do not revisit known dead ends, violate known constraints, or ignore known tool quirks.
- If the change is non-trivial, invoke `/brainstorming` to explore alternatives first.
- Respect `implementation.protocol` from config:
  - `tdd_required`: create failing test first, confirm it fails, then write minimal code to pass.
  - `tdd_preferred`: follow TDD when practical, record why if skipped.
  - `direct`: no red-first required, but behavior changes need verification coverage.
- Make code changes within `boundary.mutable`. Never touch `boundary.immutable`.
- Enforce `execution_policy` from config: do not run commands matching `dangerous_commands` without human approval, never read or stage files matching `secret_patterns`, respect `network_policy` and `dependency_install` settings.
- Apply `atomicity_test`: if the change description needs "and", split into separate rounds.

### 2. Commit

- Stage only files within `boundary.mutable`.
- Commit with message: `harness(round-{N}): <description>`
- If pre-commit hook blocks: record `reverted` (reason: `hook_blocked`), move to Record.

### 3. Verify

Read `checks[]` from config.yaml. Partition into:

```
cheap    = checks where cost == "cheap" (includes probes)
review   = checks where kind == "review"
expensive = checks where cost == "expensive"
```

Execute in order:

**3a. Cheap checks** -- Run each cheap command in list order. Record per-gate verdict in `verification.gates`. On first fail: short-circuit, aggregate status = `fail`, skip 3b/3c.

**3b. Review gate** (if configured) -- Invoke `/critique` with appropriate profile. Record verdict in `verification.gates`. On fail: aggregate status = `fail`, skip 3c. On `needs_escalation`: aggregate status = `blocked`, skip 3c. On pass: increment `review_streak` counter.

**3c. Expensive checks** (when applicable) -- Run expensive checks only when:
- This is a close attempt (`close_rule` requires them and objective appears met), OR
- Round number hits `evaluation.expensive_interval` (default 10).

Record per-gate verdict. On first fail: short-circuit, aggregate status = `fail`.

**3d. Human review** (if `close_rule` includes `human_required`) -- Queue for human review. Proceed to Evaluate with current state.

Capture full stdout/stderr to `artifacts/round-{N}/`.

Write an evidence manifest to `artifacts/round-{N}/manifest.json`:

```json
{
  "round": 4,
  "commit": "m0n1o2p",
  "checks": [
    {"name": "typecheck", "cost": "cheap", "exit_code": 0},
    {"name": "lint", "cost": "cheap", "exit_code": 0},
    {"name": "review", "kind": "review", "verdict": "pass"},
    {"name": "e2e", "cost": "expensive", "exit_code": 0}
  ],
  "artifacts": ["stdout.log", "attempted.patch"],
  "diff_stat": "+42 -17 across 3 files"
}
```

Aggregate `verification.status`:
- All gates pass -> `pass`
- Any gate fail -> `fail`
- Review escalated -> `blocked`

When expensive checks are skipped in a round, their names do not appear in `verification.gates`. They were not applicable, not skipped.

### 4. Evaluate

Apply the Decision Rules (see Evaluation Rules below).

After revert (`reverted`): verify the revert itself does not break baseline.

### 5. Record

Append one event to `state.jsonl`. Required fields: `event`, `task_id`, `ts`, `round`, `commit`, `verification.status`, `verification.gates`, `verification.review_streak`, `evaluation.result`, `evaluation.reason` (when `reverted` or `blocked`), `summary`.

Before writing context.md, route information correctly:
- Contract/threshold -> `config.yaml`
- Current status/next action -> `context.md` Current State / Working Memory / Next Steps
- Durable reusable fact -> `context.md` Durable Notes
- Raw output/proof -> `artifacts/`

Update context.md:
- Overwrite Current State
- Curate Working Memory
- Rewrite Next Steps
- Append to Decisions if a new decision was made
- Update Durable Notes if this round produced cross-round knowledge

### 6. Entropy Check

#### Stop Conditions (check in order)

1. **Done**: latest round result is `done` (objective met + `close_rule` satisfied).
2. **Budget exhausted**: current round >= `termination.stop_guards.budget.max_rounds`. (`-1` disables.)
3. **Stagnation**: last N rounds all `reverted` (N = `termination.stop_guards.stagnation.rounds`, default 5).
4. **Blocked**: result is `blocked`. Pause for human input.

#### Doom Loop Detection

If the same check fails with the same error signature N times (N = `entropy.doom_loop_threshold`, default 3):

1. Invoke `/systematic-debugging` to diagnose root cause.
2. If diagnosis produces a viable new approach: pivot, record failed approach in Durable Notes, continue.
3. If diagnosis does not produce a viable approach: pause and ask the human.

Optional config hint (`entropy.doom_loop_action`): `reread_and_pivot` (default), `codex_rescue`, `ask_human`. These are preferences for escalation style, not a mandatory chain.

#### Continue

If no stop condition fires, return to step 1 (Propose).

Do not stop on a successful round while non-blocked work remains.

---

## Checks Model

Every entry in `checks[]` has exactly one classification: `cost: cheap`, `cost: expensive`, or `kind: review`. A check must not have both `cost` and `kind`. Execution order: cheap (3a) → review (3b) → expensive (3c).

### Probe Checks (optional)

A `probe` is a cheap, task-specific command check that answers "did this specific change help?" Mark with `probe: true`. Runs alongside cheap checks in step 3a. Probe failure is informational in `satisfy` mode and does not force revert. To make a probe mandatory, configure it as a regular cheap check instead.

When `objective: optimize` and no probe exists, note the gap in context.md Working Memory.

### Review Gate Rules

1. Single reviewer cannot close a task. A review pass contributes to `review_streak` only.
2. Evidence must contain mechanically checkable anchors (file:line, test name). No anchors produces `blocked` (escalation).
3. When the repo has runnable deterministic checks, review cannot be the sole check.
4. `/critique` is the review engine.
5. `review_streak`: consecutive rounds where the review gate passed. Resets to 0 on review fail or any `reverted` round. On resume, restore from `verification.review_streak` in the last `state.jsonl` event (state.jsonl is authoritative over context.md).

---

## Evaluation Rules

Verification asks "is this candidate safe?" Evaluation asks "does it improve the frontier?"

### Evaluation Results

Four values with an optional `reason` field:

| Result | Meaning | Typical reasons |
|---|---|---|
| `kept` | Change verified and frontier improved or held | -- |
| `reverted` | Change rejected and rolled back | `gate_failed`, `crash`, `hook_blocked`, `below_threshold` |
| `blocked` | Cannot proceed without external input | `escalation` |
| `done` | Objective met and `close_rule` satisfied | -- |

`baseline` appears only in `baseline_recorded` events, not in `round_completed` events. The four round-level results are `kept`, `reverted`, `blocked`, `done`.

### Decision Rules (apply in order)

1. Verification failed -> `reverted` (reason: `gate_failed`)
2. Verification command crashed -> `reverted` (reason: `crash`)
3. Pre-commit hook blocked -> `reverted` (reason: `hook_blocked`)
4. Verification blocked (review escalated) -> `blocked` (reason: `escalation`)
5. Verified but improvement below threshold -> `reverted` (reason: `below_threshold`; optimize only with `min_delta`)
6. Objective met and `close_rule` satisfied -> `done`
7. Verified and frontier improved or held -> `kept`

### close_rule

Auto-inferred from the checks list unless explicitly provided in config:

| Condition | Inferred close_rule |
|---|---|
| checks list contains at least one `cost: expensive` check | `expensive_pass` -- all expensive checks must pass |
| No expensive checks, review gate configured | `review_streak(N)` -- N consecutive review passes (N from config, default 3) |
| No expensive checks, no review gate | `all_pass` -- all cheap checks pass and acceptance criteria met |
| Config has `human_required: true` | Additionally requires human approval regardless of above |

`review_streak(N)` is satisfied when `verification.review_streak >= N` in the current round — i.e., N consecutive rounds where the review gate passed without any intervening revert.

Optional in config.yaml. When omitted, inferred from checks list. When provided, overrides inference.

### Metric Interpretation

- `increase`: higher is better
- `decrease`: lower is better
- `none`: informational only; rely on acceptance criteria

### Volatile Metrics (optimize objective only)

When `objective: optimize` and `metric.volatile: true`: run measurement N times (default `metric.samples: 3`), take median. Improvement must exceed `metric.min_delta` for `metric.confirmation_runs` consecutive rounds to be trusted.

---

## Rollback Safety

- `revert_commit` (default): `git revert HEAD --no-edit`
- `reset_to_last_pass`: `git reset --hard <last_kept_commit>` (destructive)
- `preserve_failed_experiments: true`: save diff to `artifacts/round-{N}/attempted.patch` before reverting

## Session Boundary

When ending a session mid-loop:
1. Complete the current round (do not leave a half-committed state).
2. Update context.md with full current state.
3. If mid-verify, record as `reverted` (reason: `crash`) and revert.
4. Report: current round, best result, how to resume (`/harness run`).
