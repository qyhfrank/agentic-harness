# Setup

## Step 1: Clarify the Goal

- Goal unclear -> ask before writing task state
- Broad intent -> narrow to a verifiable bug, feature, coverage target, metric target, or other decidable outcome

## Step 2: Plan Intake and Seed Strategy

Setup must produce a plan seed before writing state. Default to embedded `/brainstorming` unless the task is **very simple**; plan-first does not require long brainstorming.

Very simple means all are true: one obvious implementation path; exact target/outcome; small unambiguous mutable boundary; clear existing verification or smallest repro; no external plan, multi-step sequence, unresolved dependency, or architecture/UX/API/schema/workflow/product choice.

Collect sources:

1. **External plan**: if the user provides a plan path or inline plan, import it as advisory input.
2. **Embedded brainstorming**: run for non-very-simple tasks; include imported external plans.
3. **Investigation GSA**: prior `/fanout`/GSA plan input; snapshot as `plan_sources[].kind: investigation_gsa`.
4. **Setup synthesis**: very-simple only; create minimal seed inline (usually 1 milestone, 1 approach).

External plan import:

- Accept only explicit `.md`, `.yaml`, or inline plan input; no auto-discovery.
- Snapshot under `artifacts/setup/`; record path, digest, summary. Duplicate external files may live once in `.harness/shared/` with task-local pointer/symlink.
- For sectioned sources, record `relevant_sections`; feed only scoped extracts into brainstorming/synthesis; keep full-source digest.
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

**Normalize selected source:** snapshot selected plan; record id/digest/summary/sections in `plan.yaml.plan_sources[]`; map goal/done/boundary/protocol/check/milestone fields to config and pending milestones. `planning_context` keeps intent anchors only, not finalized contract fields or unmodified source prose.

All values remain advisory until Step 4 finalizes them. The user confirms or overrides each field during contract finalization.

## Step 3: Plan-Guided Codebase Discovery

Use the seed plan to inspect named files, implied modules, deps, risks, open questions, tests, build/lint/CI, docs/guidance, suites, and golden commands. Tighten `boundary`, `task.done_when`, `task.protocol`, `checks[]`, `evaluation`, and exit criteria before asking. If code facts invalidate the seed, revise it before Step 4; rerun embedded brainstorming unless the repair is very simple.

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

Each check shape: `{name, kind, action, cost: cheap|medium|expensive, working_directory?}`.

- `name`: unique identifier for the check, referenced by `verification.gates` and `metric.sample_check`
- `kind`: `command` for shell commands, `review` only for verdict gates such as `/critique`, `manual_probe` for probes or non-verdict skill calls
- `action`: executable shell command or review skill invocation starting with `/`; no conditional prose
- `cost`: determines execution order (`cheap -> medium -> expensive`)
- `working_directory`: optional cwd override; default implementation worktree root
- `checks[]` must contain at least 1 check before setup completes
- `checks[]` is runtime verification, not done condition. After setup, change command/order/membership in `config.yaml` before Verify.
- Future probes with unknown target/port/artifact stay in `milestones[].exit_criteria` or steps until executable.

User-requested skill gates such as `/critique` or `$critique` phrased as check/gate/must-pass/before-done/acceptance must be `checks[].action` starting with that skill command; prose/manual notes do not satisfy. If advisory, ask whether it belongs outside checks.

Review checks:

- `action` should be `/critique` plus concise selectors. Reviewer model/count and fanout mechanics belong to `/critique`.
- Optional fields: `stage: milestone|final|ad-hoc`, `target`, `base`, `scope`, `goals`, `focus`. Prefer fields over long `action` prose.
- Review gates use `/critique`, not raw `/fanout`. Bare `action: "/critique"` without stage/target/base/scope/goals blocks setup.

Manual probes are evidence notes unless the contract names probe, checklist, transcript path, commit SHA, and why no deterministic command covers it.

Filtered/no-match proof gates must show intended-surface discovery and distinguish no matches from command errors. For `go test -run`, include `go test -list` or equivalent.

When deterministic checks cannot cover semantic behavior, design intent, or cross-cutting risk, suggest `/critique` as a check. It runs during Verify; place before expensive checks when useful.

### Optimize contract

