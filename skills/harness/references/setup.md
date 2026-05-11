# Setup

## Step 1: Clarify the Goal

- Goal unclear -> ask before writing task state
- Broad intent -> narrow to a verifiable bug, feature, coverage target, metric target, or other decidable outcome

## Step 2: Plan Intake and Seed Strategy

Setup must produce a plan seed before writing task state. Default to embedded `/brainstorming` unless the task is **very simple**; plan-first does not mean always run long brainstorming.

Very simple means all are true: one obvious implementation path; exact target/outcome; small unambiguous mutable boundary; clear existing verification or smallest repro; no external plan, multi-step sequence, unresolved dependency, or architecture/UX/API/schema/workflow/product choice.

Collect sources:

1. **External plan**: if the user provides a plan path or inline plan, import it as advisory input.
2. **Embedded brainstorming**: run for every non-very-simple task; include imported external plans in its context when present.
3. **Setup synthesis**: only for very-simple work; create a minimal seed plan inline (usually 1 milestone, 1 approach).

External plan import:

- Accept only explicit `.md`, `.yaml`, or inline plan input; no auto-discovery.
- Snapshot under `artifacts/setup/`; record path, digest, summary in `plan.yaml.plan_sources[]`.
- Extract goal/done condition, boundary hints, checks, milestones, approaches, non-goals, constraints, assumptions, decisions.
- External files are snapshots, not live links; later changes require explicit re-import/replan.

**Dispatch preamble:**

```
mode: embedded
task_id: <task_id or provisional task slug>
caller: /harness setup
goal: "<raw user goal>"
plan_sources: [<external plan summaries, if any>]
```

**Normalize selected source:** snapshot the full external or embedded plan under `artifacts/setup/`; record source id, digest, and summary in `plan.yaml.plan_sources[]`; map `goal` -> `task.description`, `done_when_hint` -> `task.done_when`, `boundary_hints` -> `boundary`, `protocol_hint` -> `task.protocol`, `check_hints` -> `checks[]`, `planning_context` -> `plan.yaml.planning_context`, and `milestones` -> pending `plan.yaml.milestones[]`.

All values remain advisory until Step 4 finalizes them. The user confirms or overrides each field during contract finalization.

**Milestone handoff:** `setup:new` writes a pending seed; run Bootstrap activates/enriches it and records deviations.

## Step 3: Plan-Guided Codebase Discovery

Use the seed plan to inspect named files, implied modules, dependencies, checks, risks, open questions, relevant tests, build/lint/CI, docs/guidance, verification suites, and golden commands. Tighten `boundary`, `task.done_when`, `task.protocol`, `checks[]`, `evaluation`, and milestone exit criteria before asking. If code facts invalidate the seed, revise it before Step 4; use embedded brainstorming again unless the repaired task is very simple.

## Step 4: Finalize the Contract

Ask before writing the contract. Default values may be suggested upfront, but fields that change behavior must not be guessed. If an unresolved answer would change `boundary`, the observable done condition, `checks`, `evaluation`, `termination`, `rollback`, or `execution_policy`, stay in setup.

Base questions on the normalized plan plus code discovery. Batch independent questions; serialize only when answer A changes question B. Do not ask code-answerable questions.

Finalize in this order:

1. `boundary.mutable` and `boundary.immutable`
2. Observable done condition for the task objective (`task.done_when`, separate from checks)
3. `task.protocol` (default `direct`; when behavior-changing work can be proven with a failing test or the smallest reproducible script, ask the user whether to use `tdd_preferred` or `tdd_required`)
4. `checks[]`
5. `evaluation.objective` (discrete acceptance defaults to `satisfy`; metric improvement defaults to `optimize`)
6. `evaluation.metric.*` (only when objective is `optimize`; target defaults to unset, optimize continuously until budget or stagnation)

### Checks contract

Each check shape: `{name, kind, action, cost: cheap|medium|expensive}`.

- `name`: unique identifier for the check, referenced by `verification.gates` and `metric.sample_check`
- `kind`: `command` for shell commands, `review` for skill-backed review gates such as `/critique`
- `action`: command or skill to execute (e.g., shell command, `/critique -a gpt-5.5:6`)
- `cost`: determines execution order (`cheap -> medium -> expensive`)
- `checks[]` must contain at least 1 check before setup completes
- `checks[]` is the runtime verification contract, not the done condition. After setup, change the command, order, or membership in `config.yaml` before Verify uses the new shape.

Example: `{name: unit-tests, kind: command, action: "pytest tests/", cost: cheap}`

User-requested named skill gates (for example `/critique` or `$critique`) phrased as check/gate/must-pass/before-done/acceptance must be written as `checks[].action` starting with that skill command; prose/manual-review notes do not satisfy the gate. If the mention is only advisory, ask whether it belongs outside harness checks.

When deterministic checks cannot adequately verify correctness (for example semantic behavior, design intent, cross-cutting concerns), suggest adding `/critique` as a check. It runs during Verify like any other check; place it before expensive checks when useful.

### Optimize contract

When `evaluation.objective: optimize`, setup must clarify the optimize contract.
Do not mix correctness checks with the optimize metric: correctness checks verify whether the implementation is acceptable; the optimize metric is read from experiment-related outputs only after `sample_check` runs.

