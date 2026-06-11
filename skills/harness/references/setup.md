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
- Extract goal/done condition, root problem/motivation, boundary hints, checks, milestones/rationales, approaches, non-goals, constraints, assumptions, decisions.
- External files are snapshots, not live links; later changes require explicit re-import/replan.

**Dispatch preamble:**

```
mode: embedded
task_id: <task_id or provisional task slug>
caller: /harness setup
goal: "<raw user goal>"
motivation: "<why the task is needed, if stated>"
plan_sources: [<external plan summaries, if any>]
```

**Normalize selected source:** snapshot selected plan; record id/digest/summary/sections in `plan.yaml.plan_sources[]`; map goal/done/boundary/protocol/check/milestone fields to config and pending milestones. `planning_context` stores compact intent anchors incl. task motivation; it excludes finalized contract fields and source prose. If absent motivation would affect boundary, done condition, or milestone order, ask; otherwise set null and do not invent. Values stay advisory until Step 4; user confirms/overrides each field.

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

Each check shape: `{name, kind, action, cost: cheap|medium|expensive, when?, working_directory?}`.

- `name`: unique identifier for the check, referenced by `verification.gates` and `metric.sample_check`
- `kind`: `command` for shell commands, `review` only for verdict gates such as `/audit`, `manual_probe` for probes or non-verdict skill calls
- `action`: executable shell command or review skill invocation starting with `/`; no conditional prose
- `cost`: determines execution order (`cheap -> medium -> expensive`)
- `when`: `preflight|every_round|milestone_exit|final_exit`; default `every_round` for command/manual checks; review checks must set it or derive it from `stage`
- `working_directory`: optional cwd override; default implementation worktree root
- `checks[]` must contain at least 1 check before setup completes
- `checks[]` is runtime verification, not done condition. After setup, change command/order/membership in `config.yaml` before Verify.
- Future probes with unknown target/port/artifact stay in `milestones[].exit_criteria` or steps until executable.

User-requested skill gates such as `/audit` or `$audit` phrased as check/gate/must-pass/before-done/acceptance must be `checks[].action` starting with that skill; prose/manual notes do not satisfy. If advisory, ask whether it belongs outside checks.

Default review gates:

- Non-very-simple tasks get a final `/audit` check by default: `kind: review`, `stage: final`, `when: final_exit`, usually `cost: expensive`.
- Add milestone `/audit` checks by default when the task has multiple milestones, changes 3+ files, crosses modules/interfaces, touches API/schema/protocol, user-visible workflow, durable state/data writes, permissions/trust boundaries, concurrency, retry, rollback, or idempotency.
- Very-simple mechanical tasks may omit audit when deterministic checks cover the whole change; record the narrow omission reason in setup decisions.
- `/audit` supplements deterministic checks. Do not replace build, test, lint, smoke, metric, or repro commands with review gates.

Review checks:

- `action` should be `/audit` plus concise selectors. Reviewer model/count and fanout mechanics belong to `/audit`.
- Optional fields: `stage: milestone|final|ad-hoc`, `target`, `base`, `scope`, `goals`, `focus`, `when`. Prefer fields over long `action` prose.
- Review gates use `/audit`, not raw `/fanout`. Bare `action: "/audit"` without stage/target/base/scope/goals/when blocks setup.

Manual probes are evidence notes unless the contract names probe, checklist, transcript path, commit SHA, and why no deterministic command covers it.

Filtered/no-match proof gates must show intended-surface discovery and distinguish no matches from command errors. For `go test -run`, include `go test -list` or equivalent.

When deterministic checks cannot cover semantic behavior, design intent, or cross-cutting risk, suggest `/audit` as a check. It runs during Verify; place before expensive checks when useful.

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

Choose `.harness/` location before state:

- Single-repo task -> confirm a location that does not create tracked repo surface; prefer repo-adjacent or established local over new in-worktree state dir
- Multi-repo/cross-worktree task -> nearest common parent of touched repos/worktrees
- Do not default to the current repo just because setup runs there
- Dependent future work -> create only the next executable carrier by default; keep later work as portfolio/lean pending until prerequisites are verifiable.

### `setup:new`

1. Confirm the directory containing `.harness/`; create if missing
2. Generate `task_slug`, allocate unique NNN `task_id`, create `.harness/tasks/<task_id>/` + `artifacts/setup/` atomically. Scan sibling tasks for same worktree/branch; reuse needs `continuation_of`, base commit, and parent disposition. Race -> next NNN; duplicates -> user choice.
3. If inside a git repo, prefer `.git/info/exclude` for `.harness/` and `.worktree/`
4. Write `config.yaml`: `task.id`, `description`, `done_when`, `base_branch`, remaining skeleton fields. Run blockers become checks or blocking `open_questions`, not context-only notes. No placeholders
5. Write `context.md`: no-events Current State per `references/run.md`; manual sections empty; `last_action: "setup completed; ready for run preflight"`; Next Steps: `Run preflight; on success, record baseline and enter round 1.`
6. Write `state.jsonl`: initialize as empty file
7. Write `plan.yaml`: use `references/plan.md`; include `plan_sources[]`, `planning_context`, and seeded `milestones[]` from the normalized plan. Set `strategy.status: pending`, `active_milestone_id: null`. Empty `milestones[]` is only allowed for legacy repair when no plan source exists.

### `setup:repair`

Reuse existing `task_id` and task directory. Do not delete existing files or artifacts.

- `config.yaml`: fill gaps, infer check `kind` (`/audit` -> `review`; other slash probes -> `manual_probe`; else `command`), infer missing `when` from review `stage` or default command/manual checks to `every_round`, normalize cost aliases, move strategy prose to `plan.yaml`, no placeholders
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

checks: []  # {name, kind, action, cost: cheap|medium|expensive, when?, working_directory?}; execution order cheap -> medium -> expensive

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