When `evaluation.objective: optimize`, setup must clarify the optimize contract.
Do not mix correctness checks with optimize metric: checks decide acceptability; metric is read from experiment outputs after `sample_check`.

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

- Single-repo task -> confirm a state location that does not create tracked repo surface; prefer repo-adjacent or established local location over new in-worktree state dir
- Multi-repo or cross-worktree task -> place `.harness/` under the nearest common parent directory of the touched repos/worktrees
- Do not default to the currently open repo just because setup is being run from there
- For dependent future work, create only the next executable carrier by default; keep later work as portfolio/lean pending until prerequisites are verifiable.

### `setup:new`

1. Confirm the directory containing `.harness/`; create if missing
2. Generate `task_slug`, allocate `task_id`, ensure unique NNN prefix, then create `.harness/tasks/<task_id>/` and `artifacts/setup/` atomically. Scan sibling tasks for same worktree/branch; reuse requires `continuation_of`, base commit, and parent disposition. On race, pick next unused NNN; existing duplicates require user choice.
3. If inside a git repo, prefer `.git/info/exclude` for `.harness/` and `.worktree/`
4. Write `config.yaml`: `task.id`, `description`, `done_when`, `base_branch`, remaining skeleton fields. Run blockers become checks or blocking `open_questions`, not context-only notes. No placeholders
5. Write `context.md`: no-events Current State per `references/run.md`; manual sections empty; `last_action: "setup completed; ready for run preflight"`; Next Steps: `Run preflight; on success, record baseline and enter round 1.`
6. Write `state.jsonl`: initialize as empty file
7. Write `plan.yaml`: use `references/plan.md`; include `plan_sources[]`, `planning_context`, and seeded `milestones[]` from the normalized plan. Set `strategy.status: pending`, `active_milestone_id: null`. Empty `milestones[]` is only allowed for legacy repair when no plan source exists.

### `setup:repair`

Reuse existing `task_id` and task directory. Do not delete existing files or artifacts.

- `config.yaml`: fill gaps, infer check `kind` (`/critique` -> `review`; other slash probes -> `manual_probe`; else `command`), normalize cost aliases, move strategy prose to `plan.yaml`, no placeholders
- `plan.yaml`: missing+events -> build single-milestone active plan from task goal/current code, emit `strategy_updated(reason=bootstrap, trigger=legacy_migration)`, then run; missing/empty+no events -> pending seed from plan sources/setup synthesis; unstarted missing provenance -> `setup_synthesis` or explicit re-import; started missing provenance -> append `legacy_repair` source, not reconstructed setup evidence; normalize status aliases only with ledger/controller evidence; version drift -> restore active pointers from latest controller/strategy events, then immediate replan on run; unrecoverable corruption -> escalate
- `context.md`: no run events -> reset to `setup:new` initial snapshot; has run events -> fill in missing fields only, do not reset
- `state.jsonl`: create as empty file only if missing; leave existing history append-only. Legacy invalid events (`type` instead of `event`, non-canonical result/gate values, terminal aliases such as `task_completed` or `release_completed`) are read-only migration evidence, not canonical resume state.

## Step 6: Hand Off to Run

Setup is done when all of these hold:

- Task state files complete (including `plan.yaml`)
- `config.yaml` has no placeholders, bare skill actions, invalid plan source kinds, or generated/cache/profile dirs in `boundary.mutable`
- Contract is complete enough for run
- `done_when` matches the milestone set and feature/count claims match their enumerated lists

After setup handoff, run initialization activates or enriches the pending seed plan. See `references/run.md` Bootstrap Strategy section.

## Appendix A: Canonical `config.yaml` Skeleton

```yaml
task: { id: "<task_id>", description: "<goal from user>", done_when: "<observable done condition>", protocol: direct, base_branch: "<branch>" }  # direct|tdd_preferred|tdd_required; direct = no TDD overlay, preferred = fail-first by default, required = fail-first required for behavior-changing implementation rounds

boundary: { mutable: [], immutable: [] }  # implementation files only; generated/cache/profile dirs (`dist/`, `node_modules/`, coverage, `.playwright-cli/profiles/`, `.worktree/`, `.harness/`) are excluded unless the user explicitly makes them the target

checks: []  # {name, kind, action, cost: cheap|medium|expensive, working_directory?}; execution order cheap -> medium -> expensive

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
