# Run

initialize -> [propose -> cleanup -> commit -> verify -> evaluate -> investigate? -> adapt -> record -> replan-check -> stop-check]* -> disposition

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

- `Current State`: **generated from state.jsonl + plan.yaml at Record and lifecycle boundaries**. `phase` and `round` always from ledger; `active_milestone`, `active_approach`, `version`, `current_objective`, `best_result` reconciled against ledger (if drift, ledger wins). Generation mapping by last event:

  | Last event | phase | round | active fields | best_result |
  |---|---|---|---|---|
  | no events (empty ledger) | `setup (complete)` | `0 / -` | from plan.yaml if exists, else pending | `baseline not recorded yet` |
  | `baseline_recorded` | `run` | `0 / propose` | from plan.yaml | baseline summary |
  | `round_completed` | `run` | `N / <eval result>` | from controller | latest kept/done summary |
  | `harness_stopped` | `stopped` | `N / stopped` | last controller | last kept/done summary |
  | `task_disposed` | `disposed` | `N / stopped` | last controller | disposition summary |
  | `task_completed` (legacy) | `done` | `N / done` | last controller | completion summary |

  `best_result` for satisfy: latest `kept|done` round summary. For optimize: summary of event with current `best_value`.

- `Working Memory`: manual, prune every round
- `Durable Notes`: manual, persist across rounds
- `Decisions`: manual, append-only
- `Next Steps`: manual during active run; **generated from ledger after stopped/disposed**

### `state.jsonl`

Append-only. `metric` field present only for optimize. Add `evaluation.reason` when `reverted|escalated`.

```json
{"event":"baseline_recorded","task_id":"…","ts":"…","round":0,"commit":"<sha>","verification":{"status":"pass","gates":{"<check>":"pass"}},"evaluation":{"result":"baseline"},"metric":{"value":0,"delta":0},"summary":"…"}
```

```json
{"event":"round_completed","task_id":"…","ts":"…","round":1,"commit":"<sha|null>","verification":{"status":"pass|fail|escalated","gates":{"<check>":"pass|fail|escalated"}},"evaluation":{"result":"kept|reverted|escalated|done"},"controller":{"version":1,"milestone_id":"M1","approach_id":"A1","approach_decision":"continue|demote|failed|complete|task_done|blocked","strategy_signal":"none|all_approaches_exhausted|new_constraint","next_milestone_id":"M2|null"},"metric":{"value":0,"delta":0},"summary":"…"}
```

```json
{"event":"harness_stopped","task_id":"…","ts":"…","round":3,"reason":"done|escalated|max_rounds|stagnation","summary":"…"}
```

```json
{"event":"task_disposed","task_id":"…","ts":"…","round":3,"disposition":"merged|kept|discarded","summary":"…"}
```

```json
{"event":"strategy_updated","task_id":"…","ts":"…","round":3,"version":2,"reason":"bootstrap|replan|reopen","trigger":"<freetext>","active_milestone_id":"M2","summary":"…"}
```

Invariants:

- Exactly 1 `baseline_recorded`, at `round: 0`
- `round_completed.round` strictly increments by 1
- `harness_stopped`, if present, appears exactly once and immediately after the final `round_completed`
- `task_disposed`, if present, appears exactly once and must be the last event; must follow `harness_stopped`
- `kept|done` requires `verification.status: pass`
- Milestone advance is encoded in `controller.next_milestone_id` (set when `approach_decision = complete`, null otherwise); `strategy_updated` is reserved for non-linear transitions (bootstrap, replan, reopen)
- `strategy_updated` may follow `round_completed` or `baseline_recorded` for non-linear transitions; `version` strictly increments on `reason: replan`
- `trigger` is freetext describing why the strategy changed; recommended values: `initial`, `replan`, `reopen`, `new_constraint`. Routing uses structured fields, not trigger matching
- final task completion is encoded as `evaluation.result: done` with `controller.approach_decision: task_done`; do not encode terminal completion as `kept + strategy_signal: done`
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
3. Regenerate `context.md` Current State from ledger (see generation mapping)
4. Rewrite `Next Steps`, point to round 1

`baseline_recorded` anchors the starting point; it does not mean the task is complete.

### Bootstrap Strategy (fresh + recovery)

