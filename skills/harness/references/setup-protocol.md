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
  protocol: tdd_required    # tdd_required | tdd_preferred | direct

checks:
  # Each check: {name, command, cost} for commands, or {name, kind: review} for review gates
  # cost: cheap (every round) | expensive (close attempt + every N rounds)
  # Optional: probe: true for task-specific cheap probes
  # Example:
  # - {name: typecheck, command: "make typecheck", cost: cheap}
  # - {name: lint, command: "ruff check .", cost: cheap}
  # - {name: unit-tests, command: "pytest tests/unit", cost: cheap}
  # - {name: review, kind: review}
  # - {name: e2e, command: "pytest tests/e2e", cost: expensive}

evaluation:
  objective: satisfy         # satisfy | optimize
  acceptance_criteria: []
  close_rule: ~              # omit to auto-infer; see run-protocol.md close_rule
  expensive_interval: 10     # run expensive checks every N rounds (in addition to close attempts)

termination:
  stop_guards:
    budget: { max_rounds: 50 }
    stagnation: { rounds: 5 }

entropy:
  doom_loop_threshold: 3
  doom_loop_action: reread_and_pivot
  atomicity_test: "one-sentence"

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

When `objective: optimize`, add metric fields to the evaluation block:

```yaml
evaluation:
  objective: optimize
  acceptance_criteria: []
  metric:
    name: ""
    direction: increase        # increase | decrease | none
    volatile: false
    samples: 3
    min_delta: 0
    confirmation_runs: 2
  close_rule: ~
```

Pre-fill what can be inferred:
- `implementation.protocol`: `tdd_required` for features/bugs/refactors; `direct` for config/docs
- `evaluation.objective`: `optimize` for open-ended metric improvement; `satisfy` for concrete criteria
- `boundary` from repo analysis when unambiguous
- `checks`: auto-discover from repo assessment (see below)

### Check Auto-Discovery

From the scaffold gap report, auto-populate `checks[]`:
- Test runner found and `ready` → add as cheap check (e.g., `{name: unit-tests, command: "pytest tests/", cost: cheap}`)
- Linter found and `ready` → add as cheap check
- Typecheck found and `ready` → add as cheap check
- e2e or integration suite found → add as expensive check
- Benchmark found and `objective: optimize` → add as expensive check or cheap probe depending on runtime cost

When test infrastructure is `missing` or `partial`:
- Ask the user: "No test infrastructure found. Want to add basic tests as part of this task?"
- User says yes: add test creation to `acceptance_criteria`, pre-fill `checks` with the anticipated test command (`cost: cheap`) and a review gate.
- User says no: add a review gate to `checks`, set `close_rule: review_streak(3)`. Note in context.md Working Memory that the task relies on review-only verification.

When `objective: optimize` and no probe check exists, note the gap in context.md Working Memory.

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
- review_streak: 0

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

Append a harness section to AGENTS.md listing the task layout (configs, state, context, artifacts paths). Do not duplicate if already present.

### 9. Report

```
Scaffold complete.

Task: <task_id>
Harness root: /path/to/repo
Created: config.yaml (draft), context.md, state.jsonl

Repository assessment:
  [ready]   Test: jest (42 test files)
  [ready]   Build: npm
  [partial] Lint: eslint exists, no formatter
  [missing] CI: none found

Checks auto-discovered:
  - typecheck (cheap): npx tsc --noEmit
  - lint (cheap): npx eslint .
  - unit-tests (cheap): npx jest
  - review: /critique

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

**4. Checks Configuration** — Walk through the `checks` list:
- Show auto-discovered checks from scaffold (lint, typecheck, test commands).
- For each: confirm command string, confirm cost assignment (`cheap` vs `expensive`).
- Ask: "Should we add a `/critique` review gate between cheap and expensive checks?"
- If `objective: optimize` and no probe exists: "No task-specific probe found. Want to add a quick measurement command?"
- Dry-run each check command. Resolve failures before finalizing.

**5. Evaluation Strategy** — `satisfy` (fill acceptance_criteria) or `optimize` (fill metric + volatile settings). Show inferred `close_rule`. Ask if override needed:
- Has expensive checks: "Close requires expensive checks to pass. OK?"
- No expensive checks: "Close requires N consecutive review passes. Adjust N?"
- High risk: "Want to require human sign-off for closure?"

**6. Stop Guards** — Budget (default 50), stagnation (default 5).

**7. Entropy** — Review doom_loop defaults. "If same error 3 times, agent diagnoses and pivots. Adjust?"

**8. Finalize** — Vocabulary check, show complete config, dry-run checks, confirm. Remove `draft: true`. Update context.md phase.

### Cross-Session Continuity

On re-entry: read context.md and config.yaml, resume from first unfilled section.

---

## Preflight

Run before entering the harness loop. On any mandatory failure, do NOT enter the loop.

### Mandatory Checks

| # | Check | Rule |
|---|---|---|
| 1 | Repo integrity | Git repo exists, not in detached HEAD, no stale `index.lock`. |
| 2 | Clean working tree | No uncommitted changes. |
| 3 | Baseline verification | All checks (both cheap and expensive) pass on current HEAD. |
| 4 | Scope resolution | Every path in `boundary.mutable` and `boundary.immutable` exists on disk. |

On failure: report which check failed, suggest fix, do not auto-remediate.

### Baseline Recording

After all checks pass:
1. Run all checks against current HEAD.
2. Evaluate baseline objective state and metric.
3. Append a `baseline_recorded` event to `state.jsonl`.
4. If baseline verification fails, return to plan mode.
5. Update context.md Current State.
