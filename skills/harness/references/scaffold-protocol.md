# Scaffold Protocol

Phase 0: diagnose the repository and create task-scoped `.harness/` infrastructure.

## Entry Conditions

- User has provided a goal (explicit or inferred from conversation)
- Current directory is a git repository

If not in a git repo, stop and tell the user.

If the goal is missing, ask exactly one targeted question before creating any harness files:

`What task should this harness pursue in this repo? For example: fix a bug, add a feature, raise test coverage, or optimize a metric.`

Rules:
- `/harness`, `set up a harness`, and similar setup phrases are not valid goals
- broad intents such as `optimize this repo`, `make it better`, or `improve quality` are not yet concrete goals
- when the user provides only a broad intent, ask exactly one focused narrowing question before scaffold:
  `What specific outcome should this harness optimize here? For example: reduce test failures, add smoke coverage for one area, or improve a named metric.`
- Do not fabricate `task.description` from generic setup language
- Do not create `.harness/` until the user supplies a goal

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

For each dimension, classify as: `ready` (config exists and runnable), `partial` (config exists but incomplete), `missing` (not found).

Record findings in a gap report. Do not attempt to fix gaps -- only report them.

### 2. Detect Tech Stack

Infer primary language and framework from:
- File extensions (`.ts`, `.py`, `.rs`, `.go`, `.java`)
- Package manifests (`package.json`, `pyproject.toml`, `Cargo.toml`, `go.mod`)
- Framework markers (`next.config.*`, `django`, `actix`, `gin`)

If ambiguous, list candidates and let the user confirm during plan phase.

### 3. Resolve or Create Current Task

Harness always uses task-scoped state.

Default scaffold behavior:
- Create `.harness/` if missing
- Derive a task ID from the user's goal per the Task ID Generation rules in `SKILL.md` (e.g., "fix the auth timeout bug" → `fix-auth-timeout`)
- Create `.harness/tasks/<task_id>/` using the derived ID
- Write the derived ID to `.harness/current-task`

If a task is already resolved, scaffold only fills gaps for that task.

### 4. Create Task Directory

```
.harness/
├── current-task
└── tasks/
    └── <task_id>/
        ├── config.yaml      # draft config
        ├── context.md       # initial context
        ├── state.jsonl      # empty; baseline event added during preflight
        └── artifacts/       # empty, for future round outputs
```

Idempotency rules:
- If `.harness/` exists, check each file individually
- Never overwrite an existing file
- Only create missing files
- If all files exist, report "scaffold already complete" and suggest `plan` mode

### 5. Generate config.yaml Draft

**Important:** The template below uses `<task_id>` and `<goal from user>` as placeholders. When writing the actual file, replace ALL angle-bracket placeholders with real values. No `<...>` tokens may remain in the output file.

```yaml
draft: true  # remove this line when config is finalized

task:
  id: "<task_id>"
  name: ""
  description: "<goal from user>"
  source: ""

boundary:
  mutable: []
  immutable: []

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
  mandatory: []
  guard: []
  escalation: []

rollback:
  granularity: commit
  strategy: revert_commit
  preserve_failed_experiments: true

state:
  ledger: ".harness/tasks/<task_id>/state.jsonl"
```

Pre-fill what can be inferred:
- `task.description` from the user's stated goal
- `task.source` if the user referenced a plan or spec file
- `evaluation.objective`: use `optimize` if the goal is open-ended metric improvement; use `satisfy` if the goal has concrete completion criteria
- `boundary.mutable` and `boundary.immutable` from repo analysis when the structure is unambiguous

Leave everything else empty with `# fill during plan` comments when helpful.

### 6. Generate Initial context.md

```markdown
# Harness Context

## Current State
- phase: scaffold (complete)
- round: 0 / -
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

- Current task pointer: `.harness/current-task`
- Task configs: `.harness/tasks/<task_id>/config.yaml`
- Task state: `.harness/tasks/<task_id>/state.jsonl`
- Task context: `.harness/tasks/<task_id>/context.md`
- Task artifacts: `.harness/tasks/<task_id>/artifacts/`

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

Created:
  .harness/current-task
  .harness/tasks/fix-auth-timeout/config.yaml (draft)
  .harness/tasks/fix-auth-timeout/context.md
  .harness/tasks/fix-auth-timeout/state.jsonl
  .harness/tasks/fix-auth-timeout/artifacts/

Repository assessment:
  [ready]   Test: jest (42 test files, coverage configured)
  [ready]   Build: npm
  [partial] Lint: eslint config exists, no format config
  [missing] CI: no CI configuration found

Next: run `/harness plan` to finalize config.yaml
```

## Feedback Note

Before reporting scaffold complete, write a structured feedback note per `feedback-protocol.md`:

```json
// .harness/tasks/<task_id>/feedback-note.json (overwritten each phase)
{
  "uncertainties": [],
  "workarounds": [],
  "needs_review": []
}
```

Auto-detect: check all config values against defined vocabularies. If any value is out-of-vocab, append a `bad_default` event to `~/.asb/state/harness-feedback/events.jsonl`.
