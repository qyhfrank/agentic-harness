# Setup Protocol

Covers phases 0 (scaffold) and A (plan), plus preflight checks before the run loop.

## Phase 0: Scaffold

Diagnose the repository and create task-scoped `.harness/` infrastructure.

### Entry Conditions

- Goal is known (enforced by SKILL.md Missing Goal Gate)
- Current directory is a git repository
- Harness root is resolved (see SKILL.md Harness Root)

### 1. Diagnose Repository

Assess agent-friendliness across these dimensions:

| Dimension | What to check | How |
|---|---|---|
| Test infrastructure | Test runner, test files, coverage config | Glob for `*test*`, `*spec*`, `jest.config*`, `pytest.ini`, `vitest*`, `.mocharc*`, `Cargo.toml [dev-dependencies]` |
| Build system | Package manager, build scripts, lockfile | Check `package.json`, `Makefile`, `Cargo.toml`, `pyproject.toml`, `go.mod`, `build.gradle` |
| Lint / format | Linter config, formatter config | Glob for `.eslintrc*`, `.prettierrc*`, `ruff.toml`, `.golangci*`, `rustfmt.toml` |
| CI | CI config files | Glob for `.github/workflows/*`, `.gitlab-ci.yml`, `Jenkinsfile`, `.circleci/*` |
| Structure | AGENTS.md / README, directory layout | Check root for guidance files |
| Verification methods | Benchmarks, profiling tools, e2e suites | Glob for `bench*`, `perf*`, `e2e*`, `integration*`, `smoke*`; check package scripts |
| Golden Path Commands | `make test`, `make lint`, `make bootstrap` or equivalents | Check Makefile, package.json scripts, pyproject.toml scripts for standard entry points |
| ADR / decision records | Architecture Decision Records | Check for `docs/adr/`, `docs/decisions/`, `ADR-*.md` patterns |
| Machine-readable CLI | `--json` or structured output support | Check if primary CLI tools offer `--json` flags (advisory, not blocking) |

Classify each as: `ready`, `partial`, or `missing`. Record findings in a gap report.

### 2. Detect Tech Stack

Infer from file extensions, package manifests, and framework markers. If ambiguous, list candidates for user to confirm during plan.

### 3. Resolve or Create Task

- Create `<harness_root>/.harness/` if missing
- Derive `task_slug` and allocate `task_id` per SKILL.md Task ID Generation
- Create `<harness_root>/.harness/tasks/<task_id>/`
- Write `.harness-task` in the worktree root (if in a harness-managed worktree)
- Initialize `.harness/current-task` only when it does not yet exist and this is the sole task

If a task is already resolved, scaffold only fills gaps.

### 4. Create Task Directory

```
<harness_root>/.harness/tasks/<task_id>/
├── config.yaml      # draft config
├── context.md       # initial context
├── state.jsonl      # empty; baseline added during preflight
└── artifacts/       # optional; create lazily
```

Idempotent: never overwrite an existing file.

### 5. Generate config.yaml Draft

Replace ALL `<...>` placeholders with real values.

```yaml
draft: true

harness_root: "<absolute path>"

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

verification:
  mandatory:
    # Each gate: {type, name, command, frequency}
    # frequency: every_round (default) | milestone | final
  guard: []
  escalation: []
  architecture_guard:       # optional command gate for layer/dependency enforcement
    # command: ""           # e.g., "python scripts/check_layers.py" or "deptrac analyse"
    # frequency: milestone  # typically milestone, not every_round

rollback:
  granularity: commit
  strategy: revert_commit
  preserve_failed_experiments: true

execution_policy:
  dangerous_commands: []       # patterns requiring human approval (e.g., "rm -rf", "DROP TABLE")
  secret_patterns: [".env", "*.pem", "credentials.*"]  # never read or stage these
  network_policy: allow        # allow | deny | prompt
  dependency_install: prompt   # allow | deny | prompt — controls npm install, pip install, etc.
```

