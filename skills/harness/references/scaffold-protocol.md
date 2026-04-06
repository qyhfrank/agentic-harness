# Scaffold Protocol

Phase 0: diagnose the repository and create task-scoped `.harness/` infrastructure.

## Entry Conditions

- Goal is known (enforced by SKILL.md Missing Goal Gate before scaffold loads)
- Current directory is a git repository
- Harness root is resolved (see SKILL.md Harness Root)

If not in a git repo, stop and tell the user.

## Procedure

### 1. Diagnose Repository

Assess agent-friendliness across these dimensions:

| Dimension | What to check | How |
|---|---|---|
| Test infrastructure | Test runner, test files, coverage config | Glob for `*test*`, `*spec*`, `jest.config*`, `pytest.ini`, `vitest*`, `.mocharc*`, `Cargo.toml [dev-dependencies]` |
| Build system | Package manager, build scripts, lockfile | Check `package.json`, `Makefile`, `Cargo.toml`, `pyproject.toml`, `go.mod`, `build.gradle` |
| Lint / format | Linter config, formatter config | Glob for `.eslintrc*`, `.prettierrc*`, `ruff.toml`, `.golangci*`, `rustfmt.toml` |
| CI | CI config files | Glob for `.github/workflows/*`, `.gitlab-ci.yml`, `Jenkinsfile`, `.circleci/*` |
| Structure | AGENTS.md / README, directory layout | Check root for guidance files, assess directory depth and naming |
| Verification methods | Benchmark scripts, profiling tools, e2e suites, smoke tests | Glob for `bench*`, `benchmark*`, `perf*`, `profile*`, `e2e*`, `integration*`, `smoke*`; check `package.json`/`Makefile`/`pyproject.toml` scripts for `bench`, `perf`, `e2e` |

For each dimension, classify as: `ready` (config exists and runnable), `partial` (config exists but incomplete), `missing` (not found).

Record findings in a gap report. Do not attempt to fix gaps -- only report them.

### 2. Detect Tech Stack

Infer primary language and framework from:
- File extensions (`.ts`, `.py`, `.rs`, `.go`, `.java`)
- Package manifests (`package.json`, `pyproject.toml`, `Cargo.toml`, `go.mod`)
- Framework markers (`next.config.*`, `django`, `actix`, `gin`)

If ambiguous, list candidates and let the user confirm during plan phase.

### 3. Resolve or Create Task

Harness always uses task-scoped state. Use the local-first task resolution order from SKILL.md to check whether a task already exists for this context.

Default scaffold behavior:
- Create `<harness_root>/.harness/` if missing
- Derive `task_slug` from the user's goal per the Task ID Generation rules in `SKILL.md`
- Allocate `task_id` as `NNN-<task_slug>`. Before creating the task directory, re-scan `<harness_root>/.harness/tasks/` to verify the candidate numeric prefix is still unused (avoids race conditions when multiple agents scaffold concurrently).
- Create `<harness_root>/.harness/tasks/<task_id>/` using the allocated ID
- If in a harness-managed worktree, write the allocated ID to `<worktree_root>/.harness-task`
- Initialize `<harness_root>/.harness/current-task` only when the file does not yet exist and this is the sole task. Do not overwrite an existing `current-task` — it is a root-scoped default, not an active-task pointer.

If a task is already resolved, scaffold only fills gaps for that task.

### 4. Create Task Directory

All paths below are at the resolved harness root.

```
<harness_root>/
└── .harness/
    ├── current-task          # optional; initialized only on first sole-task bootstrap
    └── tasks/
        └── <task_id>/
            ├── config.yaml      # draft config
            ├── context.md       # initial context
            ├── state.jsonl      # empty; baseline event added during preflight
            └── artifacts/       # optional; create lazily when a round emits outputs

<worktree_root>/
└── .harness-task             # worktree-local task affinity (written when in a harness-managed worktree)
```

Idempotency rules:
- If `<harness_root>/.harness/` exists, check each file individually
- Never overwrite an existing file
- Only create missing files
- If all files exist, report "scaffold already complete" and suggest `plan` mode

### 5. Generate config.yaml Draft

**Important:** The template below uses `<task_id>` and `<goal from user>` as placeholders. When writing the actual file, replace ALL angle-bracket placeholders with real values. No `<...>` tokens may remain in the output file.

```yaml
draft: true  # remove this line when config is finalized

harness_root: "<absolute path to harness root>"

task:
  id: "<task_id>"
  name: ""
  description: "<goal from user>"
  source: ""

boundary:
  mutable: []
  immutable: []

implementation:
  protocol: tdd_required

evaluation:
  objective: satisfy
  acceptance_criteria: []
  metric:
    name: ""
    direction: increase
    volatile: false
    samples: 3
    min_delta: 0
    confirmation_runs: 2
  close_authority:
    type: command_backed

termination:
  stop_guards:
    budget: { max_rounds: 50 }
    stagnation: { rounds: 5 }

entropy:
  doom_loop_threshold: 3
  doom_loop_action: reread_and_pivot
  simplicity_rule: ""
  atomicity_test: "one-sentence"

preflight:
  repo_integrity: true
  clean_tree: true
  baseline_verify: true
  scope_resolve: true

verification:
  mandatory:
    # Each gate: {type, name, command, frequency}
    # frequency: every_round (default) | milestone | final
    # See verification-gate.md Verification Tiers for classification guidance
  guard: []
  escalation: []

rollback:
  granularity: commit
  strategy: revert_commit
  preserve_failed_experiments: true

state:
  ledger: "<harness_root>/.harness/tasks/<task_id>/state.jsonl"
```