After Baseline passes:

1. Read the pending seed in `plan.yaml`.
2. Validate `plan_sources[]`, `planning_context`, ids/statuses, `source_refs`, and verifiable `exit_criteria`. If contract-blocking `open_questions` remain, stop and return to setup repair.
3. If `milestones[]` is populated: activate the first pending milestone and top-ranked candidate approach; add approaches/steps only where sparse. Preserve valid imported rankings/source refs; record any split, reorder, insert, or demotion in `strategy_updated.summary`.
4. If `milestones[]` is empty: generate a legacy seed from the goal, mark source as `legacy_repair`, then activate it.
5. Write `plan.yaml` with `strategy.status: active`, `active_milestone_id: M1`
6. Emit `strategy_updated(reason=bootstrap, trigger=<dominant source kind or legacy_repair>)`
7. Regenerate `context.md` Current State from ledger + plan.yaml

## Round Lifecycle

### Propose

- Read `plan.yaml`: take `active_milestone_id` and its `active` approach
- If approach has `steps[]` populated and `current_step < len(steps)`: scope the round to `steps[current_step]`
- If `steps[]` exhausted (`current_step >= len(steps)`) or empty: scope to the approach `hypothesis`
- Scan `Durable Notes` for `[dead-end]`, `[constraint]` relevant to this milestone/approach
- Non-trivial changes: converge on approach first; invoke `/brainstorming` when needed
- When expanding a step into a round, follow `references/plan.md` step content standards (file mapping, intended change, verification commands, no-placeholder rules)
- Obey `task.protocol` and `execution_policy` (`dangerous_commands` require human approval; `secret_patterns` never read or staged)
- When `task.protocol` is `tdd_required` or `tdd_preferred`, or the current round writes tests, fixtures, or mocks, load `references/tdd-discipline.md`
- When `task.protocol = tdd_required` and the round changes production behavior, execute the RED proof before patching and record the failing reproduction, proof command, and failure evidence in `Decisions` or `artifacts/round-{N}/manifest.json`
- When `task.protocol = tdd_preferred`, do the same by default; for pure docs, config, or other non-behavior rounds, record the skip reason in `Decisions` before generating the patch
- Keep the implementation boundary minimal after RED evidence is recorded
- One atomic round at a time; if the description needs "and" to explain, split into multiple rounds
- **Post-revert guard:** if the previous round was `reverted`, this round's proposal must state the single hypothesis being tested and cite evidence from the failed round. Without both, investigate first — do not generate a patch
- **Reviewer-driven fix guard:** Treat `/critique`, `/fanout`, and reviewer fixes as advisory. Before patching accepted or gate-relevant findings, record a compact acceptance map in `Decisions` or the round note: `F-001 -> classification -> actionability -> chosen_action`. Auto-implement only `required_fix`; apply `optional_trim` only when deleting, narrowing, inlining, or reusing; `evidence_note` and `defer` cannot expand implementation or create code/test/schema/UI surfaces. Near-blocking does not block `done` unless the verdict is `needs_escalation`, exit criteria/checks require it, or source verification promotes it to current production risk.

### Cleanup

- After `/critique`, `/fanout`, or reviewer fixes, compare diff to goal/non-goals and acceptance map; keep only explicit requirements or `required_fix`, otherwise delete, narrow, inline, or reuse.
- Skip for small changes (< ~20 LOC, <= 3 files, no prior-round revert)
- Re-read diff, check reuse / simplicity / efficiency

### Commit

- Stage only `boundary.mutable`
- commit message: follow Conventional Commits (`<type>(<scope>): <subject>`). Choose `type` and `scope` from actual change content, not from harness metadata. Do not embed round numbers in the subject.
- hook blocked: save patch to `artifacts/round-{N}/`, reset to pre-round HEAD, mark as `reverted` (reason: `hook_blocked`)

### Verify

Execute by `cost` group: `cheap -> medium -> expensive`, in list order within each group. Any check fail short-circuits the current group and all higher-cost groups.

Checks with `action` starting with `/` are dispatched as skill calls; verdict written to `verification.gates` (`pass` / `fail` / `escalated`). Map skill verdict `needs_escalation` to gate value `escalated`.

