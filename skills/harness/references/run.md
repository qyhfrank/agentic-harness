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
- task_intent:
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

Mutation:

- `Current State`: **generated from state.jsonl + plan.yaml at Record and lifecycle boundaries**. `phase` and `round` from ledger; active fields and `best_result` reconciled against ledger (ledger wins). Mapping by last event:

  | Last event | phase | round | active fields | best_result |
  |---|---|---|---|---|
  | no events (empty ledger) | `setup (complete)` | `0 / -` | from plan.yaml if exists, else pending | `baseline not recorded yet` |
  | `baseline_recorded` | `run` | `0 / propose` | from plan.yaml | baseline summary |
  | `round_completed` | `run` | `N / <eval result>` | from controller | latest kept/done summary |
  | `harness_stopped` | `stopped` | `N / stopped` | last controller | last kept/done summary |
  | `task_disposed` | `disposed` | `N / stopped` | last controller | disposition summary |
  | `task_completed` (legacy) | `done` | `N / done` | last controller | completion summary |

  `best_result` for satisfy: latest `kept|done` round summary. For optimize: summary of event with current `best_value`.

- `Working Memory`: current handoff pointers; no per-round commit/artifact catalog.
- `task_intent`: compact `planning_context.motivation`, else task description.
- `Durable Notes`: constraints, dead ends, code maps, durable decisions; no round history or artifact/check bodies.
- `Decisions`: append-only runtime decisions; no artifact-owned finding/check bodies.
- `Next Steps`: regenerate from ledger/plan at lifecycle boundaries; after stopped/disposed, ledger-generated.
- `context.md`: only sections above unless this file adds one; evidence = path + one-line summary.

### `state.jsonl`

Append-only. `metric` field present only for optimize. Add `evaluation.reason` when `reverted|escalated`.

```json
{"event":"baseline_recorded","task_id":"…","ts":"…","round":0,"commit":"<sha>","verification":{"status":"pass","gates":{"<check>":"pass"}},"evaluation":{"result":"baseline"},"metric":{"value":0,"delta":0},"summary":"…"}
```

```json
{"event":"round_completed","task_id":"…","ts":"…","round":1,"commit":"<sha|null>","verification":{"status":"pass|fail|escalated","gates":{"<check>":"pass|fail|escalated"}},"evaluation":{"result":"kept|reverted|escalated|done"},"controller":{"version":1,"milestone_id":"M1","approach_id":"A1","approach_decision":"continue|demote|failed|complete|task_done|blocked","strategy_signal":"none|all_approaches_exhausted|new_constraint","next_milestone_id":"M2|null"},"metric":{"value":0,"delta":0},"summary":"…"}
```

Compact events:
- `harness_stopped`: `{event, task_id, ts, round, reason: done|escalated|max_rounds|stagnation, summary}`
- `task_disposed`: `{event, task_id, ts, round, disposition: merged|kept|discarded, summary}`
- `strategy_updated`: `{event, task_id, ts, round, version, reason: bootstrap|replan, trigger, active_milestone_id, summary}`
- `artifact_recorded`: optional provenance only; never controls routing

Invariants:

- Exactly 1 `baseline_recorded`, at `round: 0`
- `round_completed.round` strictly increments by 1
- `harness_stopped`, if present, appears exactly once and immediately after the final `round_completed`
- `task_disposed`, if present, appears exactly once and must be the last event; should follow `harness_stopped` (missing stop = legacy repair/audit). A task is closed only when this is the ledger tail
- `kept|done` requires `verification.status: pass`
- Milestone advance is encoded in `controller.next_milestone_id` (set when `approach_decision = complete`, null otherwise); `strategy_updated` is reserved for non-linear transitions (bootstrap, replan)
- `strategy_updated` may follow `round_completed` or `baseline_recorded` for non-linear transitions; `version` strictly increments on replan
- `trigger` is freetext; recommended values: `initial`, `replan`, `new_constraint`. Routing uses structured fields, not trigger text
- final task completion is `evaluation.result: done` with `controller.approach_decision: task_done`; never `kept + strategy_signal: done`
- `verification.gates` keys match `checks[].name`; values only `pass|fail|escalated`. Semantics like `fails_closed`/`no_matches` live in manifest.
- New writes use schema events/enums, full SHAs or `null`, and `disposition: merged|kept|discarded`. Unknown events, short commits, structural aliases, missing pre-dispose stop, and `reason: reopen` are migration evidence.

### `plan.yaml`

See `references/plan.md` for canonical schema and approach lifecycle.

`current_objective` in `context.md` tracks the active milestone objective, not the global task goal.

## Initialize

### Route Detection

Locate `.harness/` and target task, read task state, classify route:

- **fresh**: no `baseline_recorded`
- **resume**: has recorded events
- **recovery**: `state.jsonl` empty but working directory shows progress beyond setup defaults
- **legacy-audit/repair**: ledger has terminal/structural aliases, `reason: reopen`, or invalid event/result/gate enums; resume only after repair or closeout warning

### Reconcile (all routes)

- dirty working dir -> [fresh: escalate] [resume/recovery: read diff, continue if aligned with task goal, otherwise escalate]
- `HEAD` ahead of most recent recorded commit -> read new commits, reconcile with state.jsonl if they look like harness rounds, otherwise escalate
- recovery: do not delete existing task files or artifacts
- **resume: if last `round_completed.evaluation.result` is `done` or `escalated`, treat as missing `harness_stopped` and escalate; no Round Lifecycle**
- **resume: if `task_disposed` lacks prior `harness_stopped`, tail is legacy terminal, or only unknown terminal-like events exist, report legacy terminal state; do not reopen disposed tasks**
- **resume: after reconcile, enter Round Lifecycle directly**

### Preflight (fresh + recovery)

- Worktree: create if missing (`.worktree/<task_slug>/`, branch = `<task_slug>`); reuse sibling worktree/branch only with continuation evidence, else unique slug. Path anomaly -> escalate
- stale `index.lock` -> delete
- scope: `boundary.immutable` paths must exist, missing -> escalate
- run configured checks from configured cwd; record cwd, repo root, HEAD in manifest. Dependency installs/build bootstrap are setup/preflight evidence, not acceptance evidence; restore lockfile/generated churn unless in `boundary.mutable`. Negative scans fail closed on command errors. Failure -> fix/rerun; unfixable -> update `context.md` (`last_action`, `Next Steps`) then escalate

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
3. If `milestones[]` populated: activate first pending milestone and top-ranked candidate approach; add approaches/steps only where sparse. Preserve valid imported rankings/source refs; record split/reorder/insert/demotion in `strategy_updated.summary`.
4. If `milestones[]` is empty: generate a legacy seed from the goal, mark source as `legacy_repair`, then activate it.
5. Write `plan.yaml` with `strategy.status: active`, `active_milestone_id: M1`
6. Emit `strategy_updated(reason=bootstrap, trigger=<dominant source kind or legacy_repair>)`
7. Regenerate `context.md` Current State from ledger + plan.yaml

## Round Lifecycle

### Propose

- Read `plan.yaml`: take `active_milestone_id` and its `active` approach
- Read intent bundle before scoping: `task.description`, `done_when`, `planning_context.motivation`, active milestone objective/rationale/exit criteria, active approach hypothesis/rationale. Rounds serve root intent, not step mechanics only.
- If approach has `steps[]` and `current_step < len(steps)`: scope to `steps[current_step]`
- If `steps[]` exhausted (`current_step >= len(steps)`) or empty: scope to approach `hypothesis`
- Scan `Durable Notes` for `[dead-end]`, `[constraint]` relevant to this milestone/approach
- Non-trivial changes: converge on approach first; invoke `/brainstorming` when needed
- When expanding a step into a round, follow `references/plan.md` step content standards (file mapping, intended change, verification commands, no-placeholder rules)
- Obey `task.protocol` and `execution_policy` (`dangerous_commands` require human approval; `secret_patterns` never read or staged)
- Load `references/tdd-discipline.md` when `task.protocol` is `tdd_required|tdd_preferred` or the round writes tests, fixtures, or mocks
- `tdd_required` + production behavior change: run RED before patching; record failing reproduction, proof command, failure evidence in `Decisions` or manifest
- `tdd_preferred`: same by default; for pure docs/config/non-behavior rounds, record skip reason in `Decisions` before patch
- Keep implementation boundary minimal after RED evidence
- One atomic round at a time; if the description needs "and" to explain, split into multiple rounds
- Sideband/new objective: classify as dependency, scope expansion, or follow-up. Default new carrier; include here only after user confirms boundary/checks/replan update.
- **Post-revert guard:** if previous round was `reverted`, proposal must state the single hypothesis and cite failed-round evidence. Without both, investigate first; no patch.
- **Reviewer-driven fix guard:** Treat `/critique`, `/fanout`, and reviewer fixes as advisory. Before patching accepted/gate-relevant findings, record compact acceptance map: `F-001 -> classification -> actionability -> chosen_action`. Auto-implement only `required_fix`; apply `optional_trim` only for delete/narrow/inline/reuse. `evidence_note` and `defer` cannot expand implementation or create code/test/schema/UI surfaces. Near-blocking blocks `done` only when verdict is `needs_escalation`, exit criteria/checks require it, or source verification promotes it to current production risk.

### Cleanup

