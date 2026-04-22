# Setup

## Step 1: Clarify the Goal

- Goal unclear -> ask before writing task state
- Broad intent -> narrow to a verifiable bug, feature, coverage target, metric target, or other decidable outcome

## Step 2: Understand the Codebase

Read business code, understand architecture and relevant paths, scan infrastructure affecting `config.yaml` and `checks[]`. If the task turns out more complex than expected, return to Step 1 to adjust the goal with the user.

- Code: modules, entry points, dependency chains, existing test coverage related to the goal
- Test infra, Build, Lint/format, CI config
- Structure/guidance: `AGENTS.md`, `README.md`, directory layout
- Verification suites: bench, perf, e2e, integration, smoke
- Golden path commands: Makefile targets, package scripts, project scripts

## Step 3: Finalize the Contract

Ask before writing the contract. Default values may be suggested upfront, but fields that change behavior must not be guessed. If an unresolved answer would change `boundary`, `checks`, `evaluation`, `termination`, `rollback`, or `execution_policy`, stay in setup.

Finalize in this order:

1. `boundary.mutable` and `boundary.immutable`
2. `task.protocol` (default `direct`; when codebase and task fit TDD, ask the user whether to use `tdd_required`)
3. `checks[]`
4. `evaluation.objective` (discrete acceptance defaults to `satisfy`; metric improvement defaults to `optimize`)
5. `evaluation.metric.*` (only when objective is `optimize`; target defaults to unset, optimize continuously until budget or stagnation)

### Checks contract

Each check shape: `{name, action, cost: cheap|medium|expensive}`.

- `name`: unique identifier for the check, referenced by `verification.gates` and `metric.sample_check`
- `action`: command or skill to execute (e.g., shell command, `/critique -a gpt-5.4:6`)
- `cost`: determines execution order (`cheap -> medium -> expensive`)
- `checks[]` must contain at least 1 check before setup completes

Example: `{name: unit-tests, action: "pytest tests/", cost: cheap}`

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

## Step 4: Write Task State

### `setup:new`

1. Confirm the directory containing `.harness/`; create if missing
2. Generate `task_slug`, allocate `task_id`, create `.harness/tasks/<task_id>/`
3. If inside a git repo, prefer `.git/info/exclude` to ignore `.harness/` and `.worktree/`, rather than editing repo `.gitignore`
4. Write `config.yaml`: `task.id = <task_id>`, `task.description = <goal>`, `task.base_branch` = current branch, remaining fields per Appendix A skeleton. No `<...>` placeholders
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
7. Write `plan.yaml`: initialize with skeleton from `references/plan.md`; set `strategy.status: pending`, empty `milestones[]`. Strategy bootstrap happens during run initialization, not setup.

### `setup:repair`

Reuse existing `task_id` and task directory. Do not delete existing files or artifacts.

- `config.yaml`: fill in missing fields, no `<...>` placeholders
- `plan.yaml`: if missing and `state.jsonl` has recorded events, build a single-milestone single-approach active plan from the task goal and current code state, emit `strategy_updated(reason=bootstrap, trigger=legacy_migration)`, then hand off to run; if missing and no events, create skeleton with `strategy.status: pending`; if present but `version` doesn't match latest `strategy_updated.version`, restore active pointers from latest `round_completed.controller` and `strategy_updated` events then trigger an immediate replan on entering run; if structural corruption is unrecoverable, escalate
- `context.md`: no run events -> reset to `setup:new` initial snapshot; has run events -> fill in missing fields only, do not reset
- `state.jsonl`: create as empty file only if missing; leave alone if it already exists

## Step 5: Hand Off to Run

Setup is done when all of these hold:

- Task state files are complete (including `plan.yaml`)
- `config.yaml` has no unresolved placeholders
- Contract is complete enough for run

After setup handoff, run initialization will bootstrap the strategy (decompose goal into milestones, generate initial approaches). See `references/run.md` Bootstrap Strategy section.

## Appendix A: Canonical `config.yaml` Skeleton

```yaml
task: { id: "<task_id>", description: "<goal from user>", protocol: direct, base_branch: "<branch>" }  # direct|tdd_required|tdd_preferred; base_branch = branch active at setup time

boundary: { mutable: [], immutable: [] }  # repo-root-relative path list; directories mean the full subtree; no globs

checks: []  # {name, action, cost: cheap|medium|expensive}; execution order cheap -> medium -> expensive

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