Run checks exactly as written in `config.yaml`. If a check's command, order, or membership changes, update `config.yaml` before Verify runs.

Evidence: stdout/stderr + `artifacts/round-{N}/manifest.json`. Manifest verification entries list only executed checks, and each `command` matches the configured `action`. A gate passes only from its configured action on the current commit, or a same-commit manifest entry with the same action and unchanged target/scope; ad hoc/manual checks are evidence notes, not gate results. Short-circuited checks are omitted. Any other skip needs an explicit recorded reason, or the round escalates as contract drift.

### Evaluate

First match wins:

1. Any failure (check fail / crash / hook blocked / timeout) -> `reverted` (reason field distinguishes)
2. verification escalated -> `escalated`, pause for human
3. `optimize` and `metric.delta` is non-null and < `min_delta` -> `reverted` (reason: `below_threshold`)
4. objective met -> `done` only when the task objective and active milestone `exit_criteria` are satisfied and required checks pass (checks passing alone -> `kept`; optimize also needs target reached)
5. otherwise -> `kept`

**Revert post-actions (only when reverted):**

Rerun cheap checks to confirm baseline is intact.

Rollback: `revert_commit` = `git revert HEAD --no-edit`; `reset_to_last_pass` = `git reset --hard <last_kept_commit>` (requires human approval). When `preserve_failed_experiments: true`, save `artifacts/round-{N}/attempted.patch` before revert.

**Recording semantics (all outcomes):**

`round_completed.commit` = current HEAD SHA at time of recording (after any rollback). For pre-commit failures, `commit` = `null`. `last_kept_commit` = `commit` of the most recent `round_completed` where `evaluation.result` is `kept` or `done`; falls back to `baseline_recorded.commit` if none exists.

**Metric runtime (optimize):**

- `volatile: true`: rerun `N-1` additional times within the round, take median
- `metric.delta`: increase = `value - best_value`, decrease = `best_value - value`
- `best_value`: best `metric.value` among baseline and historical `kept` events; frontier updated only on `kept`
- `target`: increase = `best_value >= target`, decrease = `best_value <= target`
- `reading` cannot yield a unique float -> `escalated` (reason: `escalation`)

### Investigate (conditional)

Runs between Evaluate and Adapt only when `evaluation.result = reverted` and any of these hold:

- Same step or failure family reverted a second time
- `failure_scope` is ambiguous (cannot confidently classify as `execution`)
- Failure spans multiple components or a deep call chain
- Error symptoms are migrating across rounds

Skip when cause is self-evident: `hook_blocked`, obvious typo/wrong file, `below_threshold` in optimize mode.

Produce 5 fields before entering Adapt:

1. `observed_failure` — which check failed and the exact error
2. `reproduction` — minimal command or steps to trigger the failure
3. `suspected_layer` — where in the call chain the bad value originates
4. `working_vs_broken_diff` — what differs from a working path or reference
5. `single_hypothesis` — one falsifiable explanation for the failure

Write these to `artifacts/round-{N}/investigation.md` and reference in `Durable Notes`. Adapt uses this investigation to classify `failure_scope` with evidence, rather than defaulting to `execution` on ambiguity.

### Adapt

Translate round verdict into plan-level actions per `references/plan.md` Adapt Decision Table, then update `plan.yaml`.

Run-layer invariants: `evaluation.result = done` pairs with `approach_decision = task_done`; milestone completion uses `kept + approach_decision = complete`; `approach_decision` maps to approach status (`failed`, `blocked`, `done`). Stop-check handles escalated/blocked outcomes.

### Record

Update `state.jsonl` (append event) and `context.md` (regenerate Current State from ledger; update manual sections). Write `artifacts/round-{N}/manifest.json` when the round executed checks, changed files, had a skip/short-circuit, or is decision-only. Omit the round artifact dir only for rounds where state.jsonl summary is the sole record needed.

Refresh native progress mirror (best effort; failures never affect routing, evaluation, or stop conditions).

Decision-only rounds are allowed when no code or config change is needed and `HEAD` stays unchanged. They still write `artifacts/round-{N}/manifest.json` with `changed_files: []` and a note saying whether verification reran or was reused from the prior unchanged commit.

## Stop Conditions

Checked after each round's Record, first match wins:

1. `done`
2. `escalated`
3. round >= `termination.max_rounds` (`-1` disables)
4. stagnation:
   - satisfy: last N rounds all `reverted` and replan-check has run
   - optimize: last N metric samples did not advance frontier

`kept` is not a reason to stop.

When a stop condition fires:

1. Append `harness_stopped` to `state.jsonl` with the matching `reason`
2. Regenerate `context.md` Current State from ledger (phase: stopped); rewrite `Next Steps` to disposition prompt
3. Present disposition via AskUserQuestion with options: merge (worktree only), keep, discard

### Doom loop (tactics-level)

Same check fails with same error pattern N times (`doom_loop_threshold`) within the current milestone:

1. Record pattern in `Durable Notes` (`[dead-end][M#/A#]`)
2. `/fanout -a codex:6 -m sample` for independent diagnosis with GSA synthesis (in Claude Code, load `/codex-exec` first)
3. Fanout candidates become new approaches in current milestone (initial `score: 40`, `status: candidate`)
4. Subsequent rounds consume candidates through normal lifecycle
5. All exhausted -> `strategy_signal = all_approaches_exhausted` -> triggers replan

### Architecture escalation (strategy-level)

Separate from doom loop. Triggers when the current milestone accumulates 3+ reverted rounds covering 2 or more distinct failure families, or when each successive fix exposes coupling in a different location. This pattern suggests the problem is architectural, not tactical.

Action: `escalated` for human architecture review. Do not continue with fanout or replan — the milestone's framing may be wrong.

### Replan Protocol

**Triggers** (checked at `replan-check`, after Record):

1. `controller.next_milestone_id` set: activate next pending milestone; generate tactics for it. Not a structural replan; does not bump `plan_version`.
2. `all_approaches_exhausted`: current milestone has no `active|candidate` approaches.
3. `new_constraint`: `[constraint][strategy]` in Durable Notes invalidates milestone order or feasibility.
4. `stagnation`: current milestone has N consecutive rounds with no `kept` progress across multiple approaches.

**Not triggers**: single `reverted`, single approach switch, `[code-map]` without order implications, doom-loop candidates not yet consumed.

**Replan execution**:

1. Freeze all `status: done` milestones. Never rewrite completed prefix.
2. Suffix-only: rewrite active + pending milestones. Allowed ops: split, reorder, drop, insert prerequisite.
3. Preserve `planning_context` and `source_refs` where intent remains valid; supersede or add a plan source only for explicit external re-import or new strategy evidence.
4. Generate 2-3 ranked approaches for affected milestones.
5. `strategy.version += 1`, emit `strategy_updated(reason=replan, trigger=<matching trigger>)`. When the trigger is `new_constraint` or `stagnation`, cite the triggering check, artifact path, or Durable Note tag in `summary`.

**Anti-thrash**: at least 1 completed round between replans; only hard triggers (`new_constraint`, `escalated`) bypass cooldown.

## Disposition

Collect disposition via AskUserQuestion (not inline text). Options: `merge` (worktree only), `keep`, `discard`. After user selects:

- **`merged`** (worktree mode only): organize changes into logical commits on `task.base_branch`, delete branch and worktree. See **Commit organization** below.
- **`kept`**: preserve final code state; worktree mode reports path + branch, in-place preserves current state
- **`discarded`**: worktree mode deletes branch + worktree; in-place reverts to `baseline_recorded.commit` (**requires explicit user confirmation**)

`.harness/` is always preserved.

### Commit organization

**Grouping** — milestone → commit:

1. Diff worktree against `base_branch`
2. Map changed files/hunks to their milestone (`plan.yaml` scope and step file-lists as primary signal)
3. Cross-cutting changes (config, deps, shared types) attach to the milestone that introduced them
4. One milestone = one commit when cohesive. Split only for clearly separable concerns within a milestone

Single-milestone tasks produce one commit. Present an ordered commit plan (`subject — M#`) for user confirmation or regrouping. Then reset worktree to `base_branch`, apply commits in order (Conventional Commits), and fast-forward `base_branch`.

After applying: append `task_disposed` to `state.jsonl`; regenerate `context.md` Current State from ledger (phase: disposed, Next Steps: cleared).