- After reviewer fixes, compare diff to goal/non-goals and acceptance map; keep only explicit requirements or `required_fix`, otherwise delete, narrow, inline, or reuse.
- Skip for small changes (< ~20 LOC, <= 3 files, no prior-round revert)
- Re-read diff, check reuse / simplicity / efficiency

### Commit

- Stage only `boundary.mutable`
- commit message: Conventional Commits (`<type>(<scope>): <subject>`). Choose `type`/`scope` from change content, not harness metadata. No round numbers in subject.
- hook blocked: save patch to `artifacts/round-{N}/`, reset to pre-round HEAD, mark as `reverted` (reason: `hook_blocked`)

### Verify

Execute by `cost` group: `cheap -> medium -> expensive`, in list order within each group. Any check fail short-circuits the current group and all higher-cost groups.

`kind: review` checks dispatch skill calls; write verdict to `verification.gates` (`pass|fail|escalated`). Map `needs_escalation` to `escalated`. `manual_probe` results are evidence notes unless contract makes them acceptance evidence.

Before `/critique` review checks, assemble the capsule from `config.yaml`, `plan.yaml`, and current round evidence: intent bundle, boundary, relevant `planning_context` non-goals/constraints/decisions, check stage/target/base/scope/goals/focus, current round touched files, unresolved accepted findings/acceptance map.

Run checks exactly as written in `config.yaml`. If a check's command, order, or membership changes, update `config.yaml` before Verify runs.

Evidence: stdout/stderr + `artifacts/round-{N}/manifest.json`. Manifest verification lists only executed checks; each `command` matches configured `action`; each gate key equals `checks[].name`. Gate passes only from configured action on current commit, or same-commit manifest with same action and unchanged target/scope. Ad hoc/manual checks are evidence notes. Omit short-circuited checks. Any other skip needs recorded reason or escalates as contract drift.

Artifact hygiene: compact by default. Exclude binaries, DB snapshots, `node_modules`, `dist`, `__pycache__`, browser profiles, repeated screenshots/logs unless offline repro needs them. Store command/path/hash/size and compact logs; raw heavy artifacts only latest pass/latest failure with reason. Prefer incremental critique diffs, DOM/JSON UI proof, screenshot caps/contact sheets, one compact GSA artifact. `evidence.md` only for RED proof, source review, manual-probe transcript, critique acceptance maps; no manifest check copy.

### Evaluate

First match wins:

1. Any failure (check fail / crash / hook blocked / timeout) -> `reverted` (failed review gates with actionable findings are failures, not `strategy_updated(reason=reopen)`)
2. verification escalated -> `escalated`, pause for human
3. `optimize` and `metric.delta` is non-null and < `min_delta` -> `reverted` (reason: `below_threshold`)
4. objective met -> `done` only when the task objective and active milestone `exit_criteria` are satisfied and required checks pass (checks passing alone -> `kept`; optimize also needs target reached)
5. otherwise -> `kept`

**Revert post-actions (only when reverted):**

Rerun cheap checks to confirm baseline is intact.

Rollback: `revert_commit` = `git revert HEAD --no-edit`; `reset_to_last_pass` = `git reset --hard <last_kept_commit>` (requires human approval). When `preserve_failed_experiments: true`, save `artifacts/round-{N}/attempted.patch` before revert.

**Recording semantics (all outcomes):**

`round_completed.commit` = current HEAD SHA at recording time after rollback. Pre-commit failure => `null`. `last_kept_commit` = most recent `kept|done` commit, else `baseline_recorded.commit`.

**Metric runtime (optimize):**

- `volatile: true`: rerun `N-1` additional times within the round, take median
- `metric.delta`: increase = `value - best_value`, decrease = `best_value - value`
- `best_value`: best `metric.value` among baseline and historical `kept` events; frontier updated only on `kept`
- `target`: increase = `best_value >= target`, decrease = `best_value <= target`
- `reading` cannot yield a unique float -> `escalated` (reason: `escalation`)

### Investigate (conditional)

Runs between Evaluate and Adapt only when `evaluation.result = reverted` and any condition holds:

- Same step or failure family reverted a second time
- `failure_scope` is ambiguous (cannot confidently classify as `execution`)
- Failure spans multiple components or a deep call chain
- Error symptoms are migrating across rounds

Skip self-evident causes: `hook_blocked`, obvious typo/wrong file, `below_threshold` in optimize mode.

Produce 5 fields before entering Adapt:

1. `observed_failure` — which check failed and the exact error
2. `reproduction` — minimal command or steps to trigger the failure
3. `suspected_layer` — where in the call chain the bad value originates
4. `working_vs_broken_diff` — what differs from a working path or reference
5. `single_hypothesis` — one falsifiable explanation for the failure

Write to `artifacts/round-{N}/investigation.md` and reference in `Durable Notes`. Adapt uses it to classify `failure_scope` instead of defaulting ambiguous failures to `execution`.