Pre-fill what can be inferred:
- `harness_root` from the resolved harness root path (always absolute)
- `task.description` from the user's stated goal
- `task.source` if the user referenced a plan or spec file
- `implementation.protocol`: use `tdd_required` for features, bug fixes, refactors, and behavior changes; use `direct` for config/docs/generated-code style tasks when that is already obvious from the goal
- `evaluation.objective`: use `optimize` if the goal is open-ended metric improvement; use `satisfy` if the goal has concrete completion criteria
- `boundary.mutable` and `boundary.immutable` from repo analysis when the structure is unambiguous

When test infrastructure is `missing` or `partial`:
- Pre-fill `verification.mandatory` with the cheapest available command backstops (build, typecheck, lint, dry-run, docs render — whatever the repo supports)
- Suggest `agent_review` via `/critique --spec` and `/critique --quality` as additional mandatory gates
- Set `close_authority.type` to `confirmed_review` instead of `command_backed`
- Note in `context.md` that oracle lifting should be a priority: convert review findings into command gates as the task progresses

When verification methods are discovered during repo diagnosis, pre-classify them by tier (see `verification-gate.md` Verification Tiers):
- Tier 0 (`every_round`): typecheck, lint, unit tests
- Tier 1 (`every_round`): isolated benchmarks, profiling scripts, smoke metrics -- anything that verifies the specific change without running the full pipeline
- Tier 2 (`milestone` or `final`): full e2e suites, integration tests

When the task is optimization-focused (`evaluation.objective: optimize`) and no Tier 1 probe exists:
- Note in `context.md` Working Memory that creating an isolated probe should be a plan-phase priority
- Suggest concrete probe types based on the detected stack (e.g., a micro-benchmark script for the targeted function, a bundle-size snapshot, a cold-start timer)

Leave everything else empty with `# fill during plan` comments when helpful.

### 6. Generate Initial context.md

```markdown
# Harness Context

## Current State
- phase: scaffold (complete)
- round: 0 / -
- harness_root: <absolute path>
- worktree: <to be set when worktree is created>
- current_objective: "<goal>"
- best_result: "baseline not recorded yet"

## Repository Assessment
<gap report from step 1>

## Tech Stack
- primary: <detected language/framework>
- test runner: <detected or "unknown">
- build: <detected or "unknown">
- lint: <detected or "unknown">

## Working Memory
(empty -- will be populated during plan and run phases)

## Durable Notes
(empty -- dead ends, constraints, tool quirks added during run phase)

## Decisions
(empty)

## Next Steps
- Run `/harness plan` to finalize config.yaml
```

### 7. Generate state.jsonl

Create an empty file. Do not write a header row. The baseline event is created during preflight (run phase).

### 8. Update AGENTS.md

Append a harness section to the project's AGENTS.md (create if missing):

```markdown
## Harness

This project uses the harness autonomous iteration engine.

Harness state lives at the **harness root** (original repo root), not inside worktrees.

- Repo default task (root-scoped only): `<harness_root>/.harness/current-task`
- Worktree task affinity: `<worktree_root>/.harness-task`
- Task configs: `<harness_root>/.harness/tasks/<task_id>/config.yaml`
- Task state: `<harness_root>/.harness/tasks/<task_id>/state.jsonl`
- Task context: `<harness_root>/.harness/tasks/<task_id>/context.md`
- Task artifacts (optional): `<harness_root>/.harness/tasks/<task_id>/artifacts/`

Multi-task concurrency: one task per worktree, one controller per task. `.harness-task` binds a worktree to its task. Child implementers (dispatched by `/planning`) do not write `.harness/` state.

Run `/harness plan` to configure, `/harness run` to execute.
```

Rules:
- If AGENTS.md already has a `## Harness` section, do not duplicate
- Append at the end of the file, before any trailing newlines
- Do not modify any other section of AGENTS.md

### 9. Report

Summarize what was created and what gaps were found:

```
Scaffold complete.

Harness root: /path/to/repo
Created:
  <harness_root>/.harness/tasks/000-fix-auth-timeout-bug/config.yaml (draft)
  <harness_root>/.harness/tasks/000-fix-auth-timeout-bug/context.md
  <harness_root>/.harness/tasks/000-fix-auth-timeout-bug/state.jsonl
  <worktree_root>/.harness-task

current-task: initialized (sole task)  # or: unchanged (already exists) / not applicable (multiple tasks)

Repository assessment:
  [ready]   Test: jest (42 test files, coverage configured)
  [ready]   Build: npm
  [partial] Lint: eslint config exists, no format config
  [missing] CI: no CI configuration found

Next: run `/harness plan` to finalize config.yaml
```

Report the harness root path and worktree branch alongside the file listing so the user knows where state and code live.