Pre-fill what can be inferred:
- `implementation.protocol`: `tdd_required` for features/bugs/refactors; `direct` for config/docs
- `evaluation.objective`: `optimize` for open-ended metric improvement; `satisfy` for concrete criteria
- `boundary` from repo analysis when unambiguous

When test infrastructure is `missing` or `partial`:
- Pre-fill verification with cheapest available command backstops
- Suggest `agent_review` via `/critique`
- Set `close_authority.type` to `confirmed_review`

When the task is optimize-focused and no Tier 1 probe exists, note it in context.md Working Memory.

### 6. Generate Initial context.md

```markdown
# Harness Context

## Current State
- phase: scaffold (complete)
- round: 0 / -
- harness_root: <absolute path>
- worktree: <to be set>
- current_objective: "<goal>"
- best_result: "baseline not recorded yet"

## Repository Assessment
<gap report>

## Tech Stack
- primary: <detected>
- test runner: <detected or "unknown">
- build: <detected or "unknown">
- lint: <detected or "unknown">

## Working Memory
(empty)

## Durable Notes
(empty)

## Decisions
(empty)

## Next Steps
- Run `/harness plan` to finalize config.yaml
```

### 7. Generate state.jsonl

Create an empty file. Baseline event is created during preflight.

### 8. Update AGENTS.md

Append a harness section (see SKILL.md Completion for the template). Do not duplicate if already present.

### 9. Report

```
Scaffold complete.

Harness root: /path/to/repo
Created: config.yaml (draft), context.md, state.jsonl, .harness-task

Repository assessment:
  [ready]   Test: jest (42 test files)
  [ready]   Build: npm
  [partial] Lint: eslint exists, no formatter
  [missing] CI: none found

Next: run `/harness plan` to finalize config.yaml
```

---

## Phase A: Plan

Interactive config finalization. Guide the user from draft config.yaml to a finalized contract.

### Entry Conditions

- Task config exists with `draft: true`
- User is present for interactive Q and A

### Procedure

Walk through these sections. Ask one question at a time. Suggest concrete values based on repo analysis. If the user says "just use defaults", show complete config for confirmation.

**1. Task Definition** — Confirm `task.description` captures the intent. If broad, ask one narrowing question.

**2. Boundary Definition** — Most critical step. Ask: "Which files need to change?" then "Anything that must NOT be touched?" Validate paths exist.

**3. Implementation Protocol** — `tdd_required` (default) / `tdd_preferred` / `direct`. Explain: controls whether proposer must produce RED evidence before production code.

**4. Verification Strategy** — Suggest gates from scaffold's repo assessment. Dry-run each command. Resolve failures before finalizing.

**5. Evaluation Strategy** — `satisfy` (fill acceptance_criteria) or `optimize` (fill metric). Set `close_authority`.

**6. Stop Guards** — Budget (default 50), stagnation (default 5).

**7. Entropy** — Review doom_loop defaults. "If same error 3 times, agent pivots. Adjust?"

**8. Finalize** — Vocabulary check, show complete config, dry-run gates, confirm. Remove `draft: true`. Update context.md phase.

### Cross-Session Continuity

On re-entry: read context.md and config.yaml, resume from first unfilled section.

---

## Preflight

Run before entering the harness loop. On any mandatory failure, do NOT enter the loop.

### Mandatory Checks

| # | Check | Rule |
|---|---|---|
| 1 | Repo integrity | Git repo exists, not in detached HEAD, no stale `index.lock`. |
| 2 | Clean working tree | No uncommitted changes. `.harness-task` is exempt. |
| 3 | Baseline verification | All mandatory verification gates pass on current HEAD. |
| 4 | Scope resolution | Every path in `boundary.mutable` and `boundary.immutable` exists on disk. |

On failure: report which check failed, suggest fix, do not auto-remediate.

### Baseline Recording

After all checks pass:
1. Run all mandatory verification gates against current HEAD.
2. Evaluate baseline objective state and metric.
3. Append a `baseline_recorded` event to `state.jsonl`.
4. If baseline verification fails, return to plan mode.
5. Update context.md Current State.