### Adapt

Translate round verdict into plan-level actions per `references/plan.md` Adapt Decision Table, then update `plan.yaml`.

Verdict-to-decision mapping is canonical in `references/plan.md`; stop-check handles escalated/blocked outcomes.

### Record

Update `state.jsonl` and regenerate `context.md`. Write `artifacts/round-{N}/manifest.json` for checks, changes, skips/short-circuits, or decision-only rounds. Omit the round dir only when state summary is enough. After append, prior round artifacts are sealed; corrections go in later errata/manifest.

Refresh native progress mirror (best effort; failures never affect routing, evaluation, or stop conditions).

Decision-only rounds are allowed when no code/config change is needed and `HEAD` stays unchanged. They still write manifest with `changed_files: []` and whether verification reran or reused prior unchanged commit. Pure source-map/evidence-index updates belong in Durable Notes or next implementation manifest unless they change controller/strategy state, satisfy an exit gate, or user requests an audit point.

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
3. Present disposition (see Disposition)

Before `reason: done`, assert objective met, `controller.next_milestone_id == null`, and no pending non-dropped milestone remains. Defer suffix work by replan/drop/supersede or escalated stop; disposed tasks carry no hidden future work.

### Doom loop (tactics-level)

Same check fails with same error pattern N times (`doom_loop_threshold`) within the current milestone. Repeated critique failures by finding family count:

1. Record pattern in `Durable Notes` (`[dead-end][M#/A#]`)
2. `/fanout -a gpt:6 -m sample` for independent diagnosis with GSA synthesis (in Claude Code, load `/codex-exec` first)
3. Fanout candidates become new approaches in current milestone (initial `score: 40`, `status: candidate`)
4. Subsequent rounds consume candidates through normal lifecycle
5. All exhausted -> `strategy_signal = all_approaches_exhausted` -> triggers replan

### Architecture escalation (strategy-level)

Separate from doom loop. Triggers when the current milestone accumulates 3+ reverted rounds covering 2 or more distinct failure families, or when each successive fix exposes coupling in a different location. This pattern suggests the problem is architectural, not tactical.

Action: `escalated` for human architecture review. Do not continue with fanout or replan; milestone framing may be wrong.

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

Enter Disposition only after `harness_stopped`. Repo, plan, context, manifest, remote ref, tag, or workflow evidence may prove what happened, but never terminal harness state.

If normal user instructions already performed disposition outside AskUserQuestion (merge, push, tag, publish, cleanup, discard), treat verified action as disposition evidence and do not ask again. If `harness_stopped` is missing, recover through lifecycle first; external evidence only justifies missing terminal round/stop events.

Collect disposition via AskUserQuestion (not inline text) unless verified out-of-band disposition evidence already exists. Options: `merge` (worktree only), `keep`, `discard`. After user selects:

- **`merged`** (worktree mode only): organize changes into logical commits on `task.base_branch`, delete branch and worktree. See **Commit organization** below.
- **`kept`**: preserve final code state; worktree mode reports path + branch, in-place preserves current state
- **`discarded`**: worktree mode deletes branch + worktree; in-place reverts to `baseline_recorded.commit` (**requires explicit user confirmation**)

`.harness/` is always preserved.

After disposition, retain carriers for audit. Optional archival follows Verify artifact hygiene and preserves `config.yaml`, `plan.yaml`, `context.md`, `state.jsonl`, setup snapshots/pointers, and compact round summaries.

### Commit organization

**Grouping** — milestone → commit:

1. Diff worktree against `base_branch`
2. Map changed files/hunks to their milestone (`plan.yaml` scope and step file-lists as primary signal)
3. Cross-cutting changes (config, deps, shared types) attach to the milestone that introduced them
4. One milestone = one commit when cohesive. Split only for clearly separable concerns within a milestone

Single-milestone tasks produce one commit. Present an ordered commit plan (`subject — M#`) for user confirmation or regrouping. Then reset worktree to `base_branch`, apply commits in order (Conventional Commits), and fast-forward `base_branch`.

If disposition rewrites or relocates kept commits (rebase, cherry-pick, squash, regrouping, or applying onto an advanced base), previous `round_completed` entries remain historical only. Before `task_disposed`, rerun configured gates against final HEAD and refresh revision-sensitive evidence (build label, SHA, branch, deployment, or screenshot proof). Record the final merged/kept SHA in `task_disposed.summary` and `context.md`. Example: round 3 passed at `abc`; rebase yields `def`; verify and dispose `def`, leave `abc` unchanged.

After applying: append `task_disposed` to `state.jsonl`; regenerate `context.md` Current State from ledger (phase: disposed, Next Steps: cleared).
