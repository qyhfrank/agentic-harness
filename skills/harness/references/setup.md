# Setup

## Step 1: Clarify the Goal

- Goal unclear -> ask before writing task state
- Broad intent -> narrow to a verifiable bug, feature, coverage target, metric target, or other decidable outcome

## Step 2: Explore Requirements (optional)

When the goal involves creative work (new feature, behavior change, architectural decision) and any of these apply, dispatch `/brainstorming` in embedded mode before proceeding to Step 3:

- User's goal description is broad or has multiple valid interpretations
- The task requires choosing between 2+ distinct approaches
- Scope boundaries are unclear (what's in vs. out)

Skip when the goal is already narrow and verifiable (specific bug fix, mechanical refactor, metric optimization with clear target).

**Dispatch preamble:**

```
mode: embedded
task_id: <task_id>
caller: /harness setup
goal: "<raw user goal>"
```

**Consuming the output:** Brainstorming returns a YAML block (see brainstorming skill, Embedded Mode > Output Schema). Map fields to setup state:

| Brainstorming output | Setup action |
|---|---|
| `goal` | Use as refined `task.description` in Step 4 |
| `boundary_hints` | Pre-populate `boundary` defaults for Step 4 confirmation |
| `protocol_hint` | Pre-populate `task.protocol` default for Step 4 confirmation |
| `check_hints` | Pre-populate `checks[]` defaults for Step 4 confirmation |
| `milestones` | Pass to Bootstrap Strategy (run initialization) as seed decomposition |

All values remain advisory until Step 4 finalizes them. The user confirms or overrides each field during contract finalization.

**Milestone handoff:** When brainstorming produces `milestones`, Bootstrap Strategy uses them as the starting decomposition instead of generating from scratch. It may restructure (split, reorder, insert prerequisites) but should preserve the user-validated approach rankings.

## Step 3: Understand the Codebase

Read business code, understand architecture and relevant paths, scan infrastructure affecting `config.yaml` and `checks[]`. If the task turns out more complex than expected, return to Step 1 to adjust the goal with the user.

- Code: modules, entry points, dependency chains, existing test coverage related to the goal
- Test infra, Build, Lint/format, CI config
- Structure/guidance: `AGENTS.md`, `README.md`, directory layout
- Verification suites: bench, perf, e2e, integration, smoke
- Golden path commands: Makefile targets, package scripts, project scripts

## Step 4: Finalize the Contract

Ask before writing the contract. Default values may be suggested upfront, but fields that change behavior must not be guessed. If an unresolved answer would change `boundary`, the observable done condition, `checks`, `evaluation`, `termination`, `rollback`, or `execution_policy`, stay in setup.

Finalize in this order:

1. `boundary.mutable` and `boundary.immutable`
2. Observable done condition for the task objective (separate from checks)
3. `task.protocol` (default `direct`; when behavior-changing work can be proven with a failing test or the smallest reproducible script, ask the user whether to use `tdd_preferred` or `tdd_required`)
4. `checks[]`
5. `evaluation.objective` (discrete acceptance defaults to `satisfy`; metric improvement defaults to `optimize`)
6. `evaluation.metric.*` (only when objective is `optimize`; target defaults to unset, optimize continuously until budget or stagnation)

### Checks contract

Each check shape: `{name, action, cost: cheap|medium|expensive}`.

- `name`: unique identifier for the check, referenced by `verification.gates` and `metric.sample_check`
- `action`: command or skill to execute (e.g., shell command, `/critique -a gpt-5.5:6`)
- `cost`: determines execution order (`cheap -> medium -> expensive`)
- `checks[]` must contain at least 1 check before setup completes
- `checks[]` is the runtime verification contract, not the done condition. After setup, change the command, order, or membership in `config.yaml` before Verify uses the new shape.

Example: `{name: unit-tests, action: "pytest tests/", cost: cheap}`

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

## Step 6: Hand Off to Run

Setup is done when all of these hold:

- Task state files are complete (including `plan.yaml`)
- `config.yaml` has no unresolved placeholders
- Contract is complete enough for run

After setup handoff, run initialization will bootstrap the strategy (decompose goal into milestones, generate initial approaches). See `references/run.md` Bootstrap Strategy section.

## Appendix A: Canonical `config.yaml` Skeleton

```yaml
task: { id: "<task_id>", description: "<goal from user>", protocol: direct, base_branch: "<branch>" }  # direct|tdd_preferred|tdd_required; direct = no TDD overlay, preferred = fail-first by default, required = fail-first required for behavior-changing implementation rounds

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