Must finalize:

- `evaluation.metric.sample_check`: which check is the sampling point
- `evaluation.metric.reading`: how to read a number from experiment-related outputs
- `evaluation.metric.direction`: `increase|decrease`
- `evaluation.metric.min_delta`
- `evaluation.metric.target` (optional)
- `evaluation.metric.volatile` and `evaluation.metric.samples` (as needed)

`reading` can be a command or precise executable steps; the key is determinism, not enforcing a fixed stdout format.

## Step 5: Write Task State

Choose the `.harness/` location before creating task state:

- Single-repo task -> confirm a task-state location that does not create tracked repo surface; prefer a repo-adjacent or otherwise established local location over inventing a new in-worktree state dir
- Multi-repo or cross-worktree task -> place `.harness/` under the nearest common parent directory of the touched repos/worktrees
- Do not default to the currently open repo just because setup is being run from there

### `setup:new`

1. Confirm the directory containing `.harness/`; create if missing
2. Generate `task_slug`, allocate `task_id`, create `.harness/tasks/<task_id>/` and `artifacts/setup/`
3. If inside a git repo, prefer `.git/info/exclude` for `.harness/` and `.worktree/`
4. Write `config.yaml`: `task.id = <task_id>`, `task.description = <goal>`, `task.done_when = <observable done condition>`, `task.base_branch` = current branch, remaining fields per Appendix A skeleton. No `<...>` placeholders
5. Write `context.md` initial snapshot:
   - `phase: setup (complete)`
   - `round: 0 / -`
   - `current_objective: "<goal>"`
   - `best_result: "baseline not recorded yet"`
   - `last_action: "setup completed; ready for run preflight"`
   - `Working Memory`: empty
   - `Durable Notes`: empty
   - `Decisions`: empty
   - `Next Steps`: `Run preflight; on success, record baseline and enter round 1.`
6. Write `state.jsonl`: initialize as empty file
7. Write `plan.yaml`: use `references/plan.md`; include `plan_sources[]`, `planning_context`, and seeded `milestones[]` from the normalized plan. Set `strategy.status: pending`, `active_milestone_id: null`. Empty `milestones[]` is only allowed for legacy repair when no plan source exists.

### `setup:repair`

Reuse existing `task_id` and task directory. Do not delete existing files or artifacts.

- `config.yaml`: fill in missing fields, no `<...>` placeholders
- `plan.yaml`: if missing and `state.jsonl` has recorded events, build a single-milestone single-approach active plan from the task goal and current code state, emit `strategy_updated(reason=bootstrap, trigger=legacy_migration)`, then hand off to run; if missing/empty and no events, create a pending seed plan from plan sources or setup synthesis; if unstarted and missing provenance, add a `setup_synthesis` source or re-import explicit external/embedded source; if present but `version` doesn't match latest `strategy_updated.version`, restore active pointers from latest `round_completed.controller` and `strategy_updated` events then trigger an immediate replan on entering run; if structural corruption is unrecoverable, escalate
- `context.md`: no run events -> reset to `setup:new` initial snapshot; has run events -> fill in missing fields only, do not reset
- `state.jsonl`: create as empty file only if missing; leave alone if it already exists

## Step 6: Hand Off to Run

Setup is done when all of these hold:

- Task state files are complete (including `plan.yaml`)
- `config.yaml` has no unresolved placeholders
- Contract is complete enough for run

After setup handoff, run initialization activates or enriches the pending seed plan. See `references/run.md` Bootstrap Strategy section.

## Appendix A: Canonical `config.yaml` Skeleton

```yaml
task: { id: "<task_id>", description: "<goal from user>", done_when: "<observable done condition>", protocol: direct, base_branch: "<branch>" }  # direct|tdd_preferred|tdd_required; direct = no TDD overlay, preferred = fail-first by default, required = fail-first required for behavior-changing implementation rounds

boundary: { mutable: [], immutable: [] }  # repo-root-relative path list; directories mean the full subtree; no globs

checks: []  # {name, kind, action, cost: cheap|medium|expensive}; execution order cheap -> medium -> expensive

evaluation:
  objective: satisfy          # satisfy|optimize

termination:
  max_rounds: 50              # hard round cap; -1 disables
  doom_loop_threshold: 3      # same failure pattern N times triggers fanout diagnosis
  stagnation_rounds: -1       # consecutive no-progress rounds cap; -1 disables

rollback:
  strategy: revert_commit     # revert_commit|reset_to_last_pass (latter requires human approval)
  preserve_failed_experiments: true  # save patch evidence before rollback

execution_policy:
  dangerous_commands: []      # substring blacklist; matches require human approval
  secret_patterns: [".env", "*.pem", "credentials.*"]  # matches are never read or staged
  dependency_install: prompt  # allow|deny|prompt
```

When `evaluation.objective: optimize`, add:

```yaml
evaluation:
  objective: optimize
  metric:
    sample_check: "<metric_check_name>"
    reading: "<metric_reading>"
    direction: increase       # increase|decrease
    volatile: false
    samples: 3               # only read when volatile=true
    min_delta: 0
    target: null              # optional; unset means optimize continuously until budget or stagnation
```
